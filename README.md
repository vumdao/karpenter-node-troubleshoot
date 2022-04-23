<p align="center">
  <a href="https://dev.to/vumdao">
    <img alt="Troubleshoot Karpenter Node" src="images/cover.png" width="700" />
  </a>
</p>
<h1 align="center">
  <div><b>Troubleshoot Karpenter Node</b></div>
</h1>

## Abstract
- It often goes smoothly when using Karpenter with the fresh new EKS/kubernetes cluster as nodes are provisioned and able to join the cluster, or it works well on this region but not on others, why???
- This blog describes what issues I faced and how to troubleshoot them.

## Table Of Contents
 * [Pre-requisite](#Pre-requisite)
 * [The issues](#The-issues)
 * [Check IAM instance profile](#Check-IAM-instance-profile)
 * [Check kube-proxy and aws-node Daemonset](#Check-kube-proxy-and-aws-node-Daemonset)
 * [Check Security group and Elastic network interface](#Check-Security-group-and-Elastic-network-interface)
 * [Debug where the kubelet stuck at](#Debug-where-the-kubelet-stuck-at)
 * [Check AWS service endpoints](#Check-AWS-service-endpoints)
 * [Conclusion](#Conclusion)

---

## 🚀 **Pre-requisite** <a name="Pre-requisite"></a>
- [Better understand Karpenter and hands-on](https://dev.to/aws-builders/aws-karpenter-hands-on-custom-resources-1am9)

## 🚀 **The issues** <a name="The-issues"></a>
- Following might be one/more issues:
  - [New nodes are failing to communicate with the API server](https://github.com/aws/karpenter/issues/1683)
  - [Lease: Failed to get lease: leases.coordination.k8s.io](https://github.com/aws/karpenter/issues/1634)
  - No `kube-proxy` and `aws-node` Daemonset Pods are created on the Karpenter nodes.
  - `kubelet` failed to start so node is stuck in `NotReady`
- Following is the case by case that we need to troubleshoot to findout the rootcause

## 🚀 **Check IAM instance profile** <a name="Check-IAM-instance-profile"></a>
- The most condition for an EKS node join the cluster is IAM instance profile which provide IAM permission for the nodes to access AWS resources and call API to EKS cluster.
- Furthermore, we also need to add the instance profile to `aws-auth` configmap to grant `system:masters` permissions in the cluster's role-based access control (RBAC) configuration in the Amazon EKS control plane. Read more [Enabling IAM user and role access to your cluster](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)

  `aws-auth configmap`

  ```
  apiVersion: v1
  data:
    mapAccounts: |
      []
    mapRoles: |
      - "groups":
        - "system:nodes"
        - "system:bootstrappers"
        "rolearn": "arn:aws:iam::<accountID>:role/KarpenterNodeInstanceProfile"
        "username": "system:node:{{EC2PrivateDNSName}}"
  ```

- The right permissions that an instance profile needs
  ```
  AmazonEKSWorkerNodePolicy
  AmazonEC2ContainerRegistryReadOnly
  AmazonSSMManagedInstanceCore
  AmazonEKS_CNI_Policy
  ```

- Instance profile role's Trust Policy:
  ```
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "EKSWorkerAssumeRole",
              "Effect": "Allow",
              "Principal": {
                  "Service": "ec2.amazonaws.com"
              },
              "Action": "sts:AssumeRole"
          }
      ]
  }
  ```

## 🚀 **Check kube-proxy and aws-node Daemonset** <a name="Check-kube-proxy-and-aws-node-Daemonset"></a>
- When new node joining the cluster, 2 Daemonset Pods must be created on it
  - `kube-system` which provides critical networking functionality for Kubernetes applicationsand
  - `aws-node` which is the Amazon VPC Container Network Interface (CNI) plugin for Kubernetes  If not some checks should be performed

  So we need to verify those pods are in Running status on the worker node. If not, perform following checks.

## 🚀 **Check Security group and Elastic network interface** <a name="Check-Security-group-and-Elastic-network-interface"></a>
- We need to ensure traffic between Pods because the `coredns` Pods are in Deployment type and created on a number of nodes that the new ones must be allowed traffic to
- Checkout [Understand Pods communication](https://dev.to/aws-builders/understand-pods-communication-338c)

## 🚀 **Debug where the `kubelet` stuck at** <a name="Debug-where-the-kubelet-stuck-at"></a>
- If there's no `kube-proxy` and `aws-node` Daemonsets on the node, the `kubelet` absolutely failed to start. Look at the `kubelet` logs
  ```
  Apr 23 05:26:38 ip-10-10-11-100 containerd: time="2022-04-23T05:26:38.814171952Z" level=error msg="failed to load cni during init, please check CRI plugin status before setting up network for pods" error="cni config load failed: no network config found in /etc/cni/net.d: cni plugin not initialized: failed to load cni config"
  ```

- Karpenter creates the node object, and then Kubelet populates the necessary labels once it can connect to the APIServer.
  - `kube-proxy` nodeAffinity
    ```
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: beta.kubernetes.io/os
                operator: In
                values:
                - linux
              - key: beta.kubernetes.io/arch
                operator: In
                values:
                - amd64
    ```

- Due to `kubelet` failed to start so the node does not contains those labels
  ```
  # kubectl describe node ip-10-10-11-198.ap-southeast-1.compute.internal
  Name:               ip-10-10-11-198.ap-southeast-1.compute.internal
  Roles:              <none>
  Labels:             karpenter.sh/capacity-type=spot
                      karpenter.sh/provisioner-name=karpenter-test
                      lifecycle=spot
                      node.kubernetes.io/instance-type=t3.large
                      role=karpenter
                      topology.kubernetes.io/zone=ap-southeast-1b
                      type=test
  Annotations:        node.alpha.kubernetes.io/ttl: 0
  CreationTimestamp:  Fri, 22 Apr 2022 15:37:34 +0000
  Taints:             node.kubernetes.io/unreachable:NoExecute
                      dedicated=karpenter:NoSchedule
                      karpenter.sh/not-ready:NoSchedule
                      node.kubernetes.io/unreachable:NoSchedule
  Unschedulable:      false
  Lease:              Failed to get lease: leases.coordination.k8s.io "ip-10-10-11-198.ap-southeast-1.compute.internal" not found
  Conditions:
    Type             Status    LastHeartbeatTime                 LastTransitionTime                Reason                   Message
    ----             ------    -----------------                 ------------------                ------                   -------
    Ready            Unknown   Fri, 22 Apr 2022 15:37:34 +0000   Fri, 22 Apr 2022 15:38:34 +0000   NodeStatusNeverUpdated   Kubelet never posted node status.
    MemoryPressure   Unknown   Fri, 22 Apr 2022 15:37:34 +0000   Fri, 22 Apr 2022 15:38:34 +0000   NodeStatusNeverUpdated   Kubelet never posted node status.
    DiskPressure     Unknown   Fri, 22 Apr 2022 15:37:34 +0000   Fri, 22 Apr 2022 15:38:34 +0000   NodeStatusNeverUpdated   Kubelet never posted node status.
    PIDPressure      Unknown   Fri, 22 Apr 2022 15:37:34 +0000   Fri, 22 Apr 2022 15:38:34 +0000   NodeStatusNeverUpdated   Kubelet never posted node status.
  ```

- Now we get into the node by using SSM or SSH to check kubelet logs, get the `kubelet` bootstrap command to run directl inside the node
  ```
  # /etc/eks/bootstrap.sh dev-d1 --apiserver-endpoint https://111111111111111111111.yl4.ap-southeast-1.eks.amazonaws.com --b64-cluster-ca xxx= --container-runtime containerd --kubelet-extra-args '--node-labels=role=karpenter,type=test,karpenter.sh/provisioner-name=karpenter-test,lifecycle=spot,karpenter.sh/capacity-type=spot --register-with-taints=dedicated=karpenter:NoSchedule'
  sed: can't read /etc/eks/containerd/containerd-config.toml: No such file or directory
  Exited with error on line 469
  ```

- Ooh, we're missing configuration file for `containerd` [containerd-config.toml](https://github.com/containerd/containerd/blob/main/docs/man/containerd-config.toml.5.md). Check the status of `sandbox-image.service` and we see it failed to start
  ```
  # systemctl status sandbox-image -l
  ● sandbox-image.service - pull sandbox image defined in containerd config.toml
    Loaded: loaded (/etc/systemd/system/sandbox-image.service; enabled; vendor preset: disabled)
    Active: activating (start) since Sat 2022-04-23 08:52:45 UTC; 12min ago
  Main PID: 2965 (bash)
      Tasks: 2
    Memory: 17.0M
    CGroup: /system.slice/sandbox-image.service
            ├─2965 bash /etc/eks/containerd/pull-sandbox-image.sh
            └─3691 sleep 170

  Apr 23 09:05:42 ip-10-10-11-162.vc.p pull-sandbox-image.sh[2965]: github.com/containerd/containerd/vendor/github.com/urfave/cli.(*App).RunAsSubcommand(0xc00032c540, 0xc0000ecf20, 0x0, 0x0)
  Apr 23 09:05:42 ip-10-10-11-162.vc.p pull-sandbox-image.sh[2965]: /builddir/build/BUILD/containerd-1.4.13-2.amzn2.0.1/src/github.com/containerd/containerd/vendor/github.com/urfave/cli/app.go:404 +0x8f4
  Apr 23 09:05:42 ip-10-10-11-162.vc.p pull-sandbox-image.sh[2965]: main.main()
  Apr 23 09:05:42 ip-10-10-11-162.vc.p pull-sandbox-image.sh[2965]: github.com/containerd/containerd/cmd/ctr/main.go:37 +0x125
  ```

- Try to run the script for debuging and we see the script stuck at AWS ECR login.
  ```
  # bash -x /etc/eks/containerd/pull-sandbox-image.sh
  ++ awk '-F[ ="]+' '$1 == "sandbox_image" { print $2 }' /etc/containerd/config.toml
  + sandbox_image=123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/eks/pause:3.1-eksbuild.1
  ++ echo 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/eks/pause:3.1-eksbuild.1
  ++ cut -f4 -d .
  + region=ap-southeast-1
  ++ aws ecr get-login-password --region ap-southeast-1
  ```

- Now we near to the rootcause.

## 🚀 **Check AWS service endpoints** <a name="Check-AWS-service-endpoints"></a>
- We know that our node is getting stuck at AWS ECR login. For security, the Amazon ECR which holds production repository are often put in private network so we need to verify that our worker node can access API endpoints for Amazon ECR (and even Amazon EC2 and S3 if we don't pubic the service endpoints)
- Check the service endpoint (go to VPC -> Endpoints) to allow traffics from our worker node to the VPC endpoint.

## **Conclusion** <a name="Conclusion"></a>
- There are many usecase for the issue of worker nodes failed to join the EKS/kubernetes cluster so this blog is just guide you the way and some clue to troubleshoot.

---

References:
- [How do I resolve kubelet or CNI plugin issues for Amazon EKS?](https://aws.amazon.com/premiumsupport/knowledge-center/eks-cni-plugin-troubleshooting/)

---

<h3 align="center">
  <a href="https://dev.to/vumdao">:stars: Blog</a>
  <span> · </span>
  <a href="https://github.com/vumdao/karpenter-node-troubleshoot">Github</a>
  <span> · </span>
  <a href="https://stackoverflow.com/users/11430272/vumdao">stackoverflow</a>
  <span> · </span>
  <a href="https://www.linkedin.com/in/vu-dao-9280ab43/">Linkedin</a>
  <span> · </span>
  <a href="https://www.linkedin.com/groups/12488649/">Group</a>
  <span> · </span>
  <a href="https://www.facebook.com/CloudOpz-104917804863956">Page</a>
  <span> · </span>
  <a href="https://twitter.com/VuDao81124667">Twitter :stars:</a>
</h3>
