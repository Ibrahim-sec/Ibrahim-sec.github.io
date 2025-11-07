---
date: 2025-10-13
title: 'OTP Redaction Bypass in SMS Logs Leads to Mass Account Takeover Across E-Commerce Platform'
template: post
slug: OTP-Bypass-in-SMS-Leads-To-Mass-Account-Takeover
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical vulnerability chain that turned a seemingly innocuous translation feature into a mass account takeover exploit. This is a story of how a simple payload injected into an SMS template bypassed a security redaction mechanism, allowing me to steal One-Time Passwords (OTPs) and take over accounts across an entire multi-store e-commerce platform.

## The Discovery: The Translation Template

I was testing the administrative dashboard (the "Admin Panel") of a popular multi-store e-commerce platform. This Admin Panel is where store owners manage their store, including customizing the SMS templates used for customer communication and viewing sensitive logs. The customer-facing side is the "Storefront."

I focused on the language and translation settings, specifically the template for the login OTP message. The platform had a feature to view SMS logs, but it was designed to redact sensitive information like the OTP with asterisks (`****`) to prevent accidental exposure.

The endpoint for updating the translation template looked like this:
```
https://api.vulnerable-platform.dev/admin/v2/translation/text-editor/sections/
```

## The Vulnerability: Bypassing OTP Redaction

The core of the vulnerability lay in how the platform processed and logged the SMS message after a store owner customized the template. The default template for the login SMS was something like:

> `{code} is your login code for {store_name}. Do not share it with anyone.`

The system was designed to redact the `{code}` placeholder in the logs. However, I hypothesized that the redaction logic was looking for the exact placeholder string or a specific character sequence.

I found that by injecting a special character sequence, specifically a **null byte (`\u0000`)**, I could trick the system. The null byte would break the expected string structure, causing the system to log the entire message *before* the redaction logic could properly identify and mask the OTP.

I modified the template with the following payload:

**Payload:**
```
"field_value":"{code}\u0000a{store_name}"
```

The request body to update the template looked like this (anonymized):

```json
{"entity":"messages","language":"ar","entity_id":"auth","field_name":"otp_content","field_value":"{code}\u0000a{store_name}"}
```

## The Exploitation: Stealing the OTP

With the malicious template active on my attacker-controlled store, the exploitation path was clear:

1.  **Attacker Setup**: I, the attacker, created a free store on the platform. I then used my store's **Admin Panel** to inject the malicious template via the translation editor.
2.  **Victim Login Attempt**: I navigated to my **Storefront's** customer login page and entered a victim's phone number or email address to initiate a login.
3.  **OTP Leak**: The system sent the OTP to the victim. Crucially, because of the null byte injection, the full, unredacted OTP was logged in my attacker-store's **Admin Panel SMS Logs**.

When I checked my store's SMS logs, the message appeared with the OTP in clear text, for example: `3919a{store_name}`. The `3919` was the victim's One-Time Password.

## The Critical Flaw: Mass Account Takeover

The vulnerability escalated from a single store to a mass account takeover due to a critical design flaw in the platform's authentication mechanism:

1.  **Login as Victim**: I used the stolen OTP (`3919`) to successfully log into my attacker-controlled storefront as the victim.
2.  **JWT Theft**: Upon successful login, the platform issued a JSON Web Token (JWT) for the victim's session. I captured this JWT.
3.  **Cross-Store Access**: I discovered that this JWT was valid across the entire platform's ecosystem, including other stores and core API services.

By replaying the victim's JWT in requests to the platform's main API, I could access and modify the victim's sensitive data, regardless of which store they were originally logging into.

**Example API Requests using the stolen JWT:**

```http
GET /core/v1/customers/profile HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer [Victim's Stolen JWT]
// Response contains full PII: name, email, phone, address, etc.
```

```http
GET /core/v1/orders?page=1&per_page=20 HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer [Victim's Stolen JWT]
// Response contains full order history and purchase details.
```

This allowed for a complete **Account Takeover** and **Mass PII Disclosure** across the entire platform.

## The Impact: Critical Account Takeover and PII Leak

This vulnerability is rated Critical (CVSS 9.1) due to the following impacts:

*   **Mass Account Takeover**: An attacker can systematically steal OTPs for any user on the platform and gain full control of their accounts.
*   **Mass PII Disclosure**: The attacker can access and modify the personal information, order history, and potentially payment details of any compromised user.
*   **Low Barrier to Entry**: The attacker only needs to own a free store on the platform (a low-privilege requirement) to set up the exploit.

## Steps to Reproduce

1.  **Attacker Setup**: Create a new store on the platform and navigate to the language/translation management section.
2.  **Inject Payload**: Modify the login SMS template by injecting the null byte sequence payload (e.g., `{code}\u0000a{store_name}`) and save the changes.
3.  **Initiate Login**: Go to the attacker-controlled storefront's login page and enter the victim's email or phone number.
4.  **Steal OTP**: Immediately check the attacker-store's SMS logs. The victim's OTP will appear in clear text.
5.  **Login**: Use the stolen OTP to log into the attacker-store's **Storefront** as the victim.
6.  **Capture JWT**: Capture the victim's session JWT from the login response.
7.  **Exploit**: Use the captured JWT to make authenticated API calls to the platform's core services (e.g., `/core/v1/customers/profile`) to access and modify the victim's data.

## The Fix

The fix requires a multi-layered approach to address both the injection and the redaction logic:

1.  **Input Sanitization**: The translation template editor must strictly sanitize input, preventing the injection of null bytes or other control characters that can interfere with string processing.
2.  **Robust Redaction Logic**: The SMS logging feature must implement a more robust redaction mechanism that identifies the OTP based on the message content or the system's knowledge of the sent code, rather than relying on a simple string match of the placeholder.
3.  **JWT Scoping**: The platform should implement stricter JWT scoping, ensuring that a token issued for a customer on one store is not automatically valid for accessing core services or data related to other stores or the platform as a whole.

## Conclusion

This was a fascinating example of a logic flaw that bypassed a security control (OTP redaction) and escalated into a platform-wide account takeover. It highlights the importance of rigorous testing on all user-controlled inputs, even those that seem purely cosmetic like translation templates. Furthermore, it serves as a crucial reminder that authentication tokens must be carefully scoped to prevent cross-application or cross-store exploitation. Happy hacking!
