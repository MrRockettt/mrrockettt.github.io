---
title: "OAuth Account Takeover: Exploiting Email Change Without Re-authentication"
date: 2024-11-24 10:00:00 +0530
categories: [Bug Bounty, Web Security]
tags: [oauth, account-takeover, authentication, vulnerability]
image:
  path: /assets/img/oauth-security.png
  alt: OAuth Security Vulnerability
---

## Introduction

During a security assessment, I discovered a critical authentication vulnerability that enables complete account takeover through OAuth login flow exploitation. The vulnerability lies in how the application handles email changes for OAuth-registered accounts, allowing attackers to hijack victim accounts without any verification mechanism.

> **Disclosure**: This vulnerability was responsibly disclosed and resulted in a **$200 CAD bounty**. The vendor has since patched the issue.

## Vulnerability Overview

**Severity**: High  
**Type**: Authentication Bypass / Account Takeover  
**Attack Complexity**: Low  
**User Interaction**: None  
**Bounty Awarded**: $200 CAD

The core issue is that OAuth-registered accounts could change their email address without triggering proper re-authentication or verification mechanisms. This created a dangerous scenario where attackers could manipulate email associations while maintaining persistent access through their original OAuth credentials, effectively taking over victim accounts.

## Technical Analysis

Modern applications often implement OAuth (Open Authorization) for seamless authentication using providers like Google, Facebook, or GitHub. However, improper integration between OAuth authentication and account management features can lead to serious security vulnerabilities.

In this case, the application failed to establish strong binding between OAuth identity and email addresses. More critically, it didn't require re-verification when users changed this sensitive account information. This oversight created a perfect storm for account takeover attacks.

### The Attack Chain

The exploitation follows a four-phase process that bypasses normal security controls:

**Phase 1: OAuth Account Creation**

The attacker creates a legitimate account using Google OAuth with their own email address (attacker@gmail.com). At this point, everything appears normal—the account is properly registered and functional.

**Phase 2: Password Establishment**

OAuth accounts typically don't have passwords since authentication is delegated to the OAuth provider. However, the application offered a password reset feature that didn't properly validate whether the account was OAuth-based. The attacker exploits this by:
- Initiating password reset via "Forgot Password"
- Receiving a reset link for their OAuth account
- Setting a new password that will later authorize critical changes

**Phase 3: Email Hijacking**

With password control established, the attacker navigates to account settings and changes the email from attacker@gmail.com to the victim's email (victim@example.com). The application accepts this change after password verification but critically fails to:
- Send verification to the new email address
- Require confirmation from the victim
- Invalidate existing OAuth sessions
- Trigger any security alerts

**Phase 4: Persistent Access**

After logging out, the attacker can still authenticate using their original Google OAuth credentials (attacker@gmail.com). Despite the email being changed to the victim's address, the OAuth token mapping remains intact. The attacker now has full access to what is effectively the victim's account.

## Proof of Concept

Here's the step-by-step reproduction:

### Step 1: Initial Setup
```plaintext
1. Navigate to target application's registration page
2. Select "Sign in with Google"
3. Authenticate using attacker@gmail.com
4. Account successfully created
```

### Step 2: Password Control
```plaintext
1. Log out from the application
2. Click "Forgot Password"
3. Enter attacker@gmail.com
4. Receive reset link via email
5. Set password: "SecurePass123!"
6. Confirm password is working
```

### Step 3: Email Manipulation
```plaintext
1. Navigate to Account Settings
2. Change email to victim@example.com
3. Enter password "SecurePass123!" for authorization
4. Submit changes
5. No verification email sent to victim
6. Change accepted immediately
```

### Step 4: Verify Takeover
```plaintext
1. Log out completely
2. Select "Sign in with Google"
3. Authenticate with attacker@gmail.com
4. Successfully logged in
5. Account displays victim@example.com
6. Full account access maintained
```

## Security Impact

This vulnerability enables several attack vectors:

**Account Takeover**: Attackers gain complete control over victim accounts by simply knowing their email address. No victim interaction is required, making this a silent takeover vulnerability.

**Data Exposure**: Once inside, attackers can access personal information, view order history, retrieve saved payment methods, and modify account settings. Any sensitive data stored in the account becomes compromised.

**Identity Theft**: The compromised account can be used for impersonation, social engineering against other users or support staff, and potentially gaining access to linked services or loyalty programs.

**Persistence**: The victim may attempt to log in using legitimate credentials and find themselves locked out or confused by the altered account state, while the attacker maintains silent access through the OAuth backdoor.

## Root Cause Analysis

The vulnerability stems from multiple security oversights:

**Missing Email Verification**: The application didn't send verification emails to new addresses when users changed their email, a fundamental security requirement for identity-critical information.

**Weak Session Management**: OAuth authentication tokens remained valid even after critical account changes, violating proper session handling principles.

**Insufficient Re-authentication**: Email changes should require strong authentication, potentially including current OAuth provider confirmation, not just a password check.

**Poor Identity Binding**: The application failed to properly link OAuth identity with account email, creating ambiguity about true account ownership.

## Recommended Remediation

**Immediate Actions:**
- Implement email verification for all email changes
- Send notifications to the old email address
- Invalidate all active sessions when email changes
- Require OAuth re-authentication for sensitive changes

**Long-term Improvements:**
- Enable Multi-Factor Authentication for account modifications
- Add comprehensive logging for email change attempts
- Implement rate limiting on account modification endpoints
- Conduct regular security audits of authentication flows

## Disclosure Timeline

- **Discovery Date**: November 24, 2024
- **Initial Report**: November 24, 2024
- **Vendor Response**: November 27, 2024
- **Bounty Awarded**: $200 CAD
- **Patch Deployed**: December 24, 2024

## Key Takeaways

This finding reinforces several critical security principles:

**Defense in Depth**: Multiple security controls should protect critical operations. Single control failures shouldn't lead to complete compromise.

**OAuth Integration Complexity**: OAuth simplifies user experience but adds complexity to authentication flows. Every integration point requires careful security consideration.

**Thorough Testing Pays Off**: Testing authentication flows comprehensively, including edge cases and state transitions, often reveals critical vulnerabilities that automated scanners miss.

**Responsible Disclosure Works**: Working collaboratively with vendors through responsible disclosure programs benefits everyone—security researchers gain recognition and compensation, while organizations can fix issues before exploitation.

For security researchers, this case study emphasizes the importance of understanding how features interact, thinking like an attacker to find unexpected state transitions, documenting findings thoroughly with clear reproduction steps, and maintaining professionalism throughout the disclosure process.

## References

- [OAuth 2.0 Security Best Practices](https://tools.ietf.org/html/draft-ietf-oauth-security-topics)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [CWE-287: Improper Authentication](https://cwe.mitre.org/data/definitions/287.html)

---

**Disclaimer**: This write-up is for educational purposes only. The vulnerability has been patched by the vendor. Always practice responsible disclosure and only test systems you have explicit permission to assess.