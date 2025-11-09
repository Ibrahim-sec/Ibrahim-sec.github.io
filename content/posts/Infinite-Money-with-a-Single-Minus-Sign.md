---
date: 2024-05-29
title: 'Infinite Money with a Single Minus Sign A Business Logic Nightmare'
template: post
slug: Infinite-Money-with-a-Single-Minus-Sign
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and bug bounty hunters! Today, I'm breaking down a critical business logic vulnerability I found in an e-commerce application. This is a story of how a single minus sign in a quantity field led to a complete checkout bypass, allowing me to get products for free and even generate a negative order total. Let's get into it.

## The Discovery The Shopping Cart

I was testing a standard e-commerce site. I added a product to my shopping cart, a typical first step. The product I chose was priced at $99. When I went to my cart, I saw the usual options to change the quantity.

This is where my hacker senses started tingling. Many applications have solid client-side validation (e.g., you can't type "-1" in the quantity box), but they sometimes forget to validate on the server-side. I fired up Burp Suite to see what was happening when I updated the quantity.

## The Vulnerability Negative Quantities and Missing Server-Side Checks

When I updated the quantity in my cart, I intercepted the following `POST` request to the `/Order/SetActiveLineItem/{id}` endpoint:

```http
POST /Order/SetActiveLineItem/3 HTTP/1.1
Host: vulnerable-e-commerce-site.com
Cookie: [redacted]
Content-Type: application/x-www-form-urlencoded

LicenseType-3=Single&Quantity-3=1&...
```

The request body was a standard form-urlencoded string with a `Quantity-3` parameter set to `1`. The `3` here corresponds to the `id` of the line item in the cart.

I decided to see what would happen if I changed that `1` to a `-1`.

```http
POST /Order/SetActiveLineItem/3 HTTP/1.1
Host: vulnerable-e-commerce-site.com
Cookie: [redacted]
Content-Type: application/x-www-form-urlencoded

LicenseType-3=Single&Quantity-3=-1&...
```

I forwarded the request, and when I refreshed the shopping cart page, the price of the $99 item was now **-$99**. The application had accepted the negative quantity and calculated a negative price.

## The Escalation From Negative Price to Negative Total

Now that I had a negative price for one item, I could add other items to the cart to see how it affected the total. I added another product, this time priced at $50. My cart total was now:

*   Item 1: -$99
*   Item 2: $50
*   **Total: -$49**

My shopping cart had a negative total. I could essentially get the $50 product for free and still have a credit of $49. Depending on how the payment processor handles negative totals, this could lead to a refund being issued to my card for an order I never paid for.

## The Impact Critical Financial Loss

This is a critical business logic vulnerability with a direct financial impact. An attacker could exploit this to:

*   **Get products for free**: By balancing the cart to a total of $0.
*   **Generate fraudulent refunds**: By creating orders with a negative total.
*   **Cause database inconsistencies** and accounting nightmares for the company.

This could be easily automated to perform mass purchases at zero or negative cost, leading to significant financial loss.

## Steps to Reproduce

1.  **Log in** to an account on the target e-commerce site.
2.  **Add any product** to the shopping cart.
3.  **Intercept the request** that updates the cart quantity (e.g., using Burp Suite).
4.  **Modify the `Quantity` parameter** for the item from a positive value to a negative one (e.g., `Quantity-3=1` to `Quantity-3=-1`).
5.  **Forward the request** and refresh the shopping cart page.
6.  **Observe** that the item now has a negative price.
7.  **Add other items** to the cart to create a zero or negative total and proceed to checkout.

## The Fix

The fix for this is simple but crucial: **always perform server-side validation**. The application should check if the quantity is a positive integer. Any value less than or equal to zero should be rejected.

## Conclusion

This was a classic example of a business logic flaw. It's a good reminder that you should always test for edge cases, especially when it comes to features that have a direct financial impact. Don't trust client-side validation, and always check what happens when you send unexpected values to the server. Happy hacking!
