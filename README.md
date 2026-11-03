# Postfix

Postfix SMTP image for a virtual-mail stack. It uses
[`ghcr.io/nforceroh/alpine-base:3.24`](https://github.com/nforceroh/alpine-base)
and s6-overlay to render SQL map templates, configure Postfix at container
startup, and supervise the mail daemon.

## What it provides

- Postfix with MySQL maps, PCRE, SPF policy, OpenDKIM, and `mailx`
- SMTP on `25`, SMTPS on `465`, message submission on `587`, and a trusted
  reinjection listener on `10025`
- Optional Rspamd milter integration
- Dovecot LMTP and SASL integration
- A persistent `/var/spool/postfix` volume

## Required configuration

The initialization services require these variables:

| Variable | Image default | Description |
| --- | --- | --- |
| `MYHOSTNAME` | required | Fully qualified mail server name |
| `MYNETWORKS` | `127.0.0.0/8 10.0.0.0/8 172.16.0.0/16 192.168.0.0/16` | Trusted client networks |
| `PROXY_MAILNAME` | required | Proxy/public mail name configured as `proxy_interfaces` |
| `DOVECOT_HOST` | `10.0.0.101` | Dovecot service host |
| `DOVECOT_LMTP` | `24` | Dovecot LMTP port |
| `DOVECOT_SASL` | `12345` | Dovecot SASL endpoint port |
| `DB_HOST` | `10.0.0.101` | SQL server host for virtual-mail maps |
| `DB_DATABASE` | `mail` | Mail database name |
| `DB_USER` | `root` | SQL map account |
| `DB_PASSWORD` | `CHANGE_THIS_PASSWORD` | SQL map account password; replace using a runtime secret |
| `SSL_CERT` | `/etc/postfix/certs/tls.crt` | Mounted TLS certificate path |
| `SSL_KEY` | `/etc/postfix/certs/tls.key` | Mounted TLS private-key path |

`init-templates` exits with an error unless all four SQL settings are
provided. The `DB_USER=root` image default is only an example; use a
least-privilege SQL account. `init-postconf` also requires `MYHOSTNAME`, `MYNETWORKS`,
`DOVECOT_HOST`, `DOVECOT_LMTP`, `DOVECOT_SASL`, `SSL_CERT`, `SSL_KEY`, and
`PROXY_MAILNAME`. The default network and host values are examples; review
them for your environment.

## Optional configuration

| Variable | Image default | Description |
| --- | --- | --- |
| `MAILNAME` | `mail.example.com` | Declared value; the bundled startup script configures `MYHOSTNAME` instead |
| `RSPAMD_ENABLE` | `false` | Set to `true` to enable the Rspamd milter |
| `RSPAMD_HOST` | `10.0.0.101` | Rspamd host when milter integration is enabled |
| `RSPAMD_PORT` | `11334` | Rspamd milter port |
| `WAITSTART_TIMEOUT` | `1m` | Declared value; not consumed by the bundled Postfix startup scripts |
| `RELAYHOST` | `mail.twc.com` | Upstream SMTP relay; external-forward rewrite requires a relay |
| `RELAY_USERNAME` | unset | Optional upstream relay username |
| `RELAY_PASSWORD` | unset | Optional upstream relay password; set as a secret |
| `POSTMASTER_ALIAS` | derived from `MYHOSTNAME` | Destination for postmaster and root aliases |
| `FILTER_MIME` | `false` | Set to `true` to enable the bundled MIME header checks |
| `EXTERNAL_FORWARD_RECIPIENTS` | empty | Comma-separated recipient addresses for the dedicated forwarding transport |
| `EXTERNAL_FORWARD_ENVELOPE_SENDER` | `forwarded-bounces@martintwingles.com` | Envelope sender used by the forwarding transport |
| `EXTERNAL_FORWARD_TRANSPORT_NAME` | `external-forward` | Name of the generated Postfix transport |
| `EXTERNAL_FORWARD_HEADER_DISPLAY_NAME` | `Forwarded via martintwingles.com` | Display name for the rewritten `From` header |
| `EXTERNAL_FORWARD_HEADER_FROM` | empty | Optional verified address for the rewritten `From` header |

When external forwarding is enabled, `RELAYHOST` must be set: initialization
refuses to configure direct-to-recipient delivery. Configure TLS files and
relay credentials securely; do not bake credentials into an image.
The image's `RELAYHOST` and external-forward sender defaults are examples;
replace them with your provider and verified sender address, or disable the
optional forwarding rewrite.

## Usage

Connect Postfix, Dovecot, the SQL server, and (optionally) Rspamd to the same
private network. The following illustrates the required settings; replace the
example hosts, networks, credentials, and certificate mount with your actual
deployment values:

```sh
docker run -d --name postfix \
  --restart unless-stopped \
  --network mail \
  -p 25:25 -p 465:465 -p 587:587 -p 10025:10025 \
  -v postfix-spool:/var/spool/postfix \
  -v "$PWD/certs:/etc/postfix/certs:ro" \
  -e MYHOSTNAME=mail.example.com \
  -e PROXY_MAILNAME=mail.example.com \
  -e MYNETWORKS='127.0.0.0/8 10.0.0.0/8' \
  -e DB_HOST=mariadb -e DB_DATABASE=mail -e DB_USER=postfix \
  -e DB_PASSWORD='replace-with-a-secret' \
  -e DOVECOT_HOST=dovecot -e DOVECOT_LMTP=24 -e DOVECOT_SASL=12345 \
  ghcr.io/nforceroh/postfix:latest
```

Do not expose port `10025` publicly. It is configured as a trusted
mail-reinjection listener for internal services.

## Local development

```sh
make build   # build ghcr.io/nforceroh/postfix:dev
make test    # build and run the s6/boot smoke tests
make run     # publish the four SMTP ports
```

The local `make run` target is a basic container launch and does not provide
the required database, Dovecot, TLS, or identity settings by itself. CI
publishes through the shared workflow in
[`nforceroh/.github`](https://github.com/nforceroh/.github).
