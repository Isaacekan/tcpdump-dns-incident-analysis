# Network Traffic Analysis: DNS & ICMP Incident Report

Analysing a `tcpdump` packet capture to find out why customers could not reach a company website, then writing it up as a formal cybersecurity incident report.

> Completed as a portfolio activity for the **Google Cybersecurity Professional Certificate**. The organisation, IP addresses and domain in this scenario are fictional.

**Skills shown:** packet capture analysis (tcpdump) · TCP/IP model · DNS, UDP and ICMP · root-cause analysis · incident documentation

📄 Full report: [Word](report/incident-report.docx) · [PDF](report/incident-report.pdf)

---

## Scenario

I'm a cybersecurity analyst at an IT services provider. Several customers of a client reported that they could not open **www.yummyrecipesforme.com**. After the page tried to load, they saw the error **"destination port unreachable"**.

I reproduced the error myself and then captured the traffic with `tcpdump` while loading the page again. My job was to find out which network protocol and service were affected and to document what I found.

## How the page should load

1. The browser sends a **DNS query over UDP (port 53)** to the DNS server, asking for the IP address of the domain (an A record).
2. The DNS server replies with the IP address.
3. The browser sends an **HTTPS request (port 443)** to that IP address, and the web server returns the page.

The capture showed that the process broke at **step 1**.

## What the tcpdump log showed

| Field | Value | Meaning |
|---|---|---|
| Timestamp | `13:24:32.192571` | 1:24 p.m. and 32.19 seconds |
| Source | `192.51.100.15` | My computer (the client) |
| Destination | `203.0.113.2.domain` | The DNS server, on the DNS port (53) |
| Query | `35084+ A?` | Query ID 35084, with flags set, requesting an A record |
| Response protocol | `ICMP` | An error message, not a DNS answer |
| Error | `udp port 53 unreachable` | Nothing on the DNS server accepted the query on port 53 |

The query was sent **three times**. Every attempt came back with the same ICMP error.

## Part 1: Summary of the problem

The DNS queries sent over UDP from the client (192.51.100.15) to the DNS server (203.0.113.2) were not being delivered. Each attempt to resolve the A record for the domain failed. The ICMP response returned the error **"udp port 53 unreachable"**.

**Port 53 is used for DNS**, the service that translates domain names into IP addresses. The browser never received an IP address, so it could not send the HTTPS request to the web server. The web server itself may be working normally, but users cannot find it.

**Most likely issue:** the DNS server is not responding to DNS requests because no service is listening on UDP port 53.

## Part 2: Analysis and likely cause

**Time of incident:** The failure was captured at 1:24 p.m. (`13:24:32.192571`). Customers had reported the problem before this capture was taken.

**How the issue was found:** Customers reported "destination port unreachable" when trying to open the website. I reproduced the error, captured the traffic with `tcpdump` while loading the page again, and traced the outgoing DNS queries (UDP) and the error responses that came back (ICMP). I reported the findings to my supervisor, and the incident was passed to the security engineers.

**Key findings:**
- DNS A-record queries were sent over UDP to 203.0.113.2 on port 53.
- Each query was answered with an ICMP "udp port 53 unreachable" error, and two retries failed in the same way.
- No HTTPS (port 443) traffic was ever sent, so **the fault is with DNS, not with the web server**.

**Likely cause:** The DNS service on 203.0.113.2 is not listening on UDP port 53. Either the service has stopped, crashed or been misconfigured, or a firewall rule is blocking inbound traffic on port 53. A denial-of-service (DoS) attack against the DNS server should also be ruled out.

**Recommended next steps:**
1. Check whether the DNS service is running and listening on port 53, and restart it if needed.
2. Review the firewall and access control rules for any recent change that blocks port 53.
3. Check the DNS server's logs and traffic volumes for signs of a DoS attack or resource exhaustion.
4. Once the service is restored, confirm that the domain resolves and that the site loads. Consider adding a secondary DNS server for redundancy.

## What I took from this

- **An "unreachable" error in the browser doesn't always mean the web server is down.** Following the request layer by layer showed that the failure happened before the browser ever contacted the web server.
- **ICMP is useful for diagnosis.** This error was a *Destination Unreachable* message (type 3, code 3: port unreachable). It identified the exact host, protocol and port involved.
- **DNS is a single point of failure.** It is worth monitoring, and having redundant DNS servers improves resilience. This links into availability, the "A" in the CIA triad.
