---
title: "Simple and secure deployment with Github Actions OpenID Connect (OIDC)"
author: "Moritz Körber"
date: "28 November 2021"
categories: [DevOps, Github Actions, Tutorial]
tags: [DevOps, Github Actions, OIDC]
---

Continuous delivery (CD) workflows implemented Github Actions help deploy software, create and update cloud infrastructure, or make use of various services of cloud providers like Amazon Web Services (AWS). To do this, a workflow needs to authenticate itself to the cloud provider through credentials to gain access to those resources and services.

Up until now, you would create credentials in the cloud provider and then store them again hardcoded as Github secrets, which can be presented to the cloud provider to grant the workflow access and permissions every time the workflow runs. Github recently [announced in their blog](https://github.blog/2021-11-23-secure-deployments-openid-connect-github-actions-generally-available/) that deployments with OpenID Connect (OIDC) through Github Actions are now generally available. With OIDC, your workflows can securely deploy after obtaining a short-lived access token directly from the cloud provider without the necessity to store long-lived credentials as GitHub secrets. Actions can utilize this access token, which expires at job completion, to connect and deploy to the cloud provider. This approach is not only simpler but also more secure as it gives you more control over your workflows' permissions and lets you forget about [synchronization, expiration, or rotation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect).

All the major players such as AWS, Azure, or GCP support OIDC. In this post, I show how to do it in AWS. To make it work, you first need to add a GitHub OIDC Provider (featuring its thumbprint) to a CloudFormation stack once (a single OIDC provider is enough to serve multiple roles). Audience/ClientId must be `sts.amazonaws.com` in this case because we will use the [official action](https://github.com/aws-actions/configure-aws-credentials); however, in general, it can be the URL of a Github user or organization.
```
GithubOIDCProvider:
  Type: AWS::IAM::OIDCProvider
  Properties:
    ClientIdList:
      - sts.amazonaws.com
    ThumbprintList:
      - a031c46782e6e6c662c2c87c76da9aa62ccabd8e
    Url: 'https://token.actions.githubusercontent.com'
```

Then, add a role to be assumed by GitHub Actions:
```
GithubOIDCRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: GithubOIDCRole
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Action: sts:AssumeRoleWithWebIdentity
          Principal:
            Federated: !Sub 'arn:aws:iam::${AWS::AccountId}:oidc-provider/token.actions.githubusercontent.com'
          Condition:
            StringLike:
              token.actions.githubusercontent.com:sub: repo:my-org/my-repo:*
```

`Condition` is important: With this field, you filter incoming requests, ensuring that only trusted repositories listed under `Condition` can request and obtain access tokens. `sub` here stands for "subject" and contains metadata about the workflow (organization, repository, job environment, etc.) in a specific format. Github Actions sends its request via a JSON Web Token (JWT) and AWS will check whether the JWT's subject matches the set conditions. Add permissions to this role as you like.

To let Github Actions fetch an OIDC token in the workflow, it first must have permissions to do so. Depending on your default permissions configuration, you may need to add
```
permissions:
  id-token: write
```

either to your workflow or directly in the job, together with all other permissions you need. Then, you can assume the role in the workflow with [Github's official action](https://github.com/aws-actions/configure-aws-credentials):
```
- uses: aws-actions/configure-aws-credentials@v1
  with:
    role-to-assume: arn:aws:iam::XXX:role/GithubOIDCRole
    aws-region: eu-west-1
```

(Replace XXX and eu-west-1 with your AWS account id and region, respectively.)

Happy deploying!
