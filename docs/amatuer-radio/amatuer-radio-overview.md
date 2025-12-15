# Amatuer Radio Overview

## Why Use Amateur Radio?
Amateur radio gives schools a resilient communication layer that works when commercial internet or cellular backhaul is too expensive, unreliable, or simply unavailable. By operating on shared RF spectrum, students learn how to pass traffic responsibly without consuming paid bandwidth or depending on cloud infrastructure.

### No Reliance on the Public Internet
- **Independent links:** Two stations can exchange data directly over RF without touching routers or paid transit.
- **Bandwidth friendly:** AX.25 packet links operate at modest bit rates (1200–9600 baud), making it practical to run on low-power equipment and solar-backed nodes.

### Safe, Transparent, and Educational
- **Legally non-encrypted:** Regulations require content to remain in the clear. This transparency teaches students to design protocols that are safe even when everyone can monitor the channel.
- **Controlled access:** Only licensed operators can transmit, so schools can gate participation through their amateur-radio clubs and logbooks.
- **Hands-on RF literacy:** Learners gain practical experience with propagation, modulation, and station setup instead of assuming the internet always exists.

### Sustainability and Community Projects
Schools can tie amateur radio into sustainability challenges—environmental sensor data, emergency drills, or shared STEM experiments—and keep collaboration alive even if local infrastructure fails. Because the hardware draws little power, it pairs well with solar trailers, battery banks, or other green-energy initiatives.

## Linking Schools Through LinBPQ Extensions
LinBPQ already routes AX.25 traffic, but you can extend it to coordinate joint sustainability projects between campuses:

1. **Enable an external application port** in `bpq32.cfg` that listens on localhost (for example, `APPLICATION 3,SUSTAIN,,CALLSIGN-15,SUSTAIN,255`).
2. **Write a small Python service** that connects to LinBPQ’s TCP application interface. The service can:
	- Ingest sensor readings or project updates from local students.
	- Publish structured messages (JSON, CSV, APRS-like text) to partner schools via RF or Telnet.
	- Archive incoming reports for later analysis.
3. **Share sustainability tasks**—water-quality snapshots, energy usage logs, reforestation goals—between two or more schools so each site contributes to a common dataset without relying on commercial backbones.

By combining a Python extension with LinBPQ’s routing, the network remains open, observable, and regulation-compliant while still giving students modern tooling to collaborate on long-term sustainability projects.


