# AWS CloudFormation Shared VPC Model

## What it does
* Provides a sample template for building a 3 tier VPC with public/private subnets and a managed prefix list that are shared to targeted member accounts using Resource Access Manager. Using the [StackSet resource type](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cloudformation-stackset.html), the template also creates [Systems Manager (SSM) Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) keys in the member accounts for use in referencing VPC related values in workload templates.
* Provides a sample template that deploys an EC2 instance and Security Group that demonstrates using the shared VPC resources by referencing the SSM Parameter Store keys.

## Prerequisites
1.	An AWS Organization with a management account and at least two member accounts. One will be your network account and the other(s) designated for workload resources.
2.	[AWS Command Line Interface](https://aws.amazon.com/cli/) installed and configured for access to all Organization accounts.
3.	A text editor for updating YAML files.
4.	A S3 bucket accessible from your Organization's accounts.  You can find an example of how to define an appropriate S3 bucket policy in this [AWS Security Blog post](https://aws.amazon.com/blogs/security/control-access-to-aws-resources-by-using-the-aws-organization-of-iam-principals/).

## Deployment Steps
### Setup the roles for cross-account access
1. From the AWS Organizations manangement account, create an AWS CloudFormation stack set using the [network-role-stackset.yaml](./network-role-stackset.yaml). Deploy the StackSet using the CLI commands below. Replace the S3 bucket domain, account IDs, organization ID, region ID with values from your environment.
```
aws cloudformation create-stack-set --stack-set-name network-parameter-role \
--parameters ParameterKey=networkAccountId,ParameterValue=network-account-id \
ParameterKey=organizationId,ParameterValue=organization-id \
--template-body file://network-role-stackset.yaml
aws cloudformation create-stack-instances --stack-set-name network-parameter-role \
--accounts network-account-id workload-account1-id workload-account2-id --regions region-id
```

### Deploy a Shared VPC from the Network Account
1.	Upload the [ssm-parameter-stackset.yaml](./ssm-parameter-stackset.yaml) to your S3 bucket.
2.	Edit the [shared-vpc-input.json](./shared-vpc-input.json) and provide the parameter values specific to your AWS Organization.
3.	From the Network account, create the stack using the CLI command below.
```
aws cloudformation deploy --template-file shared-vpc.yaml \
--stack-name shared-vpc --parameter-overrides file://shared-vpc-input.json \
--capabilities CAPABILITY_AUTO_EXPAND CAPABILITY_NAMED_IAM CAPABILITY_IAM
```

### Deploy EC2 instance in a Member Account using the Shared Subnet
1.	Edit the [workload-instance-input.json](./workload-instance-input.yaml) and provide the parameter values specific to your workload account.
2.	From the Workload account, create the stack using the CLI command below.
```
aws cloudformation deploy --template-file workload-instance.yaml \
--stack-name workload-instance --parameter-overrides file://workload-instance-input.json \
--capabilities CAPABILITY_IAM
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
Once complete, switch your CLI profile to the Organization's management account and use the following commands to delete the resources deployed to this account.  Update the account ID and region ID values to match those used during your initial deployment.
```
aws cloudformation delete-stack-instances --stack-set-name network-parameter-role \
--accounts network-account-id app-account1-id app-account2-id \
--regions region-id --no-retain-stacks
aws cloudformation delete-stack-set --stack-set-name network-parameter-role
```