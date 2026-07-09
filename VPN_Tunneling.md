# Virtual Private Networks and Tunneling: Securing Communication Across Untrusted Networks

When a packet leaves your laptop for a server on the other side of the world, it passes through a dozen or more routers you neither own nor trust. Any of them can read the contents, note where the packet came from and where it is headed, or quietly alter what passes through. For a plain HTTP request or an old-style email, that is exactly what happens: the data travels in the clear. A Virtual Private Network exists to change that. It builds a protected path across the same untrusted internet, so two machines can talk as if they shared a private cable, even though every byte between them is crossing public infrastructure.

The idea sounds almost contradictory — a private network running over a public one. The trick that makes it work is tunneling.

## What tunneling actually means

Tunneling is encapsulation: putting one packet inside another. Suppose a computer on an office network at 10.0.0.5 wants to reach a file server at 10.0.0.9 in a branch office across the country. Those 10.x.x.x addresses are private. They mean nothing on the public internet, and no router out there will forward them. So the VPN software takes the entire original packet, addresses and all, and wraps it inside a new packet whose outer header carries the two public IP addresses of the office gateways. To every router along the way, it looks like ordinary traffic between two public hosts. Only at the far end does the gateway strip off the outer wrapper and release the original packet onto the local network, where 10.0.0.9 finally receives it.

Encapsulation alone would hide the addressing but not the contents. That is why real VPNs pair tunneling with encryption. Before the inner packet is wrapped, it is encrypted, so even if someone captures the outer packet mid-route and unwraps it, they find ciphertext rather than readable data. Encapsulate, encrypt, forward — that sequence is the core of every VPN protocol, and the differences between protocols come down to how each one handles those three steps.

## The main protocols, and how they differ

IPsec is the oldest of the widely used standards and still the backbone of most corporate site-to-site links. It operates at the network layer, which means it can protect any traffic riding on top of IP without the applications knowing anything about it. IPsec is really a suite of parts: ESP (Encapsulating Security Payload) does the encryption and integrity work, while IKE (Internet Key Exchange) negotiates keys before any data moves. It runs over UDP port 500, switching to 4500 when it has to cross NAT. IPsec is powerful and well-proven, but anyone who has configured it by hand knows it is also fiddly. A mismatch in one of a dozen parameters and the tunnel silently refuses to come up, with logs that rarely tell you why.

TLS-based VPNs take a different route. OpenVPN, the best-known example, wraps traffic in the same TLS that secures HTTPS and typically sends it over UDP 1194 or, when it needs to slip past restrictive firewalls, TCP 443. Because 443 is the port normal web traffic uses, this kind of tunnel is hard to tell apart from someone simply browsing a secure website. That matters in networks that try to block VPNs outright.

WireGuard is the newcomer, and it has shifted expectations considerably. Its entire codebase runs to only a few thousand lines, against the hundreds of thousands in mature OpenVPN and IPsec implementations, which makes it far easier to audit for security flaws. It fixes its cryptography instead of negotiating it: ChaCha20 for encryption, Curve25519 for key exchange, no menu of options to get wrong. It lives inside the Linux kernel, runs over UDP, and in most benchmarks it is faster with lower latency than the older options. For a homelab or a personal setup, it has become the default choice, and deservedly so.

A couple of older names still turn up in textbooks. L2TP operates at the data-link layer and, on its own, provides no encryption at all. It is almost always paired as L2TP/IPsec, with IPsec doing the actual securing. PPTP, once common on Windows, should be treated as a museum piece; its MS-CHAPv2 authentication was broken years ago and can now be cracked in hours, so it has no business protecting anything you care about.

## Two shapes of VPN

Deployments generally fall into one of two patterns. A site-to-site VPN joins whole networks together — the office-to-branch case from earlier — with gateways at each end doing the tunneling, invisibly to the individual computers behind them. A remote-access VPN instead connects a single device to a network: the laptop of someone working from a café runs a client that builds a tunnel back to the company gateway, and from that point the laptop behaves as though it were plugged into the office wall. The remote-access model is what most people mean when they say they are "using a VPN," and its everyday use exploded around 2020, when remote work became the norm almost overnight.

## The costs, and what a VPN does not do

None of this is free. Wrapping and encrypting every packet adds processing work at both ends and extra bytes to every transmission. WireGuard, for instance, adds roughly 60 bytes of overhead, which is why tunnel interfaces usually lower their MTU to around 1420 to avoid fragmentation. Throughput drops and latency rises, though on modern hardware the penalty is often small enough to ignore.

There is a more important misconception worth correcting. A VPN does not make you anonymous, and it does not remove the need to trust anyone. It relocates that trust. Your traffic is hidden from your local network and your internet provider, yes, but the VPN server on the other end can now see everything your ISP used to. You have simply chosen to trust the VPN operator instead. A commercial provider that logs your connections, or lets your DNS queries leak outside the tunnel, can quietly undo much of the protection you thought you were paying for. The tool secures the channel. It says nothing about whoever sits at the far end of it.

That is the distinction worth carrying away from the topic. Tunneling and encryption solve a specific, real problem — moving data safely across infrastructure you cannot trust — and they solve it well. What they cannot do is decide, on your behalf, who deserves to be trusted once the data arrives.
