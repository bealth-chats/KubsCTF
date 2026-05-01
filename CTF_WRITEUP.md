# Capy-Capy CTF Challenge Write-up

## Challenge Description Analysis
The challenge provided the following hint:
> "Lately, the theme change is buggy. Maybe it's because of the bugs? Did the site work the same way before?"

We were also given three IP addresses:
- `http://193.42.127.24`
- `http://159.194.209.128`
- `http://159.194.199.71`

The description implies two main things:
1. There might be a vulnerability related to the "theme change" feature (potentially LFI, insecure deserialization, or SSRF).
2. "Did the site work the same way before?" hints that we should look closely at the different IPs, as one of them might represent an "older" or "buggy" version of the application that leaks more information or contains the flag directly.

## Step-by-Step Investigation

1. **Initial Exploration:**
   I first curled the homepage of the main IP (`http://193.42.127.24`) to understand the application. It was a simple blog with a dark/light theme toggle.

2. **Investigating the Theme Feature:**
   I noticed the theme change worked by making a request to `?set_theme=dark`, which then set a `theme` cookie in the browser. I attempted various Local File Inclusion (LFI) payloads like `../../../../../etc/passwd` and `php://filter/...` in the `theme` cookie to see if the application was directly loading the theme file and vulnerable to LFI. However, the application didn't seem to be directly vulnerable to simple LFI in the cookie on the first server.

3. **Analyzing Differences Between IPs:**
   Following the hint "Did the site work the same way before?", I realized I should compare the responses from all three servers. I fetched the source code of the homepage (`index.php`) from all three IPs.

4. **Discovering the Flag:**
   I noticed that the content length of the response from `http://159.194.209.128/` was significantly larger (15012 bytes) compared to the other two servers (12837 bytes).

   I inspected the source code of `http://159.194.209.128/index.php` and found that at the very bottom of the HTML, there were comments left behind, likely from a debugging session or an older vulnerable version.

   ```html
   <!--OUT:/proc/sys/kernel/acpi_video_flags
   ... [truncated system files listing] ...
   var
   :END-->
   <!--FLAG:KubSTU(capybl0g_php_d3s3r1al1zat10n)
   END-->
   ```

   The flag `KubSTU(capybl0g_php_d3s3r1al1zat10n)` was exposed in an HTML comment on the second IP address.

## Conclusion
The challenge required us to realize that the "buggy" theme feature might have led to debugging outputs being left on an older version of the site (the second IP). By simply inspecting the source code of the different servers, the flag was found in the HTML comments. The flag content (`capybl0g_php_d3s3r1al1zat10n`) also hints that the actual vulnerability on the current server might involve PHP deserialization via the theme cookie, but finding the debugging output on the older server was a direct shortcut to the flag!
