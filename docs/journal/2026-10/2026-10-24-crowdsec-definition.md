Added definition of CrowdSec, to remind myself how it works as much as anything. While thinking it through I wondered whether I needed it on CozyLab, which hosts a few servers that are public-facing, but since the ones that answer web requests are already connected to the Internet through Nginx Proxy Manager (which lives in vm1-web), it's not really an issue there.

I do intend to add a CrowdSec agent to my OPNsense router, however.

The router was acting up in the last two weeks in a way that made me nervous. I don't see any evidence of anything untoward, but then, I'm still learning how to identify such things. What it was doing was grinding down to a halt, essentially, and needing to be reset by hand. For now I've just applied all its updates, but there is more to do. (There is always more to do.)
