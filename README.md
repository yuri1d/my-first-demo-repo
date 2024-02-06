
# Setup cluster-autoscaler to Scale Cluster Node Group Automatically

This guide outlines the step-by-step configure and implement the cluster-autoscaler automation for AWS EKS clusters to enable automatic scaling of the cluster node group, adhering to AWS best practices. All configurations will be deployed through GitHub Actions in a separate repository.

## Overview and Architecture

**What is Cluster Autoscaler**: The Cluster Autoscaler automatically adds or removes nodes in a cluster based on resource requests from pods. The Cluster Autoscaler doesn’t directly measure CPU and memory usage values to make a scaling decision. Instead, it checks every 10 seconds to detect any pods in a pending state, suggesting that the scheduler could not assign them to a node due to insufficient cluster capacity. 

**How Cluster Autoscaler works**: In the scaling-up scenario, CA automatically kicks in when the number of pending (un-schedulable) pods increases due to resource shortages and works to add additional nodes to the cluster.

![App Screenshot](https://www.kubecost.com/images/ca-process.png)

The four steps involved in scaling up a cluster are as follows:

- When Cluster Autoscaler is active, it will check for pending pods. The default scan interval is 10 seconds, which is configurable using the --scan-interval flag.
- If there are any pending pods and the cluster needs more resources, CA will extend the cluster by launching a new node as long as it is within the constraints configured by the administrator (more on this in our example). Public cloud providers like AWS, Azure, GCP also support the Kubernetes Cluster Autoscaler functionality. For example, AWS EKS integrates into Kubernetes using its AWS Auto Scaling group functionality to automatically add and remove EC2 virtual machines that serve as cluster nodes.
- Kubernetes registers the newly provisioned node with the control plane to make it available to the Kubernetes scheduler for assigning pods.
- Finally, the Kubernetes scheduler allocates the pending pods to the new node.


## Requirements

1. **Cluster Version**: Make sure you have the same version on your cluster as on your deployment yaml file.

2. **ASG Tags**: Make sure the tags as shown below associated with the Auto Scaling group. CA relies on these labels to identify the AWS Auto Scaling groups intended for its use.

![App Screenshot](https://www.kubecost.com/images/outscaling-key-value.png)


## How to Run/Execute


### 1. Create an EKS cluster

First of all, you need to create a cluster or use an existing one in your account. In my case, the cluster is already created and I can proceed to the next step, but if you don't have a cluster yet, you can search the Internet for resources on how to quickly create a cluster.

### 2. Verification of the creation EKS cluster

We can verify using the kubectl command line:

```bash
  kubectl get svc
```

We can also verify the presence of our cluster via the AWS console:

![App Screenshot](https://www.kubecost.com/images/cluster-aws.png)


### 3. Create IAM OIDC provider

IAM OIDC is used for authorizing the Cluster Autoscaler to launch or terminate instances under an Auto Scaling group. In this section, we will see how to configure it with the EKS cluster.

- In the EKS cluster console, navigate to the configuration tab and copy the OpenID connect URL

- Then, go to the IAM console, select Identity provider and Add provider

- Click “Add provider,” select “OpenID Connect,” and click “Get thumbprint”

- Then enter the “Audience” (sts.amazonaws.com in our example pointing to the AWS STS, also known as the Security Token Service) and add the provider


### 4. Create IAM policy

Next, we need to create the IAM policy, which allows CA to increase or decrease the number of nodes in the cluster.

Create the policy with these permissions as shown below and save it in your AWS account as "AmazonEKSClusterAutoscalerPolicy"

```bash
{
  "Version": "2012-10-17",
  "Statement": [
      {
          "Action": [
              "autoscaling:DescribeAutoScalingGroups",
              "autoscaling:DescribeAutoScalingInstances",
              "autoscaling:DescribeLaunchConfigurations",
              "autoscaling:DescribeTags",
              "autoscaling:SetDesiredCapacity",
              "autoscaling:TerminateInstanceInAutoScalingGroup",
              "ec2:DescribeLaunchTemplateVersions"
          ],
          "Resource": "*",
          "Effect": "Allow"
      }
  ]
}
```


### 5. Create an IAM role for the provider

We need to create an IAM role and link it to the provider we created in Step 3

- Create IAM role. Selecting a web identity and provider which we created before.

- Select the Audience “sts.amazonaws.com” and attach the policy which you have created in Step 4.

- Then, verify the IAM role and make sure the policy is attached.

- Edit the “Trust relationships” and change the OIDC as shown below:
![App Screenshot](https://www.kubecost.com/images/change-oidc.png)

- Then click “Update Trust Policy” to save it.


### 6. Deploy Cluster Autoscaler

Next, we deploy Cluster Autoscaler. To do so, you must use the Amazon Resource Names (ARN) number of the IAM role created in our earlier step.

I was created using the helm chart but you can use any method that you want

The content intended for my helm chart templates (make sure you copy all of the content presented over the next page).

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::847845718389:role/AmazonEKSClusterAutoscalerRole
  name: cluster-autoscaler
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["events", "endpoints"]
    verbs: ["create", "patch"]
  - apiGroups: [""]
    resources: ["pods/eviction"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["endpoints"]
    resourceNames: ["cluster-autoscaler"]
    verbs: ["get", "update"]
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["watch", "list", "get", "update"]
  - apiGroups: [""]
    resources:
      - "pods"
      - "services"
      - "replicationcontrollers"
      - "persistentvolumeclaims"
      - "persistentvolumes"
    verbs: ["watch", "list", "get"]
  - apiGroups: ["extensions"]
    resources: ["replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["policy"]
    resources: ["poddisruptionbudgets"]
    verbs: ["watch", "list"]
  - apiGroups: ["apps"]
    resources: ["statefulsets", "replicasets", "daemonsets"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "csinodes"]
    verbs: ["watch", "list", "get"]
  - apiGroups: ["batch", "extensions"]
    resources: ["jobs"]
    verbs: ["get", "list", "watch", "patch"]
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["create"]
  - apiGroups: ["coordination.k8s.io"]
    resourceNames: ["cluster-autoscaler"]
    resources: ["leases"]
    verbs: ["get", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["create","list","watch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["cluster-autoscaler-status", "cluster-autoscaler-priority-expander"]
    verbs: ["delete", "get", "update", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-autoscaler
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system


---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    k8s-addon: cluster-autoscaler.addons.k8s.io
    k8s-app: cluster-autoscaler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: cluster-autoscaler
subjects:
  - kind: ServiceAccount
    name: cluster-autoscaler
    namespace: kube-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
  labels:
    app: cluster-autoscaler
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-autoscaler
  template:
    metadata:
      labels:
        app: cluster-autoscaler
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: 'false'
    spec:
      serviceAccountName: cluster-autoscaler
      containers:
        - image: k8s.gcr.io/autoscaling/cluster-autoscaler:v1.20.0 # Important to update the cluster version
          name: cluster-autoscaler
          resources:
            limits:
              cpu: 100m
              memory: 500Mi
            requests:
              cpu: 100m
              memory: 500Mi
          command:
            - ./cluster-autoscaler
            - --v=4
            - --stderrthreshold=info
            - --cloud-provider=aws
            - --skip-nodes-with-local-storage=false
            - --expander=least-waste
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/demo-ca-cluster # Update cluster name
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
          volumeMounts:
            - name: ssl-certs
              mountPath: /etc/ssl/certs/ca-certificates.crt #/etc/ssl/certs/ca-bundle.crt for Amazon Linux Worker Nodes
              readOnly: true
          imagePullPolicy: "Always"
      volumes:
        - name: ssl-certs
          hostPath:
            path: "/etc/ssl/certs/ca-bundle.crt"
```
Refer to [this file](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#what-are-the-parameters-to-ca) for a complete set of cluster configuration parameters for your future use.

Then apply the changes in my case it is the helm chart command:

```bash
helm upgrade --install cluster-autoscaler ./cluster-autoscaler/helm-chart
```

Next, verify the logs by issuing this command:

```bash
kubectl logs -l app=cluster-autoscaler -n kubesystem -f
```

CA will now check for unscheduled pods and try to schedule them. You can see those actions from the logs. Check the status of the pods by issuing the following command:

```bash
kubectl get pods -n kube-system
```

The expected results are displayed below.

![App Screenshot](https://www.kubecost.com/images/pods-status.png)

#### Congratulations! 
You have deployed the Cluster Autoscaler successfully.


## Testing 

We are going to create an simple Nginx deployment to test autoscaler functionality

Here an example of my deployment, I'm just create 40 replicas to test how cluster autoscaler works:

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-deployment
spec:
  replicas: 40
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-container
        image: nginx:latest
        ports:
        - containerPort: 80
```

Apply the changes:
```bash
kubectl apply -f nginx.yaml
```

After deployment was created, we needed to wait 10-15 min for Cluster Autoscaler to discover the Auto Scaling group to scale the cluster.

Then check again the node status to verify how CA provisions nodes in our cluster. Below you can see that a fourth node was added to the cluster automatically

#### Scale down the nodes:

We can also verify that Cluster Autoscaler can remove nodes. To do so, we delete the Nginx deployments (pods) and observe how CA responds by removing nodes from the cluster to accommodate the reduced capacity requirement. We delete the deployments by issuing the kubectl commands below:
```bash
kubectl delete -f nginx.yaml
```

After you delete the deployment, wait for a few minutes and then check the Auto Scaling group in the AWS console to verify the desired node reduction.


## Conclusion

Cluster Autoscaler plays a vital role in a Kubernetes cluster by ensuring adequate computing resources are available by adding nodes to a cluster and keeping infrastructure costs down by removing nodes. CA performs these vital functions by checking for pods in a pending state (which tells CA it needs to add capacity by scaling up) and detecting underutilized nodes (telling CA to scale down). CA is simpler to implement and maintain than VPA and HPA, but that doesn’t mean it should replace them. Kubernetes autoscaling works best with all three autoscaling vectors working in concert. The choice to use CA depends on your application’s architecture (e.g., whether your application is based on microservices or not) and scaling frequency. Therefore, we recommend you use our guide to understand each type of autoscaling supported in Kubernetes and choose the autoscalers that meet your needs.

