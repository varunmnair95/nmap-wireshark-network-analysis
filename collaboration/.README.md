# 🤝 Collaboration, Comparison & Network Analysis

## 🎯 Purpose

This document records **my individual network analysis and collaborative comparison** for the Nmap & Wireshark project.

The project uses an independent Kali Linux and Windows 11 lab environment. The same general methodology was followed by all three participants, while each participant maintained their own configuration, captures, screenshots, and results.

The main objective was not simply to run Nmap, but to understand:

```text
Nmap Command
      ↓
Generated Network Traffic
      ↓
Wireshark Capture
      ↓
Packet-Level Evidence
      ↓
Nmap Result
      ↓
Interpretation
```

This approach allows a scan result to be supported by actual network evidence.

---

# 🖥️ My Lab Environment

| Component              | Details                  |
| ---------------------- | ------------------------ |
| 🐧 Analysis / Scanning | Kali Linux               |
| 🪟 Target              | Windows 11               |
| 💻 Virtualization      | UTM                      |
| 🌐 Network             | Isolated virtual network |
| 🔍 Scanner             | Nmap                     |
| 📡 Packet Analysis     | Wireshark                |
| Kali IP                | `192.168.128.3`          |
| Windows IP             | `192.168.128.4`          |

The lab was designed as a controlled environment between the two systems. My baseline documentation contains the network configuration, connectivity verification, listening-port information, and topology.

📁 [Baseline & Topology](../baseline-and-topology/)

---

# 🔬 Analysis Method

For each experiment, I treated Nmap as the **source of the scan result** and Wireshark as the **source of packet-level evidence**.

The basic investigation questions were:

1. What command was executed?
2. What type of probe did Nmap generate?
3. What did the target return?
4. What did Nmap report?
5. Does the packet capture support the result?
6. If the result is ambiguous, what could explain it?
7. Would another scan type provide additional evidence?

---

# 01 — Host Discovery

Host discovery determines whether a target appears to be active before performing a port scan.

The important point is that Nmap's concept of "ping" is broader than the traditional `ping` command.

### ARP vs ICMP

For a target on the same local Ethernet network, Nmap can use **ARP-based discovery**.

```text
Kali
  │
  │ ARP Request
  │ "Who has 192.168.128.4?"
  ▼
Windows
  │
  │ ARP Reply
  ▼
Kali
```

This is different from ICMP Echo:

```text
Kali
  │
  │ ICMP Echo Request
  ▼
Windows
  │
  │ ICMP Echo Reply
  ▼
Kali
```

ARP is a Layer 2 mechanism used to resolve an IPv4 address to a MAC address on the local Ethernet network. ICMP Echo operates at Layer 3.

Therefore, seeing ARP instead of ICMP in a local-network Nmap capture is not unexpected.

Nmap's documentation specifically notes that ARP is used for IPv4 targets on local Ethernet networks because it is generally faster and more effective there.

### Important distinction

| Evidence        | What it means                                                                  |
| --------------- | ------------------------------------------------------------------------------ |
| ARP Reply       | Target responded at the local Ethernet level                                   |
| ICMP Echo Reply | Target responded to an ICMP Echo probe                                         |
| TCP SYN/ACK     | A TCP service responded to the discovery probe                                 |
| TCP RST         | The host responded even though that particular TCP port may be closed          |
| No response     | Insufficient evidence by itself; filtering or other conditions may be involved |

A host does **not** need to respond to ICMP Echo for Nmap to determine that it is alive.

---

# 🔎 Useful Host-Discovery Switches

The following switches are useful when investigating different discovery behaviour.

| Switch               | Purpose                                         |
| -------------------- | ----------------------------------------------- |
| `-sn`                | Host discovery only; disables port scanning     |
| `-Pn`                | Treat targets as online and skip host discovery |
| `-PE`                | ICMP Echo discovery                             |
| `-PP`                | ICMP Timestamp discovery                        |
| `-PS`                | TCP SYN discovery                               |
| `-PA`                | TCP ACK discovery                               |
| `-PU`                | UDP discovery                                   |
| `--disable-arp-ping` | Disable ARP discovery where applicable          |

These should be considered **comparison/diagnostic options**, not evidence that all were used in the completed experiments.

Nmap supports combining several discovery probe types when a network may block some forms of traffic.

### Example comparisons

```bash
nmap -sn 192.168.128.4
```

Basic host discovery.

```bash
nmap -sn -PE 192.168.128.4
```

ICMP Echo-based discovery.

```bash
nmap -sn -PS80,443 192.168.128.4
```

TCP SYN discovery against selected ports.

```bash
nmap -Pn 192.168.128.4
```

Skip host discovery and proceed as though the host is online.

### Why `-Pn` matters

If a host is alive but discovery probes are blocked, normal Nmap discovery may fail to identify it as available.

`-Pn` changes the question from:

> "Can I prove that this host is online using discovery probes?"

to:

> "Assume this host is online and continue scanning."

This can increase scan time because Nmap may scan targets that are actually unavailable.

---

# 02 — Port Scanning

Once the target is considered available, port scanning determines which TCP/UDP ports appear accessible.

The important analytical relationship is:

```text
Nmap Probe
    ↓
Target Response
    ↓
Wireshark Packet
    ↓
Port State
```

## TCP SYN Scan

A SYN scan is represented by:

```text
Kali → Windows : SYN
Windows → Kali : SYN/ACK
```

For an open TCP port, a SYN/ACK is strong evidence that a service is accepting connections.

For a closed port:

```text
Kali → Windows : SYN
Windows → Kali : RST
```

For filtering, the expected response may be unavailable.

The important point is that **filtered does not automatically prove that a specific firewall blocked the packet**. It means Nmap does not have enough usable evidence to determine the port state. Firewall filtering is one common explanation.

---

# 🔎 TCP Scan Comparison

| Scan    | Switch | Main purpose                     | Packet behaviour                  |
| ------- | ------ | -------------------------------- | --------------------------------- |
| SYN     | `-sS`  | TCP port-state discovery         | SYN → SYN/ACK or RST              |
| Connect | `-sT`  | Full TCP connection              | SYN → SYN/ACK → ACK               |
| ACK     | `-sA`  | Firewall/filtering investigation | ACK → commonly RST                |
| UDP     | `-sU`  | UDP service discovery            | UDP probe → response/ICMP/silence |

### `-sS` vs `-sT`

The important Wireshark difference is the TCP handshake.

**SYN scan:**

```text
SYN
 ↓
SYN/ACK
 ↓
scan concludes
```

**Connect scan:**

```text
SYN
 ↓
SYN/ACK
 ↓
ACK
 ↓
TCP connection established
```

A connect scan therefore completes the normal TCP handshake through the operating system.

This makes Wireshark particularly useful for proving that two Nmap scan types are not simply different names for the same packet behaviour.

---

# 🧱 Firewall Comparison

Firewall state can significantly change the observed result.

A useful controlled comparison is:

```text
Firewall State A
      ↓
Nmap Scan
      ↓
Wireshark Capture

Firewall State B
      ↓
Same Nmap Scan
      ↓
Wireshark Capture

          ↓
      Compare
```

The purpose is to determine whether the difference is caused by:

* Port state
* Firewall filtering
* Service availability
* Host behaviour
* Network conditions

The Windows firewall should therefore be treated as an experimental variable rather than assuming that every `filtered` result is automatically caused by it.

---

# 03 — Service Detection

Port scanning answers:

> "What ports appear accessible?"

Service detection asks:

> "What service/application appears to be operating on those ports?"

The principal option is:

```bash
nmap -sV 192.168.128.4
```

`-sV` sends additional service-aware probes.

This is important for packet analysis because the traffic generated by `-sV` can be more than the initial TCP SYN used for port discovery.

The investigation therefore becomes:

```text
Port discovered
      ↓
Service detection probe
      ↓
Target response
      ↓
Wireshark
      ↓
Service identification
```

A familiar port number alone does not prove that the expected service is actually operating there.

---

# 🧪 Additional Nmap Options for Analysis

These options are useful when a result requires more evidence.

| Option       | Purpose                                     |
| ------------ | ------------------------------------------- |
| `-p <ports>` | Scan specific ports                         |
| `-p-`        | Scan the full TCP port range                |
| `-F`         | Fast scan of a smaller set of common ports  |
| `-sS`        | TCP SYN scan                                |
| `-sT`        | TCP Connect scan                            |
| `-sU`        | UDP scan                                    |
| `-sV`        | Service/version detection                   |
| `-sA`        | ACK scan for filtering analysis             |
| `-sF`        | FIN scan                                    |
| `-sN`        | NULL scan                                   |
| `-sX`        | Xmas scan                                   |
| `-O`         | OS detection                                |
| `-A`         | Enables several advanced detection features |

These should be viewed as **analysis and follow-up techniques** unless they are explicitly documented in the experiment evidence as performed.

---

# 📡 UDP Analysis

UDP behaves differently from TCP because there is no TCP-style handshake.

Typical possibilities are:

```text
UDP Probe
   ↓
UDP Response
   ↓
open
```

or:

```text
UDP Probe
   ↓
ICMP Port Unreachable
   ↓
closed
```

or:

```text
UDP Probe
   ↓
No Response
   ↓
open|filtered
```

The last case is important.

`open|filtered` does **not** mean "the port is open."

It means the available evidence was insufficient to distinguish an open but silent service from filtering.

---

# 🧩 FIN / NULL / Xmas

These scans are useful as a later comparison exercise:

```text
-sF → FIN
-sN → no TCP flags
-sX → FIN + PSH + URG
```

The Wireshark analysis should focus on the exact TCP flags rather than simply trusting the Nmap output.

For example:

```text
Nmap -sF
    ↓
TCP packet with FIN
    ↓
Target response
    ↓
Nmap state
```

These scans can be less predictable depending on the target operating system and filtering behaviour, so they are better treated as comparative analysis rather than replacements for the normal SYN scan.

---

# 🔐 ACK Scan

The ACK scan is particularly useful for understanding firewall behaviour.

```bash
nmap -sA -p <PORT> 192.168.128.4
```

The important distinction is:

> **ACK scan is primarily about filtering, not finding open services.**

A common packet pattern is:

```text
Kali → Windows : TCP ACK
Windows → Kali : TCP RST
```

A response such as RST indicates that the probe reached the target TCP stack and is consistent with an **unfiltered** result.

It does **not** mean that the port is open.

Comparing:

```bash
nmap -sS -p <PORT> 192.168.128.4
```

with:

```bash
nmap -sA -p <PORT> 192.168.128.4
```

is useful because the two scans answer different questions.

---

# 🦈 Wireshark Analysis

The main Wireshark workflow is:

```text
Capture
  ↓
Filter
  ↓
Identify Probe
  ↓
Identify Response
  ↓
Inspect TCP/IP Details
  ↓
Compare with Nmap
```

Useful display filters include:

```text
ip.addr == 192.168.128.4
```

```text
arp
```

```text
icmp
```

```text
tcp
```

```text
tcp.port == 80
```

```text
tcp.flags.syn == 1
```

```text
tcp.flags.reset == 1
```

```text
udp
```

These filters reduce unrelated traffic and make it easier to correlate Nmap activity with packet evidence. Wireshark display filters operate on an existing capture; capture filters determine what is recorded in the first place.

---

# 📊 Collaborative Comparison

The three labs use the same general project progression:

```text
Baseline
   ↓
Host Discovery
   ↓
Port Scanning
   ↓
Service Detection
   ↓
Packet Analysis
```

However, the environments are independent.

| Participant | Virtualization | Kali             | Windows          |
| ----------- | -------------- | ---------------- | ---------------- |
| Varun       | UTM            | `192.168.128.3`  | `192.168.128.4`  |
| Manu        | VirtualBox     | `192.168.100.50` | `192.168.100.5`  |
| Hari        | VirtualBox     | `192.168.100.10` | `192.168.100.20` |

The different addressing and virtualization platforms mean that identical packet counts or timings should **not** be expected.

The meaningful comparison is:

* Was the same methodology followed?
* Was the expected protocol behaviour observed?
* Does Wireshark support the Nmap result?
* Can differences be explained by the environment?
* Were conclusions based on evidence rather than assumptions?

---

# 👥 Other Participants

### 👨‍💻 Hari Krishnan R K

Hari maintained an independent VirtualBox lab and performed the same three-stage analysis.

📄 [Hari — Collaboration & Network Analysis](https://github.com/harikrishnan-rk/nmap-wireshark-network-analysis)

### 👨‍💻 Manu P Nair

Manu maintained an independent VirtualBox lab with a different IP configuration and his own evidence.

📄 [Manu — Collaboration & Network Analysis](https://github.com/manunair16/nmap-wireshark-network-analysis)

Their files contain their own detailed observations and should be referred to rather than duplicating their analysis here.

---

# 🧠 My Main Analytical Takeaways

The most important lesson from the project is that **Nmap output should be treated as an interpretation of network behaviour, not as the packet evidence itself**.

For example:

```text
Nmap says: open
        ↓
Look for SYN/ACK
        ↓
Wireshark confirms response
        ↓
Evidence supports result
```

Likewise:

```text
Nmap says: filtered
        ↓
Look for missing response / filtering evidence
        ↓
Consider firewall and network conditions
        ↓
Do not automatically claim "firewall blocked it"
```

The same reasoning applies to UDP, ACK, FIN, NULL, and Xmas scans.

---

# 🎓 Skills Demonstrated

* 🔎 Nmap host discovery
* 🚪 TCP/UDP port scanning
* 🛠️ Service detection
* 📡 Wireshark packet analysis
* 🧩 TCP flag interpretation
* 🌐 ARP and ICMP analysis
* 🔥 Firewall behaviour analysis
* 📊 Cross-lab comparison
* 🧪 Controlled experimentation
* 📝 Evidence-based documentation

---

# 📌 Conclusion

My implementation demonstrates the complete process:

**Generate traffic → capture packets → identify protocol behaviour → compare with Nmap → explain the result.**

The collaborative aspect adds another layer by comparing the same methodology across three independently configured environments.

The project therefore demonstrates not only the ability to use Nmap and Wireshark, but also the more important analytical skill of **connecting tool output to observable network evidence**.

---

## ⚠️ Disclaimer

All scanning was performed against systems controlled by the participants in an isolated laboratory environment.

Nmap scanning should only be performed against systems where permission to test has been obtained.

