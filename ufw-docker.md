# Fix The Docker and UFW Security Flaw

## TL;DR

When Docker is installed, published ports bypass UFW by default. This guide shows how to fix this security issue while keeping Docker's iptables management enabled. For detailed information, see original [Solving UFW and Docker issues](https://github.com/chaifeng/ufw-docker).

## Installation & Setup

### Step 1 (optional): Rollback Previous Modifications

Before proceeding, ensure a clean state:

1. Verify Docker iptables is enabled (remove `--iptables=false` if present)
2. Reset docker related UFW FORWARD rule to DROP (default)
3. Remove old Docker rules from `/etc/ufw/after.rules`
4. Restart Docker if configuration was changed

### Step 2: Install UFW + Docker Rules

Edit `/etc/ufw/after.rules` (and `/etc/ufw/after6.rules` for IPv6):

```bash
sudo nano /etc/ufw/after.rules
```

Append the following at the end of the file:

```text
# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:ufw-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j ufw-user-forward

# Private networks returned (bypass UFW)
-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

# DNS return
-A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

# Deny public network access to private ranges
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN

# Logging & drop
-A ufw-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A ufw-docker-logging-deny -j DROP

COMMIT
# END UFW AND DOCKER
```

Restart UFW:

```bash
sudo systemctl restart ufw
sudo ufw reload
```

If rules don't take effect, reboot the system.

---

## Usage Scenarios

### Scenario 1: Allow Public Access to Container Ports

**Allow all containers on port 80 (HTTP):**

```bash
sudo ufw route allow proto tcp from any to any port 80
```

⚠️ **Important:** Use the container's internal port (80), not the host port (e.g., if mapped as 8080:80, use 80).

**Allow specific container only (172.17.0.2):**

```bash
sudo ufw route allow proto tcp from any to 172.17.0.2 port 80
```

**Allow UDP traffic (e.g., DNS):**

```bash
sudo ufw route allow proto udp from any to any port 53
sudo ufw route allow proto udp from any to 172.17.0.2 port 53
```

### Scenario 2: Allow Access from Specific IPs or Networks

**Allow from a specific IP:**

```bash
sudo ufw route allow proto tcp from 203.0.113.10 to any port 80
```

**Allow from a CIDR range:**

```bash
sudo ufw route allow proto tcp from 203.0.113.0/24 to any port 443
```

### Scenario 3: Block Container Internet Access

**Prevent a specific container from accessing the internet:**

```bash
sudo ufw route deny from 172.17.0.9 to any
```

### Scenario 4: Fresh UFW Setup

If setting up UFW for the first time:

```bash
sudo apt install ufw -y
sudo systemctl enable ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Then follow Step 2 above to add the Docker rules.

---

## Handling LAN 10.x.x.x Networks

By default, all private ranges (10/8, 172.16/12, 192.168/16) bypass UFW. If your LAN uses 10.x.x.x addresses, choose one of these options:

### Option 1: Keep Docker Safe, Filter LAN Selectively

**Use case:** Docker bridge uses 172.17.0.0/16, LAN uses 10.10.0.0/16

Edit `/etc/ufw/after.rules`:

```text
# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:ufw-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j ufw-user-forward

# Private networks returned (bypass UFW)
# -A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

# DNS return
-A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

# Deny public network access to private ranges
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d 172.16.0.0/12
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 192.168.0.0/16
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -p udp -m udp --dport 0:32767 -d 172.16.0.0/12

-A DOCKER-USER -j RETURN

# Logging & drop
-A ufw-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A ufw-docker-logging-deny -j DROP

COMMIT
# END UFW AND DOCKER
```

**Result:** LAN traffic (10.10.x.x) goes through UFW rules, Docker traffic is unaffected.

### Option 2: UFW Filters Everything (Maximum Control)

Remove all RETURN lines:

```text
# -A DOCKER-USER -j RETURN -s 10.0.0.0/8
# -A DOCKER-USER -j RETURN -s 172.16.0.0/12
# -A DOCKER-USER -j RETURN -s 192.168.0.0/16
```

**Pros:** Maximum control over all traffic

**Cons:** May break container-to-container or container-to-host communication

### Option 3: Hybrid Approach

**Use case:** LAN uses 10.10.0.0/16, Docker bridge uses 10.20.0.0/16

```text
# Allow only Docker subnets
-A DOCKER-USER -j RETURN -s 10.20.0.0/16
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16
```

**Result:** LAN 10.10.x.x goes through UFW, Docker bridge is trusted.

---

## Using the ufw-docker Utility

The `ufw-docker` utility automates rule management and supports Docker Swarm.

### Installation

```bash
sudo wget -O /usr/local/bin/ufw-docker \
  https://github.com/chaifeng/ufw-docker/raw/master/ufw-docker
sudo chmod +x /usr/local/bin/ufw-docker
```

Install the UFW rules:

```bash
sudo ufw-docker install
```

**Optional:** Include specific Docker subnets:

```bash
# Auto-detect all Docker subnets
sudo ufw-docker install --docker-subnets

# Use specific subnets
sudo ufw-docker install --docker-subnets 192.168.207.0/24 10.207.0.0/16
```

### Common Commands

| Action | Command |
|--------|---------|
| Check UFW + Docker setup | `ufw-docker check` |
| Show current allowed routes | `ufw-docker status` |
| List rules for a container | `ufw-docker list <container>` |
| Allow container port | `ufw-docker allow <container> 80/tcp` |
| Remove container port rule | `ufw-docker delete allow <container> 80/tcp` |
| Allow all ports for container | `ufw-docker allow <container>` |
| Remove all container rules | `ufw-docker delete allow <container>` |

### Docker Swarm Support

```bash
ufw-docker service allow web 80
ufw-docker service delete allow web
```

---

## Testing

### Basic Test

Run a test container:

```bash
docker run -d --name web -p 8080:80 nginx
```

Allow access:

```bash
sudo ufw route allow proto tcp from any to any port 80
```

Test from an external machine to verify the rule works.

### Vagrant Test (Local)

```bash
vagrant up
vagrant ssh master
docker service create --name web --publish 8080:80 httpd:alpine
```

Initially, the port should be inaccessible externally:

```bash
curl -v http://192.168.56.131:8080
```

Allow external access:

```bash
sudo ufw-docker service allow web 80
curl "http://192.168.56.13{0,1,2}:8080"
```

---

## Quick Reference

| Goal | Command |
|------|---------|
| Start UFW clean | `sudo ufw reset && sudo ufw enable` |
| Allow container's internal port | `ufw route allow proto tcp from any to any port <container_port>` |
| Allow from specific IP | `ufw route allow proto tcp from <IP> to any port <container_port>` |
| Allow from CIDR range | `ufw route allow proto tcp from <CIDR> to any port <port>` |
| Block container internet access | `ufw route deny from <container_ip> to any` |
| Use ufw-docker automation | `ufw-docker install` then `ufw-docker allow <container> <port>` |
| Reload UFW | `sudo ufw reload` or `sudo systemctl restart ufw` |

---

## Summary

- `ufw-docker` ensures Docker does not bypass UFW
- Use `ufw route allow/deny` for specific ports, IPs, or CIDR ranges
- LAN 10.x.x.x networks require special handling (choose Option 1, 2, or 3)
- Always restart UFW after modifications: `sudo systemctl restart ufw && sudo ufw reload`
- The `ufw-docker` utility simplifies rule management for containers and Swarm services
