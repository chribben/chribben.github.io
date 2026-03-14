---
title: "Configure OpenID Connect in AWS for GitHub Actions"
date: 2026-03-14T12:00:00+01:00
permalink: configure-oidc-for-github-actions-and-aws
description: How to configure OpenID Connect in AWS to allow GitHub Actions to deploy to AWS without long-lived credentials
categories:
  - blog
tags:
  - AWS
  - GitHub Actions
  - OIDC
---

Using OpenID Connect (OIDC) lets your GitHub Actions workflows assume an AWS IAM role directly — no stored AWS access keys needed.

## 1. Create an OIDC Identity Provider in AWS

Go to **IAM → Identity providers → Add provider** and enter:

| Field | Value |
|-------|-------|
| Provider type | OpenID Connect |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

Click **Add provider**.

## 2. Create an IAM Role

Go to **IAM → Roles → Create role**:

1. Select **Web identity** as the trusted entity type.
2. Pick the identity provider you just created and audience `sts.amazonaws.com`.
3. Under **GitHub organization**, enter your GitHub org or username (e.g. `myorg`). You can optionally filter by repository and branch as well.
4. On the **Add permissions** page, skip without selecting any policy — click **Next**.
5. Name the role, e.g. `GitHubActionsDeployRole`, and create it.

### Add permissions for CDK

If you're deploying with CDK, the role only needs permission to assume the roles that `cdk bootstrap` created. Go to the role you just created, choose **Add permissions → Create inline policy**, and use:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/cdk-*"
    }
  ]
}
```

The CDK bootstrap roles (`cdk-deploy-role-*`, `cdk-file-publishing-role-*`, etc.) already have the necessary CloudFormation, S3, Lambda, and IAM permissions scoped to CDK operations. This is much safer than granting `AdministratorAccess`.

### Edit the trust policy

Now edit the role's trust policy to restrict it to your repo. Replace the `Condition` block:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_ORG>/<REPO_NAME>:*"
        }
      }
    }
  ]
}
```

Replace `<ACCOUNT_ID>`, `<GITHUB_ORG>`, and `<REPO_NAME>` with your values. You can make the `sub` condition more specific, e.g. `repo:myorg/myrepo:ref:refs/heads/main` to only allow the `main` branch.

## 3. Use It in a GitHub Actions Workflow

```yaml
name: Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsDeployRole
          aws-region: eu-west-1

      - run: aws sts get-caller-identity  # verify it works
```

The two critical pieces are:
- **`permissions.id-token: write`** — allows the workflow to request an OIDC token.
- **`role-to-assume`** — the ARN of the role you created in step 2.

That's it. No AWS keys to rotate, no secrets to manage.
