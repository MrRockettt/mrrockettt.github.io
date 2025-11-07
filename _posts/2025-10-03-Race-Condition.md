---
title: "Bypassing Rate Limits via HTTP/2 Single-Packet Attack: A Race Condition Story"
date: 2025-10-13 11:00:00 +0530
categories: [Bug Bounty, Web Security]
tags: [race-condition, http2, rate-limit-bypass, account-takeover]
image:
  path: /assets/img/race-condition.png
  alt: Race Condition Vulnerability
---

## Introduction

Rate limiting is a fundamental security control designed to prevent brute-force attacks and credential stuffing. However, during a security assessment, I discovered a race condition vulnerability that completely bypassed the target application's rate-limiting mechanism. By leveraging HTTP/2's multiplexing capabilities and Burp Suite's Turbo Intruder, I was able to send multiple login attempts simultaneously, avoiding the 429 Too Many Requests response and successfully brute-forcing credentials.

This writeup demonstrates how race conditions in authentication systems can undermine even well-implemented security controls.

## Vulnerability Overview

**Severity**: High  
**Type**: Race Condition / Rate Limit Bypass  
**Attack Vector**: HTTP/2 Single-Packet Attack  
**Impact**: Account Takeover via Credential Brute-Force  
**CVSS**: 8.1 (High)

The application implemented rate limiting on login endpoints to prevent brute-force attacks. However, the rate-limiting logic suffered from a race condition vulnerability. When multiple requests were sent simultaneously using HTTP/2 multiplexing, the rate limiter failed to properly track and block concurrent attempts, allowing attackers to bypass the protection entirely.

## Technical Background

### Understanding Race Conditions

A race condition occurs when multiple threads or processes access shared resources concurrently, and the outcome depends on the timing of their execution. In web applications, this typically happens when:
- Multiple requests modify the same resource simultaneously
- The application doesn't properly synchronize access to shared state
- Timing-dependent logic creates exploitable windows

### HTTP/2 Multiplexing Advantage

HTTP/2 introduces request multiplexing, allowing multiple requests to be sent over a single TCP connection simultaneously. This feature, while improving performance, can be exploited to create race conditions by ensuring requests arrive at the server at nearly the same instant.

Traditional rate limiting often checks the request count, processes the request, then increments the counter. When multiple requests arrive simultaneously via HTTP/2, they may all read the same counter value before any of them increment it—effectively bypassing the limit.

## Discovery Process

### Initial Testing

I began testing the login functionality using standard brute-force techniques:

1. Captured a valid login request using Burp Suite's Proxy
2. Sent the request to Intruder for automated testing
3. Configured a password list and started the attack

**Result**: After exactly 10 requests, the server responded with `HTTP 429 Too Many Requests`. The rate limiter was working as intended—or so it seemed.

### Identifying the Race Condition

Suspecting a potential race condition, I decided to test how the application handled concurrent requests:

1. Sent the login request to Burp Repeater
2. Created a group of tabs in Repeater
3. Duplicated the request 19 times for a total of 20 parallel requests
4. Executed all requests simultaneously using the "Send group (parallel)" option

**Result**: All 20 requests completed successfully without any 429 responses. This confirmed the rate limiter could be bypassed through concurrent requests.

### Exploitation with Turbo Intruder

To weaponize this finding, I used Burp Suite's Turbo Intruder extension with the `race-single-packet-attack.py` template:

**Step 1: Prepare the Request**
```plaintext
1. Send captured login request to Turbo Intruder
2. Replace the password field with %s placeholder
3. Prepare a password wordlist
```

**Step 2: Configure the Attack**

I modified the Turbo Intruder script to leverage HTTP/2's single-packet attack capability:

```python
def queueRequests(target, wordlists):
    # Use HTTP/2 with single connection for single-packet attack
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    # Load passwords from clipboard
    passwords = wordlists.clipboard
    
    # Queue all requests with gate='1'
    # This holds requests until openGate() is called
    for password in passwords:
        engine.queue(target.req, password, gate='1')
    
    # Release all requests simultaneously
    engine.openGate('1')

def handleResponse(req, interesting):
    table.add(req)
```

**Key Configuration Details:**
- `engine=Engine.BURP2`: Enables HTTP/2 support
- `concurrentConnections=1`: Sends all requests over a single connection
- `gate='1'`: Holds requests until explicitly released
- `openGate('1')`: Sends all queued requests in a single packet burst

**Step 3: Execute the Attack**
```plaintext
1. Copy password list to clipboard
2. Launch the attack in Turbo Intruder
3. Analyze responses for variations
```

### Results

The attack successfully bypassed the rate limiter:

- All requests returned `HTTP 303 See Other` (redirect after login attempt)
- No 429 responses were triggered
- One response had a different `Content-Length` value
- This length discrepancy revealed the correct password

The varying content length acts as a **response-length oracle**, leaking information about successful authentication without explicit confirmation. This allowed me to identify valid credentials from the response metadata alone.

## Security Impact

This vulnerability has severe implications:

**Account Takeover**: Attackers can brute-force user credentials without triggering rate limits, leading to unauthorized account access.

**Credential Stuffing**: Stolen credentials from data breaches can be validated at scale, compromising numerous accounts.

**No Detection**: Traditional monitoring systems watching for 429 responses or failed login spikes won't detect this attack.

**Scalability**: The attack can be automated and parallelized across multiple accounts simultaneously.

## Root Cause Analysis

The vulnerability exists due to several implementation flaws:

**Non-Atomic Rate Limit Checks**: The application likely checks the rate limit counter, processes the request, then increments the counter—creating a time-of-check to time-of-use (TOCTOU) vulnerability.

**Lack of Synchronization**: Multiple concurrent requests can read the same counter value before any increment it, allowing all requests to pass the check.

**HTTP/2 Handling**: The server doesn't properly account for HTTP/2 multiplexing, treating simultaneous requests as if they arrived sequentially.

## Recommended Mitigations

To properly fix this vulnerability, implement the following controls:

### 1. Atomic Rate Limiting
```plaintext
Increment the rate limit counter BEFORE processing the login request.
Use atomic operations or database constraints to ensure accuracy.
Example: Redis INCR command with EXPIRE for distributed systems.
```

### 2. Resource Locking
```plaintext
Implement mutexes or locks for the same username.
Serialize login attempts to prevent parallel processing.
Use distributed locks (e.g., Redis RedLock) in clustered environments.
```

### 3. HTTP/2 Awareness
```plaintext
Treat each HTTP/2 multiplexed request as individual attempts.
Don't aggregate concurrent requests as a single attempt.
Implement per-connection rate limiting in addition to per-IP limits.
```

### 4. Server-Side Throttling
```plaintext
Add fixed delays after failed login attempts (e.g., 2-second delay).
Implement exponential backoff for repeated failures.
Use CAPTCHA after N failed attempts from the same source.
```

### 5. Additional Controls
```plaintext
Implement account lockout after X failed attempts.
Use Web Application Firewall (WAF) rules to detect burst patterns.
Monitor for unusual authentication patterns.
Enable MFA to add additional security layer.
```

## Lessons Learned

This finding highlights several important security concepts:

**Race Conditions Are Real**: Even modern applications with rate limiting can be vulnerable to timing-based attacks.

**HTTP/2 Security Implications**: New protocols introduce new attack surfaces. Security controls must evolve to handle concurrent multiplexed requests.

**Defense in Depth**: Rate limiting alone isn't sufficient. Combine it with account lockouts, CAPTCHA, and monitoring for comprehensive protection.

**Testing Matters**: Standard sequential testing wouldn't have revealed this vulnerability. Creative testing approaches are essential.


## Conclusion

Race conditions in authentication systems represent a critical security risk that's often overlooked during development and testing. This vulnerability demonstrates how concurrent request handling, especially with HTTP/2, can completely undermine rate-limiting controls designed to prevent brute-force attacks.

For security researchers, this case emphasizes the importance of testing applications with concurrent requests and exploring how modern protocols like HTTP/2 can be leveraged for exploitation. For developers, it reinforces the need for atomic operations, proper synchronization, and protocol-aware security controls.

## References

- [OWASP: Testing for Race Conditions](https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/11-Client-side_Testing/09-Testing_for_Race_Conditions)
- [PortSwigger: Race Conditions](https://portswigger.net/web-security/race-conditions)
- [HTTP/2 Specification - RFC 7540](https://tools.ietf.org/html/rfc7540)
- [Turbo Intruder Documentation](https://portswigger.net/research/turbo-intruder-embracing-the-billion-request-attack)

---