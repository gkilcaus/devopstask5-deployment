# devopstask5-deployment

## Requirements:
 - AWS CLI
 - AWS Credentials configured (if named profile used, please append --profile <profile_name> to each aws cli command)
 - git / jq

## How to

1. Clone this repository
```
git clone https://github.com/gkilcaus/devopstask5-deployment.git
```
2. Clone Lambda's repository
```
git clone https://github.com/gkilcaus/devopstask5-lambda.git
```
3. deploy deployment stack
```bash
aws cloudformation deploy --template-file ./devopstask5-deployment/deploy.yaml --stack-name codedeployenv --region eu-west-1 --capabilities CAPABILITY_NAMED_IAM CAPABILITY_IAM
```
4. Look for outputs
```bash
bucket=$(aws cloudformation describe-stacks --stack-name codedeployenv --region eu-west-1 | jq -r '.Stacks[0].Outputs[] | select(.OutputKey=="BucketName").OutputValue')
codebuildproject=$(aws cloudformation describe-stacks --stack-name codedeployenv --region eu-west-1  | jq -r '.Stacks[0].Outputs[] | select(.OutputKey=="ProjectName").OutputValue')
```
5. Copy Lambda's repository to S3
```bash
aws s3 cp ./devopstask5-lambda/ s3://${bucket}/lambda/ --recursive
```
6. Invoke CodeBuild project
```bash
aws codebuild start-build --project-name $codebuildproject --region eu-west-1
```

## Notes / Assumptions / etc
- CodeBuild job needs to be manually triggered because source is in S3 (only as part of the task)
  - GitHub would require manual setup in console for access from CodeBuild and created oauth should be then updated to template
  - CodeCommit complicates code movement from GitHub because of need for IAM User and special credentials.
    - using CodeCommit would be prefered way
- AWS SAM was used to deploy lambda and all required components (serverless application)
  - I've hit a bug in SAM where failure in first deployment of SAM Template for whatever reason would end-up ROLLCBACK_COMPLETE state and next deployments would fail (https://github.com/aws/aws-sam-cli/issues/2191). As a workaround, stack would need manual deletion first.
- Lambda function has parameters for start and stop schedules, but for sake of simplicity default values were provided and used.
- Resources do not have finite names because of secondary stack is being built during CodeBuild run, so that would fail in conflicts
- SAM accelerates creation of Serverless App but gives quite limited options for pipeline. Not using SAM would give better options to utilize CodeDeploy for lambda deployments and not using CodeBuild for deployment but just for building artifcts.
- Lambda function:
  - uses aws_lambda_powertools library for logging & tracing, x-ray is enabled
  - error handling around API calls
  - multithreading for multiple regions at the same time to better utilize CPU and lower total time
  - assumes that production instances are tagged with tag "Environment" and values ['prd', 'production'] (case not sensitive), otherwise it's treated as non prod.
  - Uses 2 EventBridge Schedules to trigger either stop or start
    - In real life scenario this could be a DynamoDB table with schedule for each individual EC2 (similar tool is provided by AWS out-of-the-box)
  - IAM Role used by lambda function has least permissions for EC2. In case EC2 instances are with encrypted storage, policy should be modified to include few "kms" permissions (otherwise, function would fail to start them).
  - writing python code didn't use list comprehension for better readability