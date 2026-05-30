# WireGuard

## 1. Install

**Fedora**

```Bash
sudo dnf install wireguard-tools
```

Check if done.

```Bash
wg --version
```

## 2. Generate server keys

```Bash
sudo mkdir -p /etc/wireguard
sudo chmod 700 /etc/wireguard

wg genkey | sudo tee /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key

sudo chmod 600 /etc/wireguard/server_private.key
```

Show public key:

```Bash
sudo cat /etc/wireguard/server_public.key
```

## 3. Create `/etc/wireguard/wg0.conf`

Create the following file: `/etc/wireguard/wg0.conf`

Add the following content into the file. The Address is the newly created VPN network's address.

```INI
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY_HERE
```

Where `SERVER_PRIVATE_KEY_HERE` is `sudo cat /etc/wireguard/server_private.key`.

Set correct permission: `sudo chmod 600 /etc/wireguard/wg0.conf`

## 4. Enable IP forwarding

This allows the server to route traffic between WireGuard clients and, if you want, the internet. Forwarding and NAT are separate things; for VPN routing through the server you normally need both.

```Bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
```

Optional IPv6 forwarding:

```Bash
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
```

## 5. Configure firewalld

Install and enable `firewalld`

```Bash
sudo dnf install firewalld
sudo systemctl enable --now firewalld
```

Open the WireGuard UDP port:
```Bash
sudo firewall-cmd --permanent --add-port=51820/udp
```

Enable masquerading/NAT so clients can access the internet through the cloud server:
```Bash
sudo firewall-cmd --permanent --add-masquerade
```

Reload
```Bash
sudo firewall-cmd --reload
```
Also make sure you are allowed inbound `51820` port via UDP.

## 6. Start WireGuard

```Bash
sudo systemctl enable --now wg-quick@wg0
```

## 7. Add your first client

Generate keys on your client device (laptop, phone, client).

```Bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

Now add the public key with the following block to the server's `/etc/wireguard/wg0.conf` file.

```INI
[Peer]
PublicKey = CLIENT_PUBLIC_KEY_HERE
AllowedIPs = 10.8.0.2/32
```

Restart WireGuard.
```Bash
sudo systemctl restart wg-quick@wg0
```

## 8. Configure your client

Find out your server's IP address. Probably something like `dev eth0`, when the following command run.

```Bash
ip route get 1.1.1.1
```

Use this to set up your client. On the client machine do the following:

```Bash
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY_HERE
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY_HERE
Endpoint = YOUR_SERVER_STATIC_PUBLIC_IP:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

`AllowedIPs = 0.0.0.0/0` means **route all cleitn internet traffic through the cloud server**.

For a split tunnel - only access your VPN network, not all internet - use:
```INI
AllowedIPs = 10.8.0.0/24
```
E.g. each machine can get a different IP, that is accessing the given network. Meaning, each has a `[Peer]` block in the server's config.

## 9. Useful checks

On server:
```Bash
sudo wg
sudo journalctl -u wg-quick@wg0 -e
sudo ss -lunp | grep 51820
```

From client:
```Bash
ping 10.8.0.1
curl ifconfig.me
```
