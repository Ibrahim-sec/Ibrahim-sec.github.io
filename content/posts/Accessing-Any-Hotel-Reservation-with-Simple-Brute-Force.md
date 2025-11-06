---
date: 2022-02-12
title: 'Accessing Any Hotel Reservation with Simple Brute Force'
template: post
slug: Accessing-Any-Hotel-Reservation-with-Simple-Brute-Force
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical vulnerability I found on a major hotel booking platform. This is a story of how a weak confirmation number system and a lack of rate limiting allowed me to access and modify any user's reservation, leading to a massive privacy breach.

## The Discovery: The Reservation Search

I started by exploring the hotel's booking management page. There was a simple form to find a reservation: you needed a "Confirmation Number" and an "Email Address." This is a common feature, but it's often a goldmine for vulnerabilities if not implemented correctly.

To understand how it worked, I made a test booking to get a valid confirmation number for my own account. My confirmation number was a simple 8-digit number, something like `12511218`. This was the first red flag. Short, numeric, and potentially sequential IDs are a recipe for disaster.

I then used the search feature with my confirmation number and email, making sure to capture the request in Burp Suite. The request sent to the backend looked like this:

```http
POST /api/Reservation/ByPmsid/get HTTP/2
Host: booking.vulnerable-hotel.com
Cookie: [redacted]
Content-Type: application/json

{"lang":"en-US","hotelCode":"001","email":"my-email@example.com","pmsid":"12511218"}
```

The request sent a JSON body with my email and the confirmation number, which was labeled `pmsid`. The lack of any CSRF token or complex identifier meant I could easily replay and manipulate this request.

## The Vulnerability: Brute-Forcing Confirmation Numbers

The core issue was twofold:

1.  **Predictable, Short IDs**: The `pmsid` was a short, numeric value, making it incredibly easy to guess or brute-force.
2.  **No Rate Limiting**: The server didn't limit the number of times I could try to find a reservation. I could send thousands of requests without being blocked.

This combination meant that if I knew a victim's email address, I could find their reservation by simply trying all possible confirmation numbers.

## The Attack Scenario: Putting it all Together

Here's how I set up the attack in Burp Suite:

1.  **Send to Intruder**: I sent the captured `POST` request to Burp Intruder.

2.  **Set the Payload Position**: I cleared the default payload positions and added one only on the value of the `pmsid` parameter:

    ```json
    {"lang":"en-US","hotelCode":"001","email":"victim@email.com","pmsid":"§12511218§"}
    ```

3.  **Configure the Payload**: I set the payload type to **Numbers** and configured it to generate a range of 8-digit numbers. Since I had a recent confirmation number (`12511218`), I started my range close to that, assuming the numbers were sequential. For example, from `12500000` to `12600000`.

4.  **Launch the Attack**: I launched the attack. Most requests came back with an error indicating the reservation was not found. However, a successful request would have a different response length and status code, making it easy to spot.

Within minutes, I found a valid confirmation number for the victim's email address. The response contained a treasure trove of sensitive information.

## The Impact: Full PII and Reservation Hijacking

A successful brute-force attack revealed:

*   **Full Booking Details**: Hotel name, room type, check-in/check-out dates, and total cost.
*   **Guest Information**: Full name, phone number, and physical address.
*   **Payment Information**: The last four digits of the credit card used for the booking.
*   **Modification Options**: The response also contained links or tokens that would allow me to **modify or even cancel the reservation**.

This is a critical privacy breach. An attacker could:

*   **Stalk a user** by knowing their exact travel plans.
*   **Cancel a user's reservation** out of malice.
*   **Use the leaked PII** for phishing or other social engineering attacks.

## Steps to Reproduce

1.  **Obtain a victim's email address**.
2.  **Navigate** to the reservation search page.
3.  **Enter an arbitrary confirmation number** and the victim's email, and capture the `POST` request to `/api/Reservation/ByPmsid/get`.
4.  **Send the request to Burp Intruder** and set the payload position on the `pmsid` value.
5.  **Configure a number range payload** to iterate through possible confirmation numbers.
6.  **Launch the attack** and monitor the responses for a change in length or status code, indicating a valid confirmation number.
7.  **Observe the sensitive PII** and booking details in the successful response.

## The Fix

The fix for this requires a multi-layered approach:

1.  **Implement Strong Rate Limiting**: The server should lock out an IP address or an account after a small number of failed attempts (e.g., 5-10).
2.  **Use Unguessable Confirmation Numbers**: Confirmation numbers should be long, alphanumeric, and randomly generated (like a UUID). They should not be sequential or predictable.
3.  **Add CAPTCHA**: After a few failed attempts, the user should be required to solve a CAPTCHA.

## Conclusion

This was a classic example of how a seemingly simple feature can have critical security flaws. Predictable identifiers combined with a lack of rate limiting is a recipe for disaster. Always test for brute-force vulnerabilities on any feature that relies on a guessable ID. Happy hacking!
