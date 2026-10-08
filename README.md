## 📌 Overview
Phase 1 establishes a privacy-focused, zero-telemetry local DNS architecture on an Ubuntu host system. By combining **AdGuard Home** (ad/malware sinkhole) and **Unbound** (recursive DNS resolver) with **UFW** (host-level network firewall), all local network DNS queries are sanitized, validated via DNSSEC, and resolved directly against global root servers without relying on upstream third-party resolvers (such as Cloudflare or Google).

     [ PHASE 1 ARCHITECTURE ]

                                  +-----------------------+
                                  |   LAN Clients / Laptops|
                                  +-----------+-----------+
                                              |
                                              | DNS Queries (Port 53 UDP/TCP)
                                              v
+---------------------------------------------------------------------------------------+
| Ubuntu Host (UFW Firewall)                                                            |
|                                                                                       |
|   +-------------------------------------------------------------------------------+   |
|   | AdGuard Home Container (Docker)                                               |   |
|   |  - Web UI: http://<HOST-IP>:80                                                |   |
|   |  - Blocks ads, tracking, and malware domains                                  |   |
|   +---------------------------------------+---------------------------------------+   |
|                                           |                                           |
|                                           | Upstream Lookups via Docker Bridge        |
|                                           v (172.17.0.1:5335)                         |
|   +-------------------------------------------------------------------------------+   |
|   | Unbound Recursive Resolver (Host Native)                                      |   |
|   |  - Performs iterative lookups (. -> .com -> domain)                           |   |
|   |  - Validates DNSSEC cryptographic signatures                                  |   |
|   |  - Protected via UFW (Restricted to 127.0.0.1 & 172.17.0.0/16 subnets)        |   |
|   +---------------------------------------+---------------------------------------+   |
|                                           |                                           |
+-------------------------------------------|-------------------------------------------+
                                            | Iterative Root Queries (Outbound Port 53)
                                            v
                                 +---------------------+
                                 |  Global Root DNS    |
                                 |  (. / Top-Level TLDs|
                                 +---------------------+
🔒 Key Features & Security ArchitectureTrue DNS Privacy & Independence: Direct iterative lookup chain eliminates upstream data logging and profiling by ISPs or public resolver operators.DNSSEC Cryptographic Enforcement: Hardened validation drops tampered or spoofed DNS records at the resolver level.DNS Rebinding Prevention: Configured private IP filtering inside Unbound blocks malicious external domains from mapping to local network subnets.Host Firewall Hardening: UFW restricts Unbound (5335) exclusively to local processes and the Docker bridge subnet (172.17.0.0/16), preventing open-resolver abuse on the local network.Persistent Storage: AdGuard settings and query log databases are mapped directly to host directories (~/adguard/conf and ~/adguard/work) for zero data loss during reboots or updates.   🛠️ Step-by-Step ImplementationStep 1: Install and Configure Native UnboundInstall Unbound on the Ubuntu host:Bashsudo apt update && sudo apt install unbound -y
Create /etc/unbound/unbound.conf.d/adguard-home.conf:Ini, TOMLserver:
    verbosity: 1
    interface: 0.0.0.0
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no

    # Access Control - Allow Local Host, LAN, and Docker Bridge Subnets
    access-control: 127.0.0.0/8 allow
    access-control: 172.16.0.0/12 allow
    access-control: 192.168.0.0/16 allow
    access-control: 10.0.0.0/8 allow

    # Security & Optimization
    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes

    # Prevent DNS Rebinding Attacks
    private-address: 192.168.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
Verify configuration syntax and restart Unbound:Bashsudo unbound-checkconf
sudo systemctl restart unbound
Step 2: Deploy AdGuard Home via DockerCreate host mount directories with proper ownership and launch the container:   Bashmkdir -p ~/adguard/work ~/adguard/conf

docker run -d \
  --name adguardhome \
  --restart unless-stopped \
  -v /home/$USER/adguard/work:/opt/adguardhome/work \
  -v /home/$USER/adguard/conf:/opt/adguardhome/conf \
  -p 53:53/tcp -p 53:53/udp \
  -p 80:80/tcp \
  -p 3000:3000/tcp \
  adguard/adguardhome
Step 3: Configure UFW Firewall RulesLock down incoming network access while protecting the Unbound port:Bash# Set Default Policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Management & Web Dashboard
sudo ufw allow 22/tcp comment 'Allow SSH'
sudo ufw allow 80/tcp comment 'Allow AdGuard Web UI'
sudo ufw allow 3000/tcp comment 'Allow AdGuard Initial Setup'

# Public LAN DNS Service
sudo ufw allow 53/tcp comment 'Allow DNS TCP'
sudo ufw allow 53/udp comment 'Allow DNS UDP'

# Protect Unbound (Port 5335) - Loopback & Docker Bridge Only
sudo ufw allow in on lo to any port 5335 proto udp comment 'Allow local Unbound UDP'
sudo ufw allow in on lo to any port 5335 proto tcp comment 'Allow local Unbound TCP'
sudo ufw allow from 172.17.0.0/16 to any port 5335 proto udp comment 'Allow Docker to Unbound UDP'
sudo ufw allow from 172.17.0.0/16 to any port 5335 proto tcp comment 'Allow Docker to Unbound TCP'

# Enable Firewall
sudo ufw enable
Step 4: Link AdGuard Home to UnboundOpen the AdGuard web UI at http://<HOST-IP>:80.   Navigate to Settings $\rightarrow$ DNS settings.Under Upstream DNS servers, enter the Docker bridge gateway pointing to Unbound:Plaintext172.17.0.1:5335
Scroll to DNS server configuration, enable DNSSEC, click Test upstreams, and click Save.🧪 Verification & TestingTest 1: Verify AdGuard to Unbound Inter-Container ResolutionTest lookup execution from inside the running container through the firewall to Unbound:Bashdocker exec -it adguardhome nslookup google.com 172.17.0.1:5335
Expected Result: Successfully returns valid A records for Google.Test 2: Verify DNSSEC Cryptographic EnforcementQuery an intentionally broken DNSSEC domain:Bashdig @127.0.0.1 -p 5335 sigfail.attester.labs.nic.cz
Expected Result: status: SERVFAIL (Proves invalid/tampered signatures are dropped).Query a valid DNSSEC-signed domain:Bashdig @127.0.0.1 -p 5335 nic.cz +dnssec
Expected Result: status: NOERROR with the ad (Authentic Data) flag present in the response header.🛡️ Firewall Policy Reference (sudo ufw status numbered)Rule / PortProtocolSourceAction / Purpose22TCPAnyAllow SSH administration80 / 3000TCPAnyAllow AdGuard Web UI   53TCP/UDPAnyAllow LAN DNS client queries   5335TCP/UDP127.0.0.1 / 172.17.0.0/16RESTRICTED: Internal Unbound resolver access only
