# RHEL 10 Initial Server Setup

| **Validation Attribute** | **Validation Details**         |
| ------------------------ | ------------------------------ |
| Target Platform          | Red Hat Enterprise Linux 10.x  |
| Validated Platform       | AlmaLinux 10.2 (Lavender Lion) |
| Validation Date          | 2026-10-09                     |

---

## 1. Scope

Establish a consistent baseline configuration for a freshly provisioned RHEL production server. This runbook will result in a secure and stable system that is ready to be extended for its intended workload.

---

## 2. Prerequisites

- [ ] **Root access.**
- [ ] **Network access to package repositories.**

---

## 3. Procedure

### 3.1 Update and Reboot the System

- [ ] **Install all available system updates:**
```bash
dnf upgrade -y
```

- [ ] **Reboot the system to complete the update process:**
```bash
systemctl reboot
```

> [!NOTE]
> If you are configuring the server remotely, wait for the system to come back online and reconnect over SSH as `root` before continuing.

### 3.2 Configure Automatic Security Updates

> [!NOTE]
> By default, the DNF Automatic configuration in this runbook will automatically download and install security updates between 6:00 and 7:00 a.m. each day. If your environment requires change control, consider configuring update notifications instead of installing updates automatically.
> [Automating software updates in RHEL](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/automating-software-updates-in-rhel)

- [ ] **Install the `dnf-automatic` package:**
```bash
dnf install dnf-automatic -y
```

- [ ] **Configure DNF Automatic to install security updates only:**
```bash
sed -i.bak 's/^upgrade_type =.*/upgrade_type = security/' /etc/dnf/automatic.conf
```

- [ ] **Confirm that security-only updates are configured:**
```bash
grep '^upgrade_type =' /etc/dnf/automatic.conf
```

- [ ] **Enable and start the DNF Automatic installation timer:**
```bash
systemctl enable --now dnf-automatic-install.timer
```

- [ ] **Confirm that the DNF Automatic timer is active:**
```bash
systemctl status dnf-automatic-install.timer
```

- [ ] **Confirm that the timer is scheduled to run:**
```bash
systemctl list-timers dnf-automatic-install.timer
```

### 3.3 Configure the Firewall

- [ ] **Install the `firewalld` package:**
```bash
dnf install firewalld -y
```

- [ ] **Enable and start `firewalld`:**
```bash
systemctl enable --now firewalld
```

- [ ] **Confirm that `firewalld` is active:**
```bash
systemctl status firewalld
```

- [ ] **Allow incoming SSH traffic in the `drop` zone:**
```bash
firewall-cmd --permanent --zone=drop --add-service=ssh
```

- [ ] **Reload `firewalld` to apply the configuration change:**
```bash
firewall-cmd --reload
```

- [ ] **Set the `drop` zone as the default firewall zone:**
```bash
firewall-cmd --set-default-zone=drop
```

- [ ] **Confirm that `drop` is the default firewall zone:**
```bash
firewall-cmd --get-default-zone
```

- [ ] **Confirm the active network interface is assigned to the `drop` zone:**
```bash
firewall-cmd --get-active-zones
```

> [!NOTE]
> If the active network interface is not assigned to the `drop` zone, assign it manually and reload `firewalld` using this command: `firewall-cmd --permanent --zone=drop --change-interface=<interface> && firewall-cmd --reload`

- [ ] **Confirm that SSH traffic is allowed in the `drop` zone:**
```bash
firewall-cmd --zone=drop --list-all
```

> [!WARNING]
> Open a new terminal on your local workstation and confirm that you can establish a new SSH connection to the server. Keep the existing `root` session open until this test succeeds to avoid being locked out. Continue using the existing `root` session for the remainder of this runbook.

### 3.4 Create and Configure an Administrative Account

- [ ] **Create the administrative account:**
```bash
adduser admin
```

- [ ] **Set a password for the administrative account:**
```bash
passwd admin
```

- [ ] **Add the administrative account to the `wheel` group:**
```bash
usermod -aG wheel admin
```

- [ ] **Create the `.ssh` directory with the appropriate ownership and permissions:**
```bash
install -d -o admin -g admin -m 0700 /home/admin/.ssh
```

- [ ] **Create the `authorized_keys` file with the appropriate ownership and permissions:**
```bash
install -o admin -g admin -m 0600 /dev/null /home/admin/.ssh/authorized_keys
```

- [ ] **Add the SSH public key to the `authorized_keys` file:**
```bash
printf '%s\n' '<public_key>' | tee /home/admin/.ssh/authorized_keys > /dev/null
```

> [!NOTE]
> Replace `<public_key>` with an existing SSH public key. If you do not already have one, generate a new SSH key pair on your local workstation using `ssh-keygen -t ed25519 -C "admin@example.tld"`. Use the contents of the generated `.pub` file as `<public_key>`.

- [ ] **Confirm that the public key was added successfully:**
```bash
grep -E '^(ssh-|ecdsa-|sk-)' /home/admin/.ssh/authorized_keys
```

- [ ] **Restore the SELinux security contexts for the `.ssh` directory and its contents:**
```bash
restorecon -RFv /home/admin/.ssh
```

- [ ] **Confirm that the `.ssh` directory has the correct SELinux security context:**
```bash
ls -ldZ /home/admin/.ssh
```

- [ ] **Confirm that the `authorized_keys` file has the correct SELinux security context:**
```bash
ls -lZ /home/admin/.ssh/authorized_keys
```

> [!NOTE]
> The SELinux security context for both the `.ssh` directory and `authorized_keys` file should contain `ssh_home_t`.

> [!WARNING]
> Open a new terminal on your local workstation and confirm that you can establish an SSH connection to the server as the new administrative user using SSH key-based authentication. Do not continue if the connection falls back to password authentication. Verify that the account has `sudo` access by running `sudo -v`. Keep the existing `root` session open until both checks succeed to avoid being locked out. Continue using the existing `root` session for the remainder of this runbook.

### 3.5 Configure SSH Access

- [ ] **Create a new SSH server configuration file:**
```bash
install -o root -g root -m 0644 /dev/null /etc/ssh/sshd_config.d/10-auth.conf
```

- [ ] **Configure SSH to require key-based authentication and prevent remote root logins:**
```bash
printf '%s\n' \
'PermitRootLogin no' \
'PubkeyAuthentication yes' \
'PasswordAuthentication no' \
'KbdInteractiveAuthentication no' \
| tee /etc/ssh/sshd_config.d/10-auth.conf > /dev/null

```

- [ ] **Confirm that the configuration file contains the correct settings:**
```bash
cat /etc/ssh/sshd_config.d/10-auth.conf
```

- [ ] **Check the SSH configuration for syntax errors:**
```bash
sshd -t
```

- [ ] **Confirm that SSH requires key-based authentication and prevents remote root logins:**
```bash
sshd -T | grep -E \
-e '^permitrootlogin ' \
-e '^pubkeyauthentication ' \
-e '^passwordauthentication ' \
-e '^kbdinteractiveauthentication '
```

- [ ] **Reload the SSH service to apply the new configuration:**
```bash
systemctl reload sshd
```

### 3.6 Configure the Hostname and Time Zone

- [ ] **Set the system hostname:**
```bash
hostnamectl set-hostname host.example.tld
```

- [ ] **Confirm that the hostname was configured correctly:**
```bash
hostnamectl --static
```

- [ ] **Set the system time zone:**
```bash
timedatectl set-timezone America/Chicago
```

> [!NOTE]
> Get a list of supported time zones using the command `timedatectl list-timezones`.

- [ ] **Confirm that the system date, time, and time zone are correct:**
```bash
timedatectl
```

- [ ] **Confirm that the system time is synchronized:**
```bash
chronyc tracking
```

> [!NOTE]
> A synchronized system should report a Leap status of `Normal`.

### 3.7 Verify SELinux Enforcement

- [ ] **Confirm that SELinux is operating in enforcing mode:**
```bash
getenforce
```

---

## 4. Validation

- [ ] **Confirm you can establish an SSH connection as the administrative user.**
- [ ] **Confirm the administrative user can execute commands with `sudo`.**
- [ ] **Confirm that you cannot establish an SSH connection using password authentication.**
- [ ] **Confirm you cannot establish an SSH connection as the `root` user.**
- [ ] **Confirm the active network interface is assigned to the `drop` zone.**
- [ ] **Confirm automatic security updates are scheduled.**

---

## 5. Recovery / Rollback

If you lose SSH access to the server, use your hosting provider's recovery environment or remote console to regain access and correct the configuration. If those options are not available, you may need local console access to the system. If you cannot access the server through any of these methods, it may need to be reprovisioned.
