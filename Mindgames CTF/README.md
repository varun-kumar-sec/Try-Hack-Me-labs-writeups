# Walkthrough: TryHackMe - Mindgames CTF
## Reconnaissance & Initial Access
### 1. OpenVPN Tunnel Establishment
The attacker machine established an active VPN connection to the target network via OpenVPN:
```bash
sudo openvpn ap-south-1-varunkr578-regular.ovpn
```
Network Parameters:
- Assigned Local IP: 192.168.135.130
- Target IP: 10.48.177.194

2. Nmap Port Scanning
A full TCP port and service version scan was executed against ```10.48.177.194```:
```bash
nmap -sV -sC -p- 10.48.177.194
```
Scan Results:

Port	  |  State	| Service	| Version / Info
22/tcp	|Open	    | SSH	    | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp	|Open	    | HTTP	  | Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
