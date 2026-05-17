
# knock – TCP Port Knocking Client

A lightweight C client that sends a TCP SYN sequence to trigger `knockd` on a remote server.

## What it does

Sends a configurable sequence of TCP SYN packets to closed ports. When the server-side `knockd` daemon receives the correct sequence, it temporarily modifies firewall rules (e.g., opens SSH port 22) for the client's IP address.

## Why this exists

- Hide SSH from port scanners (defense in depth)
- Learn raw socket programming (TCP header construction, checksums)
- Understand how malware might implement covert C2 channels

## How it works

```
Client (your Kali)          Server (your VPS/home lab)
      |                              |
      | ----- SYN to port 1000 ----> | knockd logs: "port 1000"
      | ----- SYN to port 2000 ----> | knockd logs: "port 2000"  
      | ----- SYN to port 3000 ----> | knockd logs: "port 3000"
      |                              |
      |                              | knockd: "sequence matches!"
      |                              | iptables: ACCEPT from client IP:22
      | <------ SSH handshake ------- |
```

## Requirements (build time)

- Linux (Kali, Ubuntu, Debian, WSL2)
- GCC
- Root/sudo (raw sockets require privileges)

## Quick start

### 1. Clone and compile
```bash
git clone https://github.com/phyn2-2/port-knock-client.git
cd port-knock-client
make
sudo make install  # optional: copies to /usr/local/bin
```

### 2. On your remote server (one-time setup)
```bash
sudo apt install knockd
sudo tee /etc/knockd.conf > /dev/null <<EOF
[openSSH]
    sequence = 1000,2000,3000
    seq_timeout = 10
    command = /sbin/iptables -A INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags = syn
[closeSSH]
    sequence = 3000,2000,1000
    seq_timeout = 10
    command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
    tcpflags = syn
EOF
sudo systemctl enable knockd --now
```

### 3. Knock from your Kali
```bash
sudo ./knock 192.168.1.100 1000 2000 3000
ssh user@192.168.1.100  # SSH port now open for 10 seconds
```

## Usage
```bash
./knock <target IP> <port1> [port2] [port3] ...
```

**Examples:**
```bash
./knock 10.0.0.1 7000 8000 9000
./knock example.com 12345 23456
./knock 192.168.1.50 4444 5555 6666 7777
```

## C concepts you'll learn from reading this code

| Concept | Where to find it |
|---------|------------------|
| Raw sockets | `socket(AF_INET, SOCK_RAW, IPPROTO_TCP)` |
| TCP header construction | `struct tcphdr` |
| IP header construction | `struct iphdr` |
| Checksum calculation | `in_cksum()` function |
| Command-line parsing | `argc, argv[]` |
| Network byte order | `htons()`, `htonl()` |
| Privilege checking | `geteuid()` |
| Socket options | `setsockopt()` with `IP_HDRINCL` |

## Error handling

The program checks for:
- Not running as root → exits with instruction
- Invalid IP address format → shows usage
- Ports outside 1-65535 → rejects
- Network unreachable → prints error and continues to next knock

## Limitations (v1)

- IPv4 only
- No UDP/ICMP knocking
- No response detection (fire-and-forget)
- Requires `knockd` preconfigured on server
- No retry mechanism

## Future improvements (if you extend it)

- [ ] Add UDP support
- [ ] Read port list from file
- [ ] Add delay customization between knocks
- [ ] Listen for knock acknowledgement
- [ ] Encrypted sequence (HMAC)

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `Operation not permitted` | Run with `sudo` |
| Server SSH still closed after knock | Check `sudo systemctl status knockd` on server |
| Sequence works once then stops | Ensure `closeSSH` command has reverse sequence |
| `knockd` not logging | Check `sudo journalctl -u knockd -f` |

## Legal & ethical use

This tool is for:
- Securing your own servers
- Authorized penetration testing
- Learning network programming

Do not use against systems you don't own or lack written permission to test.

## License

MIT License – free to use, modify, and distribute with attribution.

## Author

Baphyn Magero / phyn2-2 / Kali Linux user

## Acknowledgments

- `knockd` developers for the server-side daemon
- TCP/IP Illustrated by Stevens (raw socket reference)
