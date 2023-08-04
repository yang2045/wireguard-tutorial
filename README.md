# Setting up a VPN with Wireguard Server on AWS EC2.

## dev.to
- [Link](https://dev.to/gabrieltetzner/setting-up-a-vpn-with-wireguard-server-on-aws-ec2-4a49)

-------------------------------------------------------------

## Requirements
- An account on a cloud platform that offers a virtual machine (e.g. AWS, Azure, Google Cloud, etc.).
- Ubuntu 20.04 Server Virtual Machine.
- A public IP address assigned to your VM.
- UDP port 51820 open to incoming traffic from all sources (0.0.0.0/0).
- The region chosen to raise the machine in this tutorial was South America (São Paulo, Brazil) (sa-east-1), but choose your preferred region.

-------------------------------------------------------------

![Image 01](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/v6c4hdbq0ghqu6j83062.png)

![Image 02](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yqsuwf4g1p89hr88xnkx.png)

- **Caption: Configuring the opening of port 51820 UDP for the instance launched on AWS.**
- **Caption: Instance configuration with public IP and port 51820 UDP open.**

-------------------------------------------------------------

## **Introduction**

In this tutorial, we will set up a WireGuard VPN server on an Ubuntu 20.04 instance running on AWS.

WireGuard is a user-friendly VPN solution that utilizes end-to-end encryption, making it more efficient than IPSEC and faster than OpenVPN.

-------------------------------------------------------------

**1. Server WireGuard Configuration**

**Step 1: Launch an Instance on AWS EC2**

First, we need to launch an Ubuntu 20.04 instance on AWS.

Note: The usage is from Ubuntu 20.04, but can be from a recent version.

-------------------------------------------------------------

**Step 2: Install WireGuard**

After your instance is up and running, we can proceed to install WireGuard using the following commands in the terminal:

```
$ sudo apt update
```

Next we will install the Wireguard package:


```
$ sudo apt install wireguard
```

-------------------------------------------------------------

![Image 03](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/59b987pw7so01ybfgasb.png)

- **Caption: Running the apt update and apt install wireguard commands on the AWS instance.**

-------------------------------------------------------------

**Step 3: Configuring Wireguard Server on AWS instance**

With the package installed, we now need to set up the WireGuard server on the AWS instance.

But, we need to create the directory for generating the wireguard files, run the command: 

```
$ sudo mkdir /etc/wireguard/
```

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
PrivateKey = private_key
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

-------------------------------------------------------------

![Image 04](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xwc424fql8ugr2elcxiz.png)

- **Caption: Configuring wg0.conf file on wireguard server.**

-------------------------------------------------------------

Note: In the text above, the phrase "-o eth0" means that it is your machine's default network interface (and that would be the one that knows the AWS outgoing gateway), i.e. if it is different, you should replace it with the appropriate name. To check which is the correct interface this is, just run the command:

```
$ ip -c a
```

You should see something like this, like:

---

![Image 05](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4h51277amj0mnh9xaiwq.png)

- **Caption: Checking the machine's default network interface.**

To continue, replace <server_private_key> with the private key you generated earlier. The AllowedIPs option specifies the IP address range that will be routed through the VPN.

-------------------------------------------------------------

**Step 4: Start the WireGuard Service**

Once the configuration is complete, we can start the WireGuard service:

```
$ sudo systemctl enable wg-quick@wg0
$ sudo systemctl start wg-quick@wg0
```

-------------------------------------------------------------

**Step 5: Enable IP Forwarding**

To allow traffic to pass through the VPN, we need to enable IP forwarding with the following commands:

```
$ echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

```
$ sudo sysctl -p
```

Let me explain each setting:

<br>

- The [Interface] section defines the server's network information, including the IP address of the wg0 interface that the WireGuard server will use, the network CIDR block (10.0.0.0/24), and the port the server will listen on for incoming connections (51820).

<br>

- The SaveConfig option is set to true, so that the settings can be saved and retained after a system restart.

<br>

- The PrivateKey option is the private key for the WireGuard server, which is used for authentication and encryption of connections. It is important that this key is kept secure and not shared with anyone.

<br>

- The PostUp and PostDown options are the commands that will be executed after creating the wg0 interface and after removing it, respectively. These commands add and remove the necessary firewall rules to allow traffic to pass through the tunnel created by the VPN and to redirect traffic to the outgoing interface of the virtual machine.

**Now let's turn on this wg0 interface, use the following command:**

```
$ sudo wg-quick up wg0
```

-------------------------------------------------------------

**2. Client WireGuard Configuration**

- At the client:

- Run the following commands:

```
$ sudo apt update
```

```
$ sudo apt install wireguard
```

- Create the wireguard directory:

```
$ sudo mkdir /etc/wireguard/
```

- Generate the client's private and public keys:

```
$ wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

- Create and edit the wg0.conf file on the client:

```
$ sudo nano /etc/wireguard/wg0.conf
```

- Set the following settings:

```
[Interface]
PrivateKey = privatekey_client
Address = 10.0.0.2/24

[Peer]
PublicKey = publickey_server
Endpoint = ipaddress_server:51820
AllowedIPs = 0.0.0.0/0
```

-------------------------------------------------------------

![Image 06](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/03hqn09ek0mpdtb3y2hh.png)

- **Caption: Setting up client wireguard file.**
- **Note: The file here is named as wg7, but could be wg0.**

-------------------------------------------------------------

<br>

- The [Interface] section defines the client's network information, including the IP address of the wg0 interface that the Wireguard client will use, i.e. 10.0.0.2/24 and the client's private key.

<br>

- In the peer configuration, you should point to the server's public key by replacing it in the PublicKey field. The endpoint would be the public IP address of your WireGuard server, along with the configured port. Lastly, specify the allowed network, which in this tutorial is set as 0.0.0.0/0, meaning that all your traffic will be routed through the WireGuard server.

<br>

- Once this is done, you must now add your client's public key to your wireguard server, to do this, use the following command:

<br>

```
$ sudo wg set wg0 peer clientpublickey allowed-ips 10.0.0.2
```

- Now bring up the wireguard tunnel as a client:

```
$ sudo wg-quick up wg0
```

-------------------------------------------------------------

![Image 07](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/8tvoq2inv8hm5vp8k1fx.png)

- **Caption: Raising wireguard tunnel on client.**

-------------------------------------------------------------

**3. Tests**

- To perform some initial tests, you should:

- Ping to the wireguard gateway server:

- At the client:

```
$ ping 10.0.0.1
```

-------------------------------------------------------------

![Image 08](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/bwn70dcgnr6mvm8d7n7s.png)

- **Caption: Running a ping test to the gateway (server IP address).**

-------------------------------------------------------------

- Make sure your public IP is the same as the machine on AWS or another cloud provider:

- (https://whatismyipaddress.com/)

![Image 09](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zpx9ssc3tb4ub0mcu4af.png)

- Your IP address should be the same as the public IPv4 of your AWS machine (As the region I picked up the machine was in São Paulo, it got the address from there).

-------------------------------------------------------------

**4. References**

- https://linuxize.com/post/how-to-set-up-wireguard-vpn-on-debian-10/
- https://www.cyberciti.biz/faq/debian-10-set-up-wireguard-vpn-server/
- https://www.wireguard.com/
- https://www.expressvpn.com/pt/what-is-vpn