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
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy_latest.json

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

## Step-04: Install the AWS Load Balancer Controller using Helm V3 
### Step-04-01: Install Helm
- [Install Helm](https://helm.sh/docs/intro/install/) if not installed
- [Install Helm for AWS EKS](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)
```t
# Install Helm (if not installed) MacOS
brew install helm

# Verify Helm version
helm version
```
### Step-04-02: Install AWS Load Balancer Controller
- **Important-Note-1:** If you're deploying the controller to Amazon EC2 nodes that have restricted access to the Amazon EC2 instance metadata service (IMDS), or if you're deploying to Fargate, then add the following flags to the command that you run:
```t
--set region=region-code
--set vpcId=vpc-xxxxxxxx
```
- **Important-Note-2:** If you're deploying to any Region other than us-west-2, then add the following flag to the command that you run, replacing account and region-code with the values for your region listed in Amazon EKS add-on container image addresses.
- [Get Region Code and Account info](https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)
```t
--set image.repository=account.dkr.ecr.region-code.amazonaws.com/amazon/aws-load-balancer-controller
```
```t
# Add the eks-charts repository.
helm repo add eks https://aws.github.io/eks-charts

# Update your local repo to make sure that you have the most recent charts.
helm repo update

# Install the AWS Load Balancer Controller.
## Template
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=<region-code> \
  --set vpcId=<vpc-xxxxxxxx> \
  --set image.repository=<account>.dkr.ecr.<region-code>.amazonaws.com/amazon/aws-load-balancer-controller

## Replace Cluster Name, Region Code, VPC ID, Image Repo Account ID and Region Code  
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eksdemo1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0165a396e41e292a3 \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller
```
- **Sample output for AWS Load Balancer Controller Install steps**
```t
## Sample Ouput for AWS Load Balancer Controller Install steps
Kalyans-MacBook-Pro:08-01-Load-Balancer-Controller-Install kdaida$ helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
>   -n kube-system \
>   --set clusterName=eksdemo1 \
>   --set serviceAccount.create=false \
>   --set serviceAccount.name=aws-load-balancer-controller \
>   --set region=us-east-1 \
>   --set vpcId=vpc-0570fda59c5aaf192 \
>   --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller
NAME: aws-load-balancer-controller
LAST DEPLOYED: Wed Feb  2 10:33:57 2022
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS Load Balancer controller installed!
Kalyans-MacBook-Pro:08-01-Load-Balancer-Controller-Install kdaida$ 
```
### Step-04-03: Verify that the controller is installed and Webhook Service created
```t
# Verify that the controller is installed.
kubectl -n kube-system get deployment 
kubectl -n kube-system get deployment aws-load-balancer-controller
kubectl -n kube-system describe deployment aws-load-balancer-controller

# Sample Output
Kalyans-MacBook-Pro:08-01-Load-Balancer-Controller-Install kdaida$ kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           27s
Kalyans-MacBook-Pro:08-01-Load-Balancer-Controller-Install kdaida$ 

# Verify AWS Load Balancer Controller Webhook service created
kubectl -n kube-system get svc 
kubectl -n kube-system get svc aws-load-balancer-webhook-service
kubectl -n kube-system describe svc aws-load-balancer-webhook-service

# Sample Output
Kalyans-MacBook-Pro:aws-eks-kubernetes-masterclass-internal kdaida$ kubectl -n kube-system get svc aws-load-balancer-webhook-service
NAME                                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
aws-load-balancer-webhook-service   ClusterIP   10.100.53.52   <none>        443/TCP   61m
Kalyans-MacBook-Pro:aws-eks-kubernetes-masterclass-internal kdaida$ 

# Verify Labels in Service and Selector Labels in Deployment
kubectl -n kube-system get svc aws-load-balancer-webhook-service -o yaml
kubectl -n kube-system get deployment aws-load-balancer-controller -o yaml
Observation:
1. Verify "spec.selector" label in "aws-load-balancer-webhook-service"
2. Compare it with "aws-load-balancer-controller" Deployment "spec.selector.matchLabels"
3. Both values should be same which traffic coming to "aws-load-balancer-webhook-service" on port 443 will be sent to port 9443 on "aws-load-balancer-controller" deployment related pods. 
```

### Step-04-04: Verify AWS Load Balancer Controller Logs
```t
# List Pods
kubectl get pods -n kube-system

# Review logs for AWS LB Controller POD-1
kubectl -n kube-system logs -f <POD-NAME> 
kubectl -n kube-system logs -f  aws-load-balancer-controller-86b598cbd6-5pjfk

# Review logs for AWS LB Controller POD-2
kubectl -n kube-system logs -f <POD-NAME> 
kubectl -n kube-system logs -f aws-load-balancer-controller-86b598cbd6-vqqsk
```

### Step-04-05: Verify AWS Load Balancer Controller k8s Service Account - Internals 
```t
# List Service Account and its secret
kubectl -n kube-system get sa aws-load-balancer-controller
kubectl -n kube-system get sa aws-load-balancer-controller -o yaml
kubectl -n kube-system get secret <GET_FROM_PREVIOUS_COMMAND - secrets.name> -o yaml
kubectl -n kube-system get secret aws-load-balancer-controller-token-5w8th 
kubectl -n kube-system get secret aws-load-balancer-controller-token-5w8th -o yaml
## Decoce ca.crt using below two websites
https://www.base64decode.org/
https://www.sslchecker.com/certdecoder

## Decode token using below two websites
https://www.base64decode.org/
https://jwt.io/
Observation:
1. Review decoded JWT Token

# List Deployment in YAML format
kubectl -n kube-system get deploy aws-load-balancer-controller -o yaml
Observation:
1. Verify "spec.template.spec.serviceAccount" and "spec.template.spec.serviceAccountName" in "aws-load-balancer-controller" Deployment
2. We should find the Service Account Name as "aws-load-balancer-controller"

# List Pods in YAML format
kubectl -n kube-system get pods
kubectl -n kube-system get pod <AWS-Load-Balancer-Controller-POD-NAME> -o yaml
kubectl -n kube-system get pod aws-load-balancer-controller-65b4f64d6c-h2vh4 -o yaml
Observation:
1. Verify "spec.serviceAccount" and "spec.serviceAccountName"
2. We should find the Service Account Name as "aws-load-balancer-controller"
3. Verify "spec.volumes". You should find something as below, which is a temporary credentials to access AWS Services
CHECK-1: Verify "spec.volumes.name = aws-iam-token"
  - name: aws-iam-token
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          audience: sts.amazonaws.com
          expirationSeconds: 86400
          path: token
CHECK-2: Verify Volume Mounts
    volumeMounts:
    - mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
      name: aws-iam-token
      readOnly: true          
CHECK-3: Verify ENVs whose path name is "token"
    - name: AWS_WEB_IDENTITY_TOKEN_FILE
      value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token          
```

### Step-04-06: Verify TLS Certs for AWS Load Balancer Controller - Internals
```t
# List aws-load-balancer-tls secret 
kubectl -n kube-system get secret aws-load-balancer-tls -o yaml

# Verify the ca.crt and tls.crt in below websites
https://www.base64decode.org/
https://www.sslchecker.com/certdecoder

# Make a note of Common Name and SAN from above 
Common Name: aws-load-balancer-controller
SAN: aws-load-balancer-webhook-service.kube-system, aws-load-balancer-webhook-service.kube-system.svc

# List Pods in YAML format
kubectl -n kube-system get pods
kubectl -n kube-system get pod <AWS-Load-Balancer-Controller-POD-NAME> -o yaml
kubectl -n kube-system get pod aws-load-balancer-controller-65b4f64d6c-h2vh4 -o yaml
Observation:
1. Verify how the secret is mounted in AWS Load Balancer Controller Pod
CHECK-2: Verify Volume Mounts
    volumeMounts:
    - mountPath: /tmp/k8s-webhook-server/serving-certs
      name: cert
      readOnly: true
CHECK-3: Verify Volumes
  volumes:
  - name: cert
    secret:
      defaultMode: 420
      secretName: aws-load-balancer-tls
```

### Step-04-07: UNINSTALL AWS Load Balancer Controller using Helm Command (Information Purpose - SHOULD NOT EXECUTE THIS COMMAND)
- This step should not be implemented.
- This is just put it here for us to know how to uninstall aws load balancer controller from EKS Cluster
```t
# Uninstall AWS Load Balancer Controller
helm uninstall aws-load-balancer-controller -n kube-system 
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








