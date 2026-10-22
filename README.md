# MinIO Deployment on Ubuntu VPS

A practical guide for deploying MinIO Community Edition on an Ubuntu VPS without Docker.

> **Important about MinIO distribution:** MinIO Community Edition is currently distributed as source code through the upstream repository. This project does not include a MinIO binary. Build MinIO from the upstream source on a machine with a compatible Go toolchain, then copy the resulting binary to the server. Upstream source: <https://github.com/minio/minio>. Review the upstream project's current build and licensing instructions before using it.

## What this deployment does

- Runs MinIO as a dedicated, unprivileged Linux service user.
- Stores uploaded objects persistently in `/var/lib/minio/data`.
- Binds MinIO's S3 API and Console to loopback (`127.0.0.1`) so they are not directly exposed to the Internet.
- Uses Nginx as a reverse proxy for the MinIO Console.
- Uses Let's Encrypt and Certbot to enable HTTPS.
- Enables MinIO to start automatically after a reboot.

This example publishes the **MinIO Console** at `https://devopslabminio.duckdns.org`. Replace that example hostname with your own domain everywhere it appears.

## Requirements

- An Ubuntu VPS (Ubuntu 22.04/24.04 or a compatible current release).
- A public IPv4 address.
- Permission to configure the VPS firewall / cloud firewall.
- A domain or subdomain whose DNS A record points to the VPS.
- A build machine with Git and a Go toolchain compatible with the current MinIO source (upstream currently requires Go 1.24 or newer; verify this in the upstream repository).
- SSH access to the VPS.

### Open the required ports

In your VPS provider's cloud firewall and, if enabled, the server firewall, allow inbound:

- TCP `22` — SSH administration.
- TCP `80` — HTTP and Let's Encrypt HTTP validation.
- TCP `443` — HTTPS.

Do **not** open ports `9000` or `9001` to the Internet. In this configuration, MinIO listens on loopback only and Nginx handles public Console traffic.

---

# Part 1 — Deploy using repository files

This path uses the ready-made configuration templates in `config/`. You still need to customize the hostname and set your own strong MinIO password.

## 1. Clone this repository

On your workstation or on the VPS:

```bash
git clone https://github.com/pykalka/DevOps-Lab---MinIO-Deployment-on-Ubuntu-VPS.git
cd minio-deployment
```

Cloning this repository only downloads configuration templates; it does not install MinIO or create its data directory.

## 2. Build MinIO from the upstream source

Do this on a build machine with Git and a compatible Go toolchain installed. Building on a separate machine is recommended for small VPS instances.

```bash
git clone https://github.com/minio/minio.git
cd minio
# For reproducible builds, check out a reviewed upstream tag or commit here.
go version
go build -o minio .
```

Check the upstream README for the current minimum Go version and build guidance before building. The command above builds the checked-out source; it does not download an official prebuilt release.

Copy the binary to your VPS. From the directory containing the built `minio` executable, replace `VPS_IP` with the server's public IP:

```bash
scp ./minio root@VPS_IP:/tmp/minio
```

Then connect to the VPS and install it:

```bash
ssh root@VPS_IP
install -o root -g root -m 0755 /tmp/minio /usr/local/bin/minio
/usr/local/bin/minio --version
```

If you administer the VPS using a non-root account, use `sudo` for the `install` command and copy the file to a location that account can access.

## 3. Create the MinIO service account and persistent data directory

Run on the VPS:

```bash
id minio >/dev/null 2>&1 || useradd --system --user-group --home-dir /var/lib/minio --shell /usr/sbin/nologin minio
mkdir -p /var/lib/minio/data
chown -R minio:minio /var/lib/minio
```

The directory `/var/lib/minio/data` is **not** created by cloning either repository. It must be created on the server. MinIO will store uploaded objects there; do not delete this directory when updating the application.

## 4. Configure MinIO environment variables

Copy the example to MinIO's system environment file:

```bash
install -o root -g root -m 0600 config/minio.env.example /etc/default/minio
nano /etc/default/minio
```

Edit the file and:
1. Replace `devopslabminio.duckdns.org` with your domain.
2. Replace `REPLACE_WITH_A_LONG_RANDOM_PASSWORD` with a unique, strong password.
3. Choose a strong `MINIO_ROOT_USER` if you prefer not to use the example username.

Keep `/etc/default/minio` readable only by root (`0600`). Do not commit it or share its password.

## 5. Install the systemd service

```bash
install -o root -g root -m 0644 config/minio.service /etc/systemd/system/minio.service
nano /etc/systemd/system/minio.service
```

Replace `devopslabminio.duckdns.org` if it appears in the service file. (The hostname is configured in `/etc/default/minio`; this template's service itself normally does not need a hostname.) Then run:

```bash
systemctl daemon-reload
systemctl enable --now minio
systemctl status minio --no-pager
```

The service should show `active (running)`. If it does not, inspect the logs:

```bash
journalctl -u minio -n 100 --no-pager
```

## 6. Create a domain or subdomain and point DNS to the VPS

You can use DuckDNS for a free subdomain, or use a domain you already own.

For DuckDNS:
1. Open <https://www.duckdns.org/> and sign in using one of the supported login providers.
2. Create a subdomain, for example `myproject`.
3. Set its IPv4 address to your VPS public IP, then save/update the record.
4. Use the full hostname, for example `myproject.duckdns.org`, in the configuration below.

For a domain you own, open your DNS provider's control panel and create an **A record** for the chosen hostname pointing to the VPS public IPv4 address. DNS changes can take time to propagate.

Verify the DNS record from the VPS:

```bash
apt-get update
apt-get install -y dnsutils
dig +short myproject.duckdns.org
```

The result should contain your VPS public IPv4 address. Replace the sample hostname in the command with your own.

## 7. Configure Nginx using the repository template

Install Nginx and Certbot:

```bash
apt-get update
apt-get install -y nginx certbot python3-certbot-nginx
systemctl enable --now nginx
```

Copy the Nginx template into place and edit its hostname:

```bash
install -o root -g root -m 0644 config/nginx-minio.conf /etc/nginx/sites-available/minio
nano /etc/nginx/sites-available/minio
```

Replace `devopslabminio.duckdns.org` with your domain. Enable the site and validate the configuration:

```bash
ln -s /etc/nginx/sites-available/minio /etc/nginx/sites-enabled/minio
nginx -t
systemctl reload nginx
```

If the symbolic link already exists, do not create it again. The default Nginx site may remain enabled; Nginx selects this site when the request's hostname matches `server_name`.

## 8. Enable HTTPS with Certbot

Make sure that:
- The domain resolves to this VPS.
- Inbound TCP ports `80` and `443` are allowed by the cloud and server firewalls.
- Nginx configuration passes `nginx -t`.

Then run:

```bash
certbot --nginx -d myproject.duckdns.org
```

Replace the sample hostname. Follow Certbot's prompts, provide an email address, accept the terms, and choose HTTP-to-HTTPS redirection if offered. Certbot obtains the certificate and updates the Nginx configuration.

Open `https://YOUR_DOMAIN` in a browser and sign in with the MinIO root credentials from `/etc/default/minio`.

Check certificate renewal:

```bash
systemctl list-timers --all | grep -i certbot
certbot renew --dry-run
```

## 9. Verify the deployment

Run these on the VPS:

```bash
systemctl is-enabled minio
systemctl is-active minio
nginx -t
curl -fsS http://127.0.0.1:9000/minio/health/live
curl -I https://myproject.duckdns.org
```

Replace the sample hostname in the final command. The MinIO health endpoint should return successfully; the HTTPS request should show a successful HTTP response or a redirect.

Finally, reboot the VPS and verify that:
1. `systemctl is-active minio` returns `active`.
2. The Console opens at `https://YOUR_DOMAIN`.
3. An object uploaded before reboot is still present.

Persistent storage comes from `/var/lib/minio/data`, not from the MinIO source repository.

---

# Part 2 — If you wanna do it manually

This section performs the same setup without copying configuration templates from this repository. It assumes you have already built the MinIO executable from the upstream source and copied it to `/tmp/minio` on the VPS. See Part 1, Step 2 for build guidance.

## 1. Install the MinIO executable

```bash
install -o root -g root -m 0755 /tmp/minio /usr/local/bin/minio
/usr/local/bin/minio --version
```

## 2. Create the service account and data directory

```bash
id minio >/dev/null 2>&1 || useradd --system --user-group --home-dir /var/lib/minio --shell /usr/sbin/nologin minio
mkdir -p /var/lib/minio/data
chown -R minio:minio /var/lib/minio
```

Cloning the MinIO GitHub repository does not create `/var/lib/minio/data`. This is a separate directory on the server that must be created and owned by the `minio` service account.

## 3. Create `/etc/default/minio`

Open the file:

```bash
nano /etc/default/minio
```

Paste the following, then replace the sample domain and placeholder password. Save with `Ctrl+O`, press `Enter`, and exit with `Ctrl+X`.

```ini
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=REPLACE_WITH_A_LONG_RANDOM_PASSWORD
MINIO_VOLUMES="/var/lib/minio/data"
MINIO_OPTS="--address 127.0.0.1:9000 --console-address 127.0.0.1:9001"
MINIO_BROWSER_REDIRECT_URL="https://myproject.duckdns.org"
```

Protect the file because it contains credentials:

```bash
chown root:root /etc/default/minio
chmod 600 /etc/default/minio
```

Do not use the placeholder password in a real deployment, and never commit this file to a public repository.

## 4. Create the systemd service

```bash
nano /etc/systemd/system/minio.service
```

Paste this service definition:

```ini
[Unit]
Description=MinIO Object Storage
Wants=network-online.target
After=network-online.target

[Service]
User=minio
Group=minio
EnvironmentFile=/etc/default/minio
WorkingDirectory=/var/lib/minio
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

Save and exit, then enable and start MinIO:

```bash
systemctl daemon-reload
systemctl enable --now minio
systemctl status minio --no-pager
```

If the service fails, inspect its logs:

```bash
journalctl -u minio -n 100 --no-pager
```

## 5. Register a domain/subdomain and configure DNS

For a free DuckDNS subdomain:
1. Visit <https://www.duckdns.org/> and sign in.
2. Create a subdomain.
3. Set its IPv4 address to the VPS public IP and save/update the record.
4. Replace `myproject.duckdns.org` below with your actual hostname.

For a domain you own, create an A record in your DNS provider's control panel pointing the hostname to the VPS public IPv4 address. Do not publish your DNS-provider token in this repository.

Install a DNS lookup utility and check the result:

```bash
apt-get update
apt-get install -y dnsutils
dig +short myproject.duckdns.org
```

The returned address should match your VPS public IPv4 address.

## 6. Install Nginx and create its configuration

```bash
apt-get update
apt-get install -y nginx certbot python3-certbot-nginx
systemctl enable --now nginx
nano /etc/nginx/sites-available/minio
```

Paste the following Nginx configuration. Replace the sample domain with your own hostname:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name myproject.duckdns.org;

    location / {
        proxy_pass http://127.0.0.1:9001;

        proxy_http_version 1.1;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_read_timeout 3600;
        proxy_send_timeout 3600;
    }
}
```

Save and exit. Enable the site and test the configuration:

```bash
ln -s /etc/nginx/sites-available/minio /etc/nginx/sites-enabled/minio
nginx -t
systemctl reload nginx
```

If the link already exists, skip the `ln -s` command. The default Nginx site may remain enabled.

## 7. Obtain and configure the HTTPS certificate

Ensure DNS is correct and TCP ports `80` and `443` are open in both your cloud firewall and server firewall. Then run:

```bash
certbot --nginx -d devopslabminio.duckdns.org
```

Use your real hostname. Follow the prompts to enter an email address, accept the terms, and select the HTTPS redirect option if offered. Certbot obtains the Let's Encrypt certificate and updates Nginx; do not manually add certificate file paths before Certbot has issued the certificate.

Check the renewal timer and simulate renewal:

```bash
systemctl list-timers --all | grep -i certbot
certbot renew --dry-run
```

## 8. Verify and test persistence

```bash
systemctl is-enabled minio
systemctl is-active minio
nginx -t
curl -fsS http://127.0.0.1:9000/minio/health/live
curl -I https://devopslabminio.duckdns.org
```

Replace the sample hostname. Log in to the Console at `https://YOUR_DOMAIN`, upload a test object, reboot the VPS, and verify that MinIO starts and the object is still there.

## Notes and security

- Keep MinIO root credentials private. Use a unique, strong password.
- Do not commit `/etc/default/minio`, TLS private keys, or DNS tokens.
- Keep ports `9000` and `9001` closed to the public Internet; they are bound to loopback in this setup.
- The current configuration exposes the MinIO Console through Nginx. MinIO's S3 API is also bound to loopback, so external S3 clients cannot connect directly in this configuration. If external S3 API access is required, plan and secure a separate Nginx virtual host for the API (commonly port `9000`) and configure the appropriate public endpoint settings.
- Keep Ubuntu, Nginx, Certbot, and MinIO updated. Review upstream MinIO release/build guidance before rebuilding or upgrading.
