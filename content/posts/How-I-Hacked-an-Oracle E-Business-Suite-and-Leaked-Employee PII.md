---
date: 2022-04-15
title: 'How I Hacked an Oracle E-Business Suite and Leaked Employee PII'
template: post
slug: How-I-Hacked-an-Oracle E-Business-Suite-and-Leaked-Employee PII
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a sick find that started with some basic recon on Shodan and ended with me pulling sensitive employee data from a misconfigured Oracle E-Business Suite instance. This is a story of how a known CVE and a leaky search feature led to a full-blown PII disclosure.

## The Discovery: Shodan Recon

My process always starts with broad reconnaissance. I was scanning a target's IP ranges on Shodan, looking for interesting open ports and services. I stumbled upon a subdomain that was running Oracle E-Business Suite. The login page looked old and enterprise-y, a perfect target for known vulnerabilities.

Seeing an Oracle E-Business Suite instance exposed to the internet immediately set off alarm bells. I remembered reading about several CVEs related to these systems, specifically ones that allow unauthorized users to register on what should be an internal-only application.

## The Vulnerability: Unauthorized Registration

One of the most notorious vulnerabilities in older Oracle E-Business Suite versions allows for self-registration even when it's supposed to be disabled. The vulnerability lies in the `ibeCAcpSSOReg.jsp` page, which is often left accessible. This page allows an external user to create an account, effectively bypassing the main login and gaining a foothold inside the application.

I decided to test this theory. I navigated directly to the registration page, and to my surprise, it loaded perfectly.

## The Exploitation: From Registration to PII Leak

Now that I knew I could create an account, I followed a specific set of steps to see how deep I could get.

1.  **Register an Account**: I went to the registration page (`/OA_HTML/ibeCAcpSSOReg.jsp`) and registered as a new individual user. The system accepted my registration without any issues.

2.  **Log In**: With my new credentials, I logged into the application via the main login page (`/OA_HTML/AppsLocalLogin.jsp`). I was now an authenticated, albeit low-privileged, user inside the system.

3.  **Digging for Gold**: Once inside, I started exploring the dashboard. I navigated to the **Settings** menu, then to **Manage Proxies**. This section seemed to be for managing other users you could act on behalf of.

4.  **The Leaky Report**: In the Manage Proxies section, there was an option to "Run Proxy Report." I clicked it, and a new window popped up with a search form. This looked promising.

5.  **The Final Step**: The search form was intended to find employees. I simply clicked the "Search" button without entering any query. The application, lacking proper validation, interpreted this as a wildcard search and returned a massive list of **all employees** in the database.

Each entry in the search result contained the employee's **First Name, Last Name, Email Address, and Username**. I had hit the jackpot.

## The Impact: Mass PII Disclosure

This is a critical information disclosure vulnerability. By chaining the unauthorized registration with the leaky search feature, an attacker could:

*   **Harvest the PII** of every employee in the company.
*   **Use this information** for targeted phishing attacks, social engineering, or credential stuffing.
*   **Cause a major privacy breach** and reputational damage to the company.

## Steps to Reproduce

1.  **Navigate** to the Oracle E-Business Suite registration page: `/OA_HTML/ibeCAcpSSOReg.jsp`.
2.  **Register** as a new individual user.
3.  **Log in** to the application with the newly created credentials at `/OA_HTML/AppsLocalLogin.jsp`.
4.  **Navigate** to **Settings > Manage Proxies**.
5.  Click **Run Proxy Report**.
6.  In the new search window, click the **Search** button without entering any search terms.
7.  **Observe** the full list of employee PII (First Name, Last Name, Email, Username) being returned.

## The Fix

The fix for this requires several layers:

1.  **Patch Oracle E-Business Suite**: The most important step is to apply all relevant security patches from Oracle to fix the underlying unauthorized registration vulnerability (CVE).
2.  **Restrict Access**: The entire Oracle E-Business Suite instance should not be exposed to the public internet if it's intended for internal use only. It should be behind a VPN or firewall.
3.  **Enforce Secure Configurations**: The self-registration feature should be explicitly disabled.
4.  **Fix Leaky Search**: The search functionality should require a specific query and not allow empty or wildcard searches that dump the entire database.

## Conclusion

This was a perfect example of how important it is to keep your software patched and properly configured. A single known CVE on an exposed system can lead to a complete compromise of sensitive data. Always be on the lookout for outdated enterprise software during your recon phase—it's often a goldmine for critical vulnerabilities. Happy hacking!
