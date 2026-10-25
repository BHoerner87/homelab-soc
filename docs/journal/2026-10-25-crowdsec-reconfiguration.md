The following is adapted from a Claude report on work done, although not written by Claude.
Edit: I didn't think about the fact that Markdown doesn't output in monotype, and so the little diagram I slapped together looks bad unless you switch to Code on GitHub.I won't make this mistake again. I will probably not attempt to diagram anything again, either, because it wasn't remotely worth the effort I put in, even if it had shown correctly.

## What was done and why
The VM hosting most of my Internet-exposed services contained a crowdsec setup which only benefitted that specific VM. This offered some protection for other Docker services running through NPM, but some services exposed through NPM do not benefit from Cloudflare MFA or Zero Trust because it would break their application access / logins. (For example, if a family member wants to access Jellyfin to watch home movies, an MFA landing page would prevent their Jellyfin app from ever connecting to my URL since it isn't equipped to handle that landing page. Ditto services like Joplin.)

Now, vm1-web runs a working CrowdSec setup that *other machines* can join. Jellyfin logs real attacker IPs, the decision server is reachable only by approved hosts, and a live ban test blocked a smartphone (on its cellular network) at the firewall, and released it on command.

### At a glance
Every ban lands in one local API (LAPI) on vm1-web, and the bouncer enforces it.
                          _____________________________________________
                          | vm1-web                                   |
                          | ------------------------------------------|
Internet --> OPNsense -----> NPM            | Firewall bouncer        |
                      \____| (reverse proxy) | (blocks in DOCKER-USER) |
                       |  |-----------------|------------------^------|
  ________________     |  |____________________________________|______|
  | Agent+bouncer |<--------->                                        |
  | Part 2, edge  |    |  |  CrowdSec LAPI, port 8080                 |
  | blocking      |    |  |  stores bans; only CozyLab and            |
  |_______________|    |  |  OPNsense may connect                     |
                       |  |____________________________________^______|
                       |                                       |
                       |  _____________________________________|_______
                       |  | CozyLab                            |      |
                       |  | __________________  ____________________  |
                       |  || Jellyfin        |  | CrowdSec agent    | |
                       |  || logs the real   |  |                   | |
                       |  || client IP       |->| Part 1, reads logs| |
                       |  ||_________________|  |___________________| |
                       |__|___________________________________________|

Today bans are enforced on vm1-web. Parts 1 and 2 of the deployment guide add Jellyfin attack detection on CozyLab and blocking at the router, which feed the same LAPI.

## What was changed
Five changes, each fixing one link in the chain from "attacker tries a password" to "attacker gets blocked."

1. Added NPM to Jellyfin's Known Proxies
  - This matters because without it, Jellyfin logged every visitor as NPM's private IP address. CrowdSec ignores private IPs on purpose, so it would never have banned anyone.
  - This configuration change involved adding vm1-web (where CrowdSec lives) to the Networking panel of Jellyfin's Dashboard under the Known Proxies section of its Networking page.
2. Published the CrowdSec decision server (the LAPI) on the LAN, not just on vm1-web itself
  - This matters because other machines (CozyLab, OPNsense) need to reach it to report attacks and fetch bans. It was only listening on 127.0.0.1 (its loopback interface), which means, "this machine only."
  - This change was made in the crowdsec stack inside Portainer (Portainer resides in vm1-web) in the ports: section of the compose file.
3. Added a firewall rule so only CozyLab and OPNsense can reach port tcp/8080 (the crowdsec API port)
  - This matters because the LAPI holds my ban list and accepts reports; no other device on the LAN should be able to talk to it.
  - This change was made with an access control updating shell script, which lives at /usr/local/sbin/lapi-acl.sh/, and which is run by lapi-acl.service (which I created to run it whenever Docker is running).
4. Moved the firewall bouncer into its own stack folder
  - As a result of previous migration efforts which broke CrowdSec for a time, and the subsequent effort to clean that up, the bouncer and the LAPI ended up defined in two different places under the same name, so that either tool could have deleted the other's container by accident.
  - This data exists at /opt/stacks/crowdsec-bouncer. The newly-created stack points at that data.
5. Retired the old duplicate compose file
  - It no longer matched what was running. Running it would have silently undone change number 2.
  - The old file lives at /opt/stacks/crowdsec/docker-compose.yml.stale-2026-10-25
    - The new name is descriptive and meant to be self-explanatory if I ever look back and can't remember why it is out of use, when that change was made, etc.

## Issues encountered
Every issue today came from one of two things: a service listening somewhere other than expected, or two tools each thinking they owned the same thing.
Issues:
1. Jellyfin logged 192.168.1.1 (the router)
  - I just forgot to turn off WiFi when trying to test from my phone. The test went out, and back in through the router (NAT reflection). When WiFi was switched off as intended, the real carrier IP address showed.
2. Trying to test a connection from cozylab to crowdsec's LAPI using curl just hung.
  - Discovered the LAPI was published only on 127.0.0.1:8080. Nothing answered on the LAN address.
  - Learned that
    - docker port <name> and ss -tlnp show where a service actually listens
      - in Linux, ss is a powerful network statistics tool (literally short for Socket Statistics)
      - ss is a modern replacement for the older netstat command
      - the flags -tlnp are as follows:
        - t: TCP (only show TCP connections and listening ports)
        - l: Listening (filters the output to show only sockets that are listening for incoming connections, which is useful for finding services waiting for traffic, like a web server or database, or in our case, CrowdSec's local API)
        - n: Numeric (ensures that listed values are shown as numeric values e.g. IP address:port, instead of resolving to hostnames like vm1-web or service names like https)
        - p Processes/Programs (shows the process ID (PID) and the name of the program or container process that owns each socket (requires root or elevated perms to see details for all processes)
3. The compose file had no ports: section, but the container running *did*
  - Portainer had created the container from its own copy of the file, stored inside Portainer's data volume (/data/compose/11)
  - I learned to use ```docker inspect``` labels to show which file really created a container.
4. Portainer showed both containers but defined only one
  - The bouncer shared the "crowdsec" project name, so Portainer listed it and flagged it as an "orphan."
  - A shared project name between two tools created a sort of trap. Had I clicked the "remove orphans" button in Portainer, it would have deleted the bouncer.
5. Updating the stack through Portainer failed (the Update button initially grayed out; I considered the page stale, refreshed it, tried again, only now that it let me click Update, it threw a 500 error)
  - Portainer's name check ran before the old container was removed, and never refreshed. So it wouldn't let me update the compose file this way.
  - Having a one-line rollback ready before removing anything was beneficial. I was able to just ```docker compose up -d firewall-bouncer``` to put the bouncer back in place before my services were exposed, bouncer-less, for too long.
6. Typed <portainer-name> literally
  - I confess I got into that "chatbot speedrun" mindset where I was processing mentally what it was telling me less than I was simply copy-pasting its recommended commands impatiently. I pasted in one of its filler values in angle-brackets instead of looking at the name of the portainer stack from the previous output, and adding it to the command as intended.
  - As ever in my life, the sooner I get impatient and go too fast, the sooner things get sloppy. It's always better to stay slow and attentive. Always. Always.

### How DOCKER-USER rules work
Docker rewrites a packet's destination to the container's internal address before my rule sees it, so the rule is written to match the *original* port using conntrack (```--ctorigdstport 8080```)

### Validation testing
I banned my smartphone on purpose, watched it get blocked, and then unbanned it. Steps are as follows:
1. Ban.
    - ```cscli decisions add -i <phone IP> -d 5m``` told the LAPI to ban the phone's LTE address for 5 minutes.
2. Bouncer picked it up.
    - Within about ten seconds, the bouncer log showed the new decision list, and ```ipset test``` confirmed the IP was in the firewall's block list.
3. Traffic blocked.
    - Jellyfin would not load on the phone over LTE, and the packet counter on the CrowdSec firewall rule went up.
4. Unban
    - ```scli decisions delete``` removed the ban, and Jellyfin loaded again within about ten seconds.
5. Community blocklist
    - ```cscli metrics show decisions``` showed a large number of Community API (CAPI) bans. Those are IPs blocked because other CrowdSec users reported them, before they ever reached me.

Alongside the ban test, the health check also passed: CozyLab reached /health and got ```{"status":"up"}```, and the bouner showed a fresh last-pull. Together that proved every link:decision -> LAPI -> bouncer -> firewall -> blocked traffic.

## Where things live now
Each container now has exactly one owner.
- CrowdSec LAPI (```crowdsec```) is managed by Portainer, crowdsec stack
  - Portainer's editor (which is stored in its data volume
- Firewall bouncer (```crowdsec-firewall-bouncer```) is managed as a plain Docker Compose file; it shows in Portainer, but moving it into Portainer for full control wasn't worth the effort.
  - /opt/stacks/crowdsec-bouncer/docker-compose.yml
- Port 8080 allow-list is managed by systemd (```lapi-acl.service```), and re-runs whenever Docker restarts.
  - /user/local/sbin/lapi-acl.sh/

"Don'ts" until further notice:
- don't run docker compose in /opt/stacks/crowdsec
- don't tick "prune" or "remove orphans" on the Portainer stack.

Left to do:
- Part 1 of the deployment guide assembled by Claude: the CozyLab agent that reads Jellyfin logs.
- Part 2: the OPNsense plugin that blocks at the network edge.
- One reboot of vm1-web to prove everything will come back by itself.
