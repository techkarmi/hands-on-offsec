# Bike

| | |
|---|---|
| **Platform** | Hack The Box (Starting Point, Tier 1) |
| **OS** | Linux |
| **Difficulty** | Very Easy |
| **Techniques** | Server-Side Template Injection (Handlebars) to RCE |

## Overview
A Node.js/Express app using the Handlebars template engine. The email field is reflected
into a template, giving Server-Side Template Injection that leads to remote code
execution. The app runs as root, so RCE is instant root.

## Enumeration
```bash
nmap -sC -sV -oN nmap.txt <TARGET_IP>       # 80 http (Node.js / Express)
```
The homepage email field reflects input back ("We will contact you at: ...").

## Detect and fingerprint
Handlebars is logic-less, so `{{7*7}}` does not return 49; I fingerprinted by the error
instead. Submitting `{{` returned a stack trace full of `node_modules/handlebars/...`
and leaked `/root/Backend/routes/handlers.js`, confirming Handlebars and that the app
runs as root.

## Exploitation
Used the PayloadsAllTheThings Handlebars RCE payload to reach
`process.mainModule.require('child_process').exec(...)` and run a base64-encoded reverse
shell. Listener first:
```bash
nc -lvnp 4444
# payload email field: Handlebars block helper -> exec('echo <BASE64>|base64 -d|bash')
```

## Key Takeaway
Never render user input as a template. Use logic-less templates as data, keep input out
of the compile step, and run the app as a low-privilege user so RCE is not instant root.

## What tripped me up
This box was a masterclass in reading errors and fixing one thing at a time:
- Nested quotes broke the Handlebars string -> base64-encode the whole reverse shell so
  it contains no quotes.
- Base64 is not self-executing -> `echo <b64>|base64 -d|bash` (note the `echo`).
- `require is not defined` -> reach it via the global as `process.mainModule.require`.
- The web-spawned shell died after a few seconds -> grab the flag immediately, or detach
  with `setsid`.
Two encodings to keep straight: URL-encoding for the request body, base64 for the
command itself.
