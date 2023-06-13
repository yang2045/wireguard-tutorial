## Requirements
- An account on a cloud platform that offers a virtual machine (e.g. AWS, Azure, Google Cloud, etc.).
- Ubuntu 20.04 Server Virtual Machine.
- A public IP address assigned to your VM.
- UDP port 51820 open to incoming traffic from all sources (0.0.0.0/0).

## **Introduction**
A VPN, or Virtual Private Network, is a secure and private network connection that allows you to browse the internet safely and privately. In this tutorial, we will set up a WireGuard VPN server on an Ubuntu 20.04 server running on AWS.

Wireguard is a simple-to-use VPN that uses end-to-end encryption, is more useful than IPSEC, and promises to be faster than OpenVPN.

-------------------------------------------------------------

**1: Server WireGuard Configuration**

**Step 1: Launch an Instance on AWS EC2**

First, we need to launch an Ubuntu 20.04 server on AWS. Follow the steps to launch a new instance in the AWS EC2 console.

**Step 2: Install WireGuard**
Once your instance is up and running, we can install WireGuard using the following commands in the terminal:


**Step 3: Configurando Wireguard Server na instância da AWS**

Great! Now that we have generated the keys, we can proceed to setting up the Wireguard server on the AWS instance.

First, let's update the Ubuntu repository list. Run the following command:

```
$ sudo apt update
```

Next we will install the Wireguard package:

```
$ sudo apt install wireguard
```

With the package installed, we need to create a directory to store the Wireguard tunnel keys and settings:

```
$ sudo mkdir /etc/wireguard/
```

That done, we can now proceed to the Wireguard interface configuration.

Great! Now let's generate the appropriate private and public keys for the Wireguard server:


```
$ wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

For a greater understanding, a Wireguard tunnel is encrypted using an asymmetric, i.e. dual key, encryption system. Your private key is unique and should never be shared, unlike the public key which is what will be shared to the VPN clients.

Now we will need to create and configure the Wireguard interface file that will serve as the gateway within our VPN.

Create and edit the file wg0.conf:

```
$ sudo nano /etc/wireguard/wg0.conf
```

- wg0 is the configuration file for the wg0 interface.

Enter the following settings:

```
[Interface]
Address = 10.0.0.1/24
SaveConfig = true
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o ens3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o ens3 -j MASQUERADE
```

Replace <server_private_key> with the private key you generated earlier. The AllowedIPs option specifies the IP address range that will be routed through the VPN.

**Step 4: Start the WireGuard Service**
Once the configuration is complete, we can start the WireGuard service:

```
$ sudo systemctl enable wg-quick@wg0
$ sudo systemctl start wg-quick@wg0
```

**Step 5: Enable IP Forwarding**

We need to enable IP forwarding to allow traffic to pass through the VPN:

```
$ echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

```
$ sudo sysctl -p
```

Let me explain each setting:

- The [Interface] section defines the server's network information, including the IP address of the wg0 interface that the WireGuard server will use, the network CIDR block (10.0.0.0/24), and the port the server will listen on for incoming connections (51820).

- The SaveConfig option is set to true, so that the settings can be saved and retained after a system restart.

- The PrivateKey option is the private key for the WireGuard server, which is used for authentication and encryption of connections. It is important that this key is kept secure and not shared with anyone.

- The PostUp and PostDown options are the commands that will be executed after creating the wg0 interface and after removing it, respectively. These commands add and remove the necessary firewall rules to allow traffic to pass through the tunnel created by the VPN and to redirect traffic to the outgoing interface of the virtual machine.

- In short, these settings allow the WireGuard server to function properly, protecting and encrypting VPN traffic and allowing VPN clients to access the Internet through the server's network interface.

**Now let's turn on this wg0 interface, use the following command:**

```
$ sudo wg-quick up wg0
```

-------------------------------------------------------------

**2: Client WireGuard Configuration**

- Client:

```
$ sudo apt update
```

```
$ sudo apt install wireguard
```

```
$ wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

```
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.0.0.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = SERVER_IP_ADDRESS:51820
AllowedIPs = 0.0.0.0/0
```

- The [Interface] section defines the server's network information, including the IP address of the wg0 interface that the WireGuard server will use, the network CIDR block (10.0.0.0/24), and the port the server will listen on for incoming connections (51820).

- In the peer configuration, you should point to the server's public key by replacing it in the PublicKey field. The endpoint would be the public IP address of your WireGuard server, along with the configured port. Lastly, specify the allowed network, which in this tutorial is set as 0.0.0.0/0, meaning that all your traffic will be routed through the WireGuard server.

Now bring up the wireguard tunnel as a client:

```
$ sudo wg-quick up wg0
```

-------------------------------------------------------------

**3: Tests**

- To perform some initial tests, you should:

- Ping to the wireguard gateway server:

- At the client:
```
$ ping 10.0.0.1
```

- Make sure your public IP is the same as the machine on AWS or another cloud provider:

- (https://whatismyipaddress.com/)

-------------------------------------------------------------

**4: References**

- https://linuxize.com/post/how-to-set-up-wireguard-vpn-on-debian-10/
- https://www.expressvpn.com/pt/what-is-vpn

