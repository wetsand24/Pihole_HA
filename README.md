# Pi-hole High Availability: two Pi-holes, one IP

Two Pi-holes share one floating IP with keepalived. If the main one dies (the machine, its network, or just
Pi-hole itself), the standby takes over the IP within seconds. Your devices only ever know the floating IP,
so nothing changes on them. A small script copies the main Pi-hole's settings to the standby every hour.

Every command and file from the video, in order. The addresses below are the ones from the video:

| | IP | Role |
|---|---|---|
| `pihole1` | `10.37.0.41` | main |
| `pihole2` | `10.37.0.42` | standby |
| floating IP | `10.37.0.40` | what your devices use as DNS |

Replace them with yours.

## What you need
- Two Pi-holes (v6) on **two separate machines**, each with a static IP and the same web password.
  Not set up yet? Follow [Pihole_Install](https://github.com/wetsand24/Pihole_Install) twice.
- One free IP for the floating IP, **outside your router's DHCP range**.
- Both on the same network (same subnet).

## 1. Install keepalived (both Pi-holes)
```bash
apt install -y keepalived
ip -br a
```
Note your network card's name, the first column of `ip -br a` (e.g. `eth0`, `ens18`). In a Proxmox container it
shows as `eth0@if13`; the name to use is still `eth0`.

## 2. Health check (both Pi-holes)
Without it, the floating IP stays on a Pi-hole that crashed while its machine kept running.
```bash
nano /usr/local/bin/chk-ftl
```
Paste:
```bash
#!/bin/bash
dig @127.0.0.1 pi.hole +short +time=1 +tries=1 | grep -qE '^[0-9.]+$'
```
`Ctrl+O`, `Enter`, `Ctrl+X`, then:
```bash
chmod +x /usr/local/bin/chk-ftl
chk-ftl; echo $?
```
`0` = healthy. (It checks for a real IP in the answer. A plain "is the output empty" test doesn't work, because
`dig` prints its "connection refused" error on normal output, so a stopped Pi-hole would still look healthy.)

`dig` comes with Pi-hole. If it's missing: `apt install -y dnsutils`.

## 3. keepalived on pihole1 (main)
```bash
nano /etc/keepalived/keepalived.conf
```
Paste [keepalived-pihole1.conf](keepalived-pihole1.conf), change `interface`, `unicast_src_ip`, `unicast_peer`
and `virtual_ipaddress` to yours, save, then:
```bash
systemctl restart keepalived
ip -br a
```
pihole1 now shows two addresses: its own and the floating IP.

## 4. keepalived on pihole2 (standby)
Paste [keepalived-pihole2.conf](keepalived-pihole2.conf). Three differences from pihole1:
- `priority 100` (lower than 150)
- `unicast_src_ip` and `unicast_peer` swapped
- **no `preempt_delay` line.** With it, the standby waits before taking over. In my test it took about
  40 seconds instead of about 9.
```bash
systemctl restart keepalived
ip -br a
```
pihole2 shows only its own address. That's correct, it's the standby.

## 5. Point your devices at the floating IP
Windows: `Win + R` → `ncpa.cpl` → your adapter → Properties → IPv4 → Preferred DNS `10.37.0.40`,
leave Alternate **empty** → OK.
```
nslookup doubleclick.net
```
`0.0.0.0` from Server `10.37.0.40` = working.

Whole house: put only `10.37.0.40` as the DNS server in your router's DHCP settings. Don't add a public DNS
next to it; devices use it at random and ads get through.

## 6. Keep them in sync (pihole1 only)
[pihole-sync](pihole-sync) uses Pi-hole's own Teleporter backup: it exports from the main Pi-hole and imports
into the standby (adlists, allow/deny lists, local DNS, groups, clients, settings; not DHCP leases).
```bash
apt install -y jq
wget -O /usr/local/bin/pihole-sync https://raw.githubusercontent.com/wetsand24/Pihole_HA/main/pihole-sync
chmod +x /usr/local/bin/pihole-sync
nano /usr/local/bin/pihole-sync
```
Set the three lines at the top: `PEER` (standby IP), `FLOATING` (floating IP), `INTERFACE` (network card).

Your Pi-hole web password (the same on both):
```bash
nano /root/.pihole-sync.secret
chmod 600 /root/.pihole-sync.secret
```
Run it and check the log:
```bash
pihole-sync
tail -n 1 /var/log/pihole-sync.log
```
Run it every hour, at 17 minutes past:
```bash
echo "17 * * * * root /usr/local/bin/pihole-sync" > /etc/cron.d/pihole-sync
```
The script only runs while this Pi-hole holds the floating IP, so it never copies the standby's settings back
over the main one after a failover. Make your changes on the main Pi-hole; changes on the standby get replaced.

Prefer Docker? [Nebula Sync](https://github.com/lovelaze/nebula-sync) does the same job.

## 7. Test it
On your PC, look up a website every second:
```powershell
while ($true) { $t = Get-Date -Format HH:mm:ss; try { $r = Resolve-DnsName google.com -Server 10.37.0.40 -Type A -QuickTimeout -ErrorAction Stop; "$t  OK  $($r[0].IPAddress)" } catch { "$t  FAILED" }; Start-Sleep 1 }
```
**Test 1, the machine loses its network.** On pihole1: `ip link set eth0 down`. Lookups pause for a few
seconds, then pihole2 answers and `ip -br a` on pihole2 shows the floating IP. Bring it back with
`ip link set eth0 up` (from the console). After the 30 s `preempt_delay`, pihole1 takes the IP back.

**Test 2, only Pi-hole stops.** On pihole1: `systemctl stop pihole-FTL`. The health check fails 3 times
(about 6 s), and pihole2 takes over. `systemctl start pihole-FTL` to bring it back.

## Good to know
- Two separate machines. Two Pi-holes on one computer go down together, and a power cut takes both, so this
  doesn't replace a UPS.
- Each Pi-hole keeps its own query log and statistics.
- This only protects DNS. Anything else on the dead server stays down.

## Files
| File | Goes to |
|---|---|
| [chk-ftl](chk-ftl) | `/usr/local/bin/chk-ftl` on both |
| [keepalived-pihole1.conf](keepalived-pihole1.conf) | `/etc/keepalived/keepalived.conf` on pihole1 |
| [keepalived-pihole2.conf](keepalived-pihole2.conf) | `/etc/keepalived/keepalived.conf` on pihole2 |
| [pihole-sync](pihole-sync) | `/usr/local/bin/pihole-sync` on pihole1 |
