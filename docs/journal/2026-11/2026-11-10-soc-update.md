11-10-2026 SOC update

I did some reflecting on my exposure recently and made some calls in the spirit of caution.

I misunderstood the value of Cloudflare Zero Trust's DNS proxying as well as its MFA landing page.

I wanted the former in place for my various domain records so that my home IP address would be obfuscated by Cloudflare's own IPs. I wanted the latter so that which services I had exposed would be reachable only by people whom I wanted to reach them. I have whitelisted maybe 20 family and friends to reach services like Immich.

Unfortunately, these misunderstandings feed into each other to make a larger risk:

The MFA landing page is sort of "optional" in the sense that, if someone has my home IP address, they can reach my Internet-exposed services (under the previous arrangement) by simply using that, and the port of the service they want to reach. I tested this with a curl command from outside my home network, and damned if it didn't return the HTML header of my Immich server. This is all the scarier because I was using DNS records for more sensitive elements of my infrastructure as a simple shortcut to reach them without having to type their IP addresses into my browser's location bar. (The appropriate way to handle this would be to place DNS overrides into my DNS service, but I'll have to circle back on that.) Anyway, those records have since been deleted in a great culling. I must have killed four subdomains yesterday in order to harden (and "harden" is the most gracious way I can say, "fix my grievous mistake").

It isn't so bad as long as they don't have my home IP address, though, right? And they can't get that because Cloudflare is proxying my URLs, right? Well, yes...but also, no. I thought I was running a version of Cloudflare tunneling, but I was assuredly *not*. Instead, I was relying on TLS certificates issued through Let's Encrypt, which—you guessed it—exposed my home IP address. *Brilliant*. So, I had to tear down those proxied DNS records and revisit the matter of tunneling.

Fortunately, this was easy to set up, and one tunnel can handle many CNAME records. These CNAME records replace the A records that I had in place previously. All it takes to set this up is to spin up a Cloudflared service—which I put into my vm1-Web virtual machine, managed through Portainer—and then Cloudflared handles the business of opening an existing tunnel to Cloudflare, so that all requests to my services go through that instead of riding a proxied IP of Cloudflare's to ports forwarded from my router to Nginx Proxy Manager. Now, my visitors still have to hit the MFA wall in order to see my kid's baby pictures, but the services aren't exposing my home IP address through the certificates and thereby handing would-be attackers a hint as to how to sidestep Cloudflare's protections.

It's embarrassing to admit things were that wrong before, but this entire thing is about learning, and I am learning. There will always be more to fix where electronic security is concerned. You just have to do your best and hope no one finds these faults before you do. 
