# Log

<!--
## YYYY-MM-DD — short name of the problem
**Symptom:** exact error text / what I saw.
**Tried:** what I checked, in order, including the dead ends.
**Root cause:** what was actually wrong.
**Fix:** the change, with the command or config line.
**Proof:** log line, command output, or capture filter showing it worked.
-->

## 2026-09-28 — Lab addresses were never actually set

**Symptom:** On Ubuntu, `ens37` held `192.168.40.128/24` with `metric 1024`, but
`/etc/netplan/00-installer-config.yaml` only mentioned `ens33`. Nothing I had configured owned that
address, and Kali's lab connection was on DHCP too.

**Tried:**
- `networkctl status ens37` showed `Network File: /run/systemd/network/zzzz-dracut-default.network`.
- Ran `sudo netplan try` expecting a new `60-lab.yaml` to take over. It accepted, but `ens37` stayed
  on the dracut file. `ls /etc/netplan/` showed `60-lab.yaml` had never been created, and `ls
  /run/systemd/network/` had netplan output for `ens33` only. `sudo networkctl reconfigure ens37`
  changed nothing, because there was nothing new to switch to.
- Tried to create the file with a heredoc pasted into the VMware console. The console inserted a
  blank line and a leading space on every line, so the closing `EOF` never matched and the command
  hung until Ctrl+C. A one-line `printf` version didn't paste cleanly either.
- On Kali, `nmcli con mod "eth0" ...` failed with `unknown connection 'eth0'`, because `eth0` is the
  device and the connection is `Wired connection 1`. I then applied the address to `lo` by mistake.
  Removing it failed with `ipv4.method: method 'manual' requires at least an address`, so I set
  `lo` back to `127.0.0.1/8` instead.

**Root cause:** `ens37` was not declared anywhere. dracut writes a catch-all
`zzzz-dracut-default.network` into `/run` at every boot that runs DHCP on any interface nothing
else claims, so Ubuntu's lab address was just whatever VMware's DHCP handed out.

**Fix:**
- Ubuntu: `/etc/netplan/60-lab.yaml`, typed by hand in nano, mode 600, applied with
  `sudo netplan try`. systemd-networkd uses the first matching file by name, and
  `10-netplan-ens37.network` sorts before `zzzz-dracut-default.network`.
  ```yaml
  network:
    version: 2
    ethernets:
      ens37:
        dhcp4: false
        dhcp6: false
        addresses: [192.168.40.128/24]
  ```
- Kali: `sudo nmcli con mod "Wired connection 1" ipv4.method manual ipv4.addresses 192.168.40.129/24`

**Proof:** `networkctl status ens37` now shows
`Network File: /run/systemd/network/10-netplan-ens37.network`, and each host pings the other.

## 2026-09-28 — Kali's ssh never offered the Kerberos ticket

**Symptom:** `ssh -K alice@testserver1.lab.example` from Kali failed with
`Permission denied (publickey,gssapi-keyex,gssapi-with-mic)`. Kali held a valid
`krbtgt/LAB.EXAMPLE@LAB.EXAMPLE` ticket, and Windows could still reach the server over SSH.

**Tried:**
- `ssh -vK` showed the server offering `gssapi-with-mic`, but the client went straight to
  `Next authentication method: publickey`, then `No more authentication methods to try`. It never
  attempted GSSAPI, so there was no GSS error to read. The server side was already fine.
- `klist` on Kali: ticket valid until 09/29 04:53, so the ticket was not the problem.
- `ssh -G alice@testserver1.lab.example | grep -i gssapi` printed nothing. A client built with
  GSSAPI always lists those options, even when they are off, so this `ssh` had no GSSAPI support at
  all.
- `dpkg -l | grep openssh` showed `openssh-client 1:10.4p1-5` and `openssh-client-gssapi
  1:10.3p1-4`, which was installed but a version behind and marked architecture `all`.
- `dpkg -L openssh-client-gssapi` listed only files under `/usr/share/doc`. It was an empty stub.

**Root cause:** On Kali's current packages, the plain `openssh-client` build has no GSSAPI. Kerberos
support ships in `openssh-client-gssapi`, and the copy installed on this box was an old
architecture-`all` stub that only depended on `openssh-client`. `apt-cache show` listed the
`1:10.5p1-1` candidate as `amd64`, depending on `libgssapi-krb5-2`: that is the real build.

**Fix:** Added a temporary NAT adapter to Kali, ran `sudo apt install openssh-client-gssapi` to
move to `1:10.5p1-1`, then removed the adapter.

**Proof:** `ssh -G` now lists `gssapiauthentication yes` and the other GSSAPI options, and
`ssh -K alice@testserver1.lab.example` lands on the Ubuntu banner with no password prompt.

## 2026-09-28 — AppArmor kept tshark out of my home folder

**Symptom:** `sudo tshark -i ens37 -f "port 88 or port 22" -w ~/01-sso.pcapng` failed with
`The file to which the capture would be saved ("/home/administrator/01-sso.pcapng") could not be
opened: Permission denied`, even though it was running as root.

**Tried:**
- Had the shell write the file instead: `sudo tshark ... -w - > ~/01-sso.pcapng`. The capture
  worked, but `tshark -r ~/01-sso.pcapng` then failed with `You don't have permission to read the
  file`.
- `ls -l` showed `-rw-rw-r-- administrator administrator 43672`: my file, readable by me. File
  permissions were not the problem.
- `sudo aa-status` listed the profiles `tshark` and `tshark//dumpcap`.
- `sudo journalctl -k | grep -i denied` showed
  `apparmor="DENIED" operation="open" profile="tshark" name="/home/administrator/01-sso.pcapng"`.

**Root cause:** Ubuntu confines tshark with an AppArmor profile that doesn't allow it to open files
under `/home`, for reading or writing. AppArmor checks before normal file permissions, so being root
or owning the file made no difference.

**Fix:** Left the profile alone, since loosening a confinement profile would work against the point
of this lab. The shell opens the file and tshark only sees a stream:
- capture: `sudo tshark -i ens37 -f "port 88 or port 22" -w - > ~/01-sso.pcapng`
- read: `tshark -r - -Y kerberos < ~/01-sso.pcapng`

**Proof:** The read prints the full exchange: `AS-REQ`, `KRB5KDC_ERR_PREAUTH_REQUIRED`, `AS-REQ`,
`AS-REP`, then two `TGS-REQ`/`TGS-REP` pairs.
