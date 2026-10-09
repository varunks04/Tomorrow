# Networking Fundamentals — Technical Interview Q&A

### Q1: Walk me through the TCP 3-Way Handshake step-by-step.
> **Model Answer:**
> 1. **SYN:** Client selects an Initial Sequence Number ($ISN_c$) and sends a TCP segment with the `SYN` flag set to the server. State: Client enters `SYN_SENT`.
> 2. **SYN-ACK:** Server receives SYN, allocates socket memory, chooses its own Initial Sequence Number ($ISN_s$), sets the acknowledgment number to $ISN_c + 1$, and sends a segment with `SYN` and `ACK` flags. State: Server enters `SYN_RCVD`.
> 3. **ACK:** Client receives SYN-ACK, sets acknowledgment number to $ISN_s + 1$, and sends a segment with `ACK` set. State: Both transition to `ESTABLISHED`.
> *Security Angle:* In a **SYN Flood attack**, the attacker sends thousands of spoofed SYNs without returning the final ACK, exhausting the server's SYN backlog queue (mitigated via SYN Cookies).

---

### Q2: What is the difference between a Recursive and an Iterative DNS query?
> **Model Answer:**
> - **Recursive Query:** The client (resolver) asks the DNS server to return the complete answer. The DNS server takes on the burden of contacting other DNS servers on the client's behalf and returning the final IP.
> - **Iterative Query:** The DNS server returns the best referral it has (e.g. "I don't know the IP of example.com, but ask the .com TLD server at this IP"). The querying resolver is responsible for continuing the query chain.

---

### Q3: Calculate the usable host range and broadcast address for 192.168.10.65/26.
> **Model Answer:**
> - Prefix `/26` means 26 network bits and $32 - 26 = 6$ host bits.
> - Subnet mask: `255.255.255.192` ($128 + 64 = 192$).
> - Block size: $256 - 192 = 64$.
> - Subnet blocks: 0-63, 64-127, 128-191, 192-255.
> - Since the IP is `192.168.10.65`, it falls into the **64 to 127** subnet.
>   - **Network ID:** `192.168.10.64`
>   - **First Usable Host:** `192.168.10.65`
>   - **Last Usable Host:** `192.168.10.126`
>   - **Broadcast Address:** `192.168.10.127`
>   - **Usable Hosts:** $2^6 - 2 = 64 - 2 = 62$.

---

### Q4: How does ARP Spoofing work, and how do you mitigate it?
> **Model Answer:**
> ARP maps Layer 3 IP addresses to Layer 2 MAC addresses without authentication.
> In **ARP Spoofing**, an attacker transmits unsolicited Gratuitous ARP replies associating the default gateway's IP address with the attacker's MAC address. The victim's ARP cache updates, rerouting outbound traffic through the attacker (Man-in-the-Middle).  
> *Mitigation:* **Dynamic ARP Inspection (DAI)** on managed switches using the DHCP snooping binding table, and static ARP entries for critical systems.
