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

## The Discovery: The Admin Panel and the Storefront

I was testing a popular multi-store e-commerce platform. The platform has two main parts:

1.  **The Admin Panel**: This is where store owners (like me, the attacker) manage their store. They can customize their storefront, view orders, and, most importantly, see a log of all SMS messages sent to their customers.
2.  **The Storefront**: This is the customer-facing side of the store where customers browse products, log in, and make purchases.

My focus was on the Admin Panel. I wanted to see if I could find any way to abuse the features a store owner has to compromise their customers.

## The Translation Template: A Hidden Danger

In the Admin Panel, there's a feature that lets store owners translate their Storefront into different languages. This includes customizing the SMS messages that are sent to customers for things like login verification. For example, the default login OTP message is:

`{code} is your login code for {store_name}. Do not share it with anyone.`

The `{code}` and `{store_name}` are variables that the system fills in. The Admin Panel also has a section to view SMS logs. This is where a store owner can see all the messages sent to their customers. For example:

*   **Customer Login**: "A customer with phone number +15551234567 requested to log in."
*   **Customer Adds to Cart**: "Customer John Doe added a product to their cart."

Crucially, the platform was designed to **redact** sensitive information in these logs. So, when a customer requested an OTP, the log was supposed to look like this:

`Sent login code to +15551234567. Message: **** is your login code for My Awesome Store.`

The system correctly hid the OTP with asterisks. But I wondered if I could break this redaction.

## The Vulnerability: Bypassing OTP Redaction with a Null Byte

My theory was that the redaction logic was probably looking for the exact string `{code}` to replace it with `****`. What if I could inject a character that would break the string before the redaction logic could find the `{code}` placeholder?

I decided to use a **null byte** (`\u0000`), a classic character for breaking string processing. I went to the translation editor in my Admin Panel and modified the login SMS template to this:

`{code}\u0000a{store_name}`

The request to update the template looked like this:

```http
POST /admin/v2/translation/text-editor/sections/ HTTP/2
Host: api.vulnerable-platform.dev
Content-Type: application/json

{"entity":"messages","language":"ar","entity_id":"auth","field_name":"otp_content","field_value":"{code}\u0000a{store_name}"}
```

Now, with my malicious template saved, it was time to test it.

## The Exploitation: Stealing the OTP

Here's how I stole a victim's OTP:

1.  **Attacker Setup**: I, as the owner of my attacker-controlled store, had already injected the malicious SMS template.
2.  **Victim Login Attempt**: I went to my own Storefront's login page and entered a **victim's phone number** to initiate a login.
3.  **OTP Leak**: The system generated an OTP (e.g., `3919`) and sent it to the victim's phone. But because of my malicious template, the SMS log in *my* Admin Panel looked like this:

    `Sent login code to VICTIM_PHONE_NUMBER. Message: 3919aMy Awesome Store.`

The null byte broke the string processing. The system logged the OTP (`3919`) in clear text before the redaction could happen. I had successfully stolen the victim's OTP.

## The Critical Flaw: Mass Account Takeover with a Global JWT

This vulnerability went from bad to critical because of a flaw in the platform's authentication system:

1.  **Login as Victim**: I took the stolen OTP (`3919`) and used it to log into my own Storefront *as the victim*.
2.  **JWT Theft**: Upon successful login, the platform gave me a JSON Web Token (JWT) for the victim's session.
3.  **Cross-Store Access**: I discovered that this JWT was a **global token**. It wasn't just valid for my store; it was valid across the entire platform, for any store.

I could now use this JWT to make API calls to the platform's core services and access the victim's data on any store they had an account with.

```http
// Using the stolen JWT to get the victim's full profile
GET /core/v1/customers/profile HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer [Victim's Stolen JWT]

// Response contains full PII: name, email, phone, address, etc.
```

I had achieved a full, platform-wide account takeover.

## The Impact: Critical Account Takeover and PII Leak

This was a critical vulnerability (CVSS 9.1) because:

*   **Mass Account Takeover**: An attacker with a free store could steal the OTP of any user on the platform.
*   **Mass PII Disclosure**: The attacker could access the PII and order history of any compromised user across all stores.

## Steps to Reproduce

1.  **Create a store** on the platform.
2.  In the Admin Panel, **modify the login SMS template** with the null byte payload: `{code}\u0000a{store_name}`.
3.  Go to your Storefront's login page and **enter a victim's phone number**.
4.  **Check your Admin Panel's SMS logs** to find the victim's OTP in clear text.
5.  **Use the stolen OTP** to log in as the victim on your Storefront.
6.  **Capture the victim's JWT** from the login response.
7.  **Use the JWT** to access the victim's data on the platform's core API.

## The Fix

1.  **Input Sanitization**: The translation editor must prevent the injection of null bytes and other control characters.
2.  **Robust Redaction**: The logging system should not rely on simple string matching. It should know what the OTP is and redact it regardless of the template.
3.  **JWT Scoping**: JWTs should be scoped to a specific store or context and not be globally valid.

## Conclusion

This was a fascinating vulnerability that chained a simple injection flaw with a critical authentication design flaw. It's a powerful reminder that even seemingly harmless features like translation templates can be abused to achieve a critical impact. Happy hacking!
