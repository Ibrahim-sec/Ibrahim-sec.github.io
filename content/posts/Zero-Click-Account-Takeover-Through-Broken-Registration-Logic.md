---
date: 2025-11-04
title: 'Zero Click Account Takeover Through Broken Registration Logic'
template: post
slug: Zero-Click-Account-Takeover-Through-Broken-Registration-Logic
categories:
  - Technical
tags:
  - bug-bounty

---


What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical account takeover vulnerability I found on a mobile app that was supposed to be phone-only. This is a story of how I discovered hidden email endpoints by manipulating API requests, bypassed the entire phone verification process, and hijacked any user's account with just their phone number.

## The Discovery: A Phone-Only System

I was testing a mobile app that used phone numbers exclusively for registration and login. There was no visible option to use an email address anywhere. The flow was simple: you enter your phone number, get an OTP via SMS, and you're in. 

I started by intercepting the traffic with Burp Suite to see how this worked. The initial request to send an OTP looked something like this:

```http
POST /store/v1/auth/phone/send_verification HTTP/2
Host: api.vulnerable-platform.com
Content-Type: application/json

{"phone":"+15551234567","country_code":"SA"}
```

This was a standard request. But as a hacker, I'm always curious about what's *not* visible. What if the developers had built email functionality but hadn't enabled it in the app? 

## The First Breakthrough: Finding the Hidden Email Endpoint

I decided to test my theory. I sent the captured request to Burp Repeater and started messing with it. I made two simple changes:

1.  I changed the endpoint from `/phone/` to `/email/`.
2.  I changed the JSON body from `"phone"` to `"email"` and put my own email address.

The modified request looked like this:

```http
POST /store/v1/auth/email/send_verification HTTP/2
Host: api.vulnerable-platform.com
Content-Type: application/json

{"email":"attacker@email.com","country_code":"SA"}
```

I sent the request, and to my surprise, I received a `200 OK` response. A few seconds later, a valid OTP landed in my email inbox. This was a huge discovery. The application had a hidden, fully functional email verification system that wasn't exposed in the UI.

## The Second Breakthrough: The Flawed Registration

Now that I could get an OTP via email, I needed to find a way to use it. I went back to the app and captured the main registration request. It was also designed for phone numbers:

```http
POST /store/v1/auth/phone/register HTTP/2
Host: api.vulnerable-platform.com
Content-Type: application/json

{
  "first_name":"test",
  "last_name":"test",
  "phone":"+15551234567",
  "code":"1234", // Code from SMS
  "country_code":"SA"
}
```

I applied the same logic as before. I sent this request to Repeater and modified it:

1.  Changed the endpoint from `/phone/` to `/email/`.
2.  Added an `"email"` parameter with my email address.
3.  Crucially, I kept the `"phone"` parameter, but this time I put the **victim's phone number** in it.
4.  I used the `code` I had received in my email.

The final malicious request looked like this:

```http
POST /store/v1/auth/email/register HTTP/2
Host: api.vulnerable-platform.com
Content-Type: application/json

{
  "email":"attacker@email.com",
  "first_name":"test",
  "last_name":"test",
  "phone":"VICTIM_PHONE_NUMBER", // The victim's phone number
  "code":"6280", // The code from my email
  "country_code":"SA"
}
```

## The Account Takeover

This was the moment of truth. I sent the request, and the backend logic completely fell apart. Here's what happened:

1.  It saw the request was to the `/email/register` endpoint, so it checked the `code` against the one it had sent to my email.
2.  The code was valid, so it considered my session verified.
3.  Then, instead of creating a new account, it looked at the `phone` parameter. It saw the victim's phone number, which was already registered.
4.  Instead of returning an error like "phone number already exists," it simply **merged my verified session with the victim's existing account**.

The response to my request contained a **session cookie for the victim's account**. I was in. I had full control, all without the victim ever knowing.

## The Impact: Silent, Mass Account Takeover

This vulnerability was critical because it was a silent, no-interaction account takeover. An attacker could:

*   **Systematically take over any account** with just a phone number.
*   **Access and steal all PII**, including order history, addresses, and more.
*   **Place fraudulent orders** or manipulate the victim's account.

## Steps to Reproduce

1.  **Discover the hidden email verification endpoint** by modifying the `/phone/send_verification` request to `/email/send_verification` and changing the body to use an email parameter. Request an OTP to an attacker-controlled email.
2.  **Capture the phone registration request** to `/phone/register`.
3.  **Modify the registration request**:
    *   Change the endpoint to `/email/register`.
    *   Add your email and the OTP you received.
    *   Set the `phone` parameter to the **victim's phone number**.
4.  **Send the request** and capture the session cookie from the response.
5.  **Use the session cookie** to make authenticated requests as the victim.

## The Fix

The fix requires a fundamental change in the authentication logic:

1.  **Enforce a Single Identifier**: The registration process should not allow mixing identifiers. If the verification is done via email, the resulting account should be linked to that email, not a phone number provided in the same request.
2.  **Proper Uniqueness Checks**: When a phone number is provided, the system must check if it exists and, if so, force the user to log in or reset their password using that phone number, not bypass it with an email verification.
3.  **Disable Unused Endpoints**: If the email registration flow is not intended to be used, the endpoints should be disabled entirely.

## Conclusion

This was a thrilling vulnerability to find. It started with simple curiosity—what if I change `/phone` to `/email`?—and ended with a full account takeover. It's a powerful lesson in not trusting what you see in the UI and always probing for hidden or alternative logic flows in the backend. Happy hacking!
