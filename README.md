---
title: AWS ingress Controller Install on AWS EKS
---
 

## Step-00: Introduction
1. Create IAM Policy and make a note of Policy ARN
2. Create IAM Role and k8s Service Account and bound them together
3. Install AWS Load Balancer Controller using HELM3 CLI
4. create a default Ingress Class 

## Step-01: Create IAM Policy
- Create IAM policy for the AWS Load Balancer Controller that allows it to make calls to AWS APIs on your behalf.

# Download IAM Policy
## Download latest
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json


# Create IAM Policy using policy downloaded 
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy_latest.json
```
### Make a note of Policy ARN    
- Make a note of Policy ARN as we are going to use that in next step when creating IAM Role.
```t
# Policy ARN 
Policy ARN:  arn:aws:iam::180789647333:policy/AWSLoadBalancerControllerIAMPolicy
```


## Step-02: Create an IAM role for the AWS LoadBalancer Controller and attach the role to the Kubernetes service account 
- Applicable only with `eksctl` managed clusters
- This command will create an AWS IAM role 
- This command also will create Kubernetes Service Account in k8s cluster
- In addition, this command will bound IAM Role created and the Kubernetes service account created
### Step-02-01: Create IAM Role using eksctl
```t
# Verify if any existing service account
kubectl get sa -n kube-system
kubectl get sa aws-load-balancer-controller -n kube-system
Obseravation:
1. Nothing with name "aws-load-balancer-controller" should exist


# Replaced name, cluster and policy arn (Policy arn we took note in step-02)
eksctl create iamserviceaccount --cluster=eksdemo1 --namespace=kube-system --name=aws-load-balancer-controller --attach-policy-arn=arn:aws:iam::180789647333:policy/AWSLoadBalancerControllerIAMPolicy --override-existing-serviceaccounts --approve
```
```

### Step-02-02: Verify using eksctl cli
```t
# Get IAM Service Account
eksctl  get iamserviceaccount --cluster eksdemo1
```

## Step-03: Install the AWS Load Balancer Controller using Helm V3 
### Step-03-01: Install Helm
- [Install Helm](https://helm.sh/docs/intro/install/) if not installed
- [Install Helm for AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

### Step-03-02: Install AWS Load Balancer Controller

# Add the eks-charts repository.
helm repo add eks https://aws.github.io/eks-charts

# Update your local repo to make sure that you have the most recent charts.
helm repo update

# Install the AWS Load Balancer Controller.
## Template
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eksdemo1 --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=us-east-1 --set vpcId=vpc-0165a396e41e292a3 --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller

```


## Step-05: Ingress Class Concept
- Understand what is Ingress Class 
- Understand how it overrides the default deprecated annotation `#kubernetes.io/ingress.class: "alb"`
- [Ingress Class Documentation Reference](https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/ingress_class/)
- [Different Ingress Controllers available today](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)


## Step-06: Review IngressClass Kubernetes Manifest
- **File Location:** `08-01-Load-Balancer-Controller-Install/kube-manifests/01-ingressclass-resource.yaml`
- Understand in detail about annotation `ingressclass.kubernetes.io/is-default-class: "true"`
```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: my-aws-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb

## Additional Note
# 1. You can mark a particular IngressClass as the default for your cluster. 
# 2. Setting the ingressclass.kubernetes.io/is-default-class annotation to true on an IngressClass resource will ensure that new Ingresses without an ingressClassName field specified will be assigned this default IngressClass.  
# 3. Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.3/guide/ingress/ingress_class/
```

## Step-07: Create IngressClass Resource
```t
# Navigate to Directory
cd 08-01-Load-Balancer-Controller-Install

# Create IngressClass Resource
kubectl apply -f kube-manifests

# Verify IngressClass Resource
kubectl get ingressclass

# Describe IngressClass Resource
kubectl describe ingressclass my-aws-ingress-class
```

## References
- [AWS Load Balancer Controller Install](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html)
- [ECR Repository per region](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)







