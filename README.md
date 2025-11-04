# Docker Monitoring by Zabbix Active Agent 2

This template provides comprehensive Docker monitoring using Zabbix Active Agent 2 with the Docker plugin. It monitors Docker engine information, containers, CPU usage, memory consumption, and network statistics.

## Features

- Docker engine health monitoring
- Container discovery and monitoring
- CPU and memory usage per container
- Network traffic monitoring (RX/TX)
- Container state tracking (running, stopped, paused)
- Image count monitoring
- Active agent mode (no inbound connections required)

## Requirements

- Zabbix Server 7.4 or higher
- Zabbix Agent 2 installed on the Docker host
- Docker Engine installed and running
- Root or appropriate user permissions to access Docker socket

## Installation

### 1. Prerequisites

Ensure Docker and Zabbix Agent 2 are installed on your system:

```bash
# Verify Docker is installed
docker --version

# Verify Zabbix Agent 2 is installed
zabbix_agent2 --version
```

### 2. Add Zabbix User to Docker Group

The Zabbix agent needs permission to access the Docker socket. Add the `zabbix` user to the `docker` group:

```bash
sudo usermod -aG docker zabbix
```

**Important:** After adding the user to the group, restart the Zabbix Agent 2 service for the changes to take effect:

```bash
# For systemd-based systems (Ubuntu, Debian, RHEL, CentOS)
sudo systemctl restart zabbix-agent2

# For init.d-based systems
sudo service zabbix-agent2 restart
```

### 3. Configure Docker Plugin

Create or edit the Docker plugin configuration file:

```bash
sudo nano -w /etc/zabbix/zabbix_agent2.d/plugins.d/docker.conf
```

Add the following configuration:

```ini
### Option: Plugins.Docker.Endpoint
#       Docker API endpoint.
#
# Mandatory: no
# Default: unix:///var/run/docker.sock
Plugins.Docker.Endpoint=unix:///var/run/docker.sock

### Option: Plugins.Docker.Timeout
#       The maximum time (in seconds) for waiting when a request has to be done.
#
# Mandatory: no
# Range: 1-30
# Default:
# Plugins.Docker.Timeout=<Global timeout>
```

Alternatively, you can copy the provided configuration file:

```bash
sudo cp docker.conf /etc/zabbix/zabbix_agent2.d/plugins.d/docker.conf
sudo chown root:root /etc/zabbix/zabbix_agent2.d/plugins.d/docker.conf
sudo chmod 644 /etc/zabbix/zabbix_agent2.d/plugins.d/docker.conf
```

### 4. Configure Zabbix Agent 2 for Active Mode

Edit the main Zabbix Agent 2 configuration file:

```bash
sudo nano -w /etc/zabbix/zabbix_agent2.conf
```

You have **two options** for hostname configuration:

#### Option A: Static Hostname (Manual Configuration)

Use this if you want to set a specific hostname for each agent:

```ini
# Zabbix Server IP or hostname for active checks
ServerActive=YOUR_ZABBIX_SERVER_IP

# Static hostname - must match the host name in Zabbix frontend EXACTLY
Hostname=YOUR_HOST_NAME

# Optional: Disable passive checks if only using active mode
# Server=

# Enable Docker plugin (usually enabled by default)
Plugins.Docker.Endpoint=unix:///var/run/docker.sock
```

**Important Notes:**

- Replace `YOUR_ZABBIX_SERVER_IP` with your actual Zabbix Server IP address or hostname
- Replace `YOUR_HOST_NAME` with the exact hostname you'll use when creating the host in Zabbix
- The `Hostname` parameter is **case-sensitive** and must match exactly

#### Option B: Dynamic Hostname (Automatic Configuration)

Use this if you want the agent to automatically use the system's hostname:

```ini
# Zabbix Server IP or hostname for active checks
ServerActive=YOUR_ZABBIX_SERVER_IP

# Dynamic hostname - agent will query this item to get the hostname
HostnameItem=system.hostname

# Optional: Disable passive checks if only using active mode
# Server=

# Enable Docker plugin (usually enabled by default)
Plugins.Docker.Endpoint=unix:///var/run/docker.sock
```

**Important Notes:**

- `HostnameItem=system.hostname` will use the system's hostname (output of `hostname` command)
- The host in Zabbix frontend must be created with the **exact same name** as returned by `hostname` command
- Check your system hostname: `hostname` or `hostname -f` (for FQDN)
- Common HostnameItem options:
  - `system.hostname` - Short hostname (e.g., "server01")
  - `system.hostname[fqdn]` - Fully qualified domain name (e.g., "server01.example.com")

**Which option to choose?**

- **Option A (Hostname):** Better for manual control, testing, or when system hostname doesn't match desired Zabbix name
- **Option B (HostnameItem):** Better for automation, consistent naming, and large deployments

### 5. Verify Docker Socket Permissions

Ensure the Docker socket is accessible:

```bash
ls -l /var/run/docker.sock
```
```bash
usermod -aG docker zabbix
```

Expected output should show the `docker` group has read/write permissions:

```
srw-rw---- 1 root docker 0 Nov  4 10:00 /var/run/docker.sock
```

### 6. Test the Configuration

Test that the Zabbix agent can communicate with Docker:

```bash
# Restart the agent
sudo systemctl restart zabbix-agent2

# Check agent status
sudo systemctl status zabbix-agent2

# Test Docker plugin
zabbix_agent2 -t docker.ping
zabbix_agent2 -t docker.info
```

Expected output for `docker.ping` should be `1`, and `docker.info` should return JSON data about your Docker installation.

## Zabbix Frontend Configuration

### 1. Import the Template

1. Log in to your Zabbix frontend
2. Navigate to **Configuration → Templates**
3. Click **Import** in the top right corner
4. Choose the `template_docker_agent_2_active.yaml` file
5. Click **Import**

### 2. Create the Host

1. Navigate to **Configuration → Hosts**
2. Click **Create host** in the top right corner
3. Configure the host:
   - **Host name:** Must match the `Hostname` parameter in `zabbix_agent2.conf` (case-sensitive)
   - **Groups:** Select appropriate host groups
   - **Interfaces:** **DO NOT add any interface** (active agent doesn't need one)
4. Click **Add**

### 3. Link the Template

1. In the host configuration, go to the **Templates** tab
2. Click **Select** next to **Link new templates**
3. Find and select **Docker by Zabbix Active Agent 2**
4. Click **Add** (in the template selection popup)
5. Click **Update** to save the host configuration

### 4. Wait for Data Collection

- Active agents check in with the Zabbix server automatically
- Initial data should appear within 1-2 minutes
- Container discovery runs every 15 minutes by default
- Check **Monitoring → Latest data** to verify data collection

## Template Macros

You can customize container discovery and alerting using the following macros:

| Macro                                        | Default Value      | Description                                                             |
| -------------------------------------------- | ------------------ | ----------------------------------------------------------------------- |
| `{$DOCKER.LLD.FILTER.CONTAINER.MATCHES}`     | `.*`               | Regular expression to match container names for discovery               |
| `{$DOCKER.LLD.FILTER.CONTAINER.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Regular expression to exclude container names from discovery            |
| `{$DOCKER.CONTAINERS.STOPPED.MAX.WARN}`      | `1`                | Maximum number of stopped containers before warning trigger (threshold) |
| `{$DOCKER.CONTAINERS.STOPPED.MAX.CRIT}`      | `20`               | Critical threshold for stopped containers (high priority alert)         |

## Monitored Metrics

### Docker Engine Metrics

- Docker service status (up/down)
- Docker engine name
- Server version
- Total containers count
- Running containers count
- Stopped containers count
- Paused containers count
- Total images count
- CPU count
- Total memory

### Per-Container Metrics (Auto-discovered)

- Container running state
- Container status (running, exited, created, etc.)
- CPU usage percentage
- Memory usage (bytes)
- Network RX (bits per second)
- Network TX (bits per second)

## Triggers

### Docker Service Level

- **Docker: Service is down** (High priority) - Triggered when Docker engine is not responding

### Container Level

- **Docker: Container {#NAME} is not running** (Warning) - Triggered when a discovered container stops

### Container Capacity Management

- **Docker: Too many stopped containers** (Warning) - Triggered when the number of stopped containers exceeds `{$DOCKER.CONTAINERS.STOPPED.MAX.WARN}` (default: 1)
- **Docker: Critical number of stopped containers** (High priority) - Triggered when stopped containers exceed `{$DOCKER.CONTAINERS.STOPPED.MAX.CRIT}` (default: 20)
- **Docker: Stopped containers count increased** (Info) - Informational trigger when new containers are stopped
