# Dedicated Server Security Checklist for Businesses

A dedicated server gives your business the whole machine. No neighbours, no shared CPU, and full root access. That is exactly why security is partly your job. Nobody else is going to patch your operating system, review your logins or test your backups unless you set it up.

This checklist walks you through it, in the order I would do it on a new server. It is written for a YouStable [dedicated server](https://www.youstable.com/dedicated-servers/), but most of it applies to any bare metal machine. Use it on day one, and again every quarter.

## Who Secures What

YouStable dedicated servers are semi-managed. That means the data center side is looked after for you, and the platform on top is yours. Knowing where that line sits tells you what belongs on your checklist.

| YouStable Looks After | You Look After |
|---|---|
| Network connectivity, uplink capacity and DDoS filtering | Patching and hardening the operating system |
| A dedicated hardware firewall in front of the server | Your own firewall rules on the server |
| Server hardware, power, cooling and the data center (Tier III/IV) | Installing and updating applications and databases |
| A network operations team watching servers around the clock | Logins, keys, passwords and who has access |
| Out-of-band IPMI, iLO and KVM access for recovery | Backups and restore tests |
| 24/7 chat and ticket support from senior engineers | Reviewing logs and responding to alerts |

Everything in the right-hand column is what the rest of this guide covers.

## The Security Checklist

### Before You Order

The safest server starts with the right choices at checkout.

| Decision | What to Think About |
|---|---|
| Location | YouStable has dedicated servers in India, the USA and the Netherlands. Pick the region that matches where your customers are and where your data is allowed to live. Rules differ by sector and country, so check yours |
| Memory | If data integrity matters, look at plans with ECC memory. The E5-2683 v4 has 64 GB DDR4 ECC and the EPYC 7502P has 256 GB DDR4 ECC (marked with an asterisk on the plan page, so confirm the details with the team) |
| Operating system | Choose one your team can patch confidently: AlmaLinux, Rocky Linux, Ubuntu, Debian or Windows Server. You can also upload your own ISO |
| Extra IPs | Order an additional IPv4 if you want to separate services, or run SSL on its own address |
| Security add-ons | Decide whether you want Imunify360 and Premium Support from day one |

### The First Hour on a New Server

Do these before anything else. They close the most common doors.

- [ ] **Update everything.**

```bash
# Ubuntu or Debian
sudo apt update && sudo apt full-upgrade -y

# AlmaLinux or Rocky Linux
sudo dnf upgrade -y
```

- [ ] **Create a normal admin user** and stop using root for daily work.

```bash
# Ubuntu or Debian
sudo adduser admin
sudo usermod -aG sudo admin

# AlmaLinux or Rocky Linux
sudo useradd -m admin
sudo passwd admin
sudo usermod -aG wheel admin
```

- [ ] **Set up SSH keys.** On your own computer, create a key and copy it to the server.

```bash
ssh-keygen -t ed25519 -C "business-server"
ssh-copy-id admin@YOUR_SERVER_IP
```

- [ ] **Turn off root and password logins.** Edit `/etc/ssh/sshd_config` and set:

```bash
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

Test the file, then reload SSH. Keep your current session open and log in from a second window before you close anything.

```bash
sudo sshd -t
sudo systemctl reload sshd    # on Ubuntu or Debian, the service is called ssh
```

> If you ever lock yourself out, the IPMI and KVM access that comes with the server lets you get back in, even when the operating system is not responding.

- [ ] **Check who has full power.** Only root should have user ID 0.

```bash
awk -F: '($3==0){print $1}' /etc/passwd
```

### Access and Network

Most break-ins start with a stolen password or an open port. Close both.

| Item | What to Do |
|---|---|
| Firewall | Allow only the ports you use, usually 22 (SSH), 80 and 443. Block everything else |
| SSH access | Restrict it to your office or VPN address if you have a fixed IP |
| Brute force protection | Install fail2ban, so repeated failed logins get blocked |
| Open ports | Review what is listening and shut down what you do not need |
| IPMI and KVM credentials | Treat them like the root password. Use a strong, unique password and share it with as few people as possible |
| Shared logins | Give every person their own account. Never share one login |
| Control panel | If you add cPanel, Plesk or DirectAdmin, keep it updated and limit who can reach it |

On Ubuntu or Debian with ufw:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw limit 22/tcp
sudo ufw enable
```

On AlmaLinux or Rocky Linux with firewalld:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

To see what is listening on your server:

```bash
sudo ss -tulpn
```

For fail2ban, create `/etc/fail2ban/jail.local`:

```bash
[sshd]
enabled = true
maxretry = 5
findtime = 10m
bantime = 1h
```

```bash
sudo systemctl enable --now fail2ban
```

### Software, Data and Backups

Old software and untested backups are the two things that turn a small problem into a disaster.

| Item | What to Do |
|---|---|
| Security updates | Switch on automatic security updates, and still reboot when a kernel update needs it |
| Applications | Keep your CMS, plugins, frameworks and databases up to date. Remove anything you no longer use |
| Malware protection | Consider Imunify360 for a firewall, malware scanner and intrusion detection |
| Database access | Do not expose your database to the internet. Allow only the application server |
| HTTPS | Use valid TLS certificates, and redirect HTTP to HTTPS |
| Backups | Follow the 3-2-1 rule: three copies, on two different kinds of storage, with one copy off the server |
| Restore test | Restore a backup to a test machine every quarter. A backup you have never restored is only a hope |
| Encryption | Encrypt backups before storing them off the server, and consider encrypting volumes that hold sensitive data |

Automatic security updates:

```bash
# Ubuntu or Debian
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades

# AlmaLinux or Rocky Linux
sudo dnf install dnf-automatic
sudo systemctl enable --now dnf-automatic-install.timer
```

A simple file backup to a separate machine:

```bash
rsync -az --delete /var/www/ backup@BACKUP_HOST:/backups/www/
```

A note on disk encryption. If you encrypt the whole disk, the server will ask for a passphrase at every reboot, which you would enter through the KVM console. Think about whether you are comfortable with that before you choose it.

### Monitoring and Logs

YouStable's network operations team watches the servers. They cannot see inside your application, so logs and alerts on that side are yours.

| Item | What to Do |
|---|---|
| Failed logins | Check them weekly |
| Resource use | Watch CPU, RAM, disk and bandwidth. A sudden jump is often the first sign of trouble |
| Disk space | Alert when it passes 80% |
| Application logs | Look for errors and strange requests |
| New users and files | Review after any change |
| Premium Support | Consider it if you want priority response and proactive monitoring |

Useful commands:

```bash
# Failed SSH logins on Ubuntu or Debian
sudo grep "Failed password" /var/log/auth.log | tail -20

# Recent failed logins on any system
sudo lastb | head -20

# Disk space
df -h
```

## If Something Goes Wrong

Write this plan down now, before you need it.

| Order | Action |
|---|---|
| 1 | Contact YouStable support on live chat or a ticket, and describe what you see |
| 2 | Contain it: block the attacking address, disable the affected account and stop the affected service |
| 3 | Keep evidence: copy your logs before you change anything |
| 4 | Recover: if the server is compromised, reinstall the operating system from your control panel or use IPMI, then restore from a clean backup |
| 5 | Reset every password, SSH key and API key that the server could access |
| 6 | Find the cause, fix it, and write down what you learned |

A clean reinstall plus a good backup is often the safest way back. That is another reason to test your restore.

## What a Security Stack Costs

Security does not need to be expensive. Here is a sample monthly setup on the entry server, using 12 month prices. The total is my calculation.

| Item | Monthly Price |
|---|---|
| Xeon E3-1230v3 dedicated server (4 Core, 32 GB DDR4, 480 GB SSD, 10 TB bandwidth) | $77.99 |
| Imunify360 (firewall, malware scanner, intrusion detection) | $10.00 |
| Premium Support (priority response, proactive monitoring) | $18.86 |
| **Total** | **$106.85** |

You can add an extra IPv4 for $5.00 if you want to separate services. These are some of the plans on the [dedicated servers page](https://www.youstable.com/dedicated-servers/), on the 12 month cycle, which saves 15% over the regular monthly price.

| Plan | Cores / Speed | RAM | Storage | Bandwidth | Price per Month |
|---|---|---|---|---|---|
| E3-1230v3 | 4 Core / 3.3 GHz | 32 GB DDR4 | 480 GB SSD | 10 TB | $77.99 |
| E-2136 | 6 Core / 3.3 GHz | 32 GB DDR4 | 480 GB SSD | 10 TB | $116.99 |
| E5-2667v4 (Most Popular) | 8 Core / 3.2 GHz | 32 GB DDR4 | 960 GB NVMe SSD | 10 TB | $169.99 |
| Xeon E5-2683 v4 | 16 Core / 2.1-3.0 GHz | 64 GB DDR4 ECC | 960 GB NVMe SSD | 10 TB | $275.99 |
| AMD EPYC 7502P | 32 Core / 2.5-3.35 GHz | 256 GB DDR4 ECC* | 2 x 1.92 TB NVMe SSD | 50 TB | $604.99 |

Setup is free on every plan, and so is migration. If none of these fit, you can request a custom quote.

## How to Order and Secure Your Server (Step by Step)

1. **List what you need to protect.** Write down your data, your applications and the people who need access.
2. **Choose your location and hardware.** Use the table above, and pick ECC memory if data integrity matters most.
3. **Open the plans.** Go to the [YouStable dedicated servers page](https://www.youstable.com/dedicated-servers/) and choose your billing cycle: 12 month, 6 month, 3 month or 1 month.
4. **Click Configure Now.** Select your operating system and the add-ons you want, such as Imunify360, Premium Support or an extra IPv4.
5. **Create your account and pay.** Sign up on [YouStable](https://www.youstable.com/) and complete your order. There is no setup fee.
6. **Run the first hour checklist.** Update the system, create your admin user, set up SSH keys and switch off root and password logins.
7. **Set your firewall and fail2ban.** Open only the ports you need.
8. **Install and harden your applications.** Remove what you do not use, and keep the rest updated.
9. **Set up backups and test one restore.** Do this before real customer data goes on the server.
10. **Move your sites.** YouStable engineers migrate your websites and data for free, and confirm everything before your DNS changes.
11. **Turn on monitoring.** Set alerts for disk, CPU and failed logins.
12. **Put it in your calendar.** Repeat the quarterly review below.

Need a domain for a new project? You can register it from the same [YouStable](https://www.youstable.com/) account and manage hosting and domain together.

## The Quarterly Review

Print this and tick it off every three months.

| Done | Check |
|---|---|
| ☐ | All updates installed, and the server rebooted if needed |
| ☐ | The list of user accounts and SSH keys reviewed, and leavers removed |
| ☐ | Firewall rules and open ports reviewed |
| ☐ | A backup restored to a test machine |
| ☐ | Failed logins and unusual activity in the logs reviewed |
| ☐ | Applications, plugins and databases updated |
| ☐ | IPMI and KVM credentials changed if anyone with access has left |
| ☐ | Your incident plan re-read and the contacts updated |

## FAQs

### Is a dedicated server more secure than shared hosting?

It can be, because no other customer shares your hardware or your resources. But you take on more responsibility, since you manage the operating system, the firewall and your applications.

### What does semi-managed mean for security?

YouStable looks after the network, DDoS filtering, hardware and data center. You look after patching, your own firewall rules, applications, access and backups.

### Does YouStable include DDoS protection and a firewall?

Yes. The dedicated server plans include always-on DDoS mitigation and a dedicated hardware firewall. You should still set up your own firewall on the server for finer control.

### What if I lock myself out of my server?

Use the out-of-band IPMI, iLO or KVM access that comes with the server. You can reboot, reinstall and recover it even when the operating system is not responding.

### Do I need Imunify360?

It is optional, and billed only when you enable it, starting at $10.00 a month. It adds a firewall, malware scanner and intrusion detection, which is useful if you run websites or a control panel.

### How often should I back up?

It depends on how fast your data changes. For a busy site or database, daily is a sensible start. Whatever you choose, keep one copy off the server and test a restore regularly.

### Where can I host my dedicated server?

In India, the USA or the Netherlands. Choose the region closest to your users and the one that fits your data rules.

### Is migration free?

Yes. Setup and migration are free on every plan, and the YouStable engineers confirm everything before your DNS changes over.

## Conclusion

Server security is not one big task. It is a handful of habits: close what you do not use, keep everything updated, control who gets in, and prove that your backups work. YouStable's dedicated servers give you a solid base, with DDoS protection, a hardware firewall, out-of-band access and 24/7 engineers. This checklist covers the part that belongs to you.

Ready to set one up? Check the plans on the [YouStable dedicated servers page](https://www.youstable.com/dedicated-servers/), or visit [YouStable](https://www.youstable.com/) if you also need hosting or a domain name for your project.
