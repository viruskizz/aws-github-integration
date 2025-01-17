# From AWS Native CI/CD to Github integration

This project demo is a part pf topic **From AWS native CI/CD to Github integration** that show how to deploy AWS resource from Github Actions.
Read more in  [Presentation]

## Architect

This project is serverless that build by AWS SAM tools

- AWS SAM
- CloudFormation
- AWS Lambda
- Amazon API Gateway

<img title="diagram" alt="diagram" src="/assets/diagram.png">

## Configuration

### Setup AWS IAM Role
Github Actions will assume IAM role with OIDC identity. To create a role follow these.

1. Create `permission.json`
  ```json
  {
      "Version": "2012-10-17",
      "Statement": [
          {
              "Sid": "AmazonAPIGatewayAdministrator",
              "Effect": "Allow",
              "Action": [
                  "apigateway:*"
              ],
              "Resource": "arn:aws:apigateway:*::/*"
          },
          {
              "Sid": "AmazonS3FullAccess",
              "Effect": "Allow",
              "Action": [
                  "s3:*",
                  "s3-object-lambda:*"
              ],
              "Resource": "*"
          },
          {
              "Sid": "AWSCloudFormationFullAccess",
              "Effect": "Allow",
              "Action": [
                  "cloudformation:*"
              ],
              "Resource": "*"
          },
          {
              "Sid": "AWSLambdaFullAccess1",
              "Effect": "Allow",
              "Action": [
                  "cloudformation:DescribeStacks",
                  "cloudformation:ListStackResources",
                  "cloudwatch:ListMetrics",
                  "cloudwatch:GetMetricData",
                  "ec2:DescribeSecurityGroups",
                  "ec2:DescribeSubnets",
                  "ec2:DescribeVpcs",
                  "kms:ListAliases",
                  "iam:GetPolicy",
                  "iam:GetPolicyVersion",
                  "iam:GetRole",
                  "iam:GetRolePolicy",
                  "iam:ListAttachedRolePolicies",
                  "iam:ListRolePolicies",
                  "iam:ListRoles",
                  "lambda:*",
                  "logs:DescribeLogGroups",
                  "states:DescribeStateMachine",
                  "states:ListStateMachines",
                  "tag:GetResources",
                  "xray:GetTraceSummaries",
                  "xray:BatchGetTraces"
              ],
              "Resource": "*"
          },
          {
              "Sid": "AWSLambdaFullAccess2",
              "Effect": "Allow",
              "Action": "iam:PassRole",
              "Resource": "*",
              "Condition": {
                  "StringEquals": {
                      "iam:PassedToService": "lambda.amazonaws.com"
                  }
              }
          },
          {
              "Sid": "AWSLambdaFullAccess3",
              "Effect": "Allow",
              "Action": [
                  "logs:DescribeLogStreams",
                  "logs:GetLogEvents",
                  "logs:FilterLogEvents"
              ],
              "Resource": "arn:aws:logs:*:*:log-group:/aws/lambda/*"
          },
          {
              "Sid": "IAMFullAccess",
              "Effect": "Allow",
              "Action": [
                  "iam:*",
                  "organizations:DescribeAccount",
                  "organizations:DescribeOrganization",
                  "organizations:DescribeOrganizationalUnit",
                  "organizations:DescribePolicy",
                  "organizations:ListChildren",
                  "organizations:ListParents",
                  "organizations:ListPoliciesForTarget",
                  "organizations:ListRoots",
                  "organizations:ListPolicies",
                  "organizations:ListTargetsForPolicy"
              ],
              "Resource": "*"
          }
      ]
  }
  ```

2. create `trust-relationship.json`
```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::<AWS Account ID>:oidc-provider/token.actions.githubusercontent.com"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
				},
				"StringLike": {
					"token.actions.githubusercontent.com:sub": "repo:<username or organization>/*"
				}
			}
		}
	]
}
```
- Replace `<AWS Account-ID>` and `<username or organization>`

3. Create IAM Role with OIDC

- Create Policy
  ```bash
  aws iam create-policy \
      --policy-name SAMDeploymentPolicy \
      --policy-document file://policy.json
  ```

- Create role
  ```bash
  aws iam create-role \
      --role-name github-oidc-role \
      --assume-role-policy-document file://trust-relationship.json
  ```

Attach policy to role
```bash
aws iam attach-role-policy \
    --policy-arn arn:aws:iam::aws:policy/SAMDeploymentPolicy \
    --role-name github-oidc-role
```

### Setup Github Actions

1. Add role arn that created to Github secret

  ```txt
  ROLE_ARN: <role_arn>
  ```

2. Add `.github/workflows/main.yml`

```yaml
on:
  push:
    branches:
      - main
env:
  AWS_REGION : "ap-southeast-1"

permissions:
  id-token: write   # This is required for requesting the JWT
  contents: read    # This is required for actions/

jobs:
  Deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: aws-actions/setup-sam@v2
        with:
          use-installer: true
          token: ${{ secrets.GITHUB_TOKEN }}
      - name: configure aws credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.ROLE_ARN }}
          role-session-name: github-oidc
          aws-region: ${{env.AWS_REGION}}
      - name: Sts GetCallerIdentity
        run: |
          aws sts get-caller-identity
      - name: SAM Validate
        run: |
          sam validate
      - name: SAM Build
        run: |
          sam build --use-container
      - name: SAM Deploy
        run: |
          sam deploy \
            --no-confirm-changeset \
            --no-fail-on-empty-changeset \
            --parameter-overrides ParameterKey=GithubCommit,ParameterValue=${{ github.sha }}
```

<!-- Link -->
[Presentation]: https://docs.google.com/presentation/d/1CqHM4DPxpVNSyi96u96OyI3SLdNhdpT_/edit?usp=drive_link&ouid=117377256572461095430&rtpof=true&sd=true