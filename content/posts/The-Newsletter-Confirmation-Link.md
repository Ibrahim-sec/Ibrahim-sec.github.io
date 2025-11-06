---
date: 2024-06-09
title: "How I Leaked User Emails and Names Through a Buggy Newsletter Unsubscribe Feature_"
template: post
slug: The-Newsletter-Confirmation-Link
categories:
  - Technical
tags:
  - bug-bounty

---


What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a classic Insecure Direct Object Reference (IDOR) vulnerability I found on a hotel booking platform. This is a story of how a simple newsletter confirmation link led to the disclosure of user emails, and how I chained it with another leaky endpoint to get full names, creating a serious privacy breach.

## The Discovery: The Newsletter Confirmation Link

I started by testing the most common features on the site. I found a newsletter subscription form where you could sign up for hotel offers. I entered my email, and a few seconds later, I got a confirmation email.

The link in the email looked something like this:

```
https://cloud.news.vulnerable-site.com/confirmation?Customer_ID=<long_uuid>
```

The `Customer_ID` was a long, unguessable UUID. At first glance, this looks secure. But I've learned to never assume. What if the backend also accepts other types of IDs? I decided to test for a classic IDOR by replacing the long UUID with a simple integer, like `1`, `2`, `3`, etc.

I modified the URL to:

```
https://cloud.news.vulnerable-site.com/confirmation?Customer_ID=12345
```

To my surprise, it worked. The server accepted the simple integer ID and redirected me to an unsubscribe page for that user. This confirmed an IDOR vulnerability. The application was using guessable, sequential IDs, and it wasn't checking if the user viewing the page was the actual owner of that ID.

## The First Leak: Email Disclosure

Now that I could access any user's unsubscribe page, I needed to find a way to leak their information. The page itself didn't show the user's email, but it had a "Resubscribe" button.

I clicked the "Resubscribe" button and captured the request in Burp Suite. The request sent to the server contained the user's email address in the request body:

```http
POST /resubscribe HTTP/1.1
Host: cloud.news.vulnerable-site.com
Content-Type: application/json

{
  "customer_id": "12345",
  "email": "victim@email.com"
}
```

The application was pre-filling the resubscribe request with the victim's email address. By iterating through `Customer_ID`s, I could now collect the email address of every user in their database.


## Steps to Reproduce

1.  **Find a numeric `Customer_ID`** for a user.
2.  **Access the confirmation endpoint** with the victim's ID: `https://cloud.news.vulnerable-site.com/confirmation?Customer_ID={victim_id}`.
3.  **On the unsubscribe page, click the "Resubscribe" button** and capture the request in a proxy like Burp Suite.
4.  **Observe the victim's email address** in the body of the resubscribe request.

## The Fix

The fix for this requires a few changes:

1.  **Use unguessable identifiers**: The confirmation link should use a long, random, unguessable token instead of a sequential ID.
2.  **Tie actions to the session**: The backend should verify that the logged-in user is authorized to perform an action on a given resource. In this case, the resubscribe action should be tied to the user's session, not just the ID in the URL.
3.  **Don't leak data in requests**: The resubscribe request shouldn't need to include the user's email in the request body if the server can associate the action with the user's session.

## Conclusion

This was a great example of how two seemingly low-severity IDORs can be chained together to create a high-impact vulnerability. It's a reminder to always test for IDORs, even when you see a long UUID, and to always look for ways to combine vulnerabilities to increase their impact. Happy hacking!
