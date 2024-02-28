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
just click here for windows: https://get.helm.sh/helm-v3.14.2-windows-amd64.zip
- [Install Helm](https://helm.sh/docs/intro/install/) if not installed
- [Install Helm for AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

```


### Step-03-02: Install AWS Load Balancer Controller

# Add the eks-charts repository.
helm repo add eks https://aws.github.io/eks-charts

# Update your local repo to make sure that you have the most recent charts.
helm repo update

# Install the AWS Load Balancer Controller.
## Template
```
```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=eksdemo1 --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller --set region=us-east-1 --set vpcId=vpc-0165a396e41e292a3 --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller
```
```
```


## Step-05: Ingress Class Concept

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
```

## Step-07: Create IngressClass Resource
```t
# Create IngressClass Resource
kubectl apply -f ingress-class.yaml

# Verify IngressClass Resource
kubectl get ingressclass

# Describe IngressClass Resource
kubectl describe ingressclass my-aws-ingress-class
`````
```
```
## Step-02: Review App1 Deployment kube-manifest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1-nginx-deployment
  labels:
    app: app1-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app1-nginx
  template:
    metadata:
      labels:
        app: app1-nginx
    spec:
      containers:
        - name: app1-nginx
          image: stacksimplify/kube-nginxapp1:1.0.0
          ports:
            - containerPort: 80
```
```
```
## Step-03: Review App1 NodePort Service
- **File Location:** `01-kube-manifests-default-backend/01-Nginx-App1-Deployment-and-NodePortService.yml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: app1-nginx-nodeport-service
  labels:
    app: app1-nginx
  annotations:
#Important Note:  Need to add health check path annotations in service level if we are planning to use multiple targets in a load balancer    
#    alb.ingress.kubernetes.io/healthcheck-path: /app1/index.html
spec:
  type: NodePort
  selector:
    app: app1-nginx
  ports:
    - port: 80
      targetPort: 80  
````
```
```

## Step-04: Create Ingress resource now
```yaml
# Annotations Reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/latest/guide/ingress/annotations/
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginxapp1
  labels:
    app: app1-nginx
  annotations:
    # Ingress Core Settings
    alb.ingress.kubernetes.io/scheme: internet-facing
spec:
  ingressClassName: my-aws-ingress-class
  defaultBackend:
    service:
      name: app1-nginx-nodeport-service
      port:
        number: 80                    
```

## Step-05: Deploy kube-manifests and Verify
```t

# Deploy kube-manifests
kubectl apply -f ingress.yaml

# Check the ingress resource
kubectl get ingress








