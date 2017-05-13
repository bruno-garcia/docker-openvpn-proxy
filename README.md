Proxy through an OpenVPN client connection without routing all traffic through the tunnel (route-nopull).

This is a split tunnel adaptation of [pbrisbin/docker-openvpn-proxy](https://github.com/pbrisbin/docker-openvpn-proxy). Originally the OpenVPN client was configured as 'full tunnel'.

The routes of which you expect to be sent through the VPN can be added once the container is running.
A Docker host can be used as a gateway. Routing through the VPN only what you explicitly defined via routing table.
All traffic of a specific company (i.e Netflix) can be redirected through the tunnel. [It can also be automated](https://github.com/bruno-garcia/IPInfo.IO.IPAddressBlockParser).

## Usage

```console
docker run \
  --name vpn \
  --cap-add=NET_ADMIN \
  --publish 8118:8118 \
  --env VPN_USER=xxx \
  --env VPN_PASS=xxx \
  --env VPN_GATEWAY=ch1.vpn.giganews.com \
  --env VPN_CERTIFICATE=ca.vyprvpn.com.crt \
  --detach \
  pbrisbin/openvpn-proxy
```

**Note**: `VPN_CERTIFICATE` can be an absolute path or relative to
`/etc/openvpn`. If the cert you intend to use is not present on the basic Alpine
system, be sure to pass it in via `--volume`.

```console
% curl ifconfig.me
<your actual public IP>
% http_proxy=http://localhost:8118 curl ifconfig.me
<still your actual public IP>

<change route to ifconfig.me>
% dig +noall +answer ifconfig.me | egrep -o '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' | while read x; do route add $x dev tun0; done

% curl ifconfig.me
<The IP address of your VPN provider>
% http_proxy=http://localhost:8118 curl ifconfig.me
<The IP address of your VPN provider>
```

Devices that support HTTP proxy like a browser can use that while a smart TV can have its default gateway changed to the Docker host.

Traffic of a whole AS can be redirected with:

```console
git clone https://github.com/bruno-garcia/IPInfo.IO.IPAddressBlockParser.git
cd IPInfo.IO.IPAddressBlockParser/IPInfo.IO.IPAddressBlockParser/
dotnet run -- AS2906 | while read x; do route add -net $x dev tun0; done
```

An example on how to enable routing on a RedHat based ditro and NAT with iptables:

```console
sysctl -w net.ipv4.ip_forward=1
iptables -A FORWARD -i eth0 -o tun0 -s 192.168.1.0/24 -m conntrack --ctstate NEW -j ACCEPT
iptables -A FORWARD -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
iptables -A POSTROUTING -t nat -o tun0 -j MASQUERADE
```

Note: that assumes eth0 as interface to the local network and subnet 192.168.1.0/24
