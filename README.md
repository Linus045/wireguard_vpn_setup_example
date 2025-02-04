This is an example setup for a VPN using wireguard.
The server side utilizes iptables to open ports and forward the packets.


# Setup
The private and public keys should be created using wireguard tools.
The private key should be kept secret and the public key should be shared
with the peers (server or client respectively).

The command to create the keys is:
```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

The server and client create their own keys and only share the public key with
each other i.e. they enter the public key in the corresponding fields
in the configuration files (see comments in the VPN_Client.conf and VPN_Server.conf files).

# Usage
To start wireguard, load the config file with
```bash
sudo wg-quick up ./VPN_Client.conf
```

To stop wireguard, disable the config file with
```bash
sudo wg-quick down ./VPN_Client.conf
```
For the server, replace `VPN_Client.conf` with `VPN_Server.conf`.
The files can be moved to the `/etc/wireguard/` directory to be used with the `wg-quick`
command without specifying the full path.

# Debugging
To debug the connection (on server and client), use the following commands:
```bash
sudo wg show
```
Other tools like `ping`, `traceroute` and `tcpdump` can also be used to debug the connection.

# How wireguard works in short
How Packets from the client are then send through the tunnel to a device on the server's network:

Note: this is my basic understanding and not fully tested/debugged using network tools, so how exactly
the packet content is wrapped/forwarded might look different (e.g. when debugged with tcpdump)


This example assumes that a client wants to reach a webserver that is on a different network via the VPN tunnel.
To do that he connects with a VPN Server which then forwards his requests to
the Webserver and sends the response back to him.
```
PC 1 (connectred to VPN)
VPN IP: 192.168.5.3
local IP: 10.0.0.6 (10.0.0.0/24)


VPN Server (connected to VPN)
VPN IP: 192.168.5.1
local IP: 192.168.178.18 (192.168.178.0/24)

Webserver on server's network
local IP: 192.168.178.2

| = represents the LAN network
. = represents the local network stack (on the device itself)
|| = represents the VPN tunnel


PC 1 sends packet to Webserver 192.168.178.2 (on server's network)
| -------------------PC 1-(Request)------------- |
| Packet: SRC: 10.0.0.6 TARGET: 192.168.178.2    |
|------------------------------------------------|
                       .
                       .
            (PC 1 local network stack)
                       .
                       .
                       V
| -----------------PC 1----(Request) ----------------- |
| Packet: SRC: 10.0.0.6 TARGET: 192.168.178.2          |
|------------------------------------------------------|
since the target matches the IP in the AllowedIPs section of the client,
the packet will be send through the VPN tunnel interface wrapped as a
UDP packet with a new target and source ip corresponding to the VPN tunnels IPs.
| ------------------PC 1----------------------------------------------- |
| Packet: SRC: 192.168.5.3 TARGET: 192.168.5.1                          |
| >(contains original packet) SRC: 10.0.0.6 TARGET: 192.168.178.2       |
|-----------------------------------------------------------------------|
                       ||
                       ||
                       ||
           WIREGUARD VPN TUNNEL (192.168.5.0/24)
                       ||
                       ||
                       ||
                       VV
wireguard interface receives the packet and unwraps it:
| --------------------Server-----(Request) -------------- |
| Packet: SRC: 10.0.0.6 TARGET: 192.168.178.2             |
|---------------------------------------------------------|
                       .
                       .
            (Server's local network stack)
                       .
                       .
                       V
| --------------------Server-------(Request) ------------ |
| Packet: SRC: 10.0.0.6 TARGET: 192.168.178.2             |
|---------------------------------------------------------|
The firewall rules on the server will forward/nat the received packets to
allow the clients to reach devices on the server's local network e.g. a server
on 192.168.178.2
The original src ip is remembeed on the server and any response will be send back to the original src ip.
| ---------------------Server----(Request) --------- |
| Packet: SRC: 192.168.178.18 TARGET: 192.168.178.2  |
|----------------------------------------------------|
                       |
                       |
                       |
         Server's local network (192.168.178.0/24)
                       |
                       |
                       |
                       V
| -----------------Webserver----(Request) ---------- |
| Packet: SRC: 192.168.178.18 TARGET: 192.168.178.2  |
|----------------------------------------------------|
192.168.178.2 receives the packet and responds, the response is send back to the server
(src and target are simply swapped)
| ------------------Webserver---(Response)--------- |
| Packet: SRC: 192.168.178.2 TARGET: 192.168.178.18 |
|---------------------------------------------------|
                       |
                       |
                       |
         Server's local network (192.168.178.0/24)
                       |
                       |
                       |
                       V
| ------------------Server-------(Response)------------------- |
| Packet: SRC: 192.168.178.2 TARGET: 192.168.178.18            |
|--------------------------------------------------------------|
The server receives the response and sends it back to the original
source ip it remembered: 192.168.5.3
| --------------------Server------(Response)--------- |
| Packet: SRC: 192.168.178.2 TARGET: 192.168.178.5    |
|-----------------------------------------------------|
                       .
                       .
            (Server's local network stack)
                       .
                       .
                       V
since the target matches the IP of PC 1 in the AllowedIPs section of the server
the packet will be send through the VNP tunnel interface wrapped as a
UDP packet with a new target and source ip corresponding to the VPN tunnels IPs.
| ---------------------Server-------(Response)------------------- |
| Packet: SRC: 192.168.5.1 TARGET: 192.168.5.3                    |
| >(contains original packet) SRC: 192.168.178.2 TARGET: 10.0.0.6 |
|-----------------------------------------------------------------|
                       ||
                       ||
                       ||
           WIREGUARD VPN TUNNEL (192.168.5.0/24)
                       ||
                       ||
                       ||
                       VV
| ---------------------PC 1-------(Response)--------------------  |
| Packet: SRC: 192.168.5.1 TARGET: 192.168.5.3                    |
| >(contains original packet) SRC: 192.168.178.2 TARGET: 10.0.0.6 |
|-----------------------------------------------------------------|
Pc 1 receives the response on the wireguard interface and unwraps it
| ---------------------PC 1--------(Response)------------------- |
| SRC: 192.168.178.2 TARGET: 10.0.0.6                            |
|----------------------------------------------------------------|
                       .
                       .
            (PC 1 local network stack)
                       .
                       .
                       V
| ---------------------PC 1------(Response)---- |
| SRC: 192.168.178.2 TARGET: 10.0.0.6           |
|-----------------------------------------------|
Since the target ip is the device itself, it will be processed by PC 1 and we don't
need to forward anymore.
```
