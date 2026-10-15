# Run your own Istrek server

This guide shows you how to run an Istrek server on a Linux machine you control, and how to connect the Istrek Mac app and your agents to it. It takes about half an hour if you have run a web service before.

Everything you need is on the releases page: https://github.com/souriscloud/istrek-releases/releases

| File | What it is |
| --- | --- |
| `istrek-VERSION-linux-amd64.tar.gz` | The server for Linux on x86-64. It carries its own Erlang runtime. |
| `istrek-VERSION-linux-amd64.tar.gz.sha256` | Its checksum. |
| `LICENSE.md` | The terms. The archive carries a copy as well. |
| `THIRD-PARTY-NOTICES-linux.md` | The licences of the software the server archive carries (Erlang/OTP, Elixir and the libraries). The archive carries a copy as `istrek/THIRD-PARTY-NOTICES.md`. |

`VERSION` stands for the version number, for example `0.9.4`. Only the amd64 build of the server is published.

The Mac app (`Istrek-VERSION.dmg`) and the command-line tool for the Mac (`istrekctl-VERSION.zip`) are not published yet. They need to be signed and notarized by Apple first, and will appear on the same releases page when they are. Until then, the server, its web pages, its API and its MCP endpoint for agents are what you can use. Steps 7 and 8 describe the Mac app and the command-line tool for the time they are published.

## Licence

Istrek is free to use and is not open source. You may run it for yourself or your business. You may not redistribute it, and it comes with no warranty. The terms are in [LICENSE.md](LICENSE.md), shipped with every release.

## What you need

- A Linux server on x86-64. Ubuntu 24.04 is the system this guide was tested on.
- PostgreSQL. Versions 16 (the one Ubuntu 24.04 installs) and 18 were tested. The server needs no extensions.
- Typst, version 0.13 or later (0.15.1 was tested). Istrek uses it to draw invoices and reports as PDF files.
- A domain name, for example `time.example.com`, pointing at the server. The Mac app refuses to talk to a remote server over plain HTTP, so you need HTTPS.
- A reverse proxy that terminates TLS. This guide uses Caddy, which gets certificates by itself.
- Optional: `poppler-utils` (the `pdftotext` program). Only the import of PDF invoice documents uses it.
- Optional: an SMTP account, if people should sign in with an e-mail link.

You need no Erlang, Elixir, Node or Docker on the server.

## 1. Download and verify

Download the archive and its checksum on the server. Replace `0.9.4` with the version you want, everywhere in these commands:

```sh
curl -LO https://github.com/souriscloud/istrek-releases/releases/download/v0.9.4/istrek-0.9.4-linux-amd64.tar.gz
curl -LO https://github.com/souriscloud/istrek-releases/releases/download/v0.9.4/istrek-0.9.4-linux-amd64.tar.gz.sha256
sha256sum -c istrek-0.9.4-linux-amd64.tar.gz.sha256
```

The last command must print `istrek-0.9.4-linux-amd64.tar.gz: OK`. If it prints `FAILED`, download the archive again.

Create a system account and unpack the archive:

```sh
sudo useradd --system --create-home --home-dir /var/lib/istrek --shell /usr/sbin/nologin istrek
sudo mkdir -p /opt/istrek
sudo tar xzf istrek-0.9.4-linux-amd64.tar.gz -C /opt/istrek --strip-components=1
```

The program is now `/opt/istrek/bin/istrek`. It can be owned by root. It writes nothing inside its own folder.

## 2. Install Typst

Typst is published at https://github.com/typst/typst/releases. This puts version 0.15.1 in `/usr/local/bin` and checks it:

```sh
curl -fsSL https://github.com/typst/typst/releases/download/v0.15.1/typst-x86_64-unknown-linux-musl.tar.xz \
  | sudo tar -xJ -C /usr/local/bin --strip-components=1 typst-x86_64-unknown-linux-musl/typst
typst --version
```

It must print 0.13 or later. The fonts that invoices and reports use come with Istrek. You do not install them.

## 3. Create the database

Install PostgreSQL if you do not have it yet, then create a role and a database. The first command makes up a password and keeps it in a shell variable:

```sh
DBPASS="$(openssl rand -hex 24)"
echo "$DBPASS"
sudo -u postgres psql -c "CREATE ROLE istrek LOGIN PASSWORD '$DBPASS'"
sudo -u postgres psql -c "CREATE DATABASE istrek OWNER istrek"
```

Write the password down. You need it in the next step. The role owns the database, which is all it needs. It does not need to be a superuser.

## 4. The environment file

Istrek is configured only through environment variables. Put them in a file that only root and the `istrek` account can read:

```sh
sudo install -d -m 0750 -o root -g istrek /etc/istrek
sudo touch /etc/istrek/istrek.env
sudo chown root:istrek /etc/istrek/istrek.env
sudo chmod 0640 /etc/istrek/istrek.env
```

Generate the two secrets:

```sh
openssl rand -base64 48 | tr -d '\n'; echo
openssl rand -base64 32 | tr -d '\n'; echo
```

The first is for `SECRET_KEY_BASE`, the second for `IMPORT_SECRET_KEY`. Now edit `/etc/istrek/istrek.env`:

```sh
DATABASE_URL=ecto://istrek:DBPASS@127.0.0.1/istrek
DATABASE_SSL=false
SECRET_KEY_BASE=PASTE-THE-FIRST-VALUE
IMPORT_SECRET_KEY=PASTE-THE-SECOND-VALUE
PORT=4000
ISTREK_BIND=127.0.0.1
ISTREK_PUBLIC_URL=https://time.example.com
ISTREK_TRUSTED_PROXIES=1
RELEASE_DISTRIBUTION=none
RELEASE_TMP=/var/lib/istrek/tmp
LANG=C.UTF-8
```

What each variable does:

| Variable | Required | Meaning |
| --- | --- | --- |
| `DATABASE_URL` | yes | `ecto://USER:PASSWORD@HOST/DATABASE`. Use a password without special characters, such as the hex value above. |
| `DATABASE_SSL` | no, default `true` | Set `false` for a database on the same machine without TLS. Leave it `true` for a database you reach over a network. |
| `SECRET_KEY_BASE` | yes | At least 64 characters. It signs the forms of the web pages and seals the single sign-on cookie. |
| `IMPORT_SECRET_KEY` | yes | At least 32 characters. It seals the credentials that import sources and transfers store. Never change it (see "Backups"). |
| `PORT` | no, default `4000` | The port the server listens on. |
| `ISTREK_BIND` | no, default `0.0.0.0` | The address the server listens on. Behind a proxy on the same machine use `127.0.0.1`, so nobody can reach the server around the proxy. |
| `ISTREK_PUBLIC_URL` | no, but you want it | The address people open in a browser. HTTPS, with no path: give Istrek a host of its own. It turns on the sign-in page and the device sign-in the Mac app uses. |
| `ISTREK_TRUSTED_PROXIES` | yes with an HTTPS public address | How many proxies sit in front of the server. Use `1` for one Caddy. Use `0` if nothing is in front. |
| `RELEASE_DISTRIBUTION` | recommended | `none` for a single machine. Without it the Erlang runtime also starts a helper process and listens for other nodes. |
| `RELEASE_TMP` | recommended | A folder the service may write to. Create it as shown below. |

Create the folder for `RELEASE_TMP`:

```sh
sudo install -d -m 0700 -o istrek -g istrek /var/lib/istrek/tmp
```

PDF files are drawn in temporary folders under the system temporary directory, which the systemd unit below keeps private to the service.

### Signing in

Without `ISTREK_PUBLIC_URL`, the only way to connect is an owner token (see step 7). With it, the server also serves a sign-in page at `/signin` and an account page at `/account`, and the Mac app can connect through your browser without a token. Both ways work at the same time.

The sign-in page works by e-mail link or, if you set it up, by an OpenID Connect provider. There are no passwords. Add one or both of these blocks.

E-mail through an SMTP server:

```sh
ISTREK_SMTP_HOST=smtp.example.com
ISTREK_SMTP_PORT=587
ISTREK_SMTP_USER=mailer@example.com
ISTREK_SMTP_PASSWORD=PASTE-THE-PASSWORD
ISTREK_SMTP_TLS=starttls
ISTREK_MAIL_FROM="Istrek <no-reply@example.com>"
```

`ISTREK_MAIL_FROM` is required as soon as `ISTREK_SMTP_HOST` is set. The port defaults to 587, and `ISTREK_SMTP_TLS` to `starttls` (`tls` on port 465, `none` only for a relay on the same machine). Without a user, the relay is used without authentication. Without `ISTREK_SMTP_HOST`, no e-mail is sent. Ask your mail provider how to set up SPF and DKIM for the sender address, or the links land in spam.

OpenID Connect (single sign-on):

```sh
ISTREK_OIDC_ISSUER=https://id.example.com
ISTREK_OIDC_CLIENT_ID=istrek
ISTREK_OIDC_CLIENT_SECRET=PASTE-THE-SECRET
ISTREK_OIDC_LABEL="Continue with Example ID"
```

All three of issuer, client id and client secret are needed together, or none. At your provider, register this redirect address, with your own host:

```
https://time.example.com/signin/oidc/callback
```

The provider must say `email_verified: true` for the address, and only for addresses it really verified. Istrek matches people by e-mail address alone. If the provider is lenient about this, anyone could enter another person's account, so use the e-mail link instead.

Who may create an account is set by `ISTREK_SIGNUP`:

| Value | Meaning |
| --- | --- |
| `closed` (default) | Nobody can create an account by signing in. Only accounts you create on the server exist. |
| `invite` | A new address needs an invite code, which you create on the server. |
| `open` | Anyone who can receive the sign-in e-mail gets an account. |

For a server that you run for yourself, leave it `closed`. `ISTREK_OIDC_SIGNUP` (`follow` by default, or `open`) says what happens to a new address that the provider let through. With `open` everyone the provider lets through gets an account, so restrict who may use the application at the provider.

### Hosted mode and billing

Leave both off. Hosted mode (`ISTREK_HOSTED=1`) is for a server that holds accounts of people who do not know each other. It turns off watched folders and changes what the landing page shows. Billing (`ISTREK_BILLING`) charges accounts through Stripe and is only meant for a service that sells access. On a server you run for yourself or your team, an unset `ISTREK_BILLING` means every account has full access and the app shows no plan.

### Other optional settings

| Variable | Meaning |
| --- | --- |
| `ISTREK_UPDATE_URL` | An HTTPS page where people get a newer Mac app. The server tells the app about it when the app is too old. |
| `ISTREK_TYPST_CONCURRENCY` | How many PDF renderings may run at once. The default is the number of CPU cores, at least 2. |
| `ISTREK_TYPST_QUEUE_MS` | How long a rendering waits for a free slot before the request answers 503. The default is 5000. |
| `ISTREK_TURNSTILE_SITEKEY`, `ISTREK_TURNSTILE_SECRET` | A [Cloudflare Turnstile](https://developers.cloudflare.com/turnstile/) widget's sitekey and secret key. Set both, and the e-mail form of the sign-in page checks that the sender is not a script before it sends a link, which keeps scripts from creating accounts and flooding mail. Create the widget in Cloudflare's dashboard in Managed mode, with your public host as its hostname; the free plan is enough. The sign-in page then loads Cloudflare's script and says so. If Cloudflare cannot be reached, the form refuses to send links until it can, so single sign-on or `Istrek.Release.signin_link` is then the way in. Leave both unset for no check. |

## 5. Run it as a service

Create `/etc/systemd/system/istrek.service`:

```ini
[Unit]
Description=Istrek server
After=network-online.target postgresql.service
Wants=network-online.target

[Service]
User=istrek
Group=istrek
EnvironmentFile=/etc/istrek/istrek.env
Environment=PATH=/usr/local/bin:/usr/bin:/bin
ExecStart=/opt/istrek/bin/istrek start
Restart=on-failure
RestartSec=5
PrivateTmp=true
ProtectSystem=strict
ReadWritePaths=/var/lib/istrek
NoNewPrivileges=true

[Install]
WantedBy=multi-user.target
```

`typst` has to be on the service's `PATH`, which the `Environment=PATH=` line takes care of. The service may write only to `/var/lib/istrek` and to a temporary folder of its own. Do not start it yet. First prepare the database.

### A helper for one-off commands

Preparing the database, creating the first credential and a few other tasks are one-off commands that need the same settings as the service. Put a small helper in `/usr/local/sbin/istrek-eval` that loads the environment file and runs one command:

```sh
sudo tee /usr/local/sbin/istrek-eval >/dev/null <<'EOF'
#!/bin/sh
# Runs one Istrek command with the settings of the service.
set -e
cd /var/lib/istrek
set -a
. /etc/istrek/istrek.env
set +a
exec /opt/istrek/bin/istrek eval "$1"
EOF
sudo chmod 0755 /usr/local/sbin/istrek-eval
```

Run it as the `istrek` account, never as root. The `cd` line matters: the runtime stops at once with `Kernel pid terminated` when it starts in a folder the account cannot read, such as your own home folder.

### Migrate the database

```sh
sudo -u istrek istrek-eval 'Istrek.Release.migrate()'
```

If a required variable is missing or too short, the command stops and names it (see "Troubleshooting").

### Create the first owner credential

```sh
sudo -u istrek istrek-eval 'Istrek.Release.bootstrap("Personal")'
```

After a few log lines it prints one line of JSON with the `account_id` and a `token`. The token starts with `ist_`. It is shown once, and the database stores only a digest of it. Copy it to a safe place. Each run of this command creates a new, separate account, so run it once.

If you set `ISTREK_PUBLIC_URL`, also attach your e-mail address to that account. Then you can sign in at your own server's web pages with a link. Replace `ACCOUNT_ID` with the `account_id` from the previous output:

```sh
sudo -u istrek istrek-eval 'Istrek.Release.attach_user("ACCOUNT_ID", "you@example.com")'
```

An account has one user, and an address belongs to one account.

If the server cannot send e-mail (no SMTP), print a sign-in link on the server instead. It works once, for 15 minutes, so treat it like a password:

```sh
sudo -u istrek istrek-eval 'Istrek.Release.signin_link("you@example.com")'
```

With `ISTREK_SIGNUP=invite`, create an invite code the same way:

```sh
sudo -u istrek istrek-eval 'Istrek.Release.invite(expires_in_days: 14, note: "for Pat")'
```

The code is printed once.

### Start the service

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now istrek
curl -s http://127.0.0.1:4000/health
```

`/health` answers with JSON that includes the server's `version`. If it does not answer, read the log with `journalctl -u istrek -e`.

Check the credential as well. Put the token from `bootstrap` in place of `ist_PASTE-THE-TOKEN`:

```sh
curl -s -H "Authorization: Bearer ist_PASTE-THE-TOKEN" http://127.0.0.1:4000/api/v1/me
```

The answer names the account (`"account_name":"Personal"`) and the role `owner`.

## 6. Put Caddy in front

Install Caddy from its own instructions (https://caddyserver.com/docs/install), then put this in `/etc/caddy/Caddyfile`:

```
time.example.com {
	reverse_proxy 127.0.0.1:4000
}
```

Reload it:

```sh
sudo systemctl reload caddy
```

This is enough because of what Caddy does by default:

- It gets and renews a certificate for `time.example.com`. Ports 80 and 443 must be open and the domain must point at the server.
- It passes WebSocket connections through, which the app uses for live updates. It also passes `/mcp`, the endpoint for agents.
- It sets no response timeout. Istrek needs at least 45 seconds, because drawing a large PDF can take a while. If you use another proxy, set its response timeout to 45 seconds or more.
- It adds the client address to `X-Forwarded-For`, which is why `ISTREK_TRUSTED_PROXIES=1` is right.
- It writes no access log unless you add a `log` directive. Keep it that way. If you must log requests, never log the `Authorization` and `Cookie` headers, the `x-istrek-import-secret` header, or the query string of `/signin/confirm`, which carries the token of a sign-in link.

Open only ports 80 and 443 in the firewall. Port 4000 and the database port must not be reachable from outside. With `ISTREK_BIND=127.0.0.1`, the server listens on the local machine only.

Check it from your own computer:

```sh
curl https://time.example.com/health
```

## 7. Connect the Mac app

The Mac app is not published yet (see the start of this guide). When it is, download `Istrek-VERSION.dmg` from the releases page, open it and drag Istrek to Applications. The first time you open the app with no connection, a sheet asks where your time entries live. Choose **My own server**. There are three ways to connect.

**With an owner token.** Type the server address (`https://time.example.com`) in **Server**, type the token that `bootstrap` printed in **Owner token**, and choose **Connect**.

**With a connection file.** Drop a connection file on the sheet, or choose **Choose file…**. A connection file is a small JSON file:

```json
{"url":"https://time.example.com","token":"ist_PASTE-THE-TOKEN"}
```

Keep it private (`chmod 600`), because the token gives full access to the account.

**With the sign-in page.** If you set `ISTREK_PUBLIC_URL`, a **Sign in with the browser…** button appears once you have typed the address. It needs no token. Istrek shows a short code and opens your server's sign-in page. Sign in with your e-mail address, check that the page shows the same code, and choose **Approve**. The Mac connects within a few seconds. A code expires after 10 minutes.

A server that is not on your own Mac must use HTTPS. The app refuses plain HTTP, except for `127.0.0.1`, `localhost` and `[::1]`.

Under Settings, Connection, the app shows the server and the address you are signed in with. **Manage account…** there opens your server's account page, where you can see and revoke every connected Mac and agent. **Rotate credential…** replaces the owner token.

## 8. Connect the command-line tool

The command-line tool is not published yet either. When it is, unzip `istrekctl-VERSION.zip` and put `istrekctl` in a folder on your `PATH`. It reads the same connection file:

```sh
export ISTREK_CONNECTION="$HOME/.config/istrek/connection.json"
istrekctl me
```

If your server has a sign-in page, `istrekctl` can create that file for you through the browser:

```sh
istrekctl login https://time.example.com
```

Always give your server's address. Without one, and without `ISTREK_URL`, `istrekctl login` signs in to the hosted Istrek service at `https://istrek.souris.cloud` instead. Its first line names the server it signs in to.

It prints an address and a code, waits for you to approve it, and writes the connection file (mode 0600, by default `~/.config/istrek/connection.json`). You can also set `ISTREK_URL` and `ISTREK_TOKEN` instead of a file. `istrekctl version` shows its own version and, when a connection is set, the server's.

## 9. Connect agents

An agent is an assistant, for example an MCP-capable AI client, that drafts time entries and invoices. It can read and propose. You approve. Agents never get your owner token. Create a limited credential for each agent with the owner token:

```sh
curl -s -H "Authorization: Bearer ist_PASTE-THE-OWNER-TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"Assistant","role":"agent"}' \
  https://time.example.com/api/v1/tokens
```

The answer carries the new credential in `token`, shown once. The roles are `agent`, `reader` (read-only) and `importer`. Once `istrekctl` is published, `istrekctl token "Assistant" ~/.config/istrek/agents/assistant.json` does the same and writes a connection file for the agent.

Two ways to connect a client:

- Over HTTP. Point the client at `https://time.example.com/mcp` and send `Authorization: Bearer YOUR_AGENT_TOKEN`.
- Through a local program, once `istrekctl` is published. Register `istrekctl` with the argument `mcp` and the path of the agent's connection file:

```json
{
  "mcpServers": {
    "istrek": {
      "command": "/absolute/path/to/istrekctl",
      "args": ["mcp"],
      "env": {"ISTREK_CONNECTION": "/absolute/path/to/assistant.json"}
    }
  }
}
```

The client lists the tools and what each one does when it connects. An `agent` credential can read and propose drafts, but it cannot approve a time entry, issue an invoice or record a payment.

## 10. Backups

Back up three things:

1. **The PostgreSQL database.** It holds everything, including every stored PDF. Nothing else is stored on disk. For example, a nightly dump:

   ```sh
   sudo install -d -m 0700 -o postgres -g postgres /var/backups/istrek
   sudo -u postgres pg_dump --format=custom --file=/var/backups/istrek/istrek-$(date +%Y%m%d).dump istrek
   ```

   The first line creates a folder the `postgres` account may write to. Run only the second line each night. Copy the dump to another machine. Test a restore into an empty database before you rely on it:

   ```sh
   sudo -u postgres createdb istrek_restore_test
   sudo -u postgres pg_restore --no-owner --dbname=istrek_restore_test /var/backups/istrek/istrek-20260101.dump
   sudo -u postgres dropdb istrek_restore_test
   ```

2. **`IMPORT_SECRET_KEY`.** It is not in the database. Keep a copy of `/etc/istrek/istrek.env` somewhere safe, apart from the dump. If you lose the key, the server keeps running, but stored import credentials cannot be read, and each import source asks you to enter its credentials again.
3. **`SECRET_KEY_BASE`** is in the same file. If you change it, people have to sign in again.

To move to a new machine, restore the dump with `pg_restore --no-owner` into an empty database, install the release, use the same environment file, and run the migrate command. Account identities and tokens are kept, so the Mac app keeps working once the address points to the new machine.

## 11. Upgrading

Update the server first. The Mac app and `istrekctl` can follow later: a server serves older apps, and an app that is too old shows "Update Istrek to continue" without losing the changes it has queued. An app that is newer than its server says so, stops syncing and carries on once you update the server.

1. Read the release notes of every version you skip, in particular "Before you update".
2. Back up the database.
3. Download and check the new archive as in step 1, and keep the old one. Unpack the new one over `/opt/istrek` (empty the folder first):

   ```sh
   sudo systemctl stop istrek
   sudo rm -rf /opt/istrek
   sudo mkdir -p /opt/istrek
   sudo tar xzf istrek-NEWVERSION-linux-amd64.tar.gz -C /opt/istrek --strip-components=1
   ```

4. Migrate and start:

   ```sh
   sudo -u istrek istrek-eval 'Istrek.Release.migrate()'
   sudo systemctl start istrek
   curl -s http://127.0.0.1:4000/health
   ```

   `/health` now shows the new `version`.

To go back, stop the server, unpack the previous archive and start it. A release leaves a database that the previous release still runs on, unless its release notes say otherwise.

## 12. Troubleshooting

**The server does not start, and the log names a variable.** The release checks its settings at the start and says what is wrong. The messages you may meet:

| Message begins | What to do |
| --- | --- |
| `SECRET_KEY_BASE is missing or shorter than 64 bytes` | Set it, for example from `openssl rand -base64 48`. |
| `IMPORT_SECRET_KEY is missing or shorter than 32 characters` | Set it to a random value of at least 32 characters. |
| `ISTREK_TRUSTED_PROXIES is not set` | Set it to the number of proxies in front of the server (`1` for Caddy, `0` for none). |
| `ISTREK_PUBLIC_URL must be an https:// address` | Use HTTPS and no path, for example `https://time.example.com`. |
| `ISTREK_PUBLIC_URL must not have a path` | Give Istrek a host of its own. |
| `ISTREK_MAIL_FROM must be the sender's address` | Set `ISTREK_MAIL_FROM` whenever `ISTREK_SMTP_HOST` is set. |
| `single sign-on needs ISTREK_OIDC_ISSUER, ISTREK_OIDC_CLIENT_ID and ISTREK_OIDC_CLIENT_SECRET together` | Set all three, or none. |
| `ISTREK_TURNSTILE_SECRET is not set` or `ISTREK_TURNSTILE_SITEKEY is not set` | Set both Turnstile keys, or neither. |
| `ISTREK_BIND must be an IP address` | Use an address such as `127.0.0.1`. |

A missing `DATABASE_URL` stops the start as well.

**A one-off command stops with `Kernel pid terminated`.** It started in a folder the `istrek` account cannot read. Use the `istrek-eval` helper, which changes to `/var/lib/istrek` first.

**The log shows `Failed open sctp dynamic library: libsctp.so.1`.** This warning is harmless. Istrek does not use SCTP. Installing the `libsctp1` package silences it.

**`/health` does not answer.** Check that the service runs (`systemctl status istrek`), that `PORT` matches the address in the Caddyfile, and that the database accepts the role and password in `DATABASE_URL`.

**The Mac app says the server cannot be reached.** Open `https://time.example.com/health` in a browser. If that works, check the address in the app for typing errors. The address must be HTTPS.

**The app says the server is too old or the app is too old.** Update the older side. See "Upgrading".

**Sign-in e-mails do not arrive.** Check `ISTREK_SMTP_HOST` and the login, then look in the spam folder. Without `ISTREK_SMTP_HOST`, nothing is sent, and the log holds one line without the link. Use `Istrek.Release.signin_link` as shown above.

**The sign-in page says "The check that you are not a robot could not be made at the moment".** The server could not reach `challenges.cloudflare.com` within five seconds, or Cloudflare refused the secret key. The log says which (`Turnstile check could not be made`). Check that the server can make outgoing HTTPS connections and that `ISTREK_TURNSTILE_SECRET` belongs to the sitekey.

**Many people appear to come from one address, or everyone is locked out of sign-in together.** `ISTREK_TRUSTED_PROXIES` is wrong. It must equal the number of proxies in front of the server.

**A PDF fails or takes long.** Check `typst --version` as the `istrek` account (version 0.13 or later), and that the service's `PATH` contains the folder of `typst`. A busy renderer answers 503; raise `ISTREK_TYPST_CONCURRENCY` if the machine has spare cores.

## Security notes

- Public traffic reaches only the HTTPS proxy. Firewall the application and database ports.
- The API has no cookie sign-in and no browser access from other sites. Cookies exist only for the web pages.
- Credentials can be revoked, and they do not expire by themselves. A credential issued through the browser approval is a full owner credential, so approve only a request that you started.
- A server you run sends nothing to Souris.CLOUD. The Mac app looks for updates in the releases repository on GitHub, and contacts the hosted service only if you connect it to that service.
- Istrek has not been audited or penetration-tested by an outside party. Keep the system and the database up to date.
