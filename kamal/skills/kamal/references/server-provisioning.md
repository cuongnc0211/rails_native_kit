# Server Provisioning for Kamal

_Docs-sourced; verified against the official Kamal docs (Kamal 2.11.x)._

Kamal is a deploy tool, not a provisioning tool. The only thing it installs on a host is Docker (during `kamal setup`). Hardening the box — users, firewall, fail2ban, swap, updates — is on you. This covers a sane baseline for a fresh Ubuntu server, runnable before or alongside your first deploy.

## What Kamal Does vs. What You Do

| Concern | Owner |
|---|---|
| Install Docker on hosts | Kamal (`kamal setup`) |
| Non-root deploy user | You |
| Firewall (UFW) | You |
| Brute-force protection (fail2ban) | You |
| Swap space | You |
| Unattended security updates | You |
| Load balancer (multi-web-host) | You / your cloud provider |

Do this once per server. The steps are idempotent-friendly, so re-running is safe.

## Docker

Kamal installs Docker for you, but you can pre-install it to keep `setup` fast and predictable:

```bash
curl -fsSL https://get.docker.com | sh
```

## Non-Root Deploy User

Running the app as root is avoidable. Create a user, give it Docker access, and copy your SSH key over. Then set `ssh.user` in `config/deploy.yml` to match.

```bash
sudo adduser --disabled-password --gecos "" deploy
sudo usermod -aG docker deploy
sudo rsync --archive --chown=deploy:deploy ~/.ssh /home/deploy/
```

```yaml
# config/deploy.yml
ssh:
  user: deploy
```

The first non-root Linux user usually gets UID/GID 1000 — handy when you `chown` a storage directory the container writes to.

## UFW Firewall

Allow only SSH and the web ports. Enable SSH **first** so you don't lock yourself out.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22       # SSH — do this before enabling
sudo ufw allow 80       # HTTP
sudo ufw allow 443      # HTTPS (Kamal Proxy TLS)
sudo ufw --force enable
```

Gotcha: on Ubuntu, Docker writes its own iptables rules that sit **ahead** of UFW, so a container that publishes a port is reachable even if UFW "denies" it. Don't rely on UFW to hide a database. Bind sensitive accessory ports to localhost instead:

```yaml
accessories:
  db:
    image: postgres:16
    host: 192.168.0.1
    port: "127.0.0.1:5432:5432"   # not reachable from the public internet
```

## fail2ban

Bans IPs after repeated failed SSH logins — cheap insurance for an internet-facing box.

```bash
sudo apt-get install -y fail2ban
sudo systemctl enable --now fail2ban
```

The default jail watches sshd. Tune `/etc/fail2ban/jail.local` if you want stricter ban times.

## Swap

A deploy briefly runs the old and new container side by side, so memory peaks above steady state. On small VMs, add swap so a deploy doesn't OOM-kill the app.

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo sysctl vm.swappiness=20      # favor RAM, use swap as a safety net
```

## Unattended Security Updates

Let the OS apply security patches on its own:

```bash
sudo apt-get install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## Lock Down SSH (Last)

Once you've confirmed a deploy works as the `deploy` user, disable password auth and root login — then restart SSH:

```bash
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

Order matters: confirm key-based login as `deploy` and at least one successful `kamal deploy` **before** disabling root, or you can lock yourself out.

## Provisioning Checklist

| Step | Why |
|---|---|
| Install Docker (or let `kamal setup`) | Containers need a runtime |
| Create `deploy` user, add to `docker` group | Don't run as root |
| Copy SSH key, set `ssh.user: deploy` | Kamal connects as this user |
| UFW: allow 22/80/443 only | Shrink attack surface |
| Bind accessory ports to `127.0.0.1` | UFW won't stop Docker-published ports |
| Install fail2ban | Throttle SSH brute force |
| Add 2G swap | Survive the two-container deploy peak |
| Enable unattended-upgrades | Automatic security patches |
| Disable root + password SSH (last) | Only after a deploy succeeds |

Multi-host note: for more than one `web` server you need a load balancer in front and SSL terminated there — Kamal won't configure a load balancer for you (see kamal-proxy.md). Next: secrets handling (see secrets-and-env.md); ongoing ops (see maintenance.md).
