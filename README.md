# Setup wireguard on your server to have your own VPN

## Install wireguard on your server

```
sudo apt install wireguard
```

## Generate the server key pair

```
wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

To see your public key

```
cat /etc/wireguard/publickey
```

To see your private key

```
cat /etc/wireguard/privatekey
```

## Create Wireguard configuration file

```
nano /etc/wireguard/wg0.conf
```

Fill with this information

```
[Interface]
Address = 10.0.0.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey =
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

`ListenPort`: 51820 is the default port used by wireguard, but you can change it to any unused port. Hackers will tend to target this port to crack wireguard.  
`PrivateKey`: Replace with the private key generated on the server.  
`PostUp/PostDown`: replace `eth0` with your server interface. run `ip a` and check for the interface that shows the server public address.

## Change permissions of wg0.conf

Make sure that no one can read the file since it contains a private key

```
chmod 600 /etc/wireguard/wg0.conf
```

## Open up port on UFW firewall

```
ufw allow 51280
```

Replace the port with your chosen port

## Enable packet forwarding

```
nano /etc/sysctl.conf
```

Uncomment this line and save

```
net.ipv4.ip_forward=1
```

Reset networking

```
sysctl -p
```

## Start Wireguard

```
wg-quick up wg0
```

## Start Wireguard on boot

If you want wireguard to always start with the server

```
systemctl enable wg-quick@wg0
```

# Setup Wireguard on your computer

## Install the wireguard client

```
https://www.wireguard.com/install
```

Select `Add empty tunnel`. It will generate a public key and private key. Keep the public key somewhere, we'll use it later.  
Add this to the configuration (without erasing what's already there)

```
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <SERVER_IP>:<SERVER_PORT>
```

`DNS` if you are using [pi-hole](https://github.com/m1rkwood/pihole-docker) or [adguard](https://github.com/m1rkwood/adguard-docker), put 10.0.0.1 as your DNS instead of `1.1.1.1` (Cloudflare).

## Add your key to the server

Go to your server and run this command, using the public key generated in Wireguard on your computer (this will be remove when you reboot the server)

```
wg set wg0 peer <COMPUTER_PUBLIC_KEY> allowed-ips 10.0.0.2
```

As an alternative, you can edit `/etc/wireguard/wg0.conf` and add these lines so that it stays permanent

```
[Peer]
PublicKey = <COMPUTER_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
```

Then run `wg addconf wg0 <(wg-quick strip wg0)` to add this config to the running wireguard.  
Note that you'll have to increment the IP for each new peer added.

## Activate

Go back to Wireguard on your computer and choose `Activate` to start the VPN.

(You can also type `wg` on the server to see your peer added)

# Remove a key from the server

To find the IDs of the peers, just type `wg`

```
wg set wg0 peer <KEY_TO_REMOVE> remove
```

# Setup Wireguard on iOS

## Generate a pair of keys for your device
It's easier to generate the keys on the server. We will use a QR Code to transfer the configuration to your device.

Go to the Wireguard folder on the server
```
cd /etc/wireguard
```
Generate a pair of keys
```
wg genkey | tee ios-privatekey | wg pubkey > ios-publickey
```
## Add the public key to the server
Same step as for the Computer setup, don't forget to increment the IP for `Address`.

## Create a configuration for your device
Still on the server (i.e. `/etc/wireguard`), we will generate a configuration file that will be sent to your device as a QR Code:
```
vim ios.conf
```

```
[Interface]
PrivateKey = <IOS_PRIVATE_KEY>
Address = 10.0.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = <SERVER_IP>:<SERVER_PORT>
```

`DNS` if you are using [pi-hole](https://github.com/m1rkwood/pihole-docker) or [adguard](https://github.com/m1rkwood/adguard-docker), put 10.0.0.1 as your DNS instead of `1.1.1.1` (Cloudflare).

## Generate a QR code
```
apt install qrencode
```
```
qrencode -t ansiutf8 < /etc/wireguard/ios.conf
```
Simply scan the code generated with the Wireguard iOS App and you're done!
