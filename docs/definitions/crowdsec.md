A quick definition of CrowdSec, which I am using for an Intrusion Detection and Intrusion PreventionSystem (IDPS).

This IDPS covers all the Internet-facing services in my home lab / on my home network because all those Internet-facing services go through Nginx Proxy Manager (which itself lives in a VM, although not all services share that VM or indeed even that host), and CrowdSec ingests access logs from Nginx, SSH login attempts, etc., and compares these with known bad patterns of behavior that are crowdsourced from other CrowdSec users.

CrowdSec operates in two pieces: there is an agent, which CrowdSec calls a Security Engine, and it mmonitors traffic, and compares to known "scenarios" of bad behavior. When it identifies such scenarios, it makes a "decision" for what to do about it, including banning source IPs.

The second piece is the CrowdSec bouncer, which handles banning malicious IPs. CrowdSec calls this bouncer a "remediation component". It adds the identified malicious IPs to the Linux kernel's firewall (iptables).

This makes Nginx Proxy Manager a security chokepoint--a single point of ingress--for unsolicited traffic, malicious and otherwise and that is why it is so important to cover with CrowdSec IDPS. I took a look at some of its current decisions, and probing from Censys is banned, as is some entity that was poking around my services looking for CVEs to exploit.
