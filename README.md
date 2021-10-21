# Kubernetes application-Terraform-aws-EKS
Deploy a full AWS EKS cluster with Terraform
# What resources are created
1.VPC
2.Internet Gateway (IGW)
3.Public and Private Subnets
4.Security Groups, Route Tables and Route Table Associations
5.IAM roles, instance profiles and policies
6.An EKS Cluster
7.EKS Managed Node group
8.Autoscaling group and Launch Configuration
9.Worker Nodes in a private Subnet
10.bastion host for ssh access to the VPC
11.The ConfigMap required to register Nodes with EKS
12.KUBECONFIG file to authenticate kubectl using the aws eks get-token command. needs awscli version 1.16.156 >
# Configuration

You can configure you config with the following input variables:
|      Name           |    Description  	                     |    Default                                  |   
|---------------------|----------------------------------------|---------------------------------------------|
|  cluster-name       |    The name of your EKS Cluster        |      eks-cluster                            |
|  aws-region         |    The AWS Region to deploy EKS        |       us-east-1                             |   
|  availability-zones |    AWS Availability Zones              |  ["us-east-1a", "us-east-1b", "us-east-1c"] |   
|  k8s-version        |The desired K8s version to launch 17.1.0|                                             |
| node-instance-type  |Worker Node EC2 instance type           |t2.micro                                     |
| root-block-size     |Size of the root EBS block device       |20                                           |
|desired-capacit      |Autoscaling Desired node capacity	     |2                                            |
|max-size             |   Autoscaling Maximum node capacity	5  |5       
|min-size             |	Autoscaling Minimum node capacity	     |1
|vpc-subnet-cidr	    |Subnet CIDR	                           |10.0.0.0/16
|private-subnet-cidr	|Private Subnet CIDR ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]|                    |
|public-subnet-cidr   |Public Subnet CIDR ["10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]|
|db-subnet-cidr       |	DB/Spare Subnet CIDR	["10.0.192.0/21", "10.0.200.0/21", "10.0.208.0/21"]|           |
|eks-cw-logging	EKS Logging Components	["api", "audit", "authenticator", "controllerManager", "scheduler"]  |
|ec2-key-public-key	  |EC2 Key Pair for bastion and nodes	     |ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQD3F6tyPEFEzV0LX3X8BsXdMsQz1x2cEikKDEY0aIj41qgxMCP/iteneqXSIFZBp5vizPvaoIR3Um9xK7PGoW8giupGn+EPuxIA4cDM4vzOqOkiMPhz5XK0whEjkVzTo4+S0puvDZuwIsdiW9mxhJc7tgBNL0cYlWSYVkz4G/fslNfRPW5mYAM49f4fhtxPb5ok4Q2Lg9dPKVHO/Bgeu5woMc7RY0p1ej6D4CKFE6lymSDJpW0YHX/wqE9+cfEauh7xZcG0q9t2ta6F6fmX0agvpFyZo8aFbXeUBr7osSCJNgvavWbM/06niWrOvYX2xwWdhXmXSrbX8ZbabVohBK41 email@example.com|
|# You can create a file called terraform.tfvars or copy variables.tf into the project root, if you would like to over-ride the defaults Or by using variables.tf or a tfvars file:|
# IAM
The AWS credentials must be associated with a user having at least the following AWS managed IAM policies
----------------------------------------------------------------------------------------------------------------
1.IAMFullAccess
2.AutoScalingFullAccess
3.AmazonEKSClusterPolicy
4.AmazonEKSWorkerNodePolicy
5.AmazonVPCFullAccess
6.AmazonEKSServicePolicy
7.AmazonEKS_CNI_Policy
8.AmazonEC2FullAccess
In addition, you will need to create the following managed policies

EKS
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "eks:*"
            ],
            "Resource": "*"
        }
    ]
} 
```
# Terraform
You need to run the following commands to create the resources with Terraform:
```
terraform init
terraform plan
terraform apply
```
TIP: you should save the plan state ```terraform plan -out eks-state ```or even better yet, setup ```remote storage ``` for Terraform state. You can store state in an S3 backend, with locking via DynamoDB

# Setup kubectl
# Setup your``` KUBECONFIG```

```terraform output kubeconfig > ~/.kube/eks-cluster
export KUBECONFIG=~/.kube/eks-cluster
```
# Authorize users to access the cluster
Initially, only the system that deployed the cluster will be able to access the cluster. To authorize other users for accessing the cluster, aws-auth config needs to be modified by using the steps given below:

 Open the aws-auth file in the edit mode on the machine that has been used to deploy EKS cluster:
``` sudo kubectl edit -n kube-system configmap/aws-auth  ```

# Add the following configuration in that file by changing the placeholders:
``` mapUsers: |
     -userarn: arn:aws:iam::111122223333:user/<username>
     username: <username>
     groups:
      - system:masters 
 ```
    
# So, the final configuration would look like this:

 ``` apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::555555555555:role/devel-worker-nodes-NodeInstanceRole-74RF4UBDUKL6
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: arn:aws:iam::111122223333:user/<username>
      username: <username>
      groups:
        - system:masters 
```
        
# Once the user map is added in the configuration we need to create cluster role binding for that user:
``` kubectl create clusterrolebinding ops-user-cluster-admin-binding-<username> --clusterrole=cluster-admin --user=<username> ```
# Replace the placeholder with proper values

# Cleaning up
You can destroy this cluster entirely by running:

```terraform plan -destroy
     terraform destroy  --force 
```
