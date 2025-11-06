---
date: 2025-11-05
title: 'Unauthorized Access to Internal Systems via Leaked Cloudflare Access Tokens'
template: post
slug: Unauthorized-Access-to-Internal-Systems-via-Leaked-Cloudflare-Access-Tokens
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical information disclosure vulnerability that provided unauthorized access to multiple internal systems. This is a story of how a simple oversight—leaving sensitive credentials in a publicly accessible JavaScript file—led to a complete bypass of the platform's internal network security and the exposure of highly sensitive PII.

## The Discovery: The Leaky JavaScript File

My investigation began with routine analysis of the platform's publicly served assets. While reviewing the source code and network traffic, I noticed a JavaScript file hosted on a third-party content delivery network (CDN) that appeared to be part of a development or staging environment.

The file's URL was highly suspicious:
```
https://cdn-assets-vulnerable-network.com/dev/dev-portal-utils-module/feature-jira-1-mirror/partners-utils.js
```

Upon inspecting the contents of this file, I found hardcoded credentials for the platform's internal access control system, specifically **Cloudflare Access**.

## The Vulnerability: Hardcoded CF-Access Credentials

The JavaScript file contained two critical, hardcoded values: the `CF-Access-Client-Id` and the `CF-Access-Client-Secret`. These tokens are designed to authenticate services and grant access to internal, protected applications without requiring a user login.

The relevant snippet found in the file was:
```javascript
ACCESS_ID:localStorage.getItem("ACCESS_ID")||"ec7586fe0fd8d8f06301379d8706dcef.access",
ACCESS_SECRET:localStorage.getItem("ACCESS_SECRET")||"d459ead056bedf891cb413f81bb74d40e4c5de1e7fa9ffe3fcb3df431b0f1259"
```

By extracting these two values, an attacker could craft a request to bypass the Cloudflare Access protection on any internal domain protected by the same configuration.

## The Exploitation: Bypassing Internal Security

I immediately tested the credentials against known internal subdomains. The attack was simple: include the stolen credentials as headers in an HTTP request to the protected resource.

### 1. Accessing Internal Mail Service

The first target was the internal mail service, which was protected by Cloudflare Access.

**Simulated Request (using leaked tokens):**
```http
GET / HTTP/2
Host: mail.vulnerable-group.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:130.0) Gecko/20100101 Firefox/130.0
Cf-Access-Client-Id: ec7586fe0fd8d8f06301379d8706dcef.access
Cf-Access-Client-Secret: d459ead056bedf891cb413f81bb74d40e*******7fa9ffe3fcb3df431b0f1259
```

**Result:** The request was successfully authenticated, granting me unauthorized access to the internal mail service.

### 2. Accessing Internal Job Queue and Leaking PII

The most critical finding was an internal job queue monitoring system. Accessing this system revealed highly sensitive data related to partner applications.

**Simulated Request (using leaked tokens):**
```http
GET /jobs/completed HTTP/2
Host: queue-nasa.vulnerable-group.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:130.0) Gecko/20100101 Firefox/130.0
Cf-Access-Client-Id: ec7586fe0fd8d8f06301379d8706dcef.access
Cf-Access-Client-Secret: d459ead056bedf891cb413f81bb74d40e*******7fa9ffe3fcb3df431b0f1259
```

**Result:** The system returned a list of completed jobs, which included the full request data from partner applications. This exposed a massive amount of PII, including:

*   **Full Name and Email**
*   **Hashed Passwords**
*   **API Tokens**
*   **Credit Card Information** (in some cases, related to subscription payments)

## The Impact: Critical Internal System Compromise

This vulnerability is rated Critical (CVSS 9.1) because it completely bypasses the intended security boundary for internal infrastructure.

*   **Unauthorized Internal Access**: Full access to multiple internal applications (mail, job queues, partner portals).
*   **Massive PII Disclosure**: Exposure of highly sensitive data, including customer PII, hashed passwords, and financial information, from the internal job queue.
*   **Persistent Access**: Even after the initial tokens were revoked, the attacker could use the generated session cookies (`CF_Authorization`) to maintain access, demonstrating a failure in session invalidation.

## Steps to Reproduce

1.  **Locate Credentials**: Navigate to the publicly accessible JavaScript file: `https://cdn-assets-vulnerable-network.com/dev/.../partners-utils.js`.
2.  **Extract Tokens**: Search for `"ACCESS_SECRET"` and extract the `Cf-Access-Client-Id` and `Cf-Access-Client-Secret` values.
3.  **Bypass Access**: Use a tool like Burp Repeater to send a request to a protected internal domain (e.g., `mail.vulnerable-group.com`) and include the extracted tokens as `Cf-Access-Client-Id` and `Cf-Access-Client-Secret` headers.
4.  **Verify PII Leak**: Access the internal job queue endpoint (e.g., `queue-nasa.vulnerable-group.com/jobs/`) to confirm the exposure of sensitive PII.

## The Fix

The fix requires immediate and comprehensive action:

1.  **Immediate Token Revocation**: The exposed Cloudflare Access tokens must be immediately revoked and new tokens generated.
2.  **Code Review and Deployment Pipeline Fix**: The development pipeline must be reviewed to ensure that hardcoded secrets are never deployed to publicly accessible environments, even staging or development CDNs. Secrets should be loaded from secure environment variables or a secret manager.
3.  **Session Invalidation**: Ensure that revoking the `Cf-Access-Client-Secret` also invalidates all associated session cookies (`CF_Authorization`) to prevent persistent access.
4.  **PII Redaction in Internal Systems**: The internal job queue system must be secured, and all sensitive PII (passwords, credit card data) must be redacted or masked before being logged, even in internal systems.

## Conclusion

This vulnerability is a stark reminder that a single misconfiguration in a development asset can compromise an entire internal network. The security chain is only as strong as its weakest link, and in this case, a publicly accessible JavaScript file was the key that unlocked multiple critical internal systems. It underscores the necessity of strict secret management and continuous monitoring of all publicly served assets for sensitive data. Happy hacking!
