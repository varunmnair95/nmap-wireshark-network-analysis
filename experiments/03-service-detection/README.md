# 🔎 Experiment 3 — Nmap Service & Version Detection

## 📌 Objective

The objective of this experiment was to use **Nmap service and version detection (`-sV`)** to identify services running on open TCP ports and observe the network traffic generated during the detection process using **Wireshark**.

This experiment builds directly on **Experiment 2 — Port Scanning**.

```text
Exp 1 → Host Discovery
        ↓
Exp 2 → Port Scanning
        ↓
Exp 3 → Service & Version Detection
```

---

## 🖥️ Lab Environment

| Component          | Details                        |
| ------------------ | ------------------------------ |
| 🐉 Kali Linux      | `192.168.128.3`                |
| 🪟 Windows 11      | `192.168.128.4`                |
| 🌐 Network         | UTM Host Only                  |
| 🛡️ Firewall       | Windows Defender Firewall — ON |
| 🔍 Tool            | Nmap 7.99                      |
| 📡 Packet Analysis | Wireshark                      |

---

## 🎯 Ports Investigated

Experiment 2 identified the following open TCP ports:

```text
135/tcp
139/tcp
445/tcp
```

These ports were used as the target ports for service detection.

---

## 🧪 Methodology

### 1️⃣ Service & Version Detection

The following command was used:

```bash
nmap -sV -p 135,139,445 192.168.128.4
```

### 2️⃣ Wireshark Capture

Wireshark was started before the Nmap scan to capture the traffic generated during service detection.

The capture was saved as:

```text
03-exp3-service-version-detection2.pcapng
```

---

## 📊 Nmap Results

```text
PORT     STATE SERVICE      VERSION
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds
```

Nmap also identified:

```text
OS: Windows
```

Port `445/tcp` was identified as `microsoft-ds`, but no specific version was reported.

---

## 📡 Wireshark Analysis

The packet capture showed communication between:

```text
192.168.128.3 → 192.168.128.4
```

The capture included:

* TCP SYN/SYN-ACK communication
* Connections to ports 135, 139 and 445
* Service-detection probes
* RPC-related traffic
* NetBIOS/SMB-related traffic
* TCP resets following individual probes

The capture demonstrates that `-sV` performs additional network communication beyond the initial port-state determination used during port scanning.

---

## 🔍 Findings

The experiment demonstrated that:

1. Nmap can identify services associated with open TCP ports.
2. Port `135` was identified as **Microsoft Windows RPC**.
3. Port `139` was identified as **Microsoft Windows netbios-ssn**.
4. Port `445` was identified as **microsoft-ds**.
5. Wireshark showed the additional traffic generated during service detection.
6. Service identification depends on receiving useful responses from the target.

---

## 📁 [Evidence](https://github.com/varunmnair95/nmap-wireshark-network-analysis/tree/main/experiments/03-service-detection/evidence)

```text
Service version detection .pcapng file
```

Supporting screenshots:

```text
Nmap 'sV' Service Version Detection Command
Wireshark Packet Capture Service Detection
```

---

## ✅ Conclusion

Nmap `-sV` was successfully used to perform service detection against the Windows 11 target.

The experiment demonstrated the progression from identifying open ports to determining the services associated with those ports, while Wireshark provided packet-level evidence of the communication generated during the detection process.

---

## ⚠️ Lab Disclaimer

This experiment was performed in an isolated virtual lab environment for learning and portfolio development.

No production systems were involved.

