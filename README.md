StormVPN version 0.0.6
============================
Copyright (c) Netstorm, 2025

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
         proxy_buffering off;
        }

        access_log  /dev/null;
        error_log /dev/null crit;
}
```

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
interface_name= iface1
mac_address=00:11:22:33:44:55
link_type=none 
```

Thus, the test1 peer contains the secret key for authorization test1key, the interface name that will be assigned at server startup - iface1, mac_address in this case will be generated automatically (unicast MAC), and link_type= bridge:bridge1:bridge2 means that the program will join the interface on the server to bridge bridge1 and on the client to bridge2 (bridges must be created in advance on both sides). As you can see, mac_address can be set manually and link_type can be none, which means that two Ethernet interfaces will be created and tunneled together.
In version 0.0.4 a new link_quality key has been added for for each individual peer, which can take the following values - auto, none and **bw**M, where **bw** is the value in megabits/second. This option enables test traffic within the VPN, thereby measuring the connection speed. The connection speed statistics are then sent to the server console and can also be displayed in the stormvpn-stats console utility, which has also been available since version 0.0.4. Test traffic stops moving through the channel as soon as real client traffic appears. This option allows you to monitor the channel quality when there is no activity in the channel.

```
Peer Name            | Connected At           | Last Activity          | Avg Up (Mbps)   | Avg Down (Mbps)
--------------------+------------------------+------------------------+-----------------+-----------------
test1                | 15:48:49 10.04.2025    | 15:49:37 10.04.2025    | 8.93            | 8.90           
                     | (48s ago)              | (0s ago)               |                 |                
--------------------+------------------------+------------------------+-----------------+-----------------
```

In version 0.0.5 an additional key was added to peers.conf - bridge_vlan, which has the format LOCAL_VLAN:REMOTE_VLAN. This key is relevant only in link_type=bridge mode. It specifies the ability to filter and retag traffic using 802.1q. LOCAL_VLAN is the server-side VLAN and REMOTE_VLAN is the client-side VLAN. Thus, these VLANs can be different, but they will enter the tunnel without a tag, and will be modified only when exiting to bridge, which allows re-tagging traffic. VLAN=0 implies a full trunk - tagged + untagged.

Example of server.crt and server.key generation:
```
# openssl req -x509 -newkey rsa:4096 -keyout server.key -out server.crt -sha256 -days 3650 -nodes -subj "/C=XX/ST=StateName/L=CityName/O=CompanyName/OU=CompanySectionName/CN=CommonNameOrHostname"
```

Since version 0.0.6, the server supports SIGHUP signal to reload the part of the configuration.

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
-x - use SOCKS5 proxy for connection (format: user:password@ip:port or ip:port).

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

**Licensing**

Without the correct key, the server operates in restricted mode and can accept only one connection. No other restrictions apply. If you need more than one connection per server, you will need a key. To obtain a key, you can contact us by providing the request code received when starting an unregistered server.

**Contacts**

Telegram: el_stormi
