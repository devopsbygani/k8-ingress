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
--attach-policy-arn=arn:aws:iam::<acoount_id>:policy/AWSLoadBalancerControllerIAMPolicy \
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


### I faced a issue where it was unable to create a ALB.
```
  Warning  FailedDeployModel  40s (x7 over 3m23s)  ingress  (combined from similar events): Failed deploy model due to operation error Elastic Load Balancing v2: CreateTargetGroup, https response error StatusCode: 403, RequestID: 76243322-411a-4a5f-8bb6-bc5f61429c09, api error AccessDenied: User: arn:aws:sts::<account id>:assumed-role/eksctl-expense-addon-iamserviceaccount-kube-s-Role1-3rOnTN0LfaQM/1748622633297434864 is not authorized to perform: elasticloadbalancing:AddTags on resource: arn:aws:elasticloadbalancing:us-east-1:<account id>:targetgroup/k8s-default-app1-c31b714ecc/* because no identity-based policy allows the elasticloadbalancing:AddTags action
```

## resolution steps.

### delete iam role and recreating.

### Step 1: List all versions of the policy.
```bash
aws iam list-policy-versions \
  --policy-arn arn:aws:iam::<acoount_id>:policy/AWSLoadBalancerControllerIAMPolicy
```

example output 
```
{
  "Versions": [
    {
      "VersionId": "v5",
      "IsDefaultVersion": true
    },
    {
      "VersionId": "v4",
      "IsDefaultVersion": false
    },
    {
      "VersionId": "v3",
      "IsDefaultVersion": false
    }
  ]
}
```

### Step 2: Delete all non-default versions
```
aws iam delete-policy-version \
  --policy-arn arn:aws:iam::<acoount_id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --version-id v4

aws iam delete-policy-version \
  --policy-arn arn:aws:iam::<acoount_id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --version-id v3
```

Note : Repeat this until only the default version is left.

### Step 3: Now delete the policy

```bash
aws iam delete-policy \
  --policy-arn arn:aws:iam::<acoount_id>:policy/AWSLoadBalancerControllerIAMPolicy
```
if no error skip step4
### step4: Identify attached entities
```
if getting error An error occurred (DeleteConflict) when calling the DeletePolicy operation: Cannot delete a policy attached to entities.
```
step4.1:  Detach the policy from each role
For each role listed under PolicyRoles, run:
```bash
aws iam detach-role-policy \
  --role-name eksctl-expense-addon-iamserviceaccount-kube-s-Role1-3rOnTN0LfaQM \
  --policy-arn arn:aws:iam::<account_id>:policy/AWSLoadBalancerControllerIAMPolicy
```
step4.2:
```bash
aws iam delete-policy \
  --policy-arn arn:aws:iam::<acoount_id>:policy/AWSLoadBalancerControllerIAMPolicy
```

### step5: Recreate the policy with the correct content:
```bash
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam-policy.json
```

### step6: if helm already installed upgrade.
```bash
helm upgrade aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=expense \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```