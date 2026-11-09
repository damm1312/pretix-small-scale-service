# pretix small-scale service

Docker Compose stack for [pretix](https://docs.pretix.eu/self-hosting/installation/docker_smallscale/):
PostgreSQL, Redis, one or more pretix replicas (`all` mode, each running `pretix cron` itself) and an nginx
reverse proxy that terminates TLS and load-balances across the replicas.

## First start
```sh
cp .env.example .env        # fill in PRETIX_URL, PRETIX_DOMAIN, POSTGRES_PASSWORD, MAIL_*
# Put your certificate chain and key in nginx/certs/fullchain.pem and nginx/certs/privkey.pem
docker compose config -q    # validate
docker compose up -d postgres redis
docker compose run --rm migrate
docker compose up -d
docker compose ps           # wait for "healthy"
```
Log in at `<PRETIX_URL>/control/` with `admin@localhost` / `admin` and **change the password immediately**.

## Scaling
```sh
docker compose up -d --scale pretix=3
```
nginx re-resolves the `pretix` service name every 10 seconds, so new replicas receive traffic without an nginx
reload. Every replica runs `pretix cron` every `CRON_INTERVAL_SECONDS`. Limit: all replicas share the `pretix_data`
volume; across several hosts it must be shared storage (e.g. NFS).

## Upgrade
Back up first (below), read the pretix release notes, then:
```sh
docker compose pull
docker compose run --rm migrate
docker compose up -d
```
Pin `PRETIX_VERSION` to a release in `.env` for reproducible deploys.

## Backup
```sh
docker compose exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' > pretix-$(date +%F).sql
docker run --rm -v pretix_pretix_data:/data -v "$PWD":/backup alpine tar czf /backup/pretix-data-$(date +%F).tgz -C /data .
```

## Notes
- Postgres and Redis are on an internal network with no published ports; only nginx publishes 80/443.
- Plugins: build an image `FROM pretix/standalone:stable`, `pip3 install` the plugin, run `make production` in `/pretix/src`, and set it as the image in `compose.yaml`.

## Email (bundled Postfix)
The `smtp` service (Postfix + OpenDKIM) is outbound-only and publishes no ports. pretix sends to `smtp:25` on the
private network. Deliverability needs DNS and network setup:

1. Outbound port 25 must be open at your hosting provider. If it is blocked, set `SMTP_RELAYHOST` (and credentials)
   to an upstream relay; the service then acts as a local queue with DKIM signing.
2. Reverse DNS (PTR) of the server IP must match `SMTP_HOSTNAME`, which needs an A record.
3. SPF TXT on `MAIL_DOMAIN`: `v=spf1 ip4:<server-ip> -all`.
4. DKIM: read the public key and publish it as a TXT record at `<selector>._domainkey.<MAIL_DOMAIN>`:
   ```sh
   docker compose exec smtp sh -c 'cat /etc/opendkim/keys/*/*.txt'
   ```
5. DMARC TXT at `_dmarc.<MAIL_DOMAIN>`: start with `v=DMARC1; p=none; rua=mailto:dmarc@<MAIL_DOMAIN>`, tighten to
   `quarantine` once reports are clean.
6. The sender domain must be able to receive bounces (MX record or mailbox).
7. New IPs have no reputation: send low volume at first and check with mail-tester.com.

Test: `docker compose logs smtp` should show `status=sent` after a test mail from the pretix control panel; the received
headers should show `spf=pass`, `dkim=pass`, `dmarc=pass`.

## Local test

> [!WARNING]
> Production deployments require proper certificates as the Let's Encrypt ones.
>
> For that one would need the following prerequisites:
> - A DNS A record for your domain points to the server's public IP.
> - Ports 80 and 443 are open in the firewall.
> - Nothing is listening on port:80. Do this before the first docker compose up, because nginx won't start without certs anyway.

### Self-sign certificate

```sh
openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout nginx/certs/privkey.pem \
  -out nginx/certs/fullchain.pem \
  -subj "/CN=<PRETIX_DOMAIN>" \
  -addext "subjectAltName=DNS:<PRETIX_DOMAIN>"
chmod 600 nginx/certs/privkey.pem
```

- Use the same hostname as PRETIX_DOMAIN in .env. For the local test, set
  PRETIX_DOMAIN=localhost and PRETIX_URL=https://localhost, and use
  /CN=localhost and DNS:localhost in the command.
- Your browser will warn about the certificate. Use curl -k or accept the warning.
