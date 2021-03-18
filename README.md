# AWS CloudFormation Shared VPC Model

## What it does
* Provides a sample template for building a 3 tier VPC with public/private subnets and a managed prefix list that are shared to targeted member accounts using Resource Access Manager. Using the [StackSet resource type](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudformation-stackset.html), the template also creates [Systems Manager (SSM) Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) keys in the member accounts for use in referencing VPC related values in workload templates.
* Provides a sample template that deploys an EC2 instance and Security Group that utilizes the shared VPC resources using the SSM Parameter Store keys.

## Prerequisites
1.	An AWS Organization with a management account and at least two member accounts. One will be your network account and the other(s) designated for workload resources.
2.	[AWS Command Line Interface](https://aws.amazon.com/cli/) installed and configured for access to all Organization accounts.
3.	A text editor for updating YAML files.
4.	A S3 bucket accessible from your Organization's accounts.  You can find an example of how to define an appropriate S3 bucket policy in this [AWS Security Blog post](https://aws.amazon.com/blogs/security/control-access-to-aws-resources-by-using-the-aws-organization-of-iam-principals/).

## Deployment Steps
### Setup the roles for cross-account access
1. Upload the [network-role-stackset.yaml](../blobs/mainline/--/network-role-stackset.yaml) to the S3 bucket identified in the prerequisites.
2. From the AWS Organizations manangement account, create an AWS CloudFormation stack set using the *network-role-stackset.yaml*. Replace the S3 bucket domain and account IDs with values from your environment.
```
aws cloudformation create-stack-set --stack-set-name network-parameter-role /
--parameters ParameterKey=networkAccountId,ParameterValue=network-account-id /
--template-url https://your-s3-bucket-domain/network-role-stackset.yaml
aws cloudformation create-stack-instances --stack-set-name network-parameter-role /
--accounts network-account-id workload-account1-id workload-account2-id
```

### Deploy a Shared VPC from the Network Account
1.	Edit the [shared-vpc.yaml](../blobs/mainline/--/shared-vpc.yaml) and replace *your-s3-bucket-domain* with the S3 bucket you set up in the prerequisites.
2.	Upload the [shared-vpc.yaml](../blobs/mainline/--/shared-vpc.yaml) and [ssm-parameter-stackset.yaml](../blobs/mainline/--/ssm-parameter-stackset.yaml) to your S3 bucket.
3.	Edit the [shared-vpc-input.yaml](../blobs/mainline/--/shared-vpc-input.yaml) and provide the parameter values specific to your AWS Organization.
4.	From the Network account, create the stack using the CLI command below.
```
aws cloudformation create-stack --stack-name shared-vpc /
--template-url https://your-s3-bucket-domain/shared-vpc.yaml /
--cli-input-yaml file://shared-vpc-input.yaml
```

### Deploy EC2 instance in a Member Account using the Shared Subnet
1.	Upload the [workload-instance.yaml](../blobs/mainline/--/workload-instance.yaml) to your S3 bucket.
2.	Edit the [workload-instance-input.yaml](../blobs/mainline/--/workload-instance-input.yaml) and provide the parameter values specific to your application account.
3.	From a Workload account, create the stack using the CLI command below.
```
aws cloudformation create-stack --stack-name workload-instance /
--template-url https://your-s3-bucket-domain/workload-instance.yaml /
--cli-input-yaml file://workload-instance-input.yaml
```

4. After your stack creation is completed, you can confirm your instance details in the console or by running the following CLI command.
```
aws ec2 describe-instances  --filter "Name=tag:Name,Values=AppServer1-Dev"
```

### Clean up
Set your CLI profile to the application account and use the following commands to delete the resources deployed to this account.
```
aws cloudformation delete-stack --stack-name workload-instance
```
Once complete, switch your CLI profile to the network account and use the following commands to delete the resources deployed to this account.
```
aws cloudformation delete-stack --stack-name shared-vpc
```
Once complete, switch your CLI profile to the Organization's management account and use the following commands to delete the resources deployed to this account.  Update the account id values to match those used during your initial deployment.
```
aws cloudformation delete-stack-instances --stack-set-name network-parameter-role /
--accounts network-account-id app-account1-id app-account2-id /
--no-retain-stacks
aws cloudformation delete-stack-set --stack-set-name network-parameter-role

```