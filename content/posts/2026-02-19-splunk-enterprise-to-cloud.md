+++
title = "Splunk Enterprise to Splunk Cloud Setup"
date = 2026-02-19T15:57:57-05:00
draft = false
+++

## Overview
Today we have a use case where firewall logs are forwarded to Splunk for centralized visibility and monitoring. The goal is to receive logs on-prem first, then securely forward them to Splunk Cloud.

## Architecture
### Components and Roles
- Firewall: Generates security logs (traffic, threat, and system events).
- Splunk Deployment Server (on-prem): Manages forwarder configuration and acts as the controlled on-prem forwarding point.
- Splunk Cloud: Central analytics and search platform for long-term operations.

### Data Flow
`Firewall -> Splunk Deployment Server (on-prem) -> Splunk Cloud`

## Environment
For this use case (firewall log forwarding through an on-prem Splunk layer into Splunk Cloud), a practical baseline is:
- At least 4 vCPU
- At least 8 GB RAM
- Sufficient free disk for queueing, logs, and short-term operational buffering

Our server is above that baseline, so it is a good fit for this deployment pattern. Since this node is forwarding-focused and not a primary indexing tier, the current sizing is appropriate for our immediate need.

## Prechecks
Before installation, I ran a structured pre-check to validate platform readiness and network reachability.

### System Pre-checks
1. `uname -a`  
Purpose: Quick OS/kernel/architecture fingerprint.

2. `echo "== Host ==" && hostnamectl && uname -r`  
Purpose: Host identity + OS version + kernel version.

3. `echo "== CPU ==" && nproc && lscpu | egrep 'Model name|Socket|Core|Thread|MHz'`  
Purpose: CPU capacity and topology (vCPU count, model, cores/threads/sockets).

4. `echo "== Memory ==" && free -h`  
Purpose: RAM and swap usage snapshot.

5. `echo "== Disk (/ /opt /var) ==" && df -h / /opt /var 2>/dev/null || df -h /`  
Purpose: Disk capacity/free space for key Splunk paths, with fallback if mounts do not exist.

6. `echo "== Inodes ==" && df -i / /opt /var 2>/dev/null || df -i /`  
Purpose: Inode usage check (prevents "disk looks free but files cannot be created" issues).

7. `echo "== Time sync ==" && timedatectl status | egrep 'System clock synchronized|NTP service|Time zone'`  
Purpose: Time/NTP health (critical for log/event timestamp accuracy).

8. `echo "== Open files limits ==" && ulimit -n && ulimit -u`  
Purpose: Process limits check (`nofile` and `nproc`) for Splunk readiness.

### Outbound Connectivity Test
```bash
# change targets to your real endpoints
TARGETS=(
  "download.splunk.com:443"
  "splunkbase.splunk.com:443"
  "es.<your_domain>.splunkcloud.com:443"
  "8.8.8.8:53"
  "time.google.com:123"
)

for t in "${TARGETS[@]}"; do
  h="${t%:*}"; p="${t##*:}"
  echo -n "$h:$p -> "
  nc -zvw3 "$h" "$p" >/dev/null 2>&1 && echo OK || echo FAIL
done
```

## Install Splunk Enterprise
Download path used:
- `https://www.splunk.com/en_us/customer-success.html` (login required)
- `Trials & Downloads` -> `Splunk Enterprise`
- Choose OS: Linux
- Choose package: `.deb` (validated using `uname -a`)
- Copy the provided `wget` URL

On the on-prem server:
- Change to `/tmp`
- Download with `wget`
- Install using `dpkg` and dependency fix

```bash
cd /tmp
wget -O splunk.deb "<copied_splunk_download_url>"
sudo dpkg -i splunk.deb
sudo apt-get -f install -y
```

Add Splunk to `PATH` so commands can run from any directory:

```bash
export SPLUNK_HOME=/opt/splunk
export PATH=$SPLUNK_HOME/bin:$PATH
```

First startup and initial setup:

```bash
cd /opt/splunk/bin
./splunk start --run-as-root
```

During first start:
- Accept the EULA
- Create the Splunk admin account for console access
- Save the password carefully to avoid reset/reinstall overhead

License install step (XML provided by Splunk Tech Ops/Support):

```bash
./splunk add licenses /tmp/<licensefilename>.xml --run-as-root
```

For hardened environments, review Splunk guidance on running as non-root and stricter service permissions. For this deployment, an internal non-routed server met the current operational and security needs.

## Configure Forwarding to Splunk Cloud (HA)
Final local `outputs.conf`:

```conf
[tcpout]
defaultGroup = splunkcloud_20260205_12345678901234567890123456789012
indexAndForward = false
```

This uses the Splunk Cloud app-provided HA input group (`inputs1.<tenant>.splunkcloud.com` through `inputs15.<tenant>.splunkcloud.com`) and avoids local indexing.

## Validate
```bash
splunk btool outputs list --debug
splunk list forward-server
```

Expected result:
- forward servers show `Active`
- events appear in Splunk Cloud search

## Notes
- A single target like `inputs1.<tenant>.splunkcloud.com` works, but the full HA group is preferred.
- `indexAndForward = false` ensures forward-only behavior.

## Next Steps
The next build step is enabling a UDP syslog input on Splunk Deployment Server / Splunk Enterprise, then pointing the firewall to that port.

In Splunk Web:
1. Go to `Settings` -> `Data inputs` -> `UDP`.
2. Click `New Local UDP`.
3. Choose UDP (a practical option for high-volume firewall logs with low overhead).
4. Select an available port on the server. I used a non-standard port such as `20512`.

To check currently listening ports before choosing one:

```bash
sudo ss -tuln
```

After input creation, configure the firewall syslog destination to the Splunk server IP and chosen UDP port.

Validation in Splunk Cloud:
- Search by index configured on the UDP input, for example: `index=<your_index> earliest=-15m`
- Or search by sourcetype, for example: `sourcetype=<your_sourcetype> earliest=-15m`

## Final Note
For a somewhat technical team, a first-time setup like this is realistic in about 2-4 hours. Our first full run took about 3 hours end-to-end.

I hope this walkthrough helps you get through it faster and avoid common mistakes, especially:
- Losing track of the console admin password (which can force a reset or reinstall)
- Overcomplicating `outputs.conf` when only two values were needed for our case

Salam, and see you on the next post.

`[ signed://ops ]` — The Cyber Engineer

