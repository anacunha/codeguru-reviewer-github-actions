# Amazon CodeGuru Reviewer with GitHub Actions

Follow these steps to integrate [Amazon CodeGuru Reviewer](https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/welcome.html) with [GitHub Actions](https://docs.github.com/en/actions):

## 1. Configure GitHub OpenID Connect (OIDC) in AWS

Your [GitHub Action Workflow](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#workflows) will need access to resources on your AWS account to create code reviews with CodeGuru Reviewer.

The recommended way to allow your workflow to access resources on your AWS account is through *short-lived credentials* using [OpenID Connect (OIDC)](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect).

To configure GitHub's OIDC provider in AWS, follow these instructions:

- [Create an OpenID Connect identity provider on AWS](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html)

    - **Provider Type**: OpenID Connect
    - **Provider URL**: `https://token.actions.githubusercontent.com`
    - **Audience**: `sts.amazonaws.com`

## Resources

- [Configuring OpenID Connect in Amazon Web Services (GitHub Docs)](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services)
- https://github.com/marketplace/actions/codeguru-reviewer
- https://github.com/marketplace/actions/configure-aws-credentials-action-for-github-actions
- https://docs.aws.amazon.com/codeguru/latest/reviewer-ug/working-with-cicd.html
