# Zero Trust Security in Practice
## 25-Minute Technical Presentation

---

## Zero Trust Foundation
**Traditional Security Model:**
- Castle-and-moat approach
- Trust internal network traffic
- Perimeter-based defenses

**Zero Trust Principles:**
- **Never trust, always verify**
- **Least privilege access**
- **Assume breach**
- **Verify explicitly**

**Today's Focus:** Two practical implementations at scale

---

## Implementation 1 - CI/CD Pipeline Security
**Problem:** Long-lived AWS keys in GitHub Secrets
- Shared credentials across teams
- Manual rotation nightmares
- Full AWS access if compromised

**Zero Trust Solution:** OpenID Connect (OIDC) Token Exchange
- GitHub proves identity via short-lived tokens
- AWS validates repository, branch, environment
- No stored credentials anywhere

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

**Why This Is Dangerous:**
- Keys stored in GitHub Secrets across multiple repos
- Same keys shared by entire team
- No rotation strategy → keys live for months/years
- **If compromised:** Full AWS account access with no audit trail

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
      "token.actions.githubusercontent.com:repository": "my-org/my-repo",
      "token.actions.githubusercontent.com:ref": "refs/heads/main"
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

## Self-Service Implementation
**Terraform Modules for Standardization:**
```hcl
# OIDC for CI/CD
module "github_oidc_role" {
  source = "internal/terraform-github-oidc"
  repository_name = "my-service"
  environments = ["prod", "staging"]
}

# Cloudflare Access for Applications
module "app_access" {
  source = "internal/terraform-cloudflare-access"
  application_domain = "internal-app.company.com"
  allowed_groups = ["developers", "qa-team"]
}
```

**Benefits:** Consistent security, reduced manual work, faster onboarding

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

## Implementation Challenges & Solutions
**Challenge 1:** Complex policy syntax across platforms
- **Solution:** Standardized Terraform modules with validation

**Challenge 2:** Emergency access scenarios
- **Solution:** Break-glass procedures with elevated monitoring

**Challenge 3:** Cross-account and multi-environment complexity
- **Solution:** Template-based role patterns and automation

**Challenge 4:** User experience and adoption
- **Solution:** Transparent authentication, performance optimization

---

## Results & Business Impact
**Security Improvements:**
- ✅ Eliminated 100% of long-lived credentials
- ✅ Reduced attack surface by 80%
- ✅ Complete audit trail for all access

**Operational Benefits:**
- ✅ 60% reduction in access-related tickets
- ✅ Faster onboarding (days → hours)
- ✅ Automated compliance reporting

**Cost Savings:**
- Reduced VPN infrastructure costs
- Lower operational overhead
- Improved developer productivity

---

## Next Steps & Roadmap
**Short Term (3 months):**
- Expand to remaining 200+ repositories
- Additional identity provider integrations
- Mobile device management integration

**Medium Term (6 months):**
- Cross-cloud implementations (Azure, GCP)
- API gateway Zero Trust integration
- Advanced threat detection

**Long Term (12 months):**
- Continuous risk assessment
- ML-based anomaly detection
- Full software supply chain security

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