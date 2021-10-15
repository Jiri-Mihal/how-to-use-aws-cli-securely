# How to use AWS CLI securely
Using your root long-term access key for the AWS CLI is very dangerous for obvious reasons. If you want to use the AWS CLI more securely, create a new IAM user with read-only privileges and allow other privileges only temporarily upon MFA.

This is a step-by-step guide inspired by the following articles:
- [Securing Local AWS Credentials](https://medium.com/starting-up-security/securing-local-aws-credentials-9589b56a0957)
- [AWS Security - Securing Your Use of the AWS CLI and Automation Tools](https://jack-vanlightly.com/blog/2018/8/14/aws-security-securing-your-use-of-the-aws-cli-and-automation-tools)

> Don't forget to replace brackets `[VALUE]` with your data!

## 1. Create a new admin user with limited privileges
1. Visit `https://console.aws.amazon.com/iamv2/home#/users`
2. Click `Add user`
3. Set the user name to `admin`
4. Select `Access key - Programmatic access`
5. Click button `Next: Permissions`
6. Click tab `Attach existing policies directly`
7. Search for `SecurityAudit` and check it
8. Click `Next: Tags`
9. Click `Next: Review`
10. Click `Create user`
11. Click `Download .csv` and keep it in a safe place

## 2. Set the admin user to require MFA
1. Visit `https://console.aws.amazon.com/iamv2/home#/users`
2. Click user `admin`
3. Click tab `Security credentials`
4. Click `Manage` next to the Assigned MFA device
5. Check `Virtual MFA device`
6. Click `Continue` and pair your device

## 3. Create a new admin role requiring MFA
1. Visit https://console.aws.amazon.com/iamv2/home#/roles
2. Click `Create role`
3. Click tab `Another AWS account`
4. Fill `Account ID` form field with your account ID. The account ID can be found under the drop-down arrow in the right corner.
5. Check `Require MFA`
6. Click `Next: Permissions`
7. Search for `AdministratorAccess` and check it
8. Click `Next: Tags`
9. Click `Next: Review`
10. Set the role name to `admin`
11. Click `Create role`

## 4. Allow the admin user to assume the admin role
You can [read more about AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) in the official docs.

1. Visit https://console.aws.amazon.com/iamv2/home#/users
2. Click user `admin`
3. Click `Add inline policy`
4. Click tab `JSON`
5. Paste the following configuration:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::[YOUR ACCOUNT ID]:role/admin",
            "Condition": {
                "Bool": {
                    "aws:MultiFactorAuthPresent": "true"
                }
            }
        }
    ]
}
```
6. Click `Review policy`
7. Set the policy name to `admin`
8. Click `Create policy`

## 5. Configure your AWS CLI
1. Run `aws configure`
2. Paste credentials you've got in step 1/11 :
```
AWS Access Key ID [None]: [ACCESS KEY ID FROM STEP 1/11]
AWS Secret Access Key [None]: [SECRET ACCESS KEY FROM STEP 1/11]
Default region name [None]: 
Default output format [None]: 
```
3. Add the following lines to file `~/.aws/config`:
```
[profile admin]
source_profile = default
role_arn = arn:aws:iam::[YOUR ACCOUNT ID]:role/admin
mfa_serial = arn:aws:iam::[YOUR ACCOUNT ID]:mfa/admin
```

## 6. Test It
1. Run `aws ec2 describe-instances --region=us-west-1`. It should succeed. It will run the command with the user you just made and `SecurityAudit` permissions.
2. Run `aws s3api create-bucket --bucket bucket-name --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1`. It should fail because `SecurityAudit` can't create, modify, or delete anything.
3. Next, we introduce `--profile` to the CLI command, which is configured to prompt for MFA into the `admin` role. This time you will be prompted for your MFA code and command should succeed:
`aws --profile admin s3api create-bucket --bucket bucket-name --region us-west-1 --create-bucket-configuration LocationConstraint=us-west-1`
4. You can dive into `~/.aws/cli/cache` and inspect the temporary credentials used to create the bucket. They're just like normal AWS credentials, with a session key. That key is useless to an attacker (and you) in only an hour.

## 7. Store temporary credentials in environment variables
When working with Terraform, Serverless etc. it may be useful storing your AWS credentials in environment variables. I prefer this simple solution. If you are looking for something more robust, check out [AWS-MFA](https://github.com/broamski/aws-mfa).

1. Make you sure you've installed `jq` CLI tool on your system.
2. Create shell script `get_aws_credentials.sh` with the following code:
```shell
#!/bin/bash

if [ $# -eq 1 ]; then
    CREDS=$(aws sts get-session-token --duration-seconds 3600 --serial-number arn:aws:iam::[YOUR ACCOUNT ID]:mfa/admin --token-code $1)
    export AWS_PROFILE=admin
    export AWS_ACCESS_KEY_ID=$(echo $CREDS | jq -r .Credentials.AccessKeyId)
    export AWS_SECRET_ACCESS_KEY=$(echo $CREDS | jq -r .Credentials.SecretAccessKey)
    export AWS_SESSION_TOKEN=$(echo $CREDS | jq -r .Credentials.SessionToken)  
    echo "You've got temporary credentials valid for 1 hour."
else
    echo "Pass your MFA token as an argument: get_aws_credentials.sh [MFA-code]"
fi
```
3. Source the script to create and store temporary credentials in environment variables:
`source get_aws_credentials.sh [MFA-code]`

5. Terraform users, don't forget to add `assume_role` to the `provider` block:
```
provider "aws" {
  region  = "us-west-1"
  profile = "default"
  assume_role {
    role_arn = "arn:aws:iam::[YOUR ACCOUNT ID]:role/admin"
  }
}
```