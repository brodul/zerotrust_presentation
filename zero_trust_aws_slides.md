# Zero Trust Security in Practice

@brodul

---

## Zero Trust Foundation

**Traditional Security Model**
- Castle-and-moat approach 
- Long lived secrets

Zero Trust Principles:
- Never trust, always verify
- Least privilege access
- Assume breach
- Verify explicitly

---

**Today's Focus:** Two practical implementations at scale

---

## Implementation 1 - CI/CD Pipeline Security

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

## Implementation 1 - CI/CD Pipeline Security

**Zero Trust Solution:** OpenID Connect (OIDC) Token Exchange
- GitHub proves identity via short-lived tokens
- AWS validates repository, branch, environment
- No stored credentials anywhere

---

```
┌──────────────┐    1. Push Code    ┌──────────────┐
│  Developer   │ ─────────────────► │ GitHub Repo  │
└──────────────┘                    └──────────────┘
                                            │
                                            │ 2. Workflow Triggers
                                            ▼
                                   ┌──────────────┐
                                   │GitHub Actions│
                                   │   Workflow   │
                                   └──────────────┘
                                            │
                                            │ 3. Request OIDC Token
                                            ▼
┌──────────────┐    4. JWT Token    ┌──────────────┐
│ GitHub OIDC  │ ◄──────────────────│GitHub Actions│
│  Provider    │ ───────────────────►│   Runner     │
└──────────────┘   5. Signed Token  └──────────────┘
                                            │
                                            │ 6. AssumeRoleWithWebIdentity
                                            ▼
                                   ┌──────────────┐
                                   │   AWS STS    │
                                   └──────────────┘
                                            │
            7. Validate Token               │ 8. Return Temp Credentials
            (repo, branch check)            │    (15 min TTL)
            ┌──────────────┐                │
            │   AWS IAM    │                │
            │Trust Policy  │                │
            └──────────────┘                ▼
                                   ┌──────────────┐
                                   │GitHub Actions│
                                   │  with AWS    │
                                   │ Credentials  │
                                   └──────────────┘
                                            │
                                            │ 9. Deploy to AWS
                                            ▼
                                   ┌──────────────┐
                                   │ AWS Services │
                                   │(S3, Lambda,  │
                                   │ ECS, etc.)   │
                                   └──────────────┘
```

Notes: **Key:** No credentials stored • 15-min token TTL • Complete audit trail

---

## OIDC Implementation - AWS Setup
**OIDC Identity Provider:**
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com
```

**IAM Role Trust Policy:**
```json
{
  "Effect": "Allow",
  "Principal": {"Federated": "arn:aws:iam::ACCOUNT:oidc-provider/token.actions.githubusercontent.com"},
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

## OIDC Implementation - GitHub Workflow
```yaml
name: Deploy to AWS
on:
  push:
    branches: [main]

permissions:
  id-token: write    # Enable OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActions-Role
          aws-region: us-east-1
      
      - run: aws s3 sync ./dist s3://my-bucket/
```

**Result:** Automatic credential exchange, no secrets stored

---

## Implementation 2 - Network Access Control
**Problem:** Traditional VPN Limitations
- Castle-and-moat security
- Full network access once connected
- Performance bottlenecks
- Complex client management

**Zero Trust Solution:** Cloudflare Zero Trust Network Access
- Application-level access control
- Identity-based authentication
- Browser-based access (no VPN client)
- Applications never exposed to internet

---

## Cloudflare ZTNA Architecture
**Components:**
- **Cloudflare Access:** Authentication gateway
- **Cloudflare Tunnel:** Secure outbound-only connections
- **Identity Providers:** SSO integration

**Access Flow:**
```
User → Cloudflare Access → Identity Provider → Policy Check → Application
```

**Cloudflare Tunnel Setup:**
```bash
cloudflared tunnel create my-app-tunnel
cloudflared tunnel --config config.yml run
```

---

## Advanced Access Policies
**Identity-Based Controls:**
```yaml
name: "Internal Application Access"
decision: "allow"
includes:
  - email_domain: "company.com"
  - group: "developers"
requires:
  - authentication_method: "mfa"
  - country: ["US", "CA"]
session_duration: "8h"
```

**Device Trust Integration:**
- OS version requirements
- Disk encryption verification
- Firewall status checks
- Certificate-based device identity

---

## Delegation Framework
**Team Ownership Model:**

**Platform Team:**
- OIDC provider setup and templates
- Cloudflare tenant configuration
- Monitoring and alerting infrastructure

**Product Teams:**
- Repository-specific IAM roles
- Application access policies
- Deployment workflows

**Security Team:**
- Policy templates and compliance
- Audit trails and reviews
- Emergency access procedures

---

## Monitoring & Compliance
**Unified Visibility:**
- **CloudTrail:** All AWS role assumptions
- **Cloudflare Analytics:** Application access patterns
- **GitHub Audit:** Workflow execution logs

**Key Metrics:**
- Credential usage by team/repository
- Failed authentication attempts
- Policy violations and exceptions
- Access pattern anomalies

**Automated Alerting:** Suspicious activity, policy drift, compliance violations

---

## Questions & Discussion
**Key Takeaways:**
1. Zero Trust requires identity verification at every step
2. Short-lived credentials eliminate most credential theft risks
3. Proper delegation enables teams to move fast securely
4. Automation and templates ensure consistent security

**Discussion Topics:**
- Implementation challenges in your environment
- Integration with existing identity systems
- Scaling considerations for large organizations

**Questions?**
