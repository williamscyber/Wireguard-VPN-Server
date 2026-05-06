# vpn-server
Fully self hosted VPN server

used an ubuntu server with a public ip

sudo apt update & sudo apt upgrade -y

now we must enable ip forwarding on the server. this allows vpn server to route traffic between vpn clients and the internet.
sudo nano /etc/sysctl.conf
uncomment "net.ipv4.ip_forward=1"
save
sudo sysctl -p
tells it to load configs from the sysctl.conf file to ensure changes persist 

WARNING :Never expose private keys publicly (GitHub, logs, screenshots)

now we add keys
cd /etc/wireguard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
make sure to run ls -l to make sure that the files are as follows:
-rw------- server_private.key
-rw------- server_public.key

now we make the wireguard config file
sudo nano /etc/wireguard/wg0.conf

[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = CL7Tjce7AtBmwl0YOdOdS7DPzTosvTScZ0FX01H/Clw=

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE

make sure you change the network interface to the main one you are using, for example if you are using AWS it is commonly ens5 instead of eth
you can run run ip -o -4 route show to default | awk '{print $5}' to check your default route (internet) interface

then run these to just lock down sensitive files
chmod 600 /etc/wireguard/*.key /etc/wireguard/wg0.conf
chmod 700 /etc/wireguard

now to set it so wireguard runs automatically on boot
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0

verify its working
sudo systemctl status wg-quick@wg0 - checks systemctl status
sudo wg show wg0 - verify wg is up/checks live view of wg interface
ip a show wg0 - verify interface exists (should be what you set, ex. 10.0.0.1)

add clients to vpn server
go to client device of your choice, i will be using an linux mint laptop
install wireguard
you can generate client keys on server or the device, but it is simpler on the server in my opinion

cd /etc/wireguard
wg genkey | tee laptop_private.key | wg pubkey > laptop_public.key

add client to server conf file
sudo nano /etc/wireguard/wg0.conf

add to bottom:
[Peer]
PublicKey = <LAPTOP_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32

on the client:
sudo nano /etc/wireguard/wg0.conf
add:
[Interface]
PrivateKey = <LAPTOP_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = <AWS_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25

then back on your server restart wireguard:
sudo systemctl restart wg-quick@wg0

then start service on client and try to connect to vpn server
sudo wg-quick up wg0

you can try to verify by using sudo wg on both the server and client
you should see something along the lines of 
"latest handshake: X seconds ago"

then you can also visit a site such as whatsmyip, if it is working then you should see the public ip of your server
you can also add as many clients as you want, just make new keypairs for all
congrats, you now own a private vpn server!

tips for extra security
if you plan to use this in place of your current vpn, make sure to add these
1. Persist WireGuard on boot
2. 2. Restrict AWS security group, only allow your client ips in the server
   3. 3. Lock down server access

Only allow:

SSH (22) ideally restricted to your IP only
WireGuard (51820 UDP)

4. Add DNS leak safety (optional)
5. In client config:
6. DNS = 1.1.1.1

7. you

🔧 Common issues
AWS Security Group not open on UDP 51820
wrong interface name (ens5 vs eth0)
missing NAT rule → VPN connects but no internet
forgot AllowedIPs = 0.0.0.0/0 (full tunnel issue)

important:
Server does NOT “dial” clients
Clients initiate connection
Server just waits and responds
