2026-11-10 Wazuh Tuning

Quick update tonight on Wazuh: the largest part of my alerts each day were supposed "brute force" attempts which closer examination determined to be calls to port 21114 from local IP addresses. I have family and customers in the area for whom I've installed RustDesk, and 21114 is its API server. Since my current installation of RustDesk is the free, rather than pro one, the service is not answering anything on that port. Clients still call out to check on it though because I misconfigured a couple of them, putting my domain in the API server field.

The best thing to do is to stop the behavior, but I have two problems:
1. I am only sure of one of the hosts. I can't speak to the other off the top of my head. It could be one of a number of people.
2. One cannot adjust RustDesk's settings while remoted in through RustDesk. This is presumably a security feature, and I appreciate it, but it means I have no recourse until I can sit with these hosts in person.

So I have tuned Wazuh's reporting to suppress alerts of single packet drops from these specific IP addresses when they are reaching out to 21114. It's something I can clean up later when I get to the machines, but for now, it reduces noise in my dashboard without over-correcting and fully allowlisting anything.
