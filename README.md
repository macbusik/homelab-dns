# Homelab DNS Server

A complete DNS and reverse proxy solution for home networks using Pi-hole, Traefik, and Docker. This project provides wildcard DNS resolution for local development and services with automatic SSL termination and service discovery.

## Prerequisites

### Router DHCP Configuration

**CRITICAL**: Your home router's DHCP server must be configured to use this DNS server, otherwise clients won't resolve your local domains.

1. **Access your router's admin panel** (usually `192.168.1.1` or `192.168.0.1`)
2. **Navigate to DHCP settings** (often under "Network" or "LAN" settings)
3. **Set Primary DNS** to your server's IP address (e.g., `192.168.10.217`)
4. **Set Secondary DNS** to a public DNS like `8.8.8.8` or `1.1.1.1` for fallback
5. **Save and restart DHCP service** or reboot router

After this change, all devices on your network will automatically use your homelab DNS server for domain resolution.

### System Requirements

- Linux server (tested on Arch Linux)
- Docker and Docker Compose
- Ansible for deployment
- Network access to modify router DHCP settings

## Architecture

This project follows SOLID principles with modular Ansible roles:

- **system-setup**: Network configuration and kernel modules
- **docker-engine**: Docker installation and service management  
- **reverse-proxy**: Traefik reverse proxy with automatic service discovery
- **dns-server**: Pi-hole DNS server with wildcard domain support
- **monitoring**: Health checks and service validation

## Features

- **Wildcard DNS**: Automatic resolution for `*.yourdomain` to your server
- **Reverse Proxy**: Traefik automatically routes traffic based on hostnames
- **Service Discovery**: Docker labels automatically configure routing
- **Ad Blocking**: Pi-hole provides network-wide ad blocking
- **Web Interfaces**: 
  - Pi-hole admin at `pihole.yourdomain`
  - Traefik dashboard at `traefik.yourdomain`
- **Health Monitoring**: Automated testing of DNS and HTTP services

## Quick Start

### 1. Configure Your Environment

Edit `host_vars/archserver.yml`:

```yaml
root_domain: "homelab"  # Your local domain
server_ip: "192.168.1.100"  # Your server's IP
install_path: "/opt/homelab"
timezone: "America/New_York"
```

### 2. Update Inventory

Edit `inventory/hosts`:

```ini
[homelab]
archserver ansible_host=192.168.1.100 ansible_user=your_user
```

### 3. Deploy

```bash
# Install Ansible dependencies
ansible-galaxy collection install community.docker

# Run the playbook
ansible-playbook -i inventory/hosts site.yml
```

### 4. Verify Installation

After deployment, test your setup:

```bash
# Test DNS resolution
dig @192.168.1.100 whoami.homelab

# Test HTTP access
curl -H "Host: whoami.homelab" http://192.168.1.100
```

## Usage

### Adding New Services

Create a `docker-compose.yml` with Traefik labels:

```yaml
services:
  myapp:
    image: nginx:alpine
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.homelab`)"
    networks:
      - proxy

networks:
  proxy:
    external: true
```

The service will automatically be available at `http://myapp.homelab`

### Accessing Web Interfaces

- **Pi-hole Admin**: `http://pihole.homelab` (DNS management and ad blocking)
- **Traefik Dashboard**: `http://traefik.homelab:8080` (routing and service status)
- **Test Services**: 
  - `http://whoami.homelab` (connection test)
  - `http://test.homelab` (nginx test)

### Customization

Each role has configurable defaults in `roles/*/defaults/main.yml`:

- **Ports**: Modify service ports
- **Images**: Change Docker images
- **Environment**: Adjust service configuration
- **Networks**: Configure network settings

Override any defaults in your `host_vars/` file.

## Troubleshooting

### DNS Not Resolving

1. **Check router DHCP**: Ensure DNS server is set correctly
2. **Verify server IP**: Confirm server is accessible on configured IP
3. **Test direct DNS**: `dig @your-server-ip domain.name`
4. **Check Pi-hole logs**: `docker logs pihole`

### Services Not Accessible

1. **Verify Traefik**: Check dashboard at `traefik.homelab:8080`
2. **Check Docker networks**: `docker network ls`
3. **Inspect service labels**: `docker inspect container-name`
4. **Review Traefik logs**: `docker logs traefik`

### Router Configuration Issues

- **DHCP lease time**: May need to wait for DHCP renewal
- **DNS caching**: Restart devices or flush DNS cache
- **Firewall**: Ensure port 53 (DNS) is accessible
- **Multiple DHCP servers**: Disable conflicting DHCP services

## Development

### Project Structure

```
homelab-dns-server/
├── site.yml                 # Main playbook
├── inventory/               # Server inventory
├── host_vars/              # Server-specific variables
├── roles/                  # Ansible roles
│   ├── system-setup/       # Network and kernel config
│   ├── docker-engine/      # Docker installation
│   ├── reverse-proxy/      # Traefik configuration
│   ├── dns-server/         # Pi-hole setup
│   └── monitoring/         # Health checks
└── test-services.yml       # Example services
```

### Testing Changes

```bash
# Dry run
ansible-playbook -i inventory/hosts site.yml --check

# Run specific role
ansible-playbook -i inventory/hosts site.yml --tags dns-server

# Verbose output
ansible-playbook -i inventory/hosts site.yml -v
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Follow SOLID principles for any new roles
4. Test changes thoroughly
5. Submit a pull request

## License

MIT License - see LICENSE file for details.

