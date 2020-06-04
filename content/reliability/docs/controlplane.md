# EKS Control Plane

Amazon Elastic Kubernetes Service (EKS) is a managed Kubernetes service that makes it easy for you to run Kubernetes on AWS without needing to install, operate, and maintain your own Kubernetes control plane. It runs upstream Kubernetes and is certified Kubernetes conformant. This conformance ensures that EKS supports the Kubernetes APIs, just like the open source community version that you can install on EC2 or on-premises. Existing applications running on upstream Kubernetes are compatible with Amazon EKS.

EKS automatically manages the availability and scalability of the Kubernetes control plane nodes and it automatically replaces unhealthy control plane nodes.

## EKS Architecture 

EKS architecture is designed to eliminate any single points of failure which may compromise the availability and durability of the Kubernetes control plane.

> Insert EKS architecture diagram here

EKS control plane runs inside an EKS managed VPC. The EKS control plane comprises the Kubernetes master nodes, etcd cluster. Kubernetes master nodes that run components like the API server, scheduler, and `kube-controller-manager` run in an auto-scaling group. This auto-scaling group is spread across a minimum of three Availability Zones (AZs). Likewise, for durability the etcd server nodes also run in an auto-scaling group that is spread across three AZs. EKS runs a NAT Gateway in each AZ and master nodes and etcd servers run in a private subnet. This ensures that an event in a single AZ doesn’t affect the availability of the etcd cluster.  

When you create a new cluster, Amazon EKS creates a highly-available endpoint for the managed Kubernetes API server that you use to communicate with your cluster (using tools like `kubectl`). The managed endpoint uses NLB to load balance Kubernetes API servers. EKS also provisions two [ENI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)s in different AZs to facilitate communication to your worker nodes.

You can [configure whether your Kubernetes cluster’s API server](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html) is reachable from the public internet (using the public endpoint) or through your VPC (using the EKS-managed ENIs) or both.  

Whether users and worker nodes connect to the API server using the public endpoint or the EKS-managed ENI, there are redundant paths for connection. 

## Recommendations

## Monitor Control Plane Metrics

Monitoring Kubernetes API metrics can give you insights into control plane performance and identify issues. Unhealthy control plane can compromise availability of the workloads running inside the cluster. For example, poorly written controllers can overload the API servers which can affect Pod autoscaling. 

Kubernetes exposes control plane metrics at the  `/metrics` endpoint. 

You can view the metrics exposed using `kubectl`:

```shell
kubectl get --raw /metrics
```

These metrics are represented in a [Prometheus text format](https://github.com/prometheus/docs/blob/master/content/docs/instrumenting/exposition_formats.md). 

You can use Prometheus to collect and store these metrics. In May 2020, CloudWatch added support for monitoring Prometheus metrics in CloudWatch Container Insights. So you can also use Amazon CloudWatch to monitor the EKS control plane. 

The Kubernetes API server metrics can be found [here](https://github.com/kubernetes/apiserver/blob/master/pkg/endpoints/metrics/metrics.go). For example, `apiserver_request_duration_seconds` can indicate how long API requests are taking to run. 

Consider monitoring these control plane metrics:

### API Server

| Metric | Description  |
|:--|:--|
|  `apiserver_request_total` | Counter of apiserver requests broken out for each verb, dry run value, group, version, resource, scope, component, client, and HTTP response contentType and code. |
| `apiserver_request_duration_seconds*`  | Response latency distribution in seconds for each verb, dry run value, group, version, resource, subresource, scope and component. |
| `rest_client_request_duration_seconds` | Request latency in seconds. Broken down by verb and URL. |
| `apiserver_admission_controller_admission_duration_seconds` | Admission controller latency histogram in seconds, identified by name and broken out for each operation and API resource and type (validate or admit). | 
| `rest_client_request_duration_seconds` | Request latency in seconds. Broken down by verb and URL. |
| `rest_client_request_duration_seconds` | Request latency in seconds. Broken down by verb and URL.
| `rest_client_requests_total`  | Number of HTTP requests, partitioned by status code, method, and host. | 

### etcd

| Metric | Description  |
|:--|:--|
| `etcd_request_duration_seconds` | Etcd request latency in seconds for each operation and object type. | 
| `etcd_request_latencies_summary` | Etcd request latency summary in microseconds for each operation and object type. |
| `etcd_helper_cache_entry_total` | Counter of etcd helper cache entries. | 
| `etcd_helper_cache_hit_total` | Counter of etcd helper cache hits.| 
| `etcd_helper_cache_miss_total` | Counter of etcd helper cache miss. | 
| `etcd_request_cache_get_duration_seconds*` | Latency in seconds of getting an object from etcd cache. | 
| `etcd_request_cache_add_duration_seconds*` | Latency in seconds of adding an object to etcd cache | 

Consider using [Grafana dashboard 12006](https://grafana.com/grafana/dashboards/12006) to visualize and monitor Kubernetes API server requests and latency and etcd latency metrics. 

## Cluster Authentication

EKS currently supports two types of authentication: [bearer/service account tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens) and IAM authentication which uses [webhook token authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication). When users call the Kubernetes API, a webhook passes an authentication token included in the request to IAM. The token, a base 64 signed URL, is generated by the AWS Command Line Interface ([AWS CLI](https://aws.amazon.com/cli/)). 

The IAM user or role that creates the EKS Cluster automatically gets full access on the cluster. You can manage access to the EKS cluster by editing the [`aws-auth` configmap](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html). 

If you misconfigure the `aws-auth` configmap and lose access to the cluster, you can still use the cluster creator’s user or role to access the cluster. 

## Handling Cluster Upgrades
Kubernetes follows a quarterly release cycle; a new minor version (like 1.**16** or 1.**17**) is released approximately every three months. Each minor version is supported for approximately nine months after it is first released. Kubernetes supports compatibility between control plane and worker nodes for at least two minor versions.

In line with the Kubernetes community support for Kubernetes versions, EKS provides at least three production-ready versions of Kubernetes at any given time, with a fourth version in deprecation.

EKS will announce the deprecation of a given Kubernetes minor version at least 60 days before the end of support date. On the end of support date, clusters running the deprecated version will begin to be automatically updated to the next EKS-supported version of Kubernetes. 

EKS performs in-place cluster upgrades for both [Kubernetes](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) and [EKS platform versions](https://docs.aws.amazon.com/eks/latest/userguide/platform-versions.html). This simplifies cluster operations and lets you take advantage of the latest Kubernetes features and apply security patches, without any downtime.

New Kubernetes versions introduce significant changes and you cannot downgrade a cluster once upgraded. So having a well documented process for handling cluster upgrades is necessary for a smooth transition to newer Kubernetes versions. 

- You should familiarize yourself with the [Kubernetes deprecation policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/) as newer versions may deprecate APIs and features that may break existing applications. 
- Before upgrading the cluster you should review the [Kubernetes change log](https://github.com/kubernetes/kubernetes/tree/master/CHANGELOG) and [Amazon EKS Kubernetes versions](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) to understand any negative impact to your workloads. 
- Consider testing the cluster upgrade in non-production environment and identify any impacts to current workloads and controllers. You can automate the testing by building a continuous integration workflow to test compatibility of your applications, controllers, and custom integrations before moving to a new Kubernetes version.
- You may also need to upgrade Kubernetes add-ons after upgrading the cluster. Review [Updating an Amazon EKS cluster Kubernetes version](https://docs.aws.amazon.com/eks/latest/userguide/update-cluster.html) to validate the compatibility of cluster add-ons with the cluster version. 
- Consider turning on [control plane logging](https://docs.aws.amazon.com/eks/latest/userguide/control-plane-logs.html) and review the logs for any errors. 
- Consider using `eksctl` to manage EKS cluster. You can use `eksctl` to [update the control plane, add-ons and worker nodes](https://eksctl.io/usage/cluster-upgrade/). 
- EKS control plane upgrade doesn’t include upgrading worker nodes. You are responsible for updating EKS worker nodes. Consider using [EKS managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) to automate the process of upgrading worker nodes. 
- If required, you can use `kubectl convert` to [convert Kubernetes manifests files between different API versions](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#convert).

## Control Plane Scaling

EKS clusters by default are sized to handle up to 200 nodes and 30 pods per node. If your cluster exceeds this size, you can request a scale up through a support ticket. The EKS team is working on automatically scaling the control plane, at which point this will not be required.

## Limits and service quotas

AWS sets service limits (an upper limit on the number of each resource your team can request) to protect you from accidentally over-provisioning resources. [Amazon EKS Service Quotas](https://docs.aws.amazon.com/eks/latest/userguide/service-quotas.html) lists the service limits. There are two types of limits, soft limits, that can be changed with proper justification via a support ticket. Hard limits cannot be changed. You should consider these values when architecting your applications. Consider reviewing these service limits periodically and incorporate them during in your application design. 

> Besides the limits from orchestration engines, there are limits in other AWS services, such as Elastic Load Balancing (ELB) and Amazon VPC, that may affect your application performance.
> More about EC2 limits here: [EC2 service limits](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-resource-limits.html). 

## Additional Resources:

- [De-mystifying cluster networking for Amazon EKS worker nodes](https://aws.amazon.com/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/)
- [Amazon EKS cluster endpoint access control](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html)
- 
- [AWS re:Invent 2019: Amazon EKS under the hood (CON421-R1)](https://www.youtube.com/watch?v=7vxDWDD2YnM)
