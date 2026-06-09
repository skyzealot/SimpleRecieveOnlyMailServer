# Receive-only mail server for your.domain

This guide walks you through setting up a mail server that catches every email
sent to `your.domain`. Anything before the `@` — `hello@`, `billing@`,
`literallyanything@` — all land in a single inbox you read with a normal
email app like Thunderbird or Apple Mail.

The server only **receives** mail. It cannot send. That's intentional: it
makes the setup simpler and there's nothing for spammers to abuse.

## What you need before starting

- A Namecheap VPS Spark (or similar) running Ubuntu 24.04.
- The VPS's public IP. In this guide it's `YOUR_VPS_IP` — replace with
  yours throughout.
- A domain registered at Namecheap.
  replace with yours throughout.
- A laptop with PowerShell (Windows) and SSH installed. SSH ships with
  Windows 10/11 by default.
- Around 30 minutes, plus DNS propagation time.

## The pieces, in plain English

- **Postfix** receives email from the internet (the "mailman").
- **Dovecot** lets your email app read the mail (the "mailbox you open with
  a key").
- **Let's Encrypt** gives you a free TLS certificate so connections are
  encrypted (the "padlock icon").
- **DNS records** at Namecheap tell the rest of the internet where to
  deliver mail for your domain.

## Two terminals — pay attention to which is which

You'll switch between two windows constantly. Every command in this guide
is labelled:

- **[LAPTOP]** = PowerShell on your Windows machine. Prompt looks like
  `PS C:\Users\You>`.
- **[VPS]** = an SSH session connected to the server. Prompt looks like
  `root@mail:~#`.

Running a [VPS] command on your laptop (or vice versa) is the single most
common mistake. Check the prompt before you press Enter.

---

## 1. Set up DNS at Namecheap

DNS records are the public signposts that tell other mail servers "send
mail for `your.domain` to this IP address." Without them, no email will
arrive.

Log in to Namecheap, go to **Domain List → Manage → Advanced DNS** for
`your.domain`, and add these five records:

| Type | Host     | Value                                   | What it does                                           |
| ---- | -------- | --------------------------------------- | ------------------------------------------------------ |
| A    | `mail`   | `YOUR_VPS_IP`                           | Points `mail.your.domain` at your VPS                  |
| MX   | `@`      | `mail.your.domain.`                     | Tells the world "send mail for your.domain to mail.your.domain." Set Priority to 10. **The trailing dot is required** |
| TXT  | `@`      | `v=spf1 mx -all`                        | SPF: declares "only the MX server sends as @your.domain" — and since we don't send, that's no one |
| TXT  | `_dmarc` | `v=DMARC1; p=reject; adkim=r; aspf=r;`  | DMARC: tells receivers to reject any spoofed mail claiming to be from us. Relaxed alignment so our own bounces don't fail authentication |
| CAA  | `@`      | Tag `issue`, value `letsencrypt.org`    | Restricts who can issue HTTPS certificates for your domain |

### Reverse DNS (PTR record)

Reverse DNS maps an IP back to a hostname. Mail servers use it as a sanity
check: "you say you're `mail.your.domain` — does the IP agree?" Without it,
Gmail in particular may delay or reject your incoming mail.

Namecheap won't let you set this from the dashboard. Open a support ticket
asking them to point `YOUR_VPS_IP` to `mail.your.domain`. They usually do
it within a few hours.

### Wait, then verify

Give DNS about 30 minutes to propagate, then check from your laptop:

**[LAPTOP]**
```powershell
Resolve-DnsName mail.your.domain -Server 8.8.8.8
Resolve-DnsName -Type MX your.domain -Server 8.8.8.8
Resolve-DnsName -Type TXT your.domain -Server 8.8.8.8
Resolve-DnsName -Type TXT _dmarc.your.domain -Server 8.8.8.8
Resolve-DnsName -Type CAA your.domain -Server 8.8.8.8
```

`-Server 8.8.8.8` queries Google's public DNS rather than your home
router's cache, so you see what the rest of the internet sees. Each
command should return the value you put in Namecheap. If something is
missing, wait longer.

---

## 2. Connect to the VPS and prepare it

### 2.1 — Connect from your laptop

**[LAPTOP]**
```powershell
ssh root@YOUR_VPS_IP
```

Type `yes` if it asks about the server's fingerprint (only on first
connect). Enter the root password Namecheap gave you. From here on, every
command is **[VPS]** unless labelled otherwise.

### 2.2 — Set the hostname

Mail servers identify themselves by hostname during SMTP conversations.
This needs to match your MX record.

**[VPS]**
```bash
hostnamectl set-hostname mail.your.domain
echo "mail.your.domain" > /etc/mailname
```

> **Important:** the file `/etc/mailname` must contain the full hostname
> `mail.your.domain`, **not** the bare domain `your.domain`. Postfix uses this
> as `myorigin`. If `myorigin` matches your virtual mail domain, Postfix
> ends up in a rewrite loop and bounces every incoming message with
> "User unknown in virtual alias table." Don't skip this detail.

### 2.3 — Update Ubuntu

Pulls in the latest security patches. Safe; can take a few minutes.

**[VPS]**
```bash
apt update && apt -y upgrade
```

### 2.4 — Open the firewall

Ubuntu ships with the UFW firewall blocking everything by default. Open
the ports our services use:

**[VPS]**
```bash
apt -y install ufw
ufw allow 22/tcp        # SSH (so you don't lock yourself out)
ufw allow 25/tcp        # SMTP — incoming mail
ufw allow 80/tcp        # HTTP — needed briefly by Let's Encrypt
ufw allow 143/tcp       # IMAP with STARTTLS
ufw allow 993/tcp       # IMAP over SSL
ufw --force enable
ufw status
```

You should see all five ports listed as `ALLOW`.

---

## 3. Get a TLS certificate

A TLS certificate proves your server is really `mail.your.domain` and lets
your email app encrypt the connection. Let's Encrypt issues these for
free.

**[VPS]**
```bash
apt -y install certbot
certbot certonly --standalone -d mail.your.domain \
    --non-interactive --agree-tos -m you@somewhere.com
```

Replace `you@somewhere.com` with your real email — Let's Encrypt uses it
to warn you if the cert is about to expire.

What the flags mean:

- `certonly` = just get the certificate; don't try to configure a web server.
- `--standalone` = certbot temporarily runs its own tiny web server on
  port 80 to prove to Let's Encrypt that you control the domain. (This is
  why we opened port 80 in the firewall.)
- `-d mail.your.domain` = the hostname the cert is for.

Verify the cert files exist:

**[VPS]**
```bash
ls -l /etc/letsencrypt/live/mail.your.domain/
```

You should see `fullchain.pem` and `privkey.pem`. If the folder doesn't
exist, certbot failed — most commonly because DNS for `mail.your.domain`
hasn't propagated yet. Re-check step 1 and try again.

---

## 4. Create the mailbox user

Every email that arrives lands in a Linux user's home directory. We'll
create a user called `sky-user` for that purpose.

**[VPS]**
```bash
adduser --disabled-password --gecos "" sky-user
passwd sky-user
```

- `--disabled-password` skips creating a Linux login password — this user
  is never going to log in via SSH, only via IMAP.
- `--gecos ""` skips the "Full name / Phone number / …" prompts.
- `passwd` asks twice. **The cursor doesn't move while you type — that's
  normal Linux behaviour.** Save this password in your password manager;
  it's what your email app will use.

> **Don't use `mail` as the username** — it collides with a system group
> of the same name and causes obscure permission errors.

Confirm:

**[VPS]**
```bash
ls -ld /home/sky-user
```

You should see something like
`drwxr-x--- … sky-user sky-user … /home/sky-user`.

---

## 5. Install and configure Postfix

Postfix is the program that listens on port 25 and accepts incoming email
from the internet. This is the longest section, so we'll go slowly.

### 5.1 — Install Postfix

**[VPS]**
```bash
DEBIAN_FRONTEND=noninteractive apt -y install postfix postfix-pcre
```

The `DEBIAN_FRONTEND=noninteractive` part stops a blue setup wizard from
popping up. If you see one anyway, pick **Internet Site** and accept any
defaults — we overwrite everything next.

### 5.2 — Back up the default config

So you can roll back if anything goes wrong:

**[VPS]**
```bash
cp /etc/postfix/main.cf /etc/postfix/main.cf.orig
```

### 5.3 — Write the new config

The block below is one big command. Copy the whole thing — from `cat >`
all the way through `EOF` — and paste it into your VPS terminal. It writes
the config file in one go.

**[VPS]**
```bash
cat > /etc/postfix/main.cf <<'EOF'
smtpd_banner = $myhostname ESMTP
biff = no
append_dot_mydomain = no
readme_directory = no
compatibility_level = 3.6

# TLS for incoming
smtpd_tls_cert_file = /etc/letsencrypt/live/mail.your.domain/fullchain.pem
smtpd_tls_key_file  = /etc/letsencrypt/live/mail.your.domain/privkey.pem
smtpd_tls_security_level = may
smtpd_tls_loglevel = 1

# Identity
myhostname = mail.your.domain
myorigin = /etc/mailname
mydestination = $myhostname, localhost
mynetworks = 127.0.0.0/8 [::1]/128
inet_interfaces = all
inet_protocols = all

# Receive-only: no relay
relayhost =
smtpd_relay_restrictions =
    permit_mynetworks
    reject_unauth_destination

# Catch-all: every address @your.domain → sky-user
virtual_alias_domains = your.domain
virtual_alias_maps = hash:/etc/postfix/virtual

# Maildir delivery (Dovecot reads this)
home_mailbox = Maildir/
mailbox_size_limit = 0
recipient_delimiter = +

# Reasonable hardening
smtpd_helo_required = yes
smtpd_recipient_restrictions =
    permit_mynetworks
    reject_unauth_destination
    reject_non_fqdn_recipient
    reject_unknown_recipient_domain
    permit
message_size_limit = 52428800
EOF
```

> **Critical detail:** `mydestination = $myhostname, localhost` includes
> `mail.your.domain` as a local destination. Combined with
> `/etc/mailname = mail.your.domain` from step 2.2, this is what makes
> `sky-user` (the bare username from the catch-all map below) resolve to
> a deliverable local mailbox instead of looping back through the virtual
> table.

### 5.4 — Create the catch-all routing rule

**[VPS]**
```bash
echo "@your.domain    sky-user" > /etc/postfix/virtual
```

> **Critical detail:** the right-hand side must be a **bare username**
> (`sky-user`), **not** `sky-user@your.domain`. If you put the domain on the
> right, Postfix re-resolves the result through the virtual table again,
> can't find an entry, and bounces every message.

### 5.5 — Compile the rule

Postfix reads a binary database file (`virtual.db`), not the text file
you just wrote. `postmap` builds the database:

**[VPS]**
```bash
postmap /etc/postfix/virtual
ls -l /etc/postfix/virtual*
```

You should now see two files: `virtual` and `virtual.db`.

> **Remember:** any time you edit `/etc/postfix/virtual`, re-run
> `postmap /etc/postfix/virtual` or your changes won't take effect.

### 5.6 — Verify the lookup works

**[VPS]**
```bash
postmap -q "@your.domain" /etc/postfix/virtual
```

This should print exactly `sky-user`. If it prints `sky-user@your.domain`,
re-run step 5.4 — your text file has the wrong content.

### 5.7 — Check the config is valid

**[VPS]**
```bash
postfix check
```

Silent output = config is fine. Any error message will name the offending
line — fix it before continuing.

### 5.8 — Start Postfix

**[VPS]**
```bash
systemctl restart postfix
systemctl status postfix --no-pager | head -5
```

Look for `Active: active (running)` or `Active: active (exited)` — both
are normal on Ubuntu. If it says `failed`, run
`journalctl -u postfix -n 50 --no-pager` to see why.

### 5.9 — Smoke test

**[VPS]**
```bash
ss -tlnp | grep ':25'
echo "QUIT" | nc -w 2 localhost 25
```

You should see Postfix listening on port 25, and a banner like
`220 mail.your.domain ESMTP` followed by `221 2.0.0 Bye`.

> **Don't send a real test email yet** — Dovecot isn't installed, so you
> couldn't read it. Finish step 6 first.

### Adding specific addresses later (optional)

The catch-all means everything already works. If you ever want
`billing@your.domain` to land in a different person's mailbox, edit
`/etc/postfix/virtual` so it looks like:

```
billing@your.domain    other-user
support@your.domain    other-user
@your.domain           sky-user
```

Specific addresses must come **above** the wildcard `@your.domain` line —
Postfix uses the first match. Re-run:

**[VPS]**
```bash
postmap /etc/postfix/virtual
systemctl reload postfix
```

(Reload, not restart, is enough when only the routing map changes.)

---

## 6. Install and configure Dovecot

Dovecot is the program that lets your email app log in over IMAP and read
the mail Postfix has stored.

### 6.1 — Install

**[VPS]**
```bash
apt -y install dovecot-core dovecot-imapd
```

### 6.2 — Set the protocol and listen address

This goes in the main `dovecot.conf`. Use `cat >>` to append (don't
overwrite — the default file has lots of useful `!include` directives):

**[VPS]**
```bash
cat >> /etc/dovecot/dovecot.conf <<'EOF'

# Receive-only mail server settings
protocols = imap
listen = *, ::
EOF
```

> **Don't try to add a block like `protocol imap { ... }`.** That syntax
> is for protocol-specific overrides, not the simple `protocols = imap`
> setting we want here. Mixing them up causes a "Garbage after '{'" error
> at startup.

### 6.3 — Set the mail location

**[VPS]**
```bash
sed -i 's|^#\?mail_location = .*|mail_location = maildir:~/Maildir|' /etc/dovecot/conf.d/10-mail.conf
grep "^mail_location" /etc/dovecot/conf.d/10-mail.conf
```

The `grep` should print: `mail_location = maildir:~/Maildir`.

### 6.4 — Lock down authentication

**[VPS]**
```bash
sed -i 's|^#\?disable_plaintext_auth = .*|disable_plaintext_auth = yes|' /etc/dovecot/conf.d/10-auth.conf
sed -i 's|^#\?auth_mechanisms = .*|auth_mechanisms = plain login|' /etc/dovecot/conf.d/10-auth.conf
grep -E "^(disable_plaintext_auth|auth_mechanisms)" /etc/dovecot/conf.d/10-auth.conf
```

The `grep` should print:
```
disable_plaintext_auth = yes
auth_mechanisms = plain login
```

### 6.5 — Point Dovecot at the Let's Encrypt cert

Dovecot ships with a self-signed cert by default. We replace those paths
with the real Let's Encrypt files:

**[VPS]**
```bash
sed -i 's|^ssl = .*|ssl = required|' /etc/dovecot/conf.d/10-ssl.conf
sed -i 's|^ssl_cert = .*|ssl_cert = </etc/letsencrypt/live/mail.your.domain/fullchain.pem|' /etc/dovecot/conf.d/10-ssl.conf
sed -i 's|^ssl_key = .*|ssl_key = </etc/letsencrypt/live/mail.your.domain/privkey.pem|' /etc/dovecot/conf.d/10-ssl.conf
sed -i 's|^#\?ssl_min_protocol = .*|ssl_min_protocol = TLSv1.2|' /etc/dovecot/conf.d/10-ssl.conf
grep -E "^(ssl|ssl_cert|ssl_key|ssl_min_protocol)" /etc/dovecot/conf.d/10-ssl.conf
```

The `grep` should print exactly:
```
ssl = required
ssl_cert = </etc/letsencrypt/live/mail.your.domain/fullchain.pem
ssl_key = </etc/letsencrypt/live/mail.your.domain/privkey.pem
ssl_min_protocol = TLSv1.2
```

The leading `<` is required — it's Dovecot's syntax for "read the value
from this file."

> **Critical detail:** Dovecot's default `10-ssl.conf` has `ssl_cert` and
> `ssl_key` lines pointing to `/etc/dovecot/private/dovecot.pem` (the
> self-signed cert). If you don't replace those, your email app will warn
> about an untrusted certificate every time it connects. The `sed`
> commands above handle this automatically.

### 6.6 — Start Dovecot

**[VPS]**
```bash
systemctl restart dovecot
systemctl status dovecot --no-pager | head -5
```

You want `Active: active (running)`. If it failed, run
`journalctl -xeu dovecot -n 30 --no-pager` and read the first error.

### 6.7 — Verify the cert being served is the real one

**[VPS]**
```bash
echo | openssl s_client -connect mail.your.domain:993 -servername mail.your.domain 2>/dev/null | openssl x509 -noout -issuer -subject -dates
```

You want:
```
issuer=C = US, O = Let's Encrypt, CN = ...
subject=CN = mail.your.domain
notAfter=...about 90 days from today...
```

If `issuer` mentions Dovecot or the dates show a 10-year validity, the
self-signed cert is still being served — re-check step 6.5 and restart
Dovecot.

---

## 7. Make the certificate auto-renew

Let's Encrypt certificates expire every 90 days. Certbot already runs a
timer that renews them, but Postfix and Dovecot won't pick up the new
cert unless we tell them to. This script does that automatically after
each renewal.

**[VPS]**
```bash
mkdir -p /etc/letsencrypt/renewal-hooks/deploy
cat > /etc/letsencrypt/renewal-hooks/deploy/reload-mail.sh <<'EOF'
#!/bin/bash
systemctl reload postfix
systemctl reload dovecot
EOF
chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-mail.sh
```

That's it. No further action — Ubuntu's pre-installed certbot timer
handles the rest.

---

## 8. Test that real mail arrives

### 8.1 — From your laptop, double-check the basics

**[LAPTOP]**
```powershell
Resolve-DnsName -Type MX your.domain -Server 8.8.8.8
Test-NetConnection mail.your.domain -Port 25
Test-NetConnection mail.your.domain -Port 993
```

The MX query should return `mail.your.domain`. Both `Test-NetConnection`s
should report `TcpTestSucceeded : True`.

### 8.2 — Watch the log live

In your SSH session, leave this running so you can see what happens in
real time:

**[VPS]**
```bash
tail -f /var/log/mail.log
```

### 8.3 — Send yourself a test email

From any external account (Gmail, Proton, your work email), send a
message to:

```
literallyanything@your.domain
```

Within a few seconds you should see the `tail -f` window print lines
ending with:

```
status=sent (delivered to maildir)
```

That's the proof end-to-end delivery works. Press `Ctrl+C` to stop the
tail.

If instead you see `status=bounced`, the message will tell you why —
re-check whichever step matches the error.

### 8.4 — Confirm the file is there

**[VPS]**
```bash
ls /home/sky-user/Maildir/new/
```

Each file is one delivered email.

### 8.5 — Read mail with an email app

In Thunderbird / Apple Mail / your phone's mail app, add a new account:

- **Account type:** IMAP
- **Server / hostname:** `mail.your.domain`
- **Port:** `993` with SSL/TLS (preferred), or `143` with STARTTLS
- **Username:** `sky-user`
- **Password:** the one you set in step 4
- **Outgoing (SMTP) server:** leave blank or disabled — this server
  doesn't send

The catch-all means every address looks like a different recipient to
your inbox. Most clients show all of them in the same folder.

### 8.6 — Read mail directly on the VPS (optional)

If you want to peek at messages without an email app:

**[VPS]**
```bash
apt -y install mutt
sudo -u sky-user mutt -f /home/sky-user/Maildir/
```

Inside mutt: arrows to navigate, `Enter` to read, `q` to quit, `?` for
help.

---

## Troubleshooting

### "I sent a test email and nothing arrived"

Check Postfix's log on the VPS:

**[VPS]**
```bash
journalctl -u postfix --since "30 min ago" --no-pager
tail -50 /var/log/mail.log
```

Common patterns and what they mean:

| Log says | Meaning | Fix |
| --- | --- | --- |
| `status=sent (delivered to maildir)` | It worked | Check `ls /home/sky-user/Maildir/new/` |
| `status=bounced (User unknown in virtual alias table)` | Virtual map is broken | Re-check steps 2.2, 5.3, 5.4 — particularly that `/etc/mailname` is `mail.your.domain` and `mydestination` includes `$myhostname` |
| `status=deferred` | Temporarily stuck — log will say why | Read the message; usually transient |
| No entries about your test email at all | Mail isn't reaching the server | DNS or port-25 problem (see below) |

### Empty log — no entries at all after sending a test

Means mail isn't reaching Postfix. Check:

**[VPS]**
```bash
ss -tlnp | grep ':25'
```

If nothing is listening, run `systemctl restart postfix`.

**[LAPTOP]**
```powershell
Test-NetConnection mail.your.domain -Port 25
```

If `TcpTestSucceeded : False`, the firewall or Namecheap is blocking
port 25. Check `ufw status` on the VPS, and contact Namecheap support
if UFW is fine.

### "My email app can't log in / shows a certificate warning"

Check Dovecot's log:

**[VPS]**
```bash
journalctl -u dovecot -n 100 --no-pager
```

Verify the right cert is being served:

**[VPS]**
```bash
echo | openssl s_client -connect mail.your.domain:993 -servername mail.your.domain 2>/dev/null | openssl x509 -noout -issuer
```

Should say `issuer=C = US, O = Let's Encrypt, ...`. If it says Dovecot,
re-run step 6.5 and restart Dovecot.

If you accepted a self-signed cert exception in your email app earlier,
clear it after fixing the server: in Thunderbird, **Settings → Privacy &
Security → Manage Certificates → Servers → delete `mail.your.domain` →
restart Thunderbird**.

### "I want to start over"

Revert Postfix:

**[VPS]**
```bash
cp /etc/postfix/main.cf.orig /etc/postfix/main.cf
systemctl restart postfix
```

Revert Dovecot (reinstall the default configs):

**[VPS]**
```bash
apt -y --reinstall install dovecot-core dovecot-imapd
systemctl restart dovecot
```

---

## Notes and gotchas

- **No DKIM here.** DKIM signs *outgoing* mail. We don't send any, so
  there's nothing to sign. The SPF `-all` record we set is the modern
  "nothing legitimately sends as `@your.domain`" signal.
- **DMARC alignment is `r` (relaxed) intentionally.** Strict alignment
  (`s`) causes your own bounce notifications to fail authentication
  because they're sent from `mail.your.domain` (subdomain) and aren't
  DKIM-signed. Relaxed alignment lets bounces through while still
  rejecting impersonation.
- **Outbound port 25 is blocked by Namecheap on most plans** — that's a
  problem for sending mail, not receiving it. Inbound port 25 is what
  matters here and is open by default.
- **Backups.** Every email lives as a file under
  `/home/sky-user/Maildir/`. If you care about retention, snapshot or
  rsync that directory somewhere else periodically.
- **The PTR record matters more than you'd think.** Until Namecheap sets
  it, expect Gmail to defer or reject incoming mail. It's not a hard
  requirement, but it's the difference between "works perfectly" and
  "sometimes mysteriously doesn't."
- **`/etc/mailname` is the FQDN, not the bare domain.** This trips up
  most catch-all setups. If you ever see "User unknown in virtual alias
  table" with `to=<sky-user@your.domain>` in the log, this is almost
  certainly the cause.
