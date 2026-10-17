# 02 — TShark → Splunk SIEM Ingestion Pipeline

**Focus:** Traffic capture, log pipeline engineering, SIEM  
**Tools:** TShark/Wireshark 4.6.6, PowerShell, Splunk Enterprise, Splunk Universal Forwarder, Windows 11

## Objective
Capture live network traffic, turn it into structured JSON, ship it through a Splunk Universal Forwarder and prove it is searchable in Splunk.

## What I did
1. Explored live traffic on the WiFi interface (`tshark -i 4 -a duration:20`): DNS, TLS 1.2/1.3 handshakes with SNI, ICMPv6 neighbour discovery, mDNS and ARP.
2. Enumerated interfaces (`tshark -D`) and confirmed the tool version.
3. Verified the local Splunk Enterprise and Universal Forwarder installs (`Get-ChildItem`, `Get-WinEvent` MsiInstaller events).
4. Built a 60-second capture using `-T ek` with explicit `-e` fields (`frame.time_epoch`, `frame.number`, `ip.src`, `ip.dst`, TCP/UDP ports, protocol, frame length), written to `C:\SplunkWireshark\network_events.json`.
5. **Troubleshooting:** `splunk enable listen 9997` on the main instance failed repeatedly at login. I pivoted to configuring the **Universal Forwarder** to forward to `127.0.0.1:9997`, which is the correct role split (forwarders ship, indexers receive).
6. Wrote `inputs.conf` by hand: a `[monitor://...]` stanza with `index = network` and `sourcetype = wireshark:json`.
7. Restarted the forwarder and checked its prerequisite checks, certificate generation and install-manifest hash validation.
8. Validated the output file (628 lines in the processed file) and inspected sample JSON records.
9. Searched `index=network` in Splunk: **22,790 events** indexed and searchable (`host`, `source`, `sourcetype` populated).

## Results
End-to-end pipeline confirmed: TShark capture → JSON → Universal Forwarder → Splunk index → Search UI.

## Observations
- Some DNS records in the JSON show `null` for source/destination IP. I did not root-cause it. A likely follow-up is checking whether those frames are IPv6 (`ipv6.src`/`ipv6.dst`) or re-exporting with `-T json`.
- The index was a single test index (`network`); in production I would separate indexes by data source, retention and access.

## Evidence
**01. TShark live capture on the WiFi interface (DNS, TLS handshakes, ICMPv6)**  
![TShark live capture on the WiFi interface (DNS, TLS handshakes, ICMPv6)](screenshots/01-tshark-live-capture.png)

**02. Continued capture: TLS 1.3 handshake and application data**  
![Continued capture: TLS 1.3 handshake and application data](screenshots/02-tshark-tls-capture.png)

**03. mDNS/ARP background traffic in the capture**  
![mDNS/ARP background traffic in the capture](screenshots/03-tshark-mdns-capture.png)

**04. TShark 4.6.6 on Windows 11**  
![TShark 4.6.6 on Windows 11](screenshots/04-tshark-version.png)

**05. Listing capture interfaces (tshark -D) and a 10 s test capture**  
![Listing capture interfaces (tshark -D) and a 10 s test capture](screenshots/05-tshark-interfaces.png)

**06. Capture command using -T ek with explicit -e field selectors, output to C:\SplunkWireshark**  
![Capture command using -T ek with explicit -e field selectors, output to C:\SplunkWireshark](screenshots/06-ek-json-capture-command.png)

**07. Verifying Splunk and Universal Forwarder installs (Get-ChildItem, Get-WinEvent)**  
![Verifying Splunk and Universal Forwarder installs (Get-ChildItem, Get-WinEvent)](screenshots/07-splunk-install-verification.png)

**08. Troubleshooting: 'enable listen 9997' failing on login, pivot to the Universal Forwarder**  
![Troubleshooting: 'enable listen 9997' failing on login, pivot to the Universal Forwarder](screenshots/08-splunk-listen-login-failure.png)

**09. Forward-server added; inputs.conf monitor stanza created; forwarder restarting**  
![Forward-server added; inputs.conf monitor stanza created; forwarder restarting](screenshots/09-forward-server-and-inputs-conf.png)

**10. Forwarder prerequisite checks, hash validation, daemon start, inputs.conf confirmed**  
![Forwarder prerequisite checks, hash validation, daemon start, inputs.conf confirmed](screenshots/10-forwarder-restart-validation.png)

**11. Capture run, and a shell mistake caught during the session**  
![Capture run, and a shell mistake caught during the session](screenshots/11-capture-run-and-shell-error.png)

**12. Final output file (628 lines), sample JSON events, forward-server and inputs.conf confirmed**  
![Final output file (628 lines), sample JSON events, forward-server and inputs.conf confirmed](screenshots/12-final-json-and-inputs-conf.png)

**13. Splunk Search UI: index=network returns 22,790 events**  
![Splunk Search UI: index=network returns 22,790 events](screenshots/13-splunk-search-index-network.png)
