---
title: "Build a full-stack app with Lovable and AWS"
date: 2026-04-05T10:30:00+01:00
permalink: full-stack-application-lovable-aws
description: Use Lovable for the frontend, then replace Supabase with AWS primitives and deploy with CDK and GitHub Actions.
categories:
  - blog
tags:
  - Lovable
  - AWS
  - Full stack
---

Lovable is very good at getting you from prompt to polished frontend quickly. The part I usually do not keep is the default backend choice.

Lovable's Supabase integration is convenient, but for side projects and bursty workloads I often prefer AWS services with usage-based pricing: S3, CloudFront, Lambda, API Gateway, DynamoDB, and Cognito. The good news is that this is usually a manageable swap because Lovable outputs normal application code. If you keep a clean seam between UI and backend, you can keep the good-looking frontend and replace the rest.

This is the flow I use:

1. Prompt and iterate on the UI in Lovable.
2. Export the project to GitHub early.
3. Replace Supabase-specific code with your own AWS-backed API and auth flow.
4. Define the infrastructure in CDK.
5. Deploy from GitHub Actions using OIDC.
6. Put the app behind your own domain.

## 1. Prompt the app in Lovable

The easiest way to make the later migration painless is to avoid coupling the UI directly to Supabase from the start.

When prompting Lovable, ask for:

- Mock data first
- A dedicated API layer
- Environment variables for backend endpoints
- Clear auth boundaries
- Loading, empty, and error states

An example prompt:

```text
Build a responsive web app for tracking personal investments.
Use mock data initially.
Put all backend calls behind a single service layer such as src/lib/api.ts.
Do not call Supabase directly from UI components.
Use environment variables for API base URL and auth settings.
Add loading states, error states, and empty states.
```

That one constraint, "do not call the backend directly from components", matters a lot. If the generated app has 40 places importing a Supabase client, your migration becomes annoying immediately.

## 2. Connect to GitHub early

As soon as the UI is roughly right, export the project to GitHub and start working locally. Do not wait until the product is "done".

Working in Git early gives you a clean before/after point for the migration and makes it much easier to refactor generated code into something you actually want to own.

Typical first step:

```bash
npm install
npm run dev
```

Before touching the backend, I like to create a branch or at least keep a commit that still reflects the original generated version. That gives you a stable fallback if you need to compare behavior later.

## 3. Do the swap: Supabase out, AWS in

The cleanest mental model is to map each Supabase concern to a specific AWS service:

| Supabase capability  | AWS replacement                  |
| -------------------- | -------------------------------- |
| Static hosting       | S3 + CloudFront                  |
| Auth                 | Cognito                          |
| Edge Functions / API | API Gateway + Lambda             |
| Postgres             | DynamoDB or Aurora Serverless v2 |
| Storage              | S3                               |
| Async jobs           | SQS / EventBridge                |

For a low-ops, mostly pay-per-use stack, my default is:

- CloudFront + S3 for the frontend
- Cognito for sign-up and sign-in
- API Gateway + Lambda for backend APIs
- DynamoDB for application data
- S3 for uploads

If the app genuinely needs relational queries, transactions across many tables, or SQL-heavy reporting, then Aurora can be the right choice. But if your app is mostly CRUD plus a few workflows, DynamoDB keeps the operating model much simpler.

### Frontend hosting

The frontend side is straightforward. Build the app into static assets and host them on S3 behind CloudFront.

That gives you:

- Cheap hosting
- HTTPS through CloudFront
- CDN caching
- Easy custom domain support

For single-page apps, remember to configure CloudFront so 403 and 404 responses fall back to `index.html`, otherwise client-side routes will break on refresh.

### Auth with Cognito

The simplest route here is usually Cognito User Pools plus the Hosted UI. That avoids re-implementing auth screens and keeps token handling standard.

On the frontend, I like to reduce auth and API access to a thin wrapper around environment variables and a token-aware fetch function.

```ts
import { fetchAuthSession } from "aws-amplify/auth";

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL;

export async function api<T>(path: string, init: RequestInit = {}): Promise<T> {
  const session = await fetchAuthSession();
  const token = session.tokens?.idToken?.toString();

  const response = await fetch(`${API_BASE_URL}${path}`, {
    ...init,
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...init.headers,
    },
  });

  if (!response.ok) {
    throw new Error(`API request failed with status ${response.status}`);
  }

  return response.json() as Promise<T>;
}
```

The key point is not the exact library choice. The key point is that the UI now talks to your API boundary, not directly to a database SDK.

### API and data layer

For many apps, one HTTP API plus a handful of Lambda functions is enough:

- `GET /items`
- `POST /items`
- `GET /me`
- `POST /uploads/presign`

Keep the Lambdas boring. Validate input, read or write data, return JSON, move on. You can always split services later if the app grows.

If the frontend previously called Supabase directly for reads and writes, convert those calls into normal `fetch()` requests against API Gateway. The resulting architecture is less magical and much easier to reason about.

## 4. Remove all traces of Lovable and Supabase

This is partly technical and partly about ownership.

After the AWS pieces are in place, search for the old integration points and delete them aggressively:

```bash
rg -n "supabase|SUPABASE|lovable|Lovable" .
```

Things I usually remove or rename:

- `@supabase/supabase-js`
- Generated Supabase client helpers
- Supabase environment variables
- Auth guards tied to Supabase session objects
- Database type files generated from the old schema
- README text describing the app as a Lovable project

If the generated code contains a folder such as `src/integrations/supabase`, delete it rather than leaving dead abstractions around. Future you does not benefit from archaeological layers.

Also check the package manifest once you are done:

```bash
npm uninstall @supabase/supabase-js
```

You do not need to erase every sign that Lovable helped build the first version. You do need to remove the parts that make the codebase confusing or misleading.

## 5. Add infrastructure as code with CDK

Once the swap works locally, codify it. Do not leave buckets, user pools, and distributions as click-ops.

I usually create a separate `infra/` directory with a small CDK app that owns:

- S3 bucket for the frontend
- CloudFront distribution
- Cognito User Pool and App Client
- API Gateway
- Lambda functions
- DynamoDB tables
- DNS and certificate resources where relevant

A minimal stack shape looks like this:

```ts
import * as cdk from "aws-cdk-lib";
import * as cloudfront from "aws-cdk-lib/aws-cloudfront";
import * as origins from "aws-cdk-lib/aws-cloudfront-origins";
import * as s3 from "aws-cdk-lib/aws-s3";
import { Construct } from "constructs";

export class AppStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const siteBucket = new s3.Bucket(this, "SiteBucket", {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
      encryption: s3.BucketEncryption.S3_MANAGED,
    });

    new cloudfront.Distribution(this, "SiteDistribution", {
      defaultBehavior: {
        origin: origins.S3BucketOrigin.withOriginAccessControl(siteBucket),
        viewerProtocolPolicy: cloudfront.ViewerProtocolPolicy.REDIRECT_TO_HTTPS,
      },
      defaultRootObject: "index.html",
      errorResponses: [
        {
          httpStatus: 403,
          responseHttpStatus: 200,
          responsePagePath: "/index.html",
        },
        {
          httpStatus: 404,
          responseHttpStatus: 200,
          responsePagePath: "/index.html",
        },
      ],
    });
  }
}
```

Even if the first version is small, CDK pays off quickly. You can create preview environments, reproduce production cleanly, and stop depending on remembered console settings.

## 6. Add CI/CD with GitHub Actions

Once infrastructure lives in CDK, deployment becomes mechanical.

My preference is to use GitHub Actions with AWS OpenID Connect so there are no long-lived AWS secrets stored in GitHub. I already wrote the OIDC setup in a separate post: [Configure OpenID Connect in AWS for GitHub Actions](/configure-oidc-for-github-actions-and-aws).

A typical deployment workflow looks like this:

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

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci
      - run: npm run build

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsDeployRole
          aws-region: eu-west-1

      - run: npm ci
        working-directory: infra

      - run: npx cdk deploy --require-approval never
        working-directory: infra
```

There are two reasonable ways to handle the frontend assets:

1. Let CDK deploy the built frontend artifacts to S3.
2. Deploy infra with CDK, then run `aws s3 sync` plus a CloudFront invalidation as a separate step.

I usually prefer the first option for small projects because the whole deployment stays in one place.

## 7. Add a domain

If you already own a domain, expose the app on a subdomain such as `app.example.com`. That is usually simpler than trying to put CloudFront directly on the root domain when your registrar is not Route 53.

For CloudFront, request the ACM certificate in `us-east-1`. That part is easy to miss and CloudFront will not use a certificate from another region.

High-level flow:

1. Request an ACM certificate for `app.example.com` in `us-east-1`.
2. Validate the certificate.
3. Attach it to the CloudFront distribution.
4. In your registrar DNS, create a `CNAME` for `app` pointing to the CloudFront distribution domain.

If you use Namecheap, the DNS entry typically looks like this:

| Type         | Host  | Value                        |
| ------------ | ----- | ---------------------------- |
| CNAME Record | `app` | `d123example.cloudfront.net` |

Once DNS propagates, the app is live on your own domain with HTTPS handled by CloudFront.

## Final thought

The real trick here is not "Lovable plus AWS" as such. It is using Lovable for what it is best at - fast UI generation - and then taking ownership of the system boundaries early.

If you export to GitHub quickly, keep backend access behind a thin API layer, and move infrastructure into CDK, you end up with a stack that keeps the speed of AI-assisted frontend generation without locking your application to the default backend choices.
