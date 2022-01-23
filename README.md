# Amazon CodeGuru Reviewer with GitHub Actions

Follow these steps to integrate [Amazon CodeGuru Reviewer](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/welcome.html) with [GitHub Actions](https://docs.github.com/en/actions):

## 1. Allow your GitHub Action Workflow to access resources in AWS

Your [GitHub Action Workflow](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#workflows) will need access to resources on your AWS account to create code reviews with CodeGuru Reviewer.

The recommended way to allow your workflow to access resources on your AWS account is through *short-lived credentials* using [OpenID Connect (OIDC)](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect).

You can use [this CloudFormation template](template.yml) to create all the resources required to configure Amazon CodeGuru Reviewer with GitHub Actions:

- An OpenID Connect (OIDC) Identity Provider for GitHub
- An Amazon S3 bucket to upload code and build artifacts for CodeGuru Reviewer
- An IAM role with access to the S3 bucket and [`AmazonCodeGuruReviewerFullAccess`](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/auth-and-access-control-iam-identity-based-access-control.html#managed-full-access) that can be assumed by the CodeGuru Reviewer workflow on your GitHub repo.

[![Launch Stack](https://s3.amazonaws.com/cloudformation-examples/cloudformation-launch-stack.png)](https://console.aws.amazon.com/cloudformation/home#/stacks/quickcreate?templateURL=https://anacunha.s3.amazonaws.com/codeguru-reviewer-github-actions-template.yml)

 If you prefer, you can also follow the instructions below:

- [Create an OpenID Connect identity provider on AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)

    - **Provider Type**: OpenID Connect
    - **Provider URL**: `https://token.actions.githubusercontent.com`
    - **Audience**: `sts.amazonaws.com`

- Create an S3 bucket with the prefix `codeguru-reviewer-` to upload your code and build artifacts for CodeGuru Reviewer.

- Create an IAM role assumed by the GitHub OIDC provider when running the [CodeGuru GitHub Action](https://github.com/marketplace/actions/codeguru-reviewer) workflow with the following trust and permissions policies:

    - Trust policy:

        ```json
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Effect": "Allow",
              "Principal": {
                "Federated": "arn:aws:iam::{AWS_ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
              },
              "Action": "sts:AssumeRoleWithWebIdentity",
              "Condition": {
                "StringLike": {
                  "token.actions.githubusercontent.com:sub": "repo:{GITHUB_ORG}/{GITHUB_REPO}:*"
                }
              }
            }
          ]
        }
        ```

    - Permissions:

        - [AmazonCodeGuruReviewerFullAccess](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/auth-and-access-control-iam-identity-based-access-control.html#managed-full-access)

        - Amazon S3 permissions for the `codeguru-reviewer-*` S3 bucket:
            - [`s3:PutObject`](https://docs.aws.amazon.com/AmazonS3/latest/API/API_PutObject.html)
            - [`s3:ListBucket`](https://docs.aws.amazon.com/AmazonS3/latest/API/API_ListObjectsV2.html)
            - [`s3:GetObject`](https://docs.aws.amazon.com/AmazonS3/latest/API/API_GetObject.html)

## 2. Create workflow file

Create you `workflow.yml` file inside `.github/workflows`:

```yml
name: CodeGuru Reviewer GitHub Actions Integration

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  CodeGuru-Reviewer-Actions:
    runs-on: ubuntu-latest

    permissions:
      # Required to interact with GitHub's OIDC Token endpoint.
      id-token: write
      # Required for Checkout action.
      contents: read
      # Required for CodeQL action (upload SARIF files).
      security-events: write

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v2
        with:
          # Required for CodeGuru Reviewer.
          fetch-depth: 0 # Fetches all history for all branches and tags.
      - name: Setup Java
        uses: actions/setup-java@v2
        with:
          distribution: 'temurin'
          java-version: '11'
      - name: Build with Maven
        run: mvn --batch-mode --update-snapshots verify
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v1
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_TO_ASSUME_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}
      - name: Amazon CodeGuru Reviewer
        uses: aws-actions/codeguru-reviewer@v1.1
        with:
          # Build artifacts directory. Only required for Java repositories.
          build_path: target
          # S3 Bucket with "codeguru-reviewer-*" prefix. Required.
          s3_bucket: ${{ secrets.AWS_CODEGURU_REVIEWER_S3_BUCKET }}
      - name: Upload review results
        uses: github/codeql-action/upload-sarif@v1
        with:
          sarif_file: codeguru-results.sarif.json
```

## Resources

- [Configuring OpenID Connect in Amazon Web Services](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)  (GitHub Docs)
- [Creating OpenID Connect (OIDC) identity providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html) (AWS Documentation)
- [Creating a role for web identity or OpenID connect federation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html) (AWS Documentation)
- [Configure AWS Credentials](https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions) (GitHub Action)
- [Amazon CodeGuru Reviewer](https://github.com/marketplace/actions/codeguru-reviewer) (GitHub Action)
- [Create code reviews with GitHub Actions](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/working-with-cicd.html) (AWS Documentation)
