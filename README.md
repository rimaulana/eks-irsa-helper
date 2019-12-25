# EKS IRSA (IAM Role for Service Account) Helper

This script (**irsa.sh**) helps create an IAM role for a service account in an EKS (Kubernetes) cluster using OpenID Connect (OIDC). The idea behind this script is to provide a better way to manage and automate IAM role creation for EKS service account. 

The current solution using eksctl create iamserviceaccount command has a limitation where the role can only be associated with managed policies. An update to this role will require to delete and recreate the role, and this will cause the pod to lose authentication to AWS API temporarily due to missing IAM policies.

irsa.sh script will not create any Kubernetes service account. It will only add annotation **eks.amazonaws.com/role-arn** into the specified service account. The script will be able to add inline policies via *--policy-document* flag and managed policies using *--policy-arn*.

## Prerequisite
- AWS CLI >= 1.16.298
- jq (any version should be fine)
- curl (any version should be fine)
- kubectl >= 1.14.0

## Minimum IAM policies for the script to work
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "eks-irsa-helper-policy",
            "Effect": "Allow",
            "Action": [
                "iam:UpdateAssumeRolePolicy",
                "iam:GetRole",
                "iam:DetachRolePolicy",
                "iam:ListAttachedRolePolicies",
                "iam:DeleteRolePolicy",
                "iam:CreateRole",
                "iam:DeleteRole",
                "eks:DescribeCluster",
                "iam:AttachRolePolicy",
                "iam:PutRolePolicy",
                "iam:ListRolePolicies"
            ],
            "Resource": [
                "arn:aws:eks:*:*:cluster/*",
                "arn:aws:iam::*:role/*"
            ]
        }
    ]
}
```

## Usage
```text
usage: ./irsa.sh [ensure|delete] [options]
Associate a service account with an IAM policy documents or IAM managed policy arns

-h,--help print this help
--cluster The name of the EKS cluster. (default: eks-fargate)
--region The EKS cluster region. (default: us-east-2)
--sa-name The name of the service account. (default: alb-ingress-controller)
--namespace The namespace of the service account. (default: kube-system)
--policy-document The name of the policy document file, can be use multiple times, if it is a URL, it will be downloaded first. For local file, use the file path without any file:// prefix
--policy-arn the arn of managed policy, can be use multiple times
```

## Example
### Example 1
Create IAM role for a service account named **cluster-autoscaler** in EKS cluster **k8s-dev** in **us-east-1** region in **kube-system** namespace.

```bash
cat > policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*"
        }
    ]
}
EOF

./irsa.sh ensure \
    --cluster k8s-dev \
    --region us-east-1 \
    --sa-name cluster-autoscaler \
    --namespace kube-system \
    --policy-document policy.json
```

### Example 2
Create IAM role for service account **alb-ingress-controller** in **kube-system** for EKS cluster **eks-fargate** in **eu-west-2** region with policy document referencing from an http file.

```bash
./irsa.sh ensure \
    --cluster eks-fargate \
    --region eu-west-2 \
    --sa-name alb-ingress-controller \
    --namespace kube-system \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/iam-policy.json
```

### Example 3
Create IAM Role for service account **s3streamer** in **streamer namespace** for EKS cluster **streamer-prod** in **us-west-2**. IAM policies will be comprises of managed policies
- arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
- arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
as well as some policy documents

```bash
cat > policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup"
            ],
            "Resource": "*"
        }
    ]
}
EOF

./irsa.sh ensure \
    --cluster streamer-prod \
    --region us-west-2 \
    --sa-name s3streamer \
    --namespace streamer \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
    --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess \
    --policy-document policy.json \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/iam-policy.json
```
## Updating and Deleting the IAM role
Both ensure and delete command are **idempotent** 

To update the IAM role, you can execute the irsa.sh ensure command with the updated parameters. If the IAM role does not exist, it will create a new one, but if it exists, the role will be adjusted based on the given *--policy-arn* and *--policy-document* parameters.

When deleting the IAM role, the required flags are *--cluster*, *--region*, *--sa-name* and *--namespace*.
