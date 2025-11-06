---
date: 2025-02-29
title: 'How I Took Over Any Account by Adding a Dot to a Phone Number'
template: post
slug: How-I-Took-Over-Any-Account-by-Adding-a-Dot-to-a-Phone-Number
categories:
  - Technical
tags:
  - bug-bounty

---
What up, fellow hackers and bug bounty hunters! Today, I'm breaking down one of the most interesting account takeover vulnerabilities I've ever found. This is a story of how a simple phone number normalization bypass allowed me to hijack any user's account on a major e-commerce platform, all because of a single misplaced dot.

## The Discovery: The Registration Flow

I was testing the user registration flow of a popular e-commerce platform. The process was standard: sign up with an email and a phone number. The system used the phone number as a key identifier for the user account.

My first test was to see what would happen if I tried to register with a phone number that was already in use. I created a victim account with my own secondary phone number. Then, I tried to create a new account with that same phone number. As expected, the application returned an error: "This phone number is already registered."


This is a good sign, but my work wasn't done. Many applications fail at **input normalization**. They might check for the existence of `123456789`, but what if I register with `123456789.` or `123-456-789`? If the backend doesn't properly sanitize and normalize the input before checking for uniqueness, you can often bypass this control.

## The Vulnerability: Bypassing the Uniqueness Check

I decided to test this normalization theory. I went back to the registration page and, for the phone number, I entered the victim's number but with a single dot (`.`) at the end.

**Victim's Number:** `+15551234567`
**My Input:** `+15551234567.`

The application accepted it! The uniqueness check saw `+15551234567.` as a new number and let me proceed. I also found that other variations worked, such as URL-encoded characters like `%2d` (hyphen) or `%0c` (form feed character).

This proved the backend was not normalizing the phone number *before* the check. But the real vulnerability was in what happened next.

## The Critical Flaw: OTP Sent to the Attacker's Email

After bypassing the phone number check, the registration flow asked me to provide an email address to verify my new account. I entered an email address that I controlled. The application then sent a One-Time Password (OTP) to **my email address**, not to the victim's phone number.

This was the critical design flaw. The system should have recognized that the (normalized) phone number already existed and sent the verification code to that trusted phone number. Instead, it trusted the new email address provided by the attacker.

## The Account Takeover

Once I received the OTP in my email, I entered it, and the registration was complete. The system then did something catastrophic: it normalized the phone number *after* registration, stripping the dot. It then merged my new account (with my email and password) with the victim's existing account, because they now shared the same primary identifier—the phone number.

I was immediately logged into the victim's account. I had full access to:

*   **Personal Information**: Full name, address, and contact details.
*   **Order History**: A complete list of all past purchases.
*   **Stored Payment Methods**: In some cases, this could include credit card information.

I had successfully taken over the account. The victim would be locked out, and their data would be completely compromised.

## The Impact: Mass Account Takeover

This is a critical account takeover vulnerability. An attacker could:

*   **Systematically take over user accounts** by using publicly available phone numbers.
*   **Steal sensitive PII** for identity theft and fraud.
*   **Place fraudulent orders** using the victim's stored information.

## Steps to Reproduce

1.  **Create a victim account** with a specific phone number.
2.  **Start a new registration** process.
3.  When asked for a phone number, **enter the victim's phone number with a dot (`.`) at the end** (or another normalization bypass character like `%2d`).
4.  When asked for an email, **enter an email address you control**.
5.  **Complete the OTP verification** using the code sent to *your* email.
6.  Upon completion, you will be logged into the **victim's account**.

## The Fix

The fix requires two key changes:

1.  **Strict Input Normalization**: The backend must normalize the phone number (by stripping all non-numeric characters) *before* performing the uniqueness check.
2.  **Verify the Existing Channel**: If a phone number is found to already exist after normalization, the verification OTP must be sent to that phone number, not to a newly provided email address.

## Conclusion

This was a fascinating vulnerability that combined a simple normalization bypass with a critical logic flaw in the verification process. It's a powerful reminder that security is about layers. Even if one check fails, the next one should catch the anomaly. Always test for normalization issues, and always question the logic of verification flows. Happy hacking!
