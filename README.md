StormVPN version 0.1.1
============================
Copyright (c) Netstorm, 2026

> Classic “unnecessary”, but what the hell - why isn't it free???

StormVPN is an implementation of Ethernet tunneling protocol over HTTP/HTTPS to provide full Ethernet connectivity. The package includes both StormVPN client and server, is distributed as binary files only and has license restrictions. Both server and client run as root only.

**Server**

The server does not require any command line parameters, all settings are taken from the configuration files located at /etc/storm-vpn.

1) /etc/storm-vpn/server.conf
```
listen_ip=0.0.0.0
url=storm-vpn-endpoint
https_port=443
http_port=80
standalone=true|false
client_timeout=30
server_crt=/etc/storm-vpn/server.crt
server_key=/etc/storm-vpn/server.key
serial=demo
```

listen_ip - the address where the server will listen. standalone - true or false defines the server operation mode, in case of standalone=true the server will work on https_port and perform encryption on its own, server_crt and server_key will also be used. This mode implies that the server works independently - without web services on one IP. In the case of standalone=false, the server works on the http_port and essentially means working behind an HTTP(s) proxy, such as Nginx. Example of Nginx configuration to work with StormVPN:

```
server {
        listen          0.0.0.0:80;
        server_name     storm-vpn.your-domain.com;
        rewrite ^(.*) https://storm-vpn.your-domain.com$1 permanent;
        return 301 https://storm-vpn.your-domain.com$request_uri;
        access_log  /dev/null;
        error_log /dev/null crit;
}

server {
        listen                  0.0.0.0:443 ssl;
        server_name             storm-vpn.your-domain.com;

        # ssl                     on;
        ssl_certificate         /etc/letsencrypt/live/your-domain.com/fullchain.pem;
        ssl_certificate_key     /etc/letsencrypt/live/your-domain.com/privkey.pem;

        location /storm-vpn-endpoint {
         proxy_pass http://<storm-vpn-ip>;
         proxy_http_version 1.1;
         proxy_set_header Upgrade $http_upgrade;
         proxy_set_header Connection "upgrade";
         proxy_set_header X-Forwarded-For $remote_addr;
         proxy_buffering off;
        }

        access_log  /dev/null;
        error_log /dev/null crit;
}
```

Note: Use X-Forwarded-For to pass the client's IP address to the server behind the Nginx reverse proxy.

url - suffix of HTTP URL to connect to VPN, all other paths will give 404 error, full path to connect will look like https://storm-vpn.your-domain.com/<url> (storm-vpn-endpoint - default). client_timeout - session timeout in seconds, when this timeout expires and there is no data and Keepalive(s) the session will be terminated.

2) /etc/storm-vpn/peers.conf
Contains a description of peers to connect to the server in INI file format, for example:
```
[test1]
secret_key=test1key
interface_name=iface1
mac_address=auto
link_type=bridge:bridge1:bridge2
link_quality=auto
bridge_vlan=15:12

[test2]
secret_key=test2key
interface_name=iface2
mac_address=00:11:22:33:44:55
link_type=none

[test3]
secret_key=test3key
interface_name=iface3
mac_address=auto
link_type=ip:30:192.168.0.1:192.168.0.2:dynamic:50
link_type=2M
bandwidth=10M
```

Thus, the test1 peer contains the secret key for authorization test1key, the interface name that will be assigned at server startup - iface1, mac_address in this case will be generated automatically (unicast MAC), and link_type=bridge:bridge1:bridge2 means that the program will join the interface on the server to bridge bridge1 and on the client to bridge2 (bridges must be created in advance on both sides). As you can see, mac_address can be set manually and link_type can be none, which means that two Ethernet interfaces will be created and tunneled together.
In version 0.0.4 a new link_quality key has been added for for each individual peer, which can take the following values - auto, none and **bw**(K|M), where **bw** is the value in (kilobits|megabits)/second. This option enables test traffic within the VPN, thereby measuring the connection speed. The connection speed statistics are then sent to the server console and can also be displayed in the stormvpn-stats console utility, which has also been available since version 0.0.4. Starting with version 0.1.0, this option stops measuring line quality when useful (DATA) traffic appears only in auto mode. When a constant value is specified for link_quality, measurement always occurs.
Starting with version 0.0.9, a bandwidth parameter was added to peers to limit the peer speed in the form of **bw**(K|M), where **bw** is the speed value in (kilobits/megabits)/second. Also, in version 0.0.9, a kick function was added to the server socket to drop user session, which was also added to stats and the web interface.
Starting with version 0.1.0, an additional parameter for peers has been introduced – ban_quality, which is set to a value of the form <INT>(K|M). When the link_quality measurement (not in auto mode) falls below ban_quality, the peer's useful (DATA) traffic is blocked. This allows unreliable peers to be excluded from the multi-link scheme. When link_quality returns to normal values, the peer is automatically unblocked.

```
Peer Name            | Client IP            | Connected At           | Last Activity          | Bandwidth (Mbps)   | LQ Up (Mbps)    | LQ Down (Mbps)  | Avg Up (Mbps)   | Avg Down (Mbps)
--------------------+----------------------+------------------------+------------------------+--------------------+-----------------+-----------------+-----------------+-----------------
test3                | 192.168.83.145:37560 | 16:17:32 19.08.2025    | 16:17:47 19.08.2025    | 10.00              | 2.00            | 2.00            | 0.00            | 0.00           
                     |                      | (15s ago)              | (0s ago)               |                    |                 |                 |                 |                
--------------------+----------------------+------------------------+------------------------+--------------------+-----------------+-----------------+-----------------+-----------------
```

In version 0.0.5 an additional key was added to peers.conf - bridge_vlan, which has the format LOCAL_VLAN:REMOTE_VLAN. This key is relevant only in link_type=bridge mode. It specifies the ability to filter and retag traffic using 802.1q. LOCAL_VLAN is the server-side VLAN and REMOTE_VLAN is the client-side VLAN. Thus, these VLANs can be different, but they will enter the tunnel without a tag, and will be modified only when exiting to bridge, which allows re-tagging traffic. VLAN=0 implies a full trunk - tagged + untagged.

Example of server.crt and server.key generation:
```
# openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=CommonNameOrHostname"
```

Since version 0.0.6, the server supports SIGHUP signal to reload the part of the configuration.

In version 0.0.7 a new link_type=ip mode was added, which solves the issue of working in IPv4 networks without additional software such as BGP.
The format of the link_type=ip:NETMASK:SERVER_IP:CLIENT_IP:MODE:METRIC entry, where:
- NETMASK is the subnet mask for the junction between peers,
  for example 30;
- SERVER_IP is the server-side IP address that will be assigned to the
  VPN interface;
- CLIENT_IP is the client-side IP address that will be assigned to the VPN
  interface;
- MODE is the mode of operation: static, dynamic, or full. Static implies
  IP assignment to interfaces, Dynamic implies IP assignment to interfaces
  and dynamic route exchange during the session, but except for Default,
  Link-Local and Connected, and Full implies additionally Connected route
  exchange. The last mode should be used carefully and most likely you
  will be satisfied with Dynamic for dynamic routing;
- METRIC is a metric for adding routes on the server and client.

**Client**

The client, on the other hand, supports only the command line. Mandatory parameters for it are:\
-u - URL to connect to the server, e.g. https://storm-vpn.your-domain.com/storm-vpn-endpoint \
-p - the name of the peer to connect to;\
-s - the secret of the peer to connect to;\
-i - name of the client-side interface.\

Optional parameters:\
-I - ignore certificate validation problems (Insecure connection);\
-k - Keepalive sending interval (seconds);\
-m - MAC address for client-side interface;\
-r - limit the number of reconnections (0 - do not limit);\
-x - use SOCKS5 proxy for connection (format: user:password@ip:port or ip:port);\
-d - DSCP value for marking outgoing HTTP(s) connection (0-63).

**Benchmarking**

Direct iperf (vm-vm), dual-test:
```
[  5]  0.0-10.0 sec  4.08 GBytes  3.51 Gbits/sec
[  4]  0.0-10.0 sec  3.33 GBytes  2.86 Gbits/sec
```

Over tunnel (vm-vm), dual-test:
```
[  5]  0.0-10.1 sec   192 MBytes   160 Mbits/sec
[  4]  0.0-10.1 sec   180 MBytes   149 Mbits/sec
```

Over tunnel (vm-vm), dual-test, with VLAN retag:
```
[  5]  0.0-10.0 sec   163 MBytes   137 Mbits/sec
[  4]  0.0-10.1 sec   154 MBytes   127 Mbits/sec
```

**Docker**

Starting with version 0.0.8, we are discontinuing platform-dependent packages and leaving support for Docker images only.
You can use Docker images to avoid installing system-dependent packages. Official repository https://hub.docker.com/r/stormotron/stormvpn-server and https://hub.docker.com/r/stormotron/stormvpn-client .

Server environment variables:
```
- LISTEN_IP      - IP, default 0.0.0.0
- HTTP_PORT      - Port, default 80
- HTTPS_PORT     - Port, default 443
- URL            - String, default storm-vpn-endpoint
- STANDALONE     - Boolean, default true
- CLIENT_TIMEOUT - Int, Timeout, default 30
- SERIAL         - String, Serial, default demo
- SERVER_CRT     - String, Server certificate, default selfsigned
- SERVER_KEY     - String, Server certificate key, default selfsigned
- CLIENT<NUM>    - String, clients description, no defaults - MUST be defined

Client examples:
CLIENT1=peer=peer01,secret=peer01secret,bandwidth=2M
CLIENT2=peer=peer02,secret=peer02secret,interface=peer02,mac=00:11:22:33:44:55
CLIENT4=peer=localpeer,secret=localpeersecret,link_quality=5M,ban_quality=2M
CLIENT5=peer=brpeer,secret=brpeersecret,link_quality=auto,link_type=bridge:mybridge:clbridge
CLIENT6=peer=brpeer2,secret=brpeersecret2,link_type=bridge:mybridge:clbridge,bridge_vlan=10:16
CLIENT7=peer=ippeer,secret=ippeersecret,link_type=ip:30:192.168.0.1:192.168.0.2:dynamic:50

- WEB_PORT       - Int, Web interface port, enables WEB interface (configuration via WEB), no default
- WEB_IP         - IP, Web interface IP, default: 0.0.0.0
- WEB_PASSWORD   - String, Web interface password, default admin
```

Starting server (without envs):
```
# docker run --cap-add=NET_ADMIN --device /dev/net/tun:/dev/net/tun --net=host --name stormvpn-server -d stormotron/stormvpn-server:latest
```

Stats:
```
# docker exec -it stormvpn-server stormvpn-stats
```

Config reload:
```
# docker exec -it stormvpn-server stormvpn-server --reload
```

Simple server configuration with envs in compose:

```
name: stormvpn-server
services:
    stormvpn-server:
        image: stormotron/stormvpn-server:latest
        cap_add:
            - NET_ADMIN
        devices:
            - /dev/net/tun:/dev/net/tun
        network_mode: host
        environment:
            - STANDALONE=false
            - HTTP_PORT=888
            - URL=custom-storm-vpn-endpoint
            - CLIENT1=peer=tunnel01,secret=SUPERSECRET,interface=tunnel01,link_type=bridge:bridges:bridgec,bridge_vlan=35:41
            - SERIAL=MYSERIAL
        restart: unless-stopped
        container_name: stormvpn-server
```

Simple server configuration with WEB interface:

```
name: stormvpn-server
services:
    stormvpn-server:
        image: stormotron/stormvpn-server:latest
        cap_add:
            - NET_ADMIN
        devices:
            - /dev/net/tun:/dev/net/tun
        network_mode: host
        environment:
            - WEB_PORT=8080
            - WEB_IP=192.168.84.5
            - WEB_PASSWORD=MY_PASSWORD
        volumes:
            - /opt/stormvpn/configs/server.conf:/etc/storm-vpn/server.conf
            - /opt/stormvpn/configs/peers.conf:/etc/storm-vpn/peers.conf
        restart: unless-stopped
        container_name: stormvpn-server
```

Client environment variables:
```
- URL       - String, default https://vpn.your.domain/storm-vpn-endpoint
- PEER      - String, default test
- SECRET    - String, default test
- INTERFACE - String, default vpn_<RANDOM NUMBER>
- INSECURE  - Boolean, default false
- PROXY     - String, format: user:password@ip:port, no default value
- DSCP      - Integer [0-63], no default value
```

Starting client (without envs):
```
# docker run --cap-add=NET_ADMIN --device /dev/net/tun:/dev/net/tun --net=host --name stormvpn-client -d stormotron/stormvpn-client:latest
```

Basic example:

Server side (IP 192.168.118.1):
```
# docker run --cap-add=NET_ADMIN --device /dev/net/tun:/dev/net/tun --net=host -e CLIENT1="peer=test,secret=test" --name stormvpn-server -d stormotron/stormvpn-server:latest
```

Client side (IP 192.168.118.2):
```
# docker run --cap-add=NET_ADMIN --device /dev/net/tun:/dev/net/tun --net=host --name stormvpn-client -e URL=https://192.168.118.1/storm-vpn-endpoint -e PEER=test -e SECRET=test -e INSECURE=true -d stormotron/stormvpn-client:latest
```

**Licensing**

Without the correct key, the server operates in limited mode and can only accept one connection. Also, starting with version 0.0.9, the unregistered version of the server has a session duration limit of 10 minutes. If you need more than one connection to the server or the session should not be interrupted, you will need a key. To obtain a key, you can contact us by providing the request code received when starting the unregistered server. Starting with version 0.0.7, a one-week trial key has been added to the licensing model, which you can order for free by contacting us. Please order a free trial key before purchasing a full key to see if the product meets your expectations.

**Contacts**

Telegram: el_stormi

**Updates:**

* 0.1.1
Starting with version 0.1.1, the server-side protection of the program has been updated, new serial keys are now in effect, and the request code format has changed. The old keys are no longer valid. The signature keys for the Android application have also changed. It is not possible to update the Android client from version 0.1.0 to version 0.1.1 while retaining data. Improvements have been made to the server side, and the Android client has been refined and improved with support for profiles in widgets.
