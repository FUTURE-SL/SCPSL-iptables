# SCP: Secret Laboratory iptables-protection
Important: This solution will NOT protect you from:
1. Spoofing
2. Bandwidth saturation
3. Large-scale or advanced botnet attacks
(if you need protection from spoofing, contact me privately)

Guide:
1. Install dependencies: `apt update && apt install -y iptables ipset`
2. Setup rules:
```
sudo ipset create sl_ports bitmap:port range 1-65535
sudo ipset create sl_sessions hash:ip,port timeout 10

sudo iptables -N SL-Filter
sudo iptables -N SL-Session
sudo iptables -N SL-Response

sudo iptables -A SL-Filter -f -j DROP
sudo iptables -A SL-Filter -m set --match-set sl_sessions src,src -j SL-Session
sudo iptables -A SL-Filter -m length --length 67:65535 -m u32 --u32 "0>>22&0x3C@8&0xFF000000=0x05000000" -m hashlimit --hashlimit-upto 2/second --hashlimit-burst 2 --hashlimit-mode srcip --hashlimit-name conn-limit -j ACCEPT
sudo iptables -A SL-Filter -m length --length 34:65535 -m u32 --u32 "0>>22&0x3C@8&0xFF000000=0xff000000" -m hashlimit --hashlimit-upto 2/second --hashlimit-burst 2 --hashlimit-mode srcip --hashlimit-name seq-limit -j ACCEPT
sudo iptables -A SL-Filter -j DROP

sudo iptables -A SL-Session -j SET --add-set sl_sessions src,src --exist
sudo iptables -A SL-Session -j ACCEPT

sudo iptables -A SL-Response -m u32 --u32 "0>>22&0x3C@8&0xFF000000=0x06000000" -j SET --add-set sl_sessions dst,dst
sudo iptables -A SL-Response -m length --length 29 -m string --hex-string "|0d|" --algo bm --from 28 --to 29 -j SET --del-set sl_sessions dst,dst
```
3. Initialization:

# For host services
```
sudo iptables -I INPUT -p udp -m set --match-set sl_ports dst -j SL-Filter
sudo iptables -I OUTPUT -p udp -m set --match-set sl_ports src -j SL-Response
```

# For Docker containers
```
sudo iptables -I DOCKER-USER -p udp -m set --match-set sl_ports dst -j SL-Filter
sudo iptables -I DOCKER-USER -p udp -m set --match-set sl_ports src -j SL-Response
```

4. Add each SL port to protection: `ipset add sl_ports <port>`

If you are NOT using NAT, Docker, or VPN tunneling, you can additionally execute:
```
sudo iptables -t raw -A PREROUTING -p udp -m set --match-set sl_ports dst -j NOTRACK
sudo iptables -t raw -A OUTPUT -p udp -m set --match-set sl_ports src -j NOTRACK
```
This bypasses the conntrack state table for these packets and reduces CPU usage during high traffic or query floods.
