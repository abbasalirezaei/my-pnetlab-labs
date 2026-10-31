# FortiGate Lab: DNAT Port Forwarding

## Scenario Overview

This lab uses a FortiGate Virtual IP (VIP) to publish an internal Apache web server. Requests sent to WAN port `8080` are forwarded to port `80` on the internal server.

> This is **not hairpin NAT**: the test client is on the WAN network and the server is on the LAN. Because they are on different subnets and the server’s default gateway is the FortiGate, the server’s replies naturally pass back through the FortiGate. Source NAT is not required for this topology.

## Network Topology and IP Addressing

| Device | Interface | IP address | Network |
|---|---|---:|---|
| Apache Server | `eth1` | `192.168.10.3` | LAN |
| LAN VPC | `eth0` | `192.168.10.2` | LAN |
| FortiGate | `port2` (LAN) | `192.168.10.1` | LAN |
| FortiGate | `port1` (WAN) | `192.168.93.154` | WAN |
| WAN VPC | `eth0` | `192.168.93.155` | WAN |
| Ubuntu Server | `eth1` | `192.168.93.156` | WAN |

## Objectives

1. Publish the internal Apache web server through the FortiGate.
2. Map WAN port `8080` to internal port `80`.
3. Test the connection from Ubuntu on the WAN network.

## Prerequisite: Apache Web Server

Make sure Apache is running on `192.168.10.3` and that its default gateway is `192.168.10.1`. To test local access, run this from a LAN host:

```bash
curl -v http://192.168.10.3
```

To edit the default web page, if needed:

```bash
sudo nano /var/www/html/index.html
```

## Create the DNAT VIP

In the FortiGate GUI, go to:

`Policy & Objects > Virtual IPs > Create New`

Configure the VIP as follows:

- **Name:** `Apache_DNAT`
- **Interface:** `port1`
- **External IP:** `192.168.93.154`
- **Port Forwarding:** Enabled
- **Protocol:** TCP
- **External Service Port:** `8080`
- **Map to Port:** `80`
- **Mapped IP:** `192.168.10.3`

CLI equivalent:

```text
config firewall vip
    edit "Apache_DNAT"
        set extip 192.168.93.154
        set extintf "port1"
        set portforward enable
        set protocol tcp
        set extport 8080
        set mappedip "192.168.10.3"
        set mappedport 80
    next
end
```

## Create the Firewall Policy

This policy allows TCP traffic arriving at the VIP on `port1` to reach the LAN server through `port2`.

In the FortiGate GUI, go to:

`Policy & Objects > Firewall Policy > Create New`

Configure the policy as follows:

- **Incoming Interface:** `port1`
- **Outgoing Interface:** `port2`
- **Source:** `all` (for this lab; in production, restrict this to permitted source IPs where possible)
- **Destination:** `Apache_DNAT` (the VIP)
- **Service:** `HTTP` (the destination port after DNAT is applied)
- **Action:** `ACCEPT`
- **NAT:** Disabled

CLI equivalent:

```text
config firewall policy
    edit 0
        set name "WAN_to_LAN_DNAT"
        set srcintf "port1"
        set dstintf "port2"
        set srcaddr "all"
        set dstaddr "Apache_DNAT"
        set action accept
        set schedule "always"
        set service "HTTP"
        set nat disable
    next
end
```

> If the FortiGate is configured for Central NAT, the relevant VIP and policy settings may differ. Option names and commands can also vary by FortiOS version.

## Test the Connection

From Ubuntu on the WAN network, run:

```bash
curl -v http://192.168.93.154:8080
```

If successful, you should receive a response similar to:

```text
< HTTP/1.1 200 OK
< Server: Apache/2.4.41 (Ubuntu)
< Content-Type: text/html

Web Server is running
```

## Troubleshooting Checklist

- Confirm Apache is listening on `192.168.10.3:80`.
- The Apache server’s default gateway should be `192.168.10.1`.
- The policy’s incoming interface should be `port1`, and its outgoing interface should be `port2`.
- The policy destination should be the VIP, `Apache_DNAT`, not the server’s IP address directly.
- The policy service should be HTTP, or a custom TCP service with destination port `80`.
- The VIP should map external port `8080` to internal port `80`.
- For this topology, leave NAT disabled on the policy so the server can see the client’s real IP address.

## Lab Files

If `DNAT.unl` and `topology.png` are included with the lab, link them here. Merely naming these files without attaching them or providing download links does not make them accessible.

## Security Note

This configuration makes the web server reachable from the WAN. In a production environment, restrict permitted source addresses where possible, allow only the required services and ports, and use HTTPS rather than unencrypted HTTP. FortiGate documentation also describes using the VIP as the firewall policy’s destination and disabling policy NAT when preserving the client’s source IP is required. <citation src="1,3"></citation>
