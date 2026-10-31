# FortiGate Lab: Destination NAT (DNAT) & Hairpin NAT

## 📌 Scenario Overview
This lab demonstrates how to securely publish an internal web server (Apache) to an external network (WAN) using **Destination NAT (DNAT / Port Forwarding)** on a FortiGate firewall. 

Because the external testing client (Ubuntu) resides on the same Layer 2 subnet as the FortiGate's WAN interface, **Hairpin NAT (Source NAT)** was also implemented to ensure return traffic correctly routes back through the firewall.

## 🗺️ Network Topology

![Network Topology](topology.png)

### IP Addressing Table

| Device | Interface | IP Address | Network Zone |
| :--- | :--- | :--- | :--- |
| **Apache2 Server** | eth1 | `192.168.10.3` | LAN (Internal) |
| **LAN VPC** | eth0 | `192.168.10.2` | LAN (Internal) |
| **FortiGate Firewall** | port2 (LAN) | `192.168.10.1` (Gateway) | LAN (Internal) |
| **FortiGate Firewall** | port1 (WAN) | `192.168.93.154` | WAN (External) |
| **WAN VPC** | eth0 | `192.168.93.155` | WAN (External) |
| **Ubuntu Server** | eth1 | `192.168.93.156` | WAN (External) |

## 🎯 Lab Objectives
1. Allow external devices to access the internal Apache web server.
2. Map WAN Port `8080` to Internal Port `80`.
3. Overcome the "Hairpin NAT" limitation since the WAN testing client (Ubuntu) is on the same subnet as the FortiGate WAN IP.

---

## 🛠️ Configuration Steps

### Step 1: Configure the Apache Server
1. Ensure the Apache server is running and has its default gateway set to the FortiGate's LAN interface (`192.168.10.1`).
2. Verify the web server is accessible locally.
3. *(Optional)* Modify the default web page text: `sudo nano /var/www/html/index.html`.

### Step 2: Configure Virtual IP (DNAT) on FortiGate
This maps the external WAN IP and port to the internal IP and port.
* **GUI Path:** `Policy & Objects > Virtual IPs > Create New`
* **CLI Configuration:**
```bash
config firewall vip
    edit "Apache_DNAT"
        set extip 192.168.93.154
        set extintf "port1"
        set portforward enable
        set extport 8080
        set mappedip "192.168.10.3"
        set mappedport 80
    next
end
Step 3: Create the Firewall Policy (with Hairpin NAT)
This allows the traffic to pass through the firewall and applies Source NAT to fix the routing for clients on the same WAN subnet.

GUI Path: Policy & Objects > Firewall Policy > Create New

Settings:

Incoming Interface: port1

Outgoing Interface: port2

Source: all

Destination: Apache_DNAT (VIP)

Service: ALL (or TCP/8080)

Action: ACCEPT

NAT: ENABLED (Crucial for Hairpin NAT)

CLI Configuration:

bash
config firewall policy
    edit 0
        set name "WAN_to_LAN_DNAT"
        set srcintf "port1"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "Apache_DNAT"
        set action accept
        set schedule "always"
        set service "ALL"
        set nat enable
    next
end
🧪 Testing and Verification
Testing was performed from the Ubuntu Server (192.168.93.156), which acts as an external WAN client.

Test Command:

bash
curl -v http://192.168.93.154:8080
Expected Output (Success):

text
*   Trying 192.168.93.154:8080...
* TCP_NODELAY set
* Connected to 192.168.93.154 (192.168.93.154) port 8080 (#0)
> GET / HTTP/1.1
> Host: 192.168.93.154:8080
> User-Agent: curl/7.68.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Sat, 31 Oct 2026 06:50:04 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Last-Modified: Mon, 08 Mar 2021 11:19:25 GMT
< ETag: "16-5bd049b34c940"
< Accept-Ranges: bytes
< Content-Length: 22
< Content-Type: text/html
< 
Web Server is running
* Connection #0 to host 192.168.93.154 left intact
Analysis of the Output
Connected to 192.168.93.154 port 8080: The client successfully connected to the FortiGate WAN IP.

HTTP/1.1 200 OK: The FortiGate successfully translated the request and forwarded it to the internal LAN.

Server: Apache/2.4.41 (Ubuntu): The response is originating directly from the internal Apache server.

The payload ("Web Server is running") proves the end-to-end data flow is intact.

📂 Lab Files & Import
If you want to test this configuration yourself, you can download the PNETLab export file:

DNAT.unl - PNETLab topology file.

How to import:

Download the .unl file.

Log into your PNETLab web interface.

Go to the main menu and select Lab Preview -> Import (or drag and drop the file into the workspace).

Upload the file and start the nodes.

💡 Key Takeaways
What is Hairpin NAT? If a client on the WAN subnet tries to access the WAN IP of the firewall, the firewall DNATs it to the LAN. However, the LAN server will try to reply directly to the client (since they are on the same subnet), bypassing the firewall, and the connection will drop. Enabling NAT on the firewall policy forces the reply to go back through the FortiGate.

Routing: Always ensure the internal server's Default Gateway points to the firewall's internal interface. Without this, the server won't know how to reply to external requests.

text

***
