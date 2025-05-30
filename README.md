### reference: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/deploy/installation/

### Create IAM OIDC provider
``` bash
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster expense \
    --approve
```
### Download IAM policy for the AWS Load Balancer Controller
```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.10.0/docs/install/iam_policy.json
```
### Create an IAM policy called AWSLoadBalancerControllerIAMPolicy

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```
### Create a IAM role and ServiceAccount for the AWS Load Balancer controller, use the ARN from the step above
```bash
eksctl create iamserviceaccount \
--cluster=expense \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::905418383993:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--region us-east-1 \
--approve
```

### Add the EKS chart repo to helm
```bash
helm repo add eks https://aws.github.io/eks-charts
```

### Install the helm chart if using IAM roles for service accounts.

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=expense --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```
* The Above step setup a ALB controller drivers for connectivity between k8 and external ALB service.

### Additional steps:
* create a ACM certificate for HTTPS:// connectivity. 
* create a route53 record after ALB is created by chhosing Alisa as Application Load Balancer and select LB .
