---
date: 2024-03-23
title: 'IDOR Into Account Takeover'
template: post
slug: IDOR-Into-Account-Takeover
categories:
  - Technical
tags:
  - bug-bounty

---


What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a wild ride of a vulnerability I found on a popular e-commerce platform. This is a story that goes beyond just the technical details of an IDOR; it's about the back-and-forth with the security team, proving the real impact, and fighting for the severity rating you know a bug deserves.

## The Discovery: The Edit Profile Endpoint

I started by signing up for an account on the platform and exploring the basic user functionalities. The "Edit Profile" page is always one of the first places I look for bugs. I updated my profile information, saved the changes, and captured the request in Burp Suite.

The request was a `PATCH` to an endpoint that looked like this:

```http
PATCH /api/v2/users/000080479494795?merchantId=9bb592ca-b1c2-4b41-af85-84f82aebba03&storeId=53559807-72c7-44d8-89bb-f4f146025b6d HTTP/2
Host: webstore-gateway.vulnerable-site.com
Authorization: Bearer [redacted]
Content-Type: application/json

{"profile":{"first_name":"USERz","last_name":"ONE","email":"nightmare@bugcrowdninja.com"}}
```

My eyes immediately locked onto the URL. It had my user ID (`000080479494795`) directly in the path. This is a classic sign of a potential IDOR. But it also had a `merchantId` that was a long, complex UUID. This could be a problem.

## The Initial Test & The Triager's Doubt

My first move was to see if I could simply replace my user ID with a victim's ID. The problem was, how would I get another user's ID? For testing, I created a second account (the "victim") and got its ID (`000039515344409`).

I swapped my ID for the victim's ID in the request and sent it. It worked. I could edit the victim's profile. But when I reported it, the triager came back with a valid question:

> "Are there two IDs that need to be brute-forced in order to exploit this? In this case there is a complex UUID (`merchantId`) and also a lengthy numeric value in the request path. If both of these need to be brute-forced in sync, this would be impractical in a real-world attack scenario."

They were right. If I needed to know both the numeric `user_id` AND the complex `merchantId` UUID for the victim, the bug would be almost impossible to exploit.

## The Breakthrough: The Useless UUID

This is where you have to push back and test further. I had a hunch: what if the `merchantId` wasn't actually being validated on the backend? What if it was just a leftover parameter from the frontend?

I went back to Burp Repeater and simply **deleted the `merchantId` and `storeId` parameters** from the URL entirely. The new request looked like this:

```http
PATCH /api/v2/users/000039515344409 HTTP/2
Host: webstore-gateway.vulnerable-site.com
Authorization: Bearer [redacted]
Content-Type: application/json

{"profile":{"first_name":"PWNED","last_name":"USER","email":"attacker@email.com"}}
```

I sent the request, and it went through successfully. The victim's profile was updated. This was the key. The UUID was completely unnecessary, which meant the only thing an attacker needed was the victim's numeric ID.

## Escalation to Account Takeover

Now that I could edit any user's profile, the path to account takeover was clear. By changing the `email` field in the JSON body to an email address I controlled, I could then go to the login page, click "Forgot Password," and reset the victim's password. Full account takeover.

## The Debate on Impact

Even after proving the `merchantId` was useless, the security team initially downplayed the severity. They argued:

*   It was on a "test store." (I later proved the endpoint was on a central gateway, `webstore-gateway.vulnerable-site.com`, affecting ALL stores).
*   It would be hard to retrieve victim IDs.
*   They had Cloudflare and rate limiting in place.

I countered by explaining that the numeric IDs, while long, could be enumerated over time with simple scripts and IP rotation to bypass rate limits. I even did the math for them:

> "With 100 IPs sending 1 request every 0.7 seconds, we can send 100,000,000 requests in about 8 days, potentially taking over 100,000,000 accounts."

This showed them I had thought through the real-world attack scenario. While they stood by their final P3 rating, my arguments were solid and based on practical attack methods.

## Steps to Reproduce

1.  **Create two accounts**: an Attacker and a Victim.
2.  **Log in** as the Attacker and go to the "Edit Profile" page.
3.  **Intercept the `PATCH` request** to `/api/v2/users/{attacker_id}`.
4.  **Modify the request** in Burp Repeater:
    *   Replace the `{attacker_id}` in the URL with the `{victim_id}`.
    *   **Delete the `merchantId` and `storeId` parameters** from the URL.
    *   Change the `email` in the JSON body to an email you control.
5.  **Send the request**. Observe a `200 OK` response.
6.  **Go to the password reset page**, enter the victim's original email, and complete the password reset using the link sent to *your* email address.

## Conclusion

This vulnerability was a great lesson in persistence. Don't let a complex-looking request discourage you. Always test removing parameters to see if they're actually being validated. And most importantly, if you believe a bug's impact is higher than what the triage team initially thinks, build a solid case and argue your point professionally. Even if they don't change the severity, you'll have sharpened your skills and proven you understand the full risk. Happy hacking!
