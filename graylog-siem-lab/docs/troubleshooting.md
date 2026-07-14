# Troubleshooting Log

Real, reproducible errors encountered during this build only.

---

### VM disk media selector confusion / duplicate .vdi files

**Date:** 2026-07-13
**Phase:** VM creation

**Symptom:**
Created a 40GB virtual disk for graylog01, but VirtualBox's storage attach dialog showed a 25GB graylog01.vdi already present, plus a separate 40GB file that had been auto-renamed (graylog01_1.vhd) to avoid a naming collision.

**Cause:**
The original disk creation attempt left a registered 25GB .vdi in VirtualBox's media registry. Creating a second disk at the correct 40GB size caused VirtualBox to auto-suffix the filename rather than overwrite, since the original was still registered.

**Fix:**
Attached the correct 40GB graylog01_1.vhd to the VM. Removed the orphaned 25GB graylog01.vdi via File > Tools > Virtual Media Manager, choosing to delete the file from disk (not just unregister it).

**Verification:**
Storage settings showed a single 40GB disk attached, virtual size 40GB / actual size ~82KB (dynamically allocated, confirmed correct).

---

### Locked out of own account - Num Lock off during password entry

**Date:** 2026-07-13
**Phase:** Ubuntu Server install / first login

**Symptom:**
"Login incorrect" at the graylog01 login prompt despite entering the correct username and password.

**Cause:**
Num Lock was off during the initial password creation in the Ubuntu installer, so numeric characters in the password did not register as typed.

**Fix:**
Verified Num Lock state before retrying login. No password reset needed once Num Lock was enabled.

**Verification:**
Successful login as cdub.

---

### DHCP lease missing default gateway route

**Date:** 2026-07-13
**Phase:** Network configuration

**Symptom:**
`ping 8.8.8.8` returned "Network is unreachable" despite graylog01 successfully receiving an IP address (192.168.56.105) via DHCP from pfSense.

**Cause:**
`ip route` showed only the local subnet route (192.168.56.0/24) with no default route to the gateway (192.168.56.2). DHCP lease did not populate the default gateway.

**Fix:**
Manually added the route at runtime (`sudo ip route add default via 192.168.56.2`) to confirm the diagnosis, then made it permanent by adding an explicit `routes` block to `/etc/netplan/00-installer-config.yaml`.

**Verification:**
`ip route` showed the default route persisting across reboots. `ping 8.8.8.8` succeeded.

---

### DHCP lease missing DNS server / hostname resolution failing

**Date:** 2026-07-13
**Phase:** Network configuration

**Symptom:**
`curl` and `ping` to hostnames (e.g. pgp.mongodb.com) failed with "Could not resolve host," while pings to raw IP addresses succeeded.

**Cause:**
`resolvectl status` showed `Current Scopes: none` and `Default Route: no` for the LAN interface - no DNS server was associated with it, despite DHCP normally providing one alongside the IP lease.

**Fix:**
Manually set DNS at runtime (`resolvectl dns enp0s3 192.168.56.2`) to confirm the diagnosis, then made it permanent by adding a `nameservers` block to the netplan config.

**Verification:**
`resolvectl status` showed a persistent "Current DNS Server" entry. Hostname resolution and `curl` to external hosts succeeded after reboot.

---

### graylog01 blocked by pfSense implicit deny

**Date:** 2026-07-13
**Phase:** Network configuration / pfSense rules

**Symptom:**
graylog01 could not reach pfSense's own LAN interface (192.168.56.2) or the internet, despite correct routing and DNS configuration. ARP resolution to pfSense succeeded (Layer 2 fine), but ICMP/TCP traffic failed (Layer 3 blocked).

**Cause:**
pfSense's LAN rule set only had an explicit allow rule scoped to Kali -> theone SSH. All other LAN traffic, including from newly added devices like graylog01, fell under the default-deny implicit rule.

**Fix:**
Added a temporary broad "Any" allow rule for graylog01 (192.168.56.105) in Firewall > Rules > LAN, to unblock package installation. Flagged for hardening to a least-privilege rule (HTTP/HTTPS only) once the Graylog stack is fully installed.

**Verification:**
`ping 192.168.56.2` and `ping 8.8.8.8` both succeeded after the rule was added and applied.

---

### MongoDB 7.0 crashes on start - SIGILL / core-dump (AVX requirement)

**Date:** 2026-07-13/14
**Phase:** MongoDB install

**Symptom:**
`systemctl status mongod` showed `Active: failed (Result: core-dump)`, with the process exiting on `code=dumped, signal=ILL` (illegal instruction) immediately after starting.

**Cause:**
MongoDB 5.0 and later require a CPU with AVX (Advanced Vector Extensions) instruction support. `grep avx /proc/cpuinfo` returned nothing, confirming AVX was not exposed to the VM - despite the host CPU (i7-8665U) natively supporting AVX2. Attempted fix via `VBoxManage modifyvm graylog01 --cpu-profile host` (confirmed applied via `showvminfo`) did not expose AVX to the guest, likely due to Hyper-V running underneath VirtualBox on the Windows host, which can limit CPU feature passthrough regardless of VirtualBox's own profile setting.

**Fix:**
Pivoted to MongoDB 4.4, the last major version released before the AVX requirement was introduced. Removed the 7.0 package/repo, added the 4.4 repo (using the `focal` release path), and reinstalled.

**Verification:**
`grep avx /proc/cpuinfo` remained empty (confirming the underlying limitation), but MongoDB 4.4 does not require AVX and installed/started without issue afterward.

---

### MongoDB 4.4 dependency conflict - libssl1.1 unavailable

**Date:** 2026-07-14
**Phase:** MongoDB install

**Symptom:**
`sudo apt install mongodb-org` failed with an unresolvable dependency: `mongodb-org-shell` requires `libssl1.1 (>= 1.1.0)`, which was not available in any configured repository.

**Cause:**
Ubuntu Server 26.04 ships with OpenSSL 3.0 by default and no longer includes `libssl1.1` in its standard repositories. MongoDB 4.4's packages were built against the older library.

**Fix:**
Manually downloaded and installed the `libssl1.1` .deb package directly from Ubuntu's package archive (security.ubuntu.com), then reran the MongoDB install.

**Verification:**
`sudo apt install -y mongodb-org` completed successfully. `sudo systemctl status mongod` showed `Active: active (running)` with no crash.

---

<!-- Add entries above this line as you hit real issues during the build -->
