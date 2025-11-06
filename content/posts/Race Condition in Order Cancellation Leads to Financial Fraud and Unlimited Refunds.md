---
date: 2025-07-12
title: "Race Condition in Order Cancellation Leads to Financial Fraud and Unlimited Refunds"
template: post
slug: Race Condition-in-Order-Cancellation-Leads-to-Financial-Fraud-and-Unlimited-Refunds
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a classic but highly impactful vulnerability: a **Race Condition** in a financial transaction flow. This is a story of how a flaw in the order cancellation logic on a major e-commerce platform allowed me to receive multiple, unauthorized refunds for a single purchase, resulting in a direct financial loss for the platform and a high-severity bounty.

## The Discovery: The Influencer Marketing Cancellation

I was testing the platform's "Influencer Marketing" feature, which allows users to purchase services from influencers. After completing a purchase, the platform provides a convenient option to **cancel the order and receive an immediate refund** to the user's internal wallet.

The process is simple:
1.  Purchase a service (e.g., from `https://platform.com/marketing/influencers/12345`).
2.  Click the "Cancel Order" button.
3.  The system processes the cancellation and refunds the full amount to the user's wallet.

The request captured in Burp Suite for the cancellation looked like this:
```http
POST /api/v1/orders/cancel HTTP/2
Host: api.vulnerable-platform.dev
Authorization: Bearer <USER_JWT>
Content-Type: application/json

{
  "order_id": "699506686",
  "reason": "Changed my mind"
}
```

This is a critical area for security testing. Any process that involves a state change (from "paid" to "canceled") and a financial transaction (a refund) is a prime target for a Race Condition attack.

## The Vulnerability: The Unprotected Refund Logic

A Race Condition occurs when a system's output is dependent on the sequence or timing of uncontrollable events. In this case, the system was failing to properly lock the transaction record before processing the refund.

The expected flow is:
1.  **Check**: Is the order eligible for a refund? (Yes, it's paid and not yet canceled).
2.  **Lock**: Lock the order record to prevent concurrent changes.
3.  **Process**: Issue the refund.
4.  **Update**: Change the order status to "Canceled" and mark it as "Refunded."
5.  **Unlock**: Release the order record.

The vulnerability was that the system was missing or had an ineffective **Lock** mechanism. This allowed an attacker to send multiple, identical cancellation requests to the server simultaneously.

## The Exploitation: Turbo Intruder Attack

To exploit this, I used a classic Race Condition attack methodology:

1.  **Capture Request**: I initiated the cancellation process and captured the HTTP request in Burp Suite.
2.  **Send to Intruder**: I sent the request to Burp's **Turbo Intruder** extension.
3.  **Synchronized Attack**: I configured Turbo Intruder to send a large number of identical requests (e.g., 30 requests) in a highly synchronized manner, using a "gate" to ensure they hit the server almost simultaneously.

The goal was to have multiple requests pass the initial **Check** (Is the order eligible for a refund?) before the first request could complete the **Update** step (Change status to "Canceled").

The server, receiving 30 requests at the same time, processed the refund logic for several of them before the order status was updated.

## The Impact: Financial Fraud and Scalable Loss

The attack was highly successful. For a single purchase, I was able to receive multiple refunds, significantly inflating the balance in my internal wallet.

*   **Direct Financial Loss**: The platform suffered a direct financial loss for every unauthorized refund processed.
*   **Scalable Fraud**: The attack could be automated and scaled. An attacker could make a large purchase, exploit the race condition to receive 10x the refund, and then use the inflated wallet balance to purchase other goods or services on the platform.
*   **Undermining Trust**: This vulnerability undermines the integrity of the platform's core financial systems, as the wallet balance no longer accurately reflects the user's actual funds.

The platform initially rated this as High severity (€600 bounty), but the researcher argued for Critical, noting that the ability to generate funds and use them to purchase anything on the platform made the impact severe enough to warrant the highest rating.

## Steps to Reproduce

1.  **Purchase Service**: Complete a payment for an Influencer Marketing service.
2.  **Capture Cancellation**: Click the "Cancel Order" button and capture the resulting HTTP request in Burp Suite.
3.  **Configure Race**: Send the request to **Turbo Intruder** and configure a synchronized attack (e.g., 30 concurrent connections, 30 requests per connection, using a gate).
4.  **Execute Attack**: Launch the attack.
5.  **Verify Refund**: Navigate to the platform's wallet page.
6.  **Observe**: The wallet balance will be credited multiple times, exceeding the original payment amount.

## The Fix

The fix for a Race Condition in a financial transaction is to implement a robust concurrency control mechanism:

1.  **Transaction Locking**: The system must implement a database-level transaction lock on the order record *before* the eligibility check and hold that lock until the order status is updated to "Canceled" and the refund is fully processed.
2.  **Atomic Operations**: The check, refund, and status update operations should be grouped into a single, atomic database transaction. If any part of the transaction fails (e.g., the order status is already "Canceled"), the entire transaction must be rolled back.
3.  **Idempotency Check**: The refund processing logic should check for a unique refund ID or transaction ID to ensure the same transaction is not processed more than once, even if multiple requests are received.

## Conclusion

This vulnerability is a powerful reminder that in modern, highly concurrent web applications, logic flaws related to timing can be just as dangerous as traditional injection flaws. Any time a user action triggers a state change and a financial transaction, developers must assume multiple requests will arrive simultaneously. Implementing proper database locking and atomic transactions is non-negotiable for maintaining the integrity of a platform's financial health. Happy hacking!
