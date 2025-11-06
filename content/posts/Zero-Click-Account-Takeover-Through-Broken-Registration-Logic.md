---
date: 2025-01-05
title: 'Zero Click Account Takeover Through Broken Registration Logic'
template: post
slug: How-I-Took Over-Any-Account-Without-User-Interaction
categories:
  - Technical
tags:
  - bug-bounty

---


What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical, no-interaction account takeover vulnerability I discovered on a major e-commerce platform. This is a story of how I chained together several flawed API endpoints to hijack any user's account simply by knowing their phone number, without them ever clicking a link or entering a password.

## The Discovery: A New Mobile App

My journey began while testing a new mobile app released by the platform. The app used phone number authentication as its primary login method. During the initial setup, the app asks for your phone number and sends you an OTP. This is a standard flow, but I wanted to see what was happening under the hood. I fired up Burp Suite to intercept the mobile app's traffic.

## The First Flaw: Unauthenticated OTP Generation

As I was exploring the app, I found an interesting endpoint that was used to send verification codes. When I requested to resend the code via email, I captured the following request:

```http
POST /store/v1/auth/email/send_verification HTTP/2
Host: api.vulnerable-platform.com
Content-Type: application/json

{"resend_by":"email","country_code":"SA","email":"attacker@email.com"}
```

The critical flaw here was that this endpoint had **no authentication**. I could send a request with any email address in the body, and the server would happily send a valid verification code to that address. This was the first piece of the puzzle.

## The Second Flaw: The Malicious Registration

Next, I dug deeper into the registration process. Through fuzzing and analyzing the app's traffic, I discovered the main registration endpoint:

```http
POST /store/v1/auth/register HTTP/2
Host: api.vulnerable-platform.com
Content-Type: application/json

{
  "email":"attacker@email.com",
  "first_name":"test",
  "code":"1234",
  "last_name":"test",
  "phone":"+15551234567",
  "verified_by":"email",
  "country_code":"SA"
}
```

This endpoint was a goldmine of flawed logic. It allowed me to specify:

*   An `email` address.
*   A `code` for verification.
*   A `phone` number to associate with the account.
*   A `verified_by` parameter, which could be set to `email`.

This is where I connected the dots. The application was letting me verify an identity using one method (email) but then link that verified session to a completely different identifier (a phone number).

## The Attack Chain: From OTP to ATO

Here is the full, three-step attack chain I used to take over any account:

**Step 1: Get a valid OTP sent to my email.**
I used the first endpoint to send a verification code to an email address I controlled.

```http
// Request to get the code
POST /store/v1/auth/email/send_verification HTTP/2
Host: api.vulnerable-platform.com

{"email":"attacker@email.com", ...}
```

**Step 2: Register with the victim's phone number.**
I took the OTP I received (`6280` in this case) and used it in the `/register` endpoint. I put my email in the `email` field, but I put the **victim's phone number** in the `phone` field. I also set `verified_by` to `email`.

```http
// Malicious registration request
POST /store/v1/auth/register HTTP/2
Host: api.vulnerable-platform.com

{
  "email":"attacker@email.com",
  "code":"6280", // The code I received
  "phone":"VICTIM_PHONE_NUMBER", // The victim's phone number
  "verified_by":"email",
  ...
}
```

The backend logic was tragically flawed. It checked my email verification code, saw it was valid, and then, instead of creating a new account, it saw the victim's phone number and simply logged me into their existing account. The response to this request contained a **session token for the victim's account**.

**Step 3: Confirm the takeover.**
With the victim's session token in hand, I could now impersonate them completely. I found a profile update endpoint and used the token to change their name, proving I had full control.

```http
// Using the leaked token to modify the victim's profile
POST /store/v1/profile/update HTTP/2
Host: api.vulnerable-platform.com
Authorization: Bearer VICTIM_SESSION_TOKEN

[...form data to change name...]
```

The request was successful. I had achieved a complete, no-interaction account takeover.

## The Impact: Critical PII and Data Manipulation

This vulnerability was critical. An attacker could:

*   **Take over any user account** with only their phone number.
*   **Access and steal sensitive PII**, including full name, address, order history, and potentially stored payment information.
*   **Modify user data**, which could be used for fraud or to cause reputational damage.

## Steps to Reproduce

1.  **Request a verification code** to an attacker-controlled email using the `/store/v1/auth/email/send_verification` endpoint.
2.  **Use the received code** in a request to the `/store/v1/auth/register` endpoint.
3.  In the registration request, set the `email` to the attacker's email, `verified_by` to `email`, and the `phone` to the **victim's phone number**.
4.  **Capture the session token** from the response.
5.  **Use the captured token** in the `Authorization` header to make authenticated requests on behalf of the victim, such as updating their profile.

## The Fix

The core of the fix is to ensure that the verification method is tied to the identifier being claimed. The backend should:

1.  **Never trust the `verified_by` parameter**. The system should decide how to verify based on the identifier provided.
2.  If a user registers with a phone number that already exists, the system must **send the OTP to that phone number**, not to an email address provided in the same request.
3.  The `/send_verification` endpoint should have authentication and rate limiting to prevent abuse.

## Conclusion

This was a textbook example of a broken authentication flow. By chaining together several logical flaws, I was able to achieve a critical impact with zero interaction from the victim. It's a powerful reminder to always map out the entire authentication and registration process and question every assumption the application makes. Happy hacking!
