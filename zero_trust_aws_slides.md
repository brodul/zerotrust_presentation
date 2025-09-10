# Zero Trust Security in Practice

@brodul

---

## Zero Trust Foundation

**Traditional Security Model**

- Castle-and-moat approach
- Long lived secrets

**Zero Trust Principles**

- Never trust, always verify
- Least privilege access
- Assume breach
- Verify explicitly

---

## Today's Focus
- Two practical implementations
- Spark interest
- So you can look it up in detail 

---
## Implementation 1 - CI/CD Pipeline Security
---

## CI/CD Pipeline Security

**Problem:** Long-lived AWS keys in GitHub Secrets

- Shared credentials across teams
- Manual rotation nightmares
- Full AWS access if compromised

---

## The Bad Practice - Long-Lived AWS Keys

**Common Anti-Pattern in GitHub Secrets:**

```yaml
name: Deploy to AWS
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - run: aws s3 sync ./dist s3://my-bucket/
```

- `ACCESS_KEY_ID` and `SECRET_KEY_ID` are used to authorize as a user
- That user has a Role with policies attach to access different services

---

## The Bad Practice - Long-Lived AWS Keys

**Why This Is Dangerous:**

- Keys stored in GitHub Secrets across multiple repos
- Same keys shared by entire team
- No rotation strategy → keys live for months/years
- **If compromised:** Full AWS account access with no audit trail

---

## AWS Security token service

- Web service that issues temporary, limited-privilege credentials
- Enables secure, time-bound access without long-term access keys
- Common use cases: cross-account access and identity federation
- Supports SAML and web identity providers (OIDC)
- Provides short-term elevated privileges for specific operations
- Credentials include: access key ID, secret access key, session token
- Token duration: configurable from minutes to hours

---
<!-- .slide: data-background="img/aws_sts_diagram.png" data-background-size="contain" -->

Notes:
- You provide some proof who you are (SAML, OIDC ...) optionally which access you need
- AWS STS validates that your proof is correct
- Once you are identified it checks if you are Authorized to "Assume a role" (trust policy)
- If you are then you get a shortlived access key ID, secret access key, and session token
- Role has permissions/policies attached on what you can do (create object on S3)

---

## Zero Trust Solution - CI/CD Pipeline Security

OpenID Connect (OIDC) Token Exchange

- GitHub proves identity via short-lived tokens
- AWS validates repository, branch, environment
- No stored credentials anywhere

---

<!-- .slide: data-background="img/oidc-flow.png" data-background-size="contain" -->

---

## OIDC Implementation - AWS IAM Role

**GitHubActions TRUST Policy (who can assume a role) :**

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:*"
    }
  }
}
```

---

**GitHubActions Policy (what can be done with the role)**

```json
...
"Statement": [
  {
    "Sid": "AllowS3SyncToBucket",
    "Effect": "Allow",
    "Action": [
      "s3:PutObject",
      "s3:GetObject", 
      "s3:DeleteObject",
      "s3:ListBucket"
    ],
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }
]
``` 

---

## OIDC Implementation - GitHub Workflow

```yaml
name: Deploy to AWS
on:
  push:
    branches: [main]

permissions:
  id-token: write # Enable OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActions
          aws-region: us-east-1
      - run: aws s3 sync ./dist s3://my-bucket/
```

**Result:** Automatic credential exchange, no secrets stored

---

## Implementation 2 - Network Access Control

---

## Network Access Control

**Problem:** Traditional VPN Limitations

- Castle-and-moat security
- Full network access once connected
- Complex client management

**Zero Trust Solution:** 

- Application-level access control
- Identity-based authentication
- Applications never exposed to internet

---

## Cloudflare ZTNA  

- Free for 50 seats
- Relatively simple 
- No need for real identity provider to start

---

## I am not affiliated with CloudFlare 

---

<!-- .slide: data-background="img/cf.svg" data-background-size="contain" -->

---

<!-- .slide: data-background="img/cloudflare-ztna.png" data-background-size="contain" -->

---


<!-- .slide: data-background="img/cloudflare-ztna-otp.png" data-background-size="contain" -->

---

<!-- .slide: data-background="img/tunnels.png" data-background-size="contain" -->

---

<!-- .slide: data-background="img/hostnames.png" data-background-size="contain" -->

---

<!-- .slide: data-background="img/app1.png" data-background-size="contain" -->

---

<!-- .slide: data-background="img/app2.png" data-background-size="contain" -->

---

## Key Takeaways

- Zero Trust requires identity verification at every step
- Short-lived credentials eliminate most credential theft risks
- Proper delegation enables teams to move fast securely
- Automation and templates ensure consistent security

