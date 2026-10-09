# RHEL 10 Rootless Caddy Reverse Proxy Setup

| **Validation Attribute** | **Validation Details**         |
| ------------------------ | ------------------------------ |
| Target Platform          | Red Hat Enterprise Linux 10.x  |
| Validated Platform       | AlmaLinux 10.2 (Lavender Lion) |
| Validation Date          | 2026-10-09                     |

---

## 1. Scope

Deploy Caddy as a rootless Podman container on a RHEL production server. This runbook will result in a secure and isolated reverse proxy configuration that is ready to route traffic to application services.

---

## 2. Prerequisites

- [ ] **Execute the `RHEL 10 Initial Server Setup` runbook.**
- [ ] **Administrative access with `sudo` privileges.**
- [ ] **Network access to package repositories.**
- [ ] **Network access to `docker.io`.**
- [ ] **Podman is installed.**

> [!IMPORTANT]
> Complete all prerequisite runbooks before proceeding. This runbook assumes those configurations are already in place. Skipping prerequisite runbooks may cause one or more steps to fail.

---

## 3. Procedure

### 3.1 Create and Configure a Service Account

- [ ] **Create the service account:**
```bash
sudo adduser \
--uid 2000 \
--create-home \
--home-dir /home/caddy \
--shell /sbin/nologin \
--user-group \
caddy
```

- [ ] **Lock the password for the service account:**
```bash
sudo passwd -l caddy
```

- [ ] **Enable lingering for the service account:**
```bash
sudo loginctl enable-linger caddy
```

- [ ] **Confirm that lingering is enabled for the service account:**
```bash
sudo loginctl show-user caddy -p Linger
```

- [ ] **Create the Quadlet directory for the service account:**
```bash
sudo install -d -o root -g root -m 0755 /etc/containers/systemd/users/2000
```

> [!NOTE]
> Podman searches `/etc/containers/systemd/users/${UID}` for rootless Quadlet files associated with a specific user. Keeping this directory owned by `root` prevents the service account from modifying its own Quadlet configuration while still allowing Podman to load it for that account.
> [podman-systemd.unit](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html#podman-rootless-unit-search-path)

- [ ] **Confirm that subordinate UID and GID ranges are assigned to the service account:**
```bash
grep '^caddy:' /etc/subuid /etc/subgid
```

> [!NOTE]
> Rootless Podman uses subordinate UID and GID ranges to map container users and groups to unprivileged IDs on the host. Confirm that both ranges are assigned to the service account before continuing. Missing subordinate IDs can prevent rootless containers from operating correctly.
> [Building, running, and managing containers](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/building_running_and_managing_containers/building_running_and_managing_containers)

#### Service Account UID Assignment

The `caddy` service account is assigned a UID of `2000` to establish a predictable numbering scheme for rootless service accounts and make this runbook easier to automate. A specific UID is not required, but the UID used in `/etc/containers/systemd/users/${UID}` must match the UID assigned to the `caddy` service account.

### 3.2 Create Caddy Directories and Configuration Files

- [ ] **Create the `/var/lib/caddy` directory with appropriate ownership and permissions:**
```bash
sudo install -d -o root -g root -m 0755 /var/lib/caddy
```

- [ ] **Create the `/srv/caddy` directory with appropriate ownership and permissions:**
```bash
sudo install -d -o root -g root -m 0755 /srv/caddy
```

- [ ] **Create the `/etc/caddy` directory with appropriate ownership and permissions:**
```bash
sudo install -d -o root -g root -m 0755 /etc/caddy
```

- [ ] **Create the `/etc/caddy/sites` directory with appropriate ownership and permissions:**
```bash
sudo install -d -o root -g root -m 0755 /etc/caddy/sites
```

- [ ] **Create `/var/lib/caddy/data` directory with appropriate ownership and permissions:**
```bash
sudo install -d -o caddy -g caddy -m 0700 /var/lib/caddy/data
```

- [ ] **Create `/var/lib/caddy/conf` directory with appropriate ownership and permissions:**
```bash
sudo install -d -o caddy -g caddy -m 0700 /var/lib/caddy/conf
```

- [ ] **Create the `/etc/caddy/Caddyfile` with appropriate ownership and permissions:**
```bash
sudo install -o root -g root -m 0644 /dev/null /etc/caddy/Caddyfile
```

- [ ] **Configure Caddy to import site configuration files from the `sites` directory:**
```bash
printf '%s\n' 'import sites/*' | sudo tee /etc/caddy/Caddyfile > /dev/null
```

- [ ] **Confirm that the Caddyfile contains the expected configuration:**
```bash
cat /etc/caddy/Caddyfile
```

- [ ] **Create the `caddy.container` Quadlet with appropriate ownership and permissions:**
```bash
sudo install \
-o root \
-g root \
-m 0644 \
/dev/null \
/etc/containers/systemd/users/2000/caddy.container
```

- [ ] **Configure the Caddy Quadlet:**
```bash
printf '%s\n' \
'[Unit]' \
'Description=Caddy' \
'' \
'[Container]' \
'ContainerName=caddy' \
'Image=docker.io/library/caddy:2.11.7-alpine' \
'' \
'PublishPort=8080:80/tcp' \
'PublishPort=8443:443/tcp' \
'PublishPort=8443:443/udp' \
'' \
'Volume=/var/lib/caddy/data:/data:Z' \
'Volume=/var/lib/caddy/conf:/config:Z' \
'Volume=/etc/caddy:/etc/caddy:ro' \
'Volume=/srv/caddy:/srv:ro' \
'' \
'[Service]' \
'Restart=always' \
'' \
'[Install]' \
'WantedBy=default.target' \
| sudo tee /etc/containers/systemd/users/2000/caddy.container > /dev/null
```

- [ ] **Confirm that the Caddy Quadlet contains the expected configuration:**
```bash
cat /etc/containers/systemd/users/2000/caddy.container
```

### 3.3 Configure QUIC UDP Buffers (Optional)

> [!NOTE]
> Caddy uses QUIC for HTTP/3. For better QUIC performance on high-bandwidth connections, the quic-go documentation recommends increasing the maximum UDP send and receive buffer sizes on the host to 7 MiB.
> [Optimizations](https://quic-go.net/docs/quic/optimizations/#udp-buffer-sizes)

- [ ] **Create the QUIC sysctl configuration file with appropriate ownership and permissions:**
```bash
sudo install -o root -g root -m 0644 /dev/null /etc/sysctl.d/90-quic.conf
```

- [ ] **Configure the maximum UDP buffer sizes:**
```bash
printf '%s\n' \
'net.core.rmem_max=7340032' \
'net.core.wmem_max=7340032' \
| sudo tee /etc/sysctl.d/90-quic.conf > /dev/null
```

- [ ] **Apply the new kernel parameters:**
```bash
sudo sysctl -p /etc/sysctl.d/90-quic.conf
```

- [ ] **Confirm that the UDP buffer size configuration is active:**
```bash
sysctl net.core.rmem_max net.core.wmem_max
```

### 3.4 Configure SELinux Contexts

> [!NOTE]
> SELinux enforces access independently of standard file ownership and permissions. Because `/etc/caddy` and `/srv/caddy` are mounted read-only into the Caddy container, they need the `container_share_t` type so the container can access their contents.

- [ ] **Install the `policycoreutils-python-utils` package:**
```bash
sudo dnf install policycoreutils-python-utils -y
```

- [ ] **Define the SELinux context for `/etc/caddy`:**
```bash
sudo semanage fcontext -a -t container_share_t '/etc/caddy(/.*)?'
```

- [ ] **Apply the SELinux context to `/etc/caddy`:**
```bash
sudo restorecon -Rv /etc/caddy
```

- [ ] **Confirm the SELinux context for `/etc/caddy`:**
```bash
ls -Zd /etc/caddy
```

- [ ] **Define the SELinux context for `/srv/caddy`:**
```bash
sudo semanage fcontext -a -t container_share_t '/srv/caddy(/.*)?'
```

- [ ] **Apply the SELinux context to `/srv/caddy`:**
```bash
sudo restorecon -Rv /srv/caddy
```

- [ ] **Confirm the SELinux context for `/srv/caddy`:**
```bash
ls -Zd /srv/caddy
```

### 3.5 Configure Firewall Forwarding

> [!NOTE]
> By default, rootless Podman containers cannot bind to privileged ports. This runbook publishes Caddy on higher host ports and uses `firewalld` forwarding rules to redirect incoming traffic from the standard HTTP and HTTPS ports.

- [ ] **Redirect traffic on port `80/tcp` to port `8080/tcp`:**
```bash
sudo firewall-cmd \
--permanent \
--zone=drop \
--add-forward-port=port=80:proto=tcp:toport=8080
```

- [ ] **Redirect traffic on port `443/tcp` to port `8443/tcp`:**
```bash
sudo firewall-cmd \
--permanent \
--zone=drop \
--add-forward-port=port=443:proto=tcp:toport=8443
```

- [ ] **Redirect traffic on port `443/udp` to port `8443/udp`:**
```bash
sudo firewall-cmd \
--permanent \
--zone=drop \
--add-forward-port=port=443:proto=udp:toport=8443
```

- [ ] **Reload `firewalld` to apply the configuration changes:**
```bash
sudo firewall-cmd --reload
```

- [ ] **Confirm that the forwarding rules are active in the `drop` zone:**
```bash
sudo firewall-cmd --zone=drop --list-forward-ports
```

### 3.6 Start and Verify Caddy

- [ ] **Reload the user service manager:**
```bash
sudo systemctl --user --machine=caddy@.host daemon-reload
```

- [ ] **Start Caddy:**
```bash
sudo systemctl --user --machine=caddy@.host start caddy.service
```

- [ ] **Confirm that Caddy is running:**
```bash
sudo systemctl --user --machine=caddy@.host status caddy.service
```

### 3.7 Configure a Temporary Validation Site

> [!IMPORTANT]
> This section creates a temporary site configuration used only to validate that Caddy is operational. The test covers the HTTP path and confirms that Caddy and the firewall forwarding rules are working as expected. Caddy can manage HTTPS automatically once a valid domain that resolves to the server is configured. Remove the temporary configuration after completing the validation section.

- [ ] **Create the site configuration file with appropriate ownership and permissions:**
```bash
sudo install -o root -g root -m 0644 /dev/null /etc/caddy/sites/validate.caddy
```

- [ ] **Configure the validation path:**
```bash
printf '%s\n' \
'http:// {' \
'  respond "Caddy is operational" 200' \
'}' \
| sudo tee /etc/caddy/sites/validate.caddy > /dev/null
```

> [!TIP]
> Run `cd /etc/caddy` before executing the validate and reload commands so the `caddy` service account can access the current working directory. Running these commands from a restricted directory may result in a `cannot chdir to <directory>: Permission denied` error.

- [ ] **Validate the Caddy configuration:**
```bash
sudo \
-u caddy \
-H env XDG_RUNTIME_DIR=/run/user/2000 \
podman exec caddy \
caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

- [ ] **Reload the Caddy configuration:**
```bash
sudo \
-u caddy \
-H env XDG_RUNTIME_DIR=/run/user/2000 \
podman exec caddy \
caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile
```

## 4. Validation

- [ ] **Confirm `http://<server-ip>` returns `Caddy is operational` from a remote host.**
- [ ] **Confirm `http://<server-ip>:8080` cannot be accessed from a remote host.**

> [!IMPORTANT]
> Run `sudo rm /etc/caddy/sites/validate.caddy` after completing these checks, then validate and reload Caddy using the commands from Section 3.7. Do not leave the validation configuration enabled on a production server.

## 5. Recovery / Rollback

If the Caddy deployment needs to be rolled back, remove the firewall forwarding rules, stop the Caddy service, and remove the Quadlet configuration. The service account, configuration files, and persistent data can remain in place unless Caddy is being permanently removed from the system.

- [ ] **Remove the redirect from port `80/tcp` to port `8080/tcp`:**
```bash
sudo firewall-cmd \
--permanent \
--zone=drop \
--remove-forward-port=port=80:proto=tcp:toport=8080
```

- [ ] **Remove the redirect from port `443/tcp` to port `8443/tcp`:**
```bash
sudo firewall-cmd \
--permanent \
--zone=drop \
--remove-forward-port=port=443:proto=tcp:toport=8443
```

- [ ] **Remove the redirect from port `443/udp` to port `8443/udp`:**
```bash
sudo firewall-cmd \
--permanent \
--zone=drop \
--remove-forward-port=port=443:proto=udp:toport=8443
```

- [ ] **Reload `firewalld` to apply the configuration changes:**
```bash
sudo firewall-cmd --reload
```

- [ ] **Confirm that the forwarding rules have been removed from the `drop` zone:**
```bash
sudo firewall-cmd --zone=drop --list-forward-ports
```

- [ ] **Stop Caddy:**
```bash
sudo systemctl --user --machine=caddy@.host stop caddy.service
```

- [ ] **Confirm that Caddy is not running:**
```bash
sudo systemctl --user --machine=caddy@.host status caddy.service
```

- [ ] **Remove the `caddy.container` Quadlet:**
```bash
sudo rm /etc/containers/systemd/users/2000/caddy.container
```

- [ ] **Reload the user service manager:**
```bash
sudo systemctl --user --machine=caddy@.host daemon-reload
```
