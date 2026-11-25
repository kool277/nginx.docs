---
title: Using NGINX or NGINX Plus as the Ingress Controller for Amazon Elastic Kubernetes Service
description: Run F5 NGINX Ingress Controller with NGINX Open Source or F5 NGINX Plus on Amazon EKS, behind an AWS Network Load Balancer with the PROXY protocol.
toc: true
weight: 400
f5-content-type: how-to
f5-product: F5 NGINX Ingress Controller
f5-keywords: "NGINX Ingress Controller, Amazon EKS, eksctl, Amazon ECR, Network Load Balancer, PROXY protocol, kubernetes-ingress"
f5-summary: >
  This guide shows how to run F5 NGINX Ingress Controller on Amazon Elastic Kubernetes Service (EKS), behind an AWS Network Load Balancer.
  It turns on the PROXY protocol, so NGINX Ingress Controller sees the IP addresses of the clients.
  It works with NGINX Open Source and F5 NGINX Plus images of NGINX Ingress Controller.
f5-audience: operator
---

## Overview

This guide installs F5 NGINX Ingress Controller in an Amazon Elastic Kubernetes Service (EKS) cluster. An AWS Network Load Balancer (NLB) sends client traffic to it. The PROXY protocol carries the IP address of each client from the NLB to NGINX Ingress Controller.

The steps work with NGINX Ingress Controller based on NGINX Open Source or on F5 NGINX Plus. Where they differ, the guide says so.

---

<span id="prereqs"></span><span id="prerequisites"></span>

## Before you begin

Before you begin, make sure you have:

- **An AWS account**: To create one, see [Create an AWS account](https://docs.aws.amazon.com/accounts/latest/reference/manage-acct-creating.html).
- **Command-line tools**: The [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html), [`eksctl`](https://eksctl.io/installation/), `kubectl`, and Docker.
- **An NGINX Ingress Controller image**:
  - For NGINX Open Source, use the image [`nginx/nginx-ingress`](https://hub.docker.com/r/nginx/nginx-ingress/) on Docker Hub, or [build your own]({{< ref "/nic/install/build.md" >}}).
  - For NGINX Plus, [pull the image from the F5 registry]({{< ref "/nic/install/images/registry-download.md" >}}) with your NGINX Ingress Controller subscription, or [build it]({{< ref "/nic/install/build.md" >}}).
- **The kubernetes-ingress repository**: Clone [kubernetes-ingress](https://github.com/nginx/kubernetes-ingress). The commands in this guide run in its directory.

{{< call-out class="caution" >}} Never push an NGINX Plus image of NGINX Ingress Controller to a public registry, such as Docker Hub. That violates your license agreement. {{< /call-out >}}

---

<span id="amazon-eks"></span>

## Create an Amazon EKS cluster

You can create a cluster with the AWS Management Console, the AWS CLI, or `eksctl`. This guide uses `eksctl`, which is the simplest option.

1. Install or update `eksctl`, as described in the [eksctl documentation](https://eksctl.io/installation/).

1. Create a cluster, as described in [Get started with Amazon EKS – eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html). Choose managed Linux nodes. Note the AWS region of the cluster for the next steps.

---

<span id="amazon-ecr"></span><span id="push-the-nginx-plus-ingress-controller-image-to-aws-ecr"></span>

## Push the image to Amazon ECR

Skip this section if your cluster pulls the image from Docker Hub, from the F5 registry, or from another registry that it can reach.

1. Create a private repository in Amazon Elastic Container Registry (ECR) called `nginx-plus-ic`, as described in [Creating an Amazon ECR private repository](https://docs.aws.amazon.com/AmazonECR/latest/userguide/repository-create.html).

1. Log Docker in to your ECR registry. The AWS CLI creates a token, and `docker login` uses it. For details, see [Private registry authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html).

    ```shell
    aws ecr get-login-password --region <AWS_REGION_CODE> | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION_CODE>.amazonaws.com
    ```

    - `<AWS_REGION_CODE>` is the region of your cluster, for example `us-west-1`.
    - `<AWS_ACCOUNT_ID>` is your AWS account ID. To find it, see [View AWS account identifiers](https://docs.aws.amazon.com/IAM/latest/UserGuide/console_account-alias.html).

1. Tag the image for the ECR repository:

    ```shell
    docker tag <IMAGE>:<VERSION_TAG> <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION_CODE>.amazonaws.com/nginx-plus-ic:<VERSION_TAG>
    ```

    - `<IMAGE>` is the name of your local image, for example `private-registry.nginx.com/nginx-ic/nginx-plus-ingress`.
    - `<VERSION_TAG>` is the tag of the image.

1. Push the image:

    ```shell
    docker push <AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION_CODE>.amazonaws.com/nginx-plus-ic:<VERSION_TAG>
    ```

---

<span id="ingress-controller"></span><span id="install-the-nginx-plus-ingress-controller"></span>

## Install NGINX Ingress Controller

Install NGINX Ingress Controller in the cluster, as described in [Installation with Manifests]({{< ref "/nic/install/manifests.md" >}}). Use your image in the Deployment or DaemonSet. Stop before you create a Service for NGINX Ingress Controller. The next section creates one for AWS.

---

<span id="use-a-network-load-balancer-in-front-of-nginx-ingress-controller"></span><span id="configure-a-loadbalancer-service-to-use-nlb"></span>

## Create the Network Load Balancer

A Service of type `LoadBalancer` sends traffic from the internet to NGINX Ingress Controller. The `loadbalancer-aws-elb.yaml` manifest of the repository creates an NLB for it:

```shell
kubectl apply -f deployments/service/loadbalancer-aws-elb.yaml
```

The `nlb-target-type` annotation of the manifest is for the [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/), which you install in the cluster:

- With the AWS Load Balancer Controller, the NLB is internal by default. To reach it from the internet, add the annotation `service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"`. You can also turn on the PROXY protocol with the annotation `service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"`, instead of in the AWS console.
- Without it, the legacy controller of Kubernetes handles the Service. The Amazon EKS documentation says that this controller gets only critical bug fixes.

For all annotations, see [Service annotations](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/service/annotations/).

---

<span id="enable-the-proxy-protocol"></span>

## Turn on the PROXY protocol

An NLB doesn't change the TCP connections, so it can't add an `X-Forwarded-For` header. The PROXY protocol sends the client's address in a header before the data instead. Turn it on in NGINX Ingress Controller and in the NLB together, because each side fails with the other's traffic.

1. In `deployments/common/nginx-config.yaml`, replace `data: {}` with this `data` block:

    ```yaml
    data:
      proxy-protocol: "True"
      real-ip-header: "proxy_protocol"
      set-real-ip-from: "0.0.0.0/0"
    ```

    The keys turn on the PROXY protocol, and use the client address from its header as the address of the client. With `set-real-ip-from: "0.0.0.0/0"`, NGINX Ingress Controller accepts the header from any address. That's safe only if nothing but the NLB can reach NGINX Ingress Controller.

1. Apply the ConfigMap:

    ```shell
    kubectl apply -f deployments/common/nginx-config.yaml
    ```

1. Turn on the PROXY protocol for the target groups of the NLB, as described in [Proxy protocol](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html#proxy-protocol) in the AWS documentation.

---

<span id="testing"></span>

## Verify the deployment

1. Find the DNS name of the load balancer. It's the `LoadBalancer Ingress` value in the output:

    ```shell
    kubectl describe svc nginx-ingress --namespace=nginx-ingress
    ```

1. Find the IP address of that DNS name. Replace `<DNS_NAME>` with the DNS name:

    ```shell
    nslookup <DNS_NAME>
    ```

1. Deploy the Cafe example application, as described in the [complete example](https://github.com/nginx/kubernetes-ingress/tree/main/examples/ingress-resources/complete-example) of the repository. Run its `kubectl` commands in the `examples/ingress-resources/complete-example` directory of the repository. In its first step, set the `IC_IP` variable to the IP address of the load balancer, and `IC_HTTPS_PORT` to `443`. Then run its `curl` command.

1. Find the pod of NGINX Ingress Controller, and show its log. Replace `<POD_NAME>` with the name of the pod:

    ```shell
    kubectl get pods --namespace=nginx-ingress
    kubectl logs <POD_NAME> --namespace=nginx-ingress
    ```

    The access log entry of your `curl` request shows the IP address of your client, not an address of the NLB. So the PROXY protocol works.

---

<span id="appendix"></span><span id="appendix-disable-the-proxy-protocol"></span>

## Turn off the PROXY protocol

To turn off the PROXY protocol, change both sides again:

1. Remove the three keys from `deployments/common/nginx-config.yaml`, and apply the ConfigMap:

    ```shell
    kubectl apply -f deployments/common/nginx-config.yaml
    ```

1. Turn off the PROXY protocol for the target groups of the NLB, in the AWS console or with the AWS CLI.

Without the PROXY protocol, NGINX Ingress Controller sees the client addresses only if the target group preserves them. With instance targets, also set `externalTrafficPolicy: Local` in the `spec` of the Service, so that the nodes don't replace the client address. For details, see [Client IP preservation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html#client-ip-preservation) in the AWS documentation.

---

## Troubleshooting

### Connections close, and the log shows "broken header"

**Symptom**: Requests through the NLB fail without a response. The log of NGINX Ingress Controller shows messages such as `broken header: "GET / HTTP/1.1" while reading PROXY protocol`.

**Cause**: NGINX Ingress Controller expects the PROXY protocol, but the target groups of the NLB don't send it.

**Fix**: Turn on the PROXY protocol for the target groups, or remove the keys from the ConfigMap.

---

### Requests return 400 Bad Request

**Symptom**: Every request through the NLB returns `400 Bad Request`.

**Cause**: The target groups send the PROXY protocol header, but NGINX Ingress Controller doesn't expect it, so it reads the header as a malformed request.

**Fix**: Add the PROXY protocol keys to the ConfigMap, or turn off the PROXY protocol for the target groups.

---

## References

- [Installation with Manifests]({{< ref "/nic/install/manifests.md" >}})
- [Download NGINX Ingress Controller from the F5 Registry]({{< ref "/nic/install/images/registry-download.md" >}})
- [ConfigMap resources]({{< ref "/nic/configuration/global-configuration/configmap-resource.md" >}})
- [Network Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html)

---

## Revision history

- Version 3 (November 2026): Rewritten to the how-to template, for current NGINX Ingress Controller images and manifests.
- Version 2 (February 2025): Initial version (NGINX Plus Release 20)
