# 7ks

Chat, voice calls, video conferencing and file sharing with anyone in the world, using nothing but their email address.

No phone number. No servers storing your messages or your files. From any device you choose.

**7ks** is the application. **7KSP** is the open and free protocol it runs on, and it belongs to everyone. The specification is in [PROTOCOL.md](PROTOCOL.md).

## Intentions

1. **The email address is the address.** Everyone has one. It is the address to chat with someone, to call them or to send them a document. Nobody needs to be signed up by anyone, nobody needs to know a phone number, and nobody needs to use the same application as you.

2. **Freedom of device.** The account is not tied to a phone or a SIM card. It is used from a phone, a computer, a tablet or a browser, on as many devices as you want, and you change devices without losing your identity.

3. **Nothing goes through servers.** Messages, files, voice and video travel directly between the devices of the people communicating, end-to-end encrypted. The network's servers only help two devices find each other.

4. **Your data is on your devices.** There is no copy of your conversations or your files on any server. The client is the storage unit. Whatever you have not delivered yet waits on your device until the recipient is available.

5. **Anyone can be part of the network.** A company, a university, an email provider or an individual can run their own node for their domain with one DNS entry, without asking anyone for permission. Their users communicate with the rest of the world like everyone else.

6. **Every domain rules its own domain.** Whoever owns `company.com` decides how `company.com` accounts are managed: sign-ups, removals, devices, recovery. The network cannot see or alter what happens inside a domain that runs its own node.

7. **The protocol belongs to everyone.** 7KSP is published as an open specification, under a free license, so that anyone can implement it in any application. 7ks is the first implementation, not the only possible one.

## How it is used

1. Install the client on your system (Windows, macOS, Linux, Android, iOS) or open the web version.
2. Type your email address.
3. You receive a link in that email account. Open it. Done.

That single step covers everything:

- If the address did not exist on the network, it signs you up.
- If it already existed, it links the new device to your account.
- If you have lost a device, you can unlink it from any other one.

From then on, anyone who knows your email can write to you, call you or send you a file, and you can do the same with them. If the other person does not use the network yet, their address simply does not answer; as soon as they sign up, with the same email, whatever you sent them is delivered.

## How it works inside

- **Identity.** Each device generates its own cryptographic keys, which never leave it. The proof that a device belongs to an email address is that its owner opened the link sent to that address.

- **Lookup.** Given an email address, the client finds out which devices it has and where they are. If the domain runs its own node, the domain's DNS points to it and that node is the authority. If not (`gmail.com`, `outlook.com`, etc.), a distributed public directory is queried, in which addresses are spread by the hash of their email.

- **Connection.** The node delivers a connection request to the recipient and helps it traverse NAT. When the two devices see each other, they open a direct encrypted channel. The node steps out.

- **Communication.** Over that direct channel goes anything: chat, calls, video, files. The protocol does not dictate what is transmitted or how; it only brings peers into contact.

- **Deferred delivery.** If the recipient is not available, whatever is to be delivered stays on the sender's device. The recipient's node notifies the sender when the recipient is back online, and delivery happens then, directly. The node never stores content.

- **Nodes.** There are two ways to take part in the network. The **public federation** is a distributed directory for addresses of domains that have no node of their own. **Domain nodes** are owned by each domain that joins the network with an entry in its DNS. Both speak the same protocol and connect to each other.

## What can be built on 7KSP

The protocol delivers a direct, authenticated channel between two devices, and identifies each one by its email. On top of that channel fits any application: messaging, telephony, video conferencing, file transfer, synchronization, remote support. 7ks implements the first ones; other applications can implement their own and remain reachable through the same email address.

The directory can also be queried over HTTPS, so any system can find out the public keys and the location of an email address without implementing the full protocol.

## Project status

Draft specification. There is no implementation yet. The planned order is:

1. Finalize the 7KSP/1 specification and publish test vectors.
2. Reference implementation of the node (`7ksd`) and the client library.
3. 7ks client for desktop, mobile and web.
4. Launch of the public federation.

## Contributing

- Open an issue to propose changes to the specification or to report errors.
- If you own a domain, run a domain node once the reference implementation exists and tell us what breaks.
- Decisions about the protocol are made in public, in this repository.

## License

All code in this project, including the reference node (`7ksd`), the client library and the 7ks clients, is free software licensed under the [GNU General Public License, version 3 or (at your option) any later version](https://www.gnu.org/licenses/gpl-3.0.html) (GPL-3.0-or-later).

The protocol specification (PROTOCOL.md) is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/), a free license compatible with the GPLv3. Anyone may implement it without permission.
