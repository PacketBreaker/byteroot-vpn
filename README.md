<div align="center">

# ByteRoot VPN

<a href="https://github.com/PacketBreaker/byteroot-vpn">
  <img src="https://readme-typing-svg.demolab.com?font=JetBrains+Mono&weight=700&size=24&duration=2800&pause=700&color=7C3AED&center=true&vCenter=true&width=760&lines=Multi-protocol+VPN+server+manager;Ubuntu+22%E2%80%9326+%7C+Debian+10%2B;Built+for+x86_64+and+ARM64+VPS" alt="Animated ByteRoot VPN tagline" />
</a>

<br />

[![Ubuntu](https://img.shields.io/badge/Ubuntu-22%E2%80%9326-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Debian](https://img.shields.io/badge/Debian-10%2B-A81D33?logo=debian&logoColor=white)](https://www.debian.org/)
[![Architecture](https://img.shields.io/badge/CPU-x86__64%20%7C%20ARM64-2EA44F?logo=linux&logoColor=white)](#intended-operating-systems-and-architectures)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**A multi-protocol VPN installer with an interactive terminal control panel, for real-world VPSs.**

Created by **Dev. Eng Abdelrahman Rabie**
Telegram: [@PacketBreaker](https://t.me/PacketBreaker)

</div>

---

ByteRoot is designed to install SSH (direct, WebSocket-style and SSL/TLS), Vmess, Vless and Trojan on a fresh Ubuntu or Debian server, wire them together behind HAProxy so everything can share port 443, and give you a `byteroot` terminal panel to create accounts, set limits and watch traffic without touching config files by hand.

> **Status:** personal project under active development. The descriptions below are design intent. No independent installation or connectivity validation has been performed on this code; see **Known limitations** for what is and is not enforced today.

## At a glance

```mermaid
flowchart LR
    VPS["☁️ VPS provider"] --> OS{"🖥️ Target OS"}
    OS --> UB["🟠 Ubuntu 22–26"]
    OS --> DEB["🔴 Debian 10+"]
    UB --> ARCH{"⚙️ CPU architecture"}
    DEB --> ARCH
    ARCH --> X["x86_64 / AMD64"]
    ARCH --> A["ARM64 / AArch64"]
    X --> BR["🚀 ByteRoot VPN"]
    A --> BR
    BR --> PANEL["🎛️ byteroot terminal panel"]
    BR --> PROTO["🔐 SSH · Vmess · Vless · Trojan"]

    classDef ubuntu fill:#E95420,color:#fff,stroke:#9c3413,stroke-width:2px;
    classDef debian fill:#A81D33,color:#fff,stroke:#741326,stroke-width:2px;
    classDef main fill:#6D28D9,color:#fff,stroke:#4C1D95,stroke-width:3px;
    class UB ubuntu;
    class DEB debian;
    class BR,PANEL,PROTO main;
```

## Features

### Supported protocols and ports

| Protocol / service | Port(s) | Notes |
|---|---:|---|
| OpenSSH | 22 | Standard SSH access |
| SSH WebSocket | 80, 8080 | WebSocket transport |
| SSH Stunnel (SSL/TLS) | 442, 443 | Port 443 is shared through HAProxy |
| BadVPN UDPGW | 7100–7900 | UDP gateway range |
| Nginx | 81 | Web server / service endpoint |
| Vmess WebSocket TLS | 443 | Path: `/vmess` |
| Vless WebSocket TLS | 443 | Path: `/vless` |
| Trojan WebSocket TLS | 443 | Path: `/trojan` |
| Vmess WebSocket (No TLS) | 80 | Path: `/vmess` |
| Vless WebSocket (No TLS) | 80 | Path: `/vless` |
| Trojan WebSocket (No TLS) | 80 | Path: `/trojan` |

### How port 443 is shared

Port **443** is shared by **Stunnel (SSH-SSL)** and **Nginx (TLS WebSocket protocols)** through **HAProxy**, which routes traffic using the TLS SNI value without decrypting the connection:

- **SNI matches the configured domain** → Nginx (WebSocket protocols)
- **No SNI or a different SNI** → Stunnel (SSH-SSL)

> **SSH-SSL client note:** In clients such as HTTP Injector or KPN Tunnel, leave the SNI/SAN field blank or enter a value other than your configured domain.

### Terminal control panel: `byteroot`

- Colorful live banner showing the date, time, domain, operating system, and CPU architecture
- Live count of currently connected SSH users
- Counters for SSH, Vmess, Vless, and Trojan accounts
- Create unlimited or restricted accounts with connection, data, and expiry limits
- Track per-account traffic usage in GB
- Enable, lock, or delete accounts
- Check service status and restart services individually or all at once
- Clear caches and system logs
- Cron-based enforcement to lock or remove expired accounts and accounts that exceed their data allowance

## Intended operating systems and architectures

These are the installer's intended targets, as designed. They are not independently verified: compatibility on a particular VPS also depends on the provider's image, available packages, and system configuration.

### Operating systems (intended)

| Distribution | Intended versions |
|---|---|
| Ubuntu | **22.04 through 26** |
| Debian | **10 and later** |

### CPU architectures (intended)

- **x86_64 / AMD64**
- **ARM64 / AArch64**

Other CPU architectures are not targeted. No specific VPS provider is certified or tested by this project; any provider offering a supported OS release on x86_64 or ARM64 may work, and no provider is endorsed.

---

## Installation

```bash
git clone https://github.com/PacketBreaker/byteroot-vpn.git
cd byteroot-vpn
chmod +x install.sh
sudo ./install.sh
```

During installation, you will be asked to enter your domain. The installer checks that its DNS A record points to the server's IP address and automatically requests an SSL certificate through Let's Encrypt.

After installation, the control panel opens automatically. You can return to it at any time by running:

```bash
byteroot
```

## Project structure

```text
byteroot-vpn/
├── install.sh              # Main self-contained installation and setup script
├── scripts/
│   ├── byteroot-menu.sh    # Reference copy of the terminal control panel
│   ├── enforce.sh          # Reference copy of expiry/data-limit enforcement (cron: every 5 minutes)
│   └── conn_limit.sh       # Reference copy of concurrent-connection limiter (cron: every minute)
├── LICENSE
└── README.md
```

> **Note:** `install.sh` is self-contained and creates the files listed above directly on the server during installation. The files in `scripts/` are reference copies for reviewing or editing the code before upload.

## Transport note: SSH "WebSocket" proxy

The SSH WebSocket proxy answers the client's HTTP request with `101 Switching Protocols` and then pipes raw traffic, without RFC 6455 WebSocket framing (the installer's own code comment calls it a "fake HTTP/WS upgrade request"). This is the style injector-type clients such as HTTP Injector or KPN Tunnel expect; standards-compliant WebSocket clients are not interoperable with it. The Vmess/Vless/Trojan paths go through Nginx and their respective cores with proper upgrade handling and are unaffected by this limitation.

## Known limitations

1. SSH data usage in GB is measured with `iptables` and counts **outbound traffic from the server only**. It is a common practical approximation, not a fully accurate measure of both upload and download traffic.
2. Connection limits for Vmess, Vless, and Trojan accounts are currently informational and are not automatically enforced. SSH connection limits are enforced through PAM and cron.
3. Expired accounts and accounts that exceed their data allowance are locked or removed by a cron job every five minutes; enforcement is not instantaneous.

## Support

For help or questions, contact us on Telegram: [@PacketBreaker](https://t.me/PacketBreaker)

[abdelrahmanrabie.24@gmail.com](mailto:abdelrahmanrabie.24@gmail.com)

## License

This project is licensed under the [MIT License](LICENSE). All rights reserved to **Dev. Eng Abdelrahman Rabie**.
