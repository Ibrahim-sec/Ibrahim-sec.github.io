---
date: 2025-04-21
title: 'Account Takeover Chain Side Channel ID Leak Bypasses 11-Digit Security'
template: post
slug: Account-Takeover-Chain-Side-Channel-ID-Leak-Bypasses-11-Digit-Security
categories:
  - Technical
tags:
  - bug-bounty
---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a vulnerability chain that proves you should never give up on an IDOR just because the IDs look unguessable. This is a story of how a seemingly secure, 11digit random user ID was leaked through a sidechannel in an "Invite User" feature, which I then used to exploit a critical Insecure Direct Object Reference (IDOR) to view and modify any user's profile.

## The Discovery: The UnGuessable ID

I was testing the user profile page on a major B2B platform. When I navigated to my own profile, the URL contained a long, numeric ID:

```
https://app.vulnerableplatform.com/profile/1299138221456
```

The ID was over 11 digits long and appeared to be randomly generated, not sequential. This usually means a direct IDOR attack is impossible because you can't bruteforce or guess the victim's ID.

However, I knew that if I could find a way to **obtain** a victim's ID, the IDOR might still be exploitable.

## The Vulnerability: The ID Leak SideChannel

My attention turned to the "Users" or "Team Management" section, which had an "Invite User" feature. This feature allowed me to invite new team members via their email or phone number.

I used a victim's known email address (`victim@email.com`) in the invite form and captured the request. The request was standard, but the **response** was the key. The application, in its eagerness to confirm the invite, returned the newly invited user's internal ID, even if that user already existed on the platform.

Here is the simulated request and the critical part of the response:

**Request:**
```http
POST /api/v1/team/invite HTTP/1.1
Host: api.vulnerableplatform.com
ContentType: application/json

{
  "invite_method": "email",
  "identifier": "victim@email.com",
  "role": "member"
}
```

**Response (Critical Snippet):**
```json
{
  "status": "success",
  "message": "Invitation sent successfully.",
  "data": {
    "invite_id": "uuid1234567890ab",
    "user_id": "4912831233789", 
    "email": "victim@email.com"
  }
}
```

Boom. The `user_id` of the victim (`4912831233789`) was now exposed to me, the attacker, simply by knowing their email address. This turned the "unguessable" ID into a "leaked" ID.

## The Exploitation: Chaining the Leak to IDOR

With the victim's ID in hand, the path to exploitation was simple. I took the leaked ID and replaced my own ID in the profile URL:

```
// Attacker's URL:
https://app.vulnerableplatform.com/profile/1299138221456

// Victim's URL (using leaked ID):
https://app.vulnerableplatform.com/profile/4912831233789
```

When I navigated to the victim's URL, the page loaded their profile. The underlying API request to fetch the profile data was completely lacking an authorization check, allowing me to view:

*   **Full Name**
*   **Email Address**
*   **Phone Number**
*   **Internal Role/Permissions**
*   **Address and Billing Information**

Furthermore, the profile page allowed me to **edit** the information. By intercepting the `PUT` or `PATCH` request to the profile endpoint and using the victim's ID, I could change their email address, effectively leading to a full **Account Takeover** via a password reset.

## The Impact: Critical PII Disclosure and Account Takeover

This vulnerability chain is critical because it bypasses the security measure of using long, random IDs and leads directly to the highest possible impact:

*   **Mass PII Disclosure**: An attacker could automate the "invite" process using a list of emails or phone numbers to harvest the internal IDs of millions of users.
*   **Full Account Takeover**: By changing the victim's email and initiating a password reset, an attacker gains complete control over the victim's account.
*   **Unauthorized Actions**: Depending on the victim's role (e.g., an admin or a highprivilege user), the attacker could perform destructive actions like deleting data or making fraudulent transactions.

## Steps to Reproduce

1.  **Obtain a victim's email or phone number** (e.g., `victim@email.com`).
2.  **Navigate to the "Invite User" feature** on the platform.
3.  **Send an invitation** using the victim's identifier.
4.  **Intercept the API response** for the invite request.
5.  **Observe the `user_id`** of the victim being returned in the response body (e.g., `4912831233789`).
6.  **Navigate to the profile page** using the leaked ID: `https://app.vulnerableplatform.com/profile/{leaked_id}`.
7.  **Observe the victim's PII** being displayed.
8.  **(Escalation)** Intercept the profile update request and change the victim's email to an attackercontrolled email to perform an Account Takeover.

## The Fix

The fix requires two distinct security improvements:

1.  **Fix the ID Leak**: The "Invite User" endpoint should **never** return the internal `user_id` of an existing user in the response. If the user already exists, it should return a generic message or an invite ID, but not the internal ID.
2.  **Fix the IDOR**: The profile viewing and editing endpoints must implement **Strict Access Control**. Before displaying or modifying a profile, the backend must verify that the authenticated user's session ID matches the `user_id` in the request path.

## Conclusion

This was a fantastic example of how a chain of lowtomedium severity flaws can combine to create a critical vulnerability. The long, random ID was a good defense, but the information leak in the invite feature completely nullified it. Always remember to test every single API response for unintended data disclosure, as that small leak can be the key to unlocking a massive exploit. Happy hacking!
