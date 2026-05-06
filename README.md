# WireGuard VPN Server
This project is a tutorial on how to set up and secure your very own self hosted VPN server using WireGuard and running on an Ubuntu server.

WireGuard is a great VPN choice because it is faster than OpenVPN, has a simple setup, and uses modern cryptography/protocols. 

**Features:** 
- Self hosted VPN using WireGuard 
- Full tunnel configuration 
- Secure key based authentication 
- Multi client support 
- Persistent connection

**Requirements:** 
- Ubuntu server (cloud or local) 
- Public IP address 
- Open UDP port 51820

# How It Works
Client → encrypted tunnel → VPN Server → Internet

- Client encrypts traffic using WireGuard
- Server decrypts and forwards it (NAT)
- Responses are sent back through the tunnel


[ Laptop ] ----->  Client \
&emsp; | \
&emsp; |  (Encrypted Tunnel - WireGuard) \
&emsp; | \
[ VPN Server ] -----> Internet

# Installation & Initial Config

Start by connecting to your server and updating.
```
sudo apt update && sudo apt upgrade -y
```

Next, install WireGuard:
```
sudo apt install wireguard -y
```

Now we must enable IP forwarding on the server. This allows the VPN server to route traffic between VPN clients and the internet.
```
sudo nano /etc/sysctl.conf
```
Make sure to uncomment ```net.ipv4.ip_forward=1``` and save/close the file.

<img width="1599" height="730" alt="Screenshot 2026-05-05 214759" src="https://github.com/user-attachments/assets/11dab89b-fedc-45ec-91e9-7768b31acd41" /> 

Now we must tell the system to load configs from the sysctl.conf file to ensure changes persist.
```
sudo sysctl -p
```

# Key Pair Generation

> [!WARNING]  
> Never expose your private keys publicly (GitHub, logs, screenshots etc)


To generate our keys we must do the following:
```
cd /etc/WireGuard
umask 077
wg genkey | tee server_private.key | wg pubkey > server_public.key
```

Make sure you run "ls -l" to make sure that the files are as follows: 
```
-rw------- server_private.key 
-rw------- server_public.key
``` 
This ensures that your keys are not exposed.

# WireGuard Configuration File & Continued Setup

Next we must modify the WireGuard configuration file so that it can run as a server.
```
sudo nano /etc/WireGuard/wg0.conf
```
Copy and paste this into your file and replace the values as needed:
```
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = [Private Key]

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE
```

> [!IMPORTANT]  
> Make sure that you change the network interface to the main one that you are using. For example if you are using AWS, it is commonly ens5 instead of eth0. You can run ```ip -o -4 route show to default | awk '{print $5}'``` to check your default route (internet) interface.


Then run these commands to lock down sensitive files:
```
chmod 600 /etc/WireGuard/*.key /etc/WireGuard/wg0.conf
chmod 700 /etc/WireGuard
```

Then finally we must make it so WireGuard runs persistently on boot:
```
sudo systemctl start wg-quick@wg0
sudo systemctl enable wg-quick@wg0
```

You can verify that WireGuard is running correctly with these commands:
```
sudo systemctl status wg-quick@wg0  # checks systemctl status
sudo wg show wg0  # verify wg is up/checks live view of wg interface
ip a show wg0  # verify interface exists (should be what you set, ex. 10.0.0.1)
```

# Adding Clients to VPN Server

> [!TIP]
> You can generate client keys on your server or the client device, but it is simpler to do so on the server in my opinion.

Start by installing WireGuard on your client device. I will go over how I did it on a Linux machine.

Begin with generating new keys on your server.
```
cd /etc/WireGuard
wg genkey | tee laptop_private.key | wg pubkey > laptop_public.key
```

Then, add your client to the bottom of the server configuration file.
```
sudo nano /etc/WireGuard/wg0.conf
```
```
[Interface]
Address = 10.0.0.1/24
ListenPort = 51820
PrivateKey = [Private Key]

PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o ens5 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o ens5 -j MASQUERADE

[Peer]
PublicKey = [Laptop Public Key]
AllowedIPs = 10.0.0.2/32
```

Now, switch over to your client and modify the WireGuard configuration file there:
```
sudo nano /etc/WireGuard/wg0.conf
```
```
[Interface]
PrivateKey = [Laptop Private Key]
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = [Server Public Key]
Endpoint = [Server Public IP]:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Then, back on your server, restart WireGuard for the changes to apply:
```
sudo systemctl restart wg-quick@wg0
```

And start the WireGuard service on the client and try to connect to the VPN server:
```
sudo wg-quick up wg0
```

You can try to verify that everything is working by using ```sudo wg``` on both the server and client. \
You should see something along the lines of:
"latest handshake: X seconds ago"

You can also visit a site such as [WhatsMyIP](https://whatismyipaddress.com/). If it is working, then you should see the public IP of your server. \
You can add as many clients as you want, just repeat the following steps and make new key pairs for each client.

**Congrats, you now own your very own private VPN server!**  🎉 

# Tips for Extra Security
> [!IMPORTANT]  
> If you plan to use this in place of your current VPN, make sure to follow these suggestions.

1. Persist WireGuard on boot 
2. Restrict security groups if possible
3. Lock down server access. Only allow: 
   SSH (22) - ideally restricted to your IP only 
   WireGuard (51820 UDP) - port that WireGuard utilizes 
4. Add DNS leak safety (optional) 
   ```
   In client config: 
   DNS = 1.1.1.1 
   ```
   I recommend 1.1.1.1 and 9.9.9.9 but you can replace it with a DNS of your choosing.

**Hardening (Recommended)**

- Disable password SSH login
- Use SSH keys only
- Change SSH port (optional)
- Use UFW or iptables firewall
- Keep system updated

**Threat Model Considerations**

- Protecting private keys from exposure
- Preventing unauthorized VPN access
- Minimizing exposed services to reduce attack surface
- Securing SSH access to the server
  
# Troubleshooting Issues & Good to Knows
- No internet but connected \
  → NAT rule or wrong interface 

- No handshake \
  → Port 51820 blocked or wrong IP 

- Handshake but no traffic \
  → AllowedIPs misconfigured 

- Works locally but not remotely \
  → Cloud firewall / port forwarding issue 

**Important:** \
Server does NOT “dial” clients \
Clients initiate connection \
Server just waits and responds 
