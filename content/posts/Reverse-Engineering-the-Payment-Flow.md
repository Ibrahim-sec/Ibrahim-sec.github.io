---
date: 2024-10-18
title: 'How I Got Free Account Credits by Reverse-Engineering the Payment Flow'
template: post
slug: Reverse-Engineering-the-Payment-Flow
categories:
  - Technical
tags:
  - bug-bounty

---
What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical payment bypass vulnerability I found on a developer platform. This is a story of how a missing server-side check allowed me to top-up my account with credits without actually paying for them. But the real trick was figuring out how the payment flow was broken in the first place.

## The Discovery: Following the Money

I was exploring a platform that allows developers to buy credits to use their API. On the billing page, there was a simple form to top-up your account. Anytime I see a payment process, I know I have to dig deeper. The only way to truly understand a payment flow is to actually go through it.

So, I decided to make a small, legitimate purchase. I entered my credit card details, selected the minimum top-up amount, and made sure Burp Suite was intercepting my traffic. I wanted to see every single request involved in the transaction.

## The Vulnerability: A Tale of Two Requests

As I watched the requests fly by in Burp, I noticed something interesting. The top-up process wasn't a single request, but two distinct API calls happening in sequence:

1.  **The Payment Request**: A request was sent to a third-party payment processor (like Stripe or Braintree) containing my credit card information and the amount to be charged.
2.  **The Credit Request**: Immediately after, a separate `POST` request was sent to the application's own backend, to an endpoint like `/api/credits/top-up`.

This second request was surprisingly simple:

```http
POST /api/credits/top-up HTTP/2
Host: vulnerable-developer-platform.com
Cookie: [redacted]
Content-Type: application/json

{"amount":50,"threshold":5}
```

This separation was the vulnerability. The application was designed with a fatal flaw: **it trusted the client to make both requests**. The backend endpoint that added credits to my account had no way of verifying that the first request—the one that actually charged my card—was successful. The two systems were completely decoupled.

## The Escalation: Replaying for Free Money

My theory was simple: what if I just skipped the payment request and only sent the credit request? To test this, I sent the `/api/credits/top-up` request to Burp Repeater.

I dropped the payment request and replayed only the credit request. The server responded with a `200 OK`. I held my breath and refreshed the billing page. My account balance had increased by $50. It worked. I had successfully bypassed the payment verification and could get free credits on demand.

## The Impact: Critical Financial Loss

This is a critical vulnerability with a direct and severe financial impact. An attacker could exploit this to:

*   **Get infinite account credits for free**, allowing them to use the platform's paid services without ever paying.
*   **Cause significant financial loss** to the company.
*   **Abuse the system** by creating multiple accounts and loading them with free credits.

This vulnerability could be easily automated to add millions of dollars in credits to an account in a matter of minutes.

## Steps to Reproduce

1.  **Log in** to an account and navigate to the billing/top-up page.
2.  **Initiate a legitimate payment** and intercept the traffic with a proxy like Burp Suite.
3.  **Identify the two separate requests**: one to the payment processor and one to the application's backend (e.g., `/api/credits/top-up`) to credit the account.
4.  **Send the credit request** to Burp Repeater.
5.  **Drop the payment request** and replay only the credit request.
6.  **Refresh the page** and observe that your account balance has increased without any charge to your card.

## The Fix

The fix for this vulnerability is to implement proper server-side payment verification. The backend should never blindly trust a request from the client to credit an account. The correct flow should be:

1.  The client initiates the payment on the frontend, which communicates with the payment provider (e.g., Stripe).
2.  The payment provider processes the payment and sends a confirmation (e.g., a webhook or a callback) directly to the application's **backend**.
3.  **Only after receiving a successful payment confirmation from the payment provider** should the backend update the user's account balance.

## Conclusion

This was a classic example of a broken payment verification flow. It's a critical reminder that you should never trust the client, especially when it comes to money. The only way to find vulnerabilities like this is to get your hands dirty, spend a little money, and analyze every step of the process. Happy hacking!
