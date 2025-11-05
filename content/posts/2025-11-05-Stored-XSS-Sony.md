---
date: 2025-11-05
title: 'From IDOR to Full Account Takeover'
template: post
slug: Chain-IDOR-Vulnerability-To-Account-Takeover
categories:
  - Technical
tags:
  - bug-bounty

---
What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical vulnerability I found in a new program. This is a story of how a simple Insecure Direct Object Reference (IDOR) in a workspace switching feature escalated into a full session hijacking, giving me complete control over another user's account.

## The Setup: Fresh Program, Fresh Scope

The program was brand new, with only two applications in scope. When a program is this fresh, you know there's a good chance of finding some low-hanging fruit. I created two accounts so I could test for authorization issues between them. Let's call them:

*   **Attacker Account:** The account I control.
*   **Victim Account:** The account I want to take over.

My main goal was to see if I could access or modify data from the Victim Account while logged in as the Attacker.

## The Discovery: The Workspace Switch

While exploring one of the apps, I found a feature that let users switch between different workspaces. This is a common feature in SaaS applications, and it's often a hotspot for IDORs. I fired up Burp Suite to see what was happening under the hood.

When I switched workspaces, I saw a `POST` request that looked something like this:

```http
POST /api/v1/workspaces/switch HTTP/1.1
Host: vulnerable-app.com
Authorization: Bearer <attacker_auth_token>
Content-Type: application/json

{
  "workspace_id": "ws_attacker_space_123"
}
```

The request was simple: it sent the `workspace_id` of the workspace I wanted to switch to. My mind immediately went to IDOR. What would happen if I replaced the `workspace_id` with one from my Victim Account?

## The Vulnerability: IDOR to Session Leak

I logged into my Victim Account, created a new workspace, and grabbed its ID (`ws_victim_space_456`). Then, I went back to my Attacker Account's session in Burp Repeater and replaced the `workspace_id` with the victim's ID.

```http
POST /api/v1/workspaces/switch HTTP/1.1
Host: vulnerable-app.com
Authorization: Bearer <attacker_auth_token>
Content-Type: application/json

{
  "workspace_id": "ws_victim_space_456" // <-- IDOR here
}
```

I sent the request, and the response was... weird. There was no JSON body, no error, just a `200 OK` status and a few headers. But one of those headers immediately caught my eye:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 0
Set-Cookie: session_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...; Path=/; HttpOnly
X-Message: Switched workspace
```

The application was telling me 
it had switched workspaces, and it was giving me a *new session token* in the `Set-Cookie` header. This was a huge red flag. Why would it give me a new session token just for switching workspaces? My theory was that this new session token belonged to the Victim Account.

## The Escalation: Session Hijacking and Full Account Takeover

Now it was time to test my theory. I copied the new `session_token` from the response and replaced my Attacker Account's session token with it in the `Cookie` header of my next request. I decided to try a sensitive action, like listing the API keys for the workspace:

```http
GET /api/v1/workspaces/ws_victim_space_456/api_keys HTTP/1.1
Host: vulnerable-app.com
Cookie: session_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9... // <-- Leaked session token
```

I sent the request, and the response was a `200 OK` with the Victim Account's API keys. It worked. I had successfully hijacked the victim's session.

To confirm the full impact, I tested other endpoints. I was able to:

*   Create, delete, and edit workspaces.
*   Create, delete, and manage API keys.
*   Invite and remove users from workspaces.

I had full control over the Victim Account, all because of a single IDOR vulnerability that leaked a session token.

## The Impact

This was a critical vulnerability. An attacker could use this to take over any user's account if they knew their workspace ID. This could lead to:

*   Data theft
*   Unauthorized access to sensitive information
*   Complete account takeover

## Steps to Reproduce

1.  **Log in** to your Attacker Account.
2.  **Find the request** that switches workspaces. In Burp Suite, this will likely be a `POST` request to an endpoint like `/api/v1/workspaces/switch`.
3.  **Change the `workspace_id`** in the request to a `workspace_id` from your Victim Account.
4.  **Send the request** and check the response headers for a `Set-Cookie` header with a new `session_token`.
5.  **Copy the new `session_token`** and use it in the `Cookie` header of your subsequent requests.
6.  **Confirm** that you can access the Victim Account's data.

## The Fix

I reported the issue to the team, and they patched it within hours. The fix was to properly validate that the user is authorized to access the workspace they are trying to switch to before generating a new session.

## Conclusion

This was a great example of how a simple IDOR can be escalated into a critical vulnerability. It also shows the importance of checking the response headers for any unexpected information, like a leaked session token. Always be curious, and always dig deeper!
