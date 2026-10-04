# Forwarding FortiFirewall Logs to Bindplane

Hands-on lab: send FortiFirewall syslog events to a Bindplane agent running on Windows, with both VMs in VirtualBox.

**Author:** Saad Qamar, SOC Analyst
**Repository:** https://github.com/SaadJavedQamar/Forwarding-FortiFirewall-logs-to-Bindplane
**Date:** 4 October 2026

**Result:** FortiFirewall syslog events (admin login, system events) reach the Bindplane agent and appear in Bindplane Live Preview.

## Contents

1. [Overview and architecture](#1-overview-and-architecture)
2. [Building the FortiFirewall VM in VirtualBox](#2-building-the-fortifirewall-vm-in-virtualbox)
3. [Connectivity test](#3-connectivity-test)
4. [Syslog forwarding setup](#4-syslog-forwarding-setup)
5. [Result: logs in Bindplane](#5-result-logs-in-bindplane)
6. [Issues and fixes](#6-issues-and-fixes)
7. [Next steps](#7-next-steps)

## 1. Overview and architecture

The Bindplane agent was already installed on a Windows VM and Windows Event Logs were flowing. The goal was to send a firewall's logs through the same agent. A Fortinet VM (FortiFirewall v8.0.1) running in VirtualBox was used as the lab firewall.

| Component | Detail |
|---|---|
| Bindplane agent (collector) | Windows VM (VirtualBox, Bridged), IP `192.168.100.42`, collector name `DESKTOP-QHSR9HE` |
| Firewall | FortiFirewall VM (VirtualBox, Bridged, built from the KVM image), IP `192.168.100.44` |
| Log path | FortiFirewall -> Syslog UDP 514 -> Bindplane agent -> Bindplane configuration `Windows_logs` |
| Destination (currently) | `Dev_Null` (test only, logs are not stored) |

## 2. Building the FortiFirewall VM in VirtualBox

### 2.1 Download the KVM image

VirtualBox has no official Fortinet image, so the **KVM** image is downloaded and converted.

You can download the KVM image from the Fortinet support site: sign in to FortiCare, then go to **Downloads -> VM Images** (`support.fortinet.com/support/#/downloads/vm`). Choose the product, set **Select Platform** to **KVM**, and pick **"New deployment ... KVM"** (not "Upgrade from previous version"). This lab used `FFW_VM64_KVM-v8.0.1.F-build0245-FORTINET.out.kvm.zip` (137.84 MB). The same page also lists FortiGate KVM images.

![Fortinet VM Images page, KVM platform, New deployment download underlined](images/01-fortinet-kvm-download.png)

The zip contains `fortios.qcow2`. Copy it to a local folder such as `C:\VMs\FFW_KVM` (not OneDrive, to avoid sync problems).

![fortios.qcow2 inside the KVM zip](images/02-kvm-zip-contents.png)

### 2.2 Convert the qcow2 image to VDI

```
cd "C:\Program Files\Oracle\VirtualBox"
VBoxManage clonemedium disk "C:\VMs\FFW_KVM\fortios.qcow2" "C:\VMs\FFW_KVM\fortios.vdi" --format VDI
```

![VBoxManage conversion succeeded](images/03-vdi-conversion.png)

### 2.3 Create the VM

New VirtualBox VM: Type **Linux**, Version **Other Linux (64-bit)**, RAM 2048 MB. On the Hard Disk step, select **"Use an Existing Virtual Hard Disk File"** and choose `fortios.vdi`. The default (create a new blank disk) will not boot.

![Create Virtual Machine wizard, Hard Disk section](images/04-create-vm-disk.png)

Then open the VM settings:

- System: Chipset PIIX3, I/O APIC enabled, EFI off, Hard Disk first in boot order
- System -> Acceleration: Paravirtualization **KVM**
- Network: **Bridged Adapter**, adapter type Intel PRO/1000 MT Desktop (so the firewall is on the same LAN as the Windows VM)

![VirtualBox Motherboard settings](images/05-virtualbox-motherboard.png)

### 2.4 First boot

Log in on the console as `admin` with a blank password, then run `get system interface physical`. port1 received `192.168.100.44` from the router, which confirms Bridged mode works. The console also warned: *"License invalid due to exceeding allowed 1 CPUs and 2048 MB RAM"* (see [Next steps](#7-next-steps)).

![First boot, port1 = 192.168.100.44](images/06-first-boot.png)

## 3. Connectivity test

The Windows VM (Bindplane agent) is at `192.168.100.42` on the same subnet. It could ping the firewall with 0% loss. Pinging from the firewall to Windows failed because Windows Firewall blocks incoming ICMP. This does not affect syslog, which uses UDP 514.

![Firewall pinging Windows: 100% loss](images/07-ping-from-firewall.png)

![Windows ping to the firewall: 4/4 replies](images/08-ping-from-windows.png)

## 4. Syslog forwarding setup

### 4.1 Syslog on the firewall

```
config log syslogd setting
set status enable
set server "192.168.100.42"
set mode udp
set port 514
end
```

![Syslog configuration in the FortiFirewall CLI](images/09-firewall-syslog-config.png)

### 4.2 Inbound rule on the Windows VM

PowerShell as Administrator:

```powershell
New-NetFirewallRule -DisplayName "Bindplane Syslog" -Direction Inbound -Protocol UDP -LocalPort 514 -Action Allow
```

![Windows Firewall rule created](images/10-windows-inbound-rule.png)

### 4.3 Syslog source in Bindplane

In the `Windows_logs` configuration, add a **Syslog** source:

| Setting | Value |
|---|---|
| Listening IP Address | `0.0.0.0` |
| Listening Port | `514` |
| Protocol | `rfc3164` |
| Transport Protocol | `udp` |
| Parse To | `body` |

Binding to a port below 1024 requires the collector to run as Administrator on Windows.

![Bindplane Add Source: Syslog](images/11-bindplane-syslog-source.png)

In the draft pipeline, the Syslog line did not reach the destination and the status was "Rollout Pending".

![Draft pipeline: Syslog not connected, rollout pending](images/12-pipeline-draft.png)

After connecting Syslog to the destination and pressing **Start Rollout**, both sources feed the destination (Windows Events 5.1 KB/m, Syslog 1.9 KB/m).

![Live pipeline after rollout](images/13-pipeline-live.png)

### 4.4 Confirm the agent is listening

```
netstat -an | findstr 514
```

![UDP 514 listening on 0.0.0.0 and [::]](images/14-netstat-514.png)

## 5. Result: logs in Bindplane

Logging in to the firewall GUI (`https://192.168.100.44`) as admin generated an event log that appeared in Bindplane Live Preview. It includes `logid="0100032001"`, `logdesc="Admin login successful"`, `srcip=192.168.100.13`, `dstip=192.168.100.44` and `user="admin"` in FortiOS key=value format.

![Live Preview: FortiOS admin login event](images/15-bindplane-live-preview.png)

![Manage Processors panel of the Syslog source (no processors yet)](images/16-bindplane-processors.png)

## 6. Issues and fixes

| Issue | Cause | Fix |
|---|---|---|
| VM would not boot from a fresh disk | Wizard default creates a blank disk | Choose "Use an Existing Virtual Hard Disk File" and select `fortios.vdi` |
| Firewall and agent on different networks | Host-only networking isolates the VM from the Windows VM | Put both VMs on Bridged Adapter, same LAN |
| Ping from firewall failed | Windows Firewall blocks ICMP | Ping is not needed for syslog; added an inbound UDP 514 rule |
| Syslog not linked to Dev_Null | Line incomplete in the draft | Connected the source to the destination and rolled out |
| License invalid warning | VM has more than 1 CPU / 2048 MB RAM | Reduce to 1 CPU and slightly less RAM, then recheck |

### GUI login and license

The GUI first showed "Authentication failure" (a password issue), and a later login worked. The GUI then showed only the License page because the VM license is invalid.

![GUI: Authentication failure](images/17-gui-auth-failure.png)

![GUI login page](images/18-gui-login.png)

![License page: VM is not licensed or license is invalid for current VM configuration](images/19-license-page.png)

> **Note:** syslog events arrived even with the license invalid, but other firewall features may stay limited until it is fixed.

## 7. Next steps

1. **Fix the license:** power off the VM, set CPUs to 1, lower RAM slightly (for example 2000 MB), then check License Status with `get system status`. If it is still invalid, a valid `.lic` file or a FortiGate-VM evaluation will be needed.
2. **Add a real destination:** `Dev_Null` discards logs. Add Google Cloud Logging, Splunk, Elastic, Loki or OTLP.
3. **Parse the logs:** they are raw key=value text. Add a Bindplane processor (Parse Key Value) so `srcip`, `dstip` and `action` become separate fields.
4. **Generate more logs:** enable *Log Allowed Traffic -> All Sessions* on a firewall policy and send traffic through it.
5. **Stable IP:** use a DHCP reservation or static IP for firewall port1 so the syslog target does not change after a restart.

## Author

Saad Qamar, SOC Analyst
