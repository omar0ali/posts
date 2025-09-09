In this document, I will go through the steps to implement a Site-to-Site Virtual Private Network
(VPN) using an on-premises network simulated as a separate Virtual Private Cloud (VPC).
And the other will be the VPC we connect to via the VPN.

## Requirements
### Topology

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/SiteToSiteVPN.png)
##### Resources
1. vpc-on-premise (10.0.0.0/16)
	1. public-subnet (10.0.0.0/24)
		1. vpc-on-premise-rtb
			1. To Internet Gateway (0.0.0.0/0)
			2. To Instance of the other side instance 10.1.0.0/16
		2. ec2-instance
			1. OS: Ubuntu
			2. Public IP
			3. VOCKEY
			4. SG-SSH - Everywhere.
	2. Internet Gateway
2. aws-vpc (10.1.0.0/16)
	1. private-subnet (10.1.0.0/24)
		1. vpc-aws-rtb
			1. To Virtual Gateway
		2. ec2-instance
			1. No public ip
			2. No key
			3. OS: Amazon Linux (default)
			4. SG-all icmp ipv4 from 10.0.0.0/24 (on-premise-public-subnet)
3. Customer Gateway (will use the public ip address of the on-premise-instance)
4. Internet Gateway (will be used in the on-premise-public-subnet)
5. Virtual Gateway (will be used on the aws-vpc private-subnet routing table)
6. Site to Site VPN Connection
	1. *Attach CustomerGW
	2. Attach VirtualGW
	3. Set static ip prefix subnet (10.0.0.0/24)
	4. Set on-premise local IPv4 CIDR (10.0.0.0/16)
	5. Set remote local IPv4 CIDR (10.1.0.0/16)
	6. In the static routes add the vpc-on-premise CIDR Block (10.0.0.0/16) or 
    the public-subnet of the on-premise (10.0.0.0/24).

## Lets Create VPCs and subnets
#### VPC-on-premise

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025192658.png)

Setting the IPv4 CIDR at 10.0.0.0/16
### Another VPC to be connected to from the VPC-on-premise

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025193011.png)

Setting the IPv4 CIDR at 10.1.0.0/16

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025193246.png)

#### Lets create the subnets

##### On-Premise Public Subnet

First subnet will be created for the VPC-on-premise. Its going to be named 
`on-premise-public-subnet`. It needs a routing table and also needs an Internet Gateway (IGW).
The on premise subnet will use the VPC-on-premise at CIDR block `10.0.0.0/16` and this subnet,
will use `10.0.0.0/24` of the vpc-on-premise CIDR block, this will give the subnet 256 hosts.
I will be using only one host, for the EC2 instance to connect to the VPN.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025195735.png)

##### AWS VPC Private Subnet

Secondly, creating a subnet for the other VPC, `vpc`. For this one, I will name it
`aws-private-subnet`. It needs a routing table as well. This subnet will use the VPC ip block 
`10.1.0.0/16` and this subnet, will use `10.1.0.0/24` from the given VPC block,
this will give the subnet 256 hosts.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025200916.png)

#### NOTE
Total of required subnets for this Site To Site VPN. Only two subnets, one will be used for the 
on-premise machine, and the other one will be used on other side, will be used to connect to the 
instances within the subnet using the tunnels by the Site To Site VPN.

>![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025201114.png)

### Lets edit or create routing table.
I will be using the routing tables that were created while creating the subnets.
You can edit them and renamed them if you want or create new ones, thats totally up to you.
I will go through editing their routings later.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025201854.png)

### Lets create an EC2 instance for vpc-on-premise
Its required since we need its public ip address for the *Customer Gateway* later. 
As well, I will be going through editing the routing table of the `on-premise-public-subnet`.


![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025204203.png)

I will name this instance `on-premise-instance` and use Ubuntu, as it includes pre-built packages.
I will also use **strongSwan** to establish the connection to the Site-to-Site VPN.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025204647.png)

I am going to be using the **vockey** provided for me. You can create a pair,
don't forget to download the private key, we will be needing that to login to the instance
through SSH. 

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025205013.png)

These are the network settings. Make sure to assign a public ip address for this instance.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025205146.png)

Make sure to add SSH security group, we will want to connect to it later.

#### NOTE
**Security Groups Rules**: You can assign your public IP address to limit access to that instance 
for secure login. To do this, make sure to select _My IP_ from _Source type_.
This ensures that only your home public IP address is allowed to connect to the instance.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025205600.png)

#### NOTE
Copy the public ip address of the created instance, we will be needing that later for the
*Customer Gateway* as well as to login to that instance. 

Also, we will need to stop Source and Destination check from the instance.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025212645.png)

#### NOTE
Stopping this will allow the instance to route traffic. It can accept data from one network and 
send it to another, acting like a bridge between different networks. 
Because that Instance will act as a VPN endpoint.


Adding more instances to that VPC, will just have to route its traffic to that VPN Endpoint.


*This instance do not originate or terminate traffic, 
its forwarding it between different networks.*

[https://www.linkedin.com/pulse/why-do-we-disable-sourcedestination-checks-nat-instance-usman-ahmad/](https://www.linkedin.com/pulse/why-do-we-disable-sourcedestination-checks-nat-instance-usman-ahmad/)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025212732.png)

### Lets create the required gateways

#### Internet Gateway

You will need to navigate to the **Internet Gateway** section and create a new Internet Gateway.
I will name this `on-premise-igw`. Remember, we are only creating one Internet Gateway to be used 
for the `vpc-on-premises` VPC.

*We will later add this to the routing table of the on-premise-vpc*

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025202626.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025202959.png)

**Don't forget to attach it to the `vpc-on-premise`**

#### IMPORTANT
**on-premise-public-subnet - Internet Access**: Before we move on, lets take a look at the
routing table of *on-premise-public-subnet*, we will need to add this **Internet Gateway** to it.
This will ensure that all instances within the `on-premise-public-subnet`, 
will have internet access.
  
![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025210453.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025210620.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025210718.png)

#### Customer Gateway
Under *Virtual Private Network* VPN section, navigate to *Customer Gateway*. Create a new one.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025211034.png)

I used the public IP address of the `on-premise-instance` I created earlier. 
This instance will serve as the on-premises site.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025211732.png)

#### Virtual Private Gateway

The virtual gateway is required to establish a Site-to-Site VPN connection. 
This virtual gateway will be used by the Site-to-Site VPN and the `vpc`.

#### NOTE
*Look at the topology.*

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026124325.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025212141.png)

This *Virtual Private Gateway* needs to be attached to the *aws-vpc* `VPC`.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025212326.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025212359.png)


![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025213009.png)

Now the VPC attached to the *Virtual Private Gateway*.

#### IMPORTANT
Attach the `VPC` not the `vpc-on-premise`


### Site To Site VPN Connection

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025213228.png)

Now we will create a site to site vpn connection.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025213606.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025214413.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025214539.png)

As you can see the tunnels are down. Lets download the configuration file for now. 
I will be using *strongSwan*.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025215224.png)

### Launch an EC2 instance in `VPC` the non-premise VPC
This instance won't have a public ip address, and no key is required. Just in the security group,
we will need to allow `Allow All ICMP IPv4` from `10.0.0.0/24`
which is the `on-premise-public-subnet`.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025220137.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025220224.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025220304.png)


Now will need to ensure this instance accepts traffic from a specific IP CIDR block. *10.0.0.0/24*


![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025220341.png)


##### Lets go back to route tables now.
Before starting to configure the strongswan, lets make sure the route tables are set correctly,
lets start with the AWS-VPC private-subnet. Since its closer to the **Virtual Private Gateway**. 
It needs to route traffic to the other on-premise-public-subnet. using the virtual private gateway.

##### Vpc-aws routing table

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025221050.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025221208.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025221129.png)

##### On-premise-public-subnet route table

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025221438.png)

We need to add the the instance of the on-premise to route to the `aws-private-subnet`
CIDR Block 10.1.0.0/24. This connection is not peering, but since Site to Site tunnels are 
connected, it should have a path to the other VPC's subnet.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241025221632.png)

### Lets configure the on-premise-instance

First, we need to login to our `on-premise-instance` to do that, we need to download our key from 
aws if you haven't. I did, and the first thing is to ensure the key can be only read. 
So using the linux command prompt type:

```
chmod 400 path/labsuser
```

Then we will need to connect to the instance, for example:

```
ssh -i labsuser.pem ubuntu@123.222.24.2
```

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026104431.png)

#### Configuration
Lets install `strongswan`

```
ubuntu@ip-10-0-0-87:~$ sudo apt install strongswan
```

##### Open `/etc/sysctl.conf`

#### NOTE
**Editor**: I am using Vim to edit these files; you can use any editor, such as `nano`,
which is also available. Please refer to the documentation for either
[Vim](https://vimdoc.sourceforge.net/htmldoc/usr_toc.html) or 
[Nano](https://www.nano-editor.org/docs.php).
 
#### IMPORTANT
**Check Your Configuration File**: Check your configuration file downloaded from the Site To Site
VPN Connection Page. You can follow the steps provided in the file. I am going through the steps 
used for strongswan.

```
sudo vim /etc/sysctl.conf
```

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026105657.png)

Uncomment `net.ipv4.ip_forward=1`

Now we need to apply the changes:

```
sudo sysctl -p
```

Next, open `etc/ipsec.conf`

```
sudo vim /etc/ipsec.conf
```

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026110112.png)

Uncomment `uniqueids = no`

And copy the following from your config file. And paste it at the end of `/etc/ipsec.conf` file.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026110256.png)

We will need to modify the last line. Uncomment the last line that starts with `leftupdown=...`
and update the `<VPC CIDR>` This CIDR range is the AWS `VPC` not the `on-premise-vpc`. 
Which is `10.1.0.0/16`

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026110555.png)

This is **tunnel 1**, we will do the same thing for **tunnel 2**.
First, find it from your configuration file.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026110735.png)

Then copy it and uncommment the last line that starts with `leftupdown=...` and update the 
`<VPC CIDR>` This CIDR range is the `AWS-VPC` not the `on-premise-vpc`. Which is `10.1.0.0/16`.

Next, open `/etc/ipsec.secrets`

```
sudo vim /etc/ipsec.secrets
```

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026112226.png)

This is the shared secret for the tunnel 1, there is also another shared secret for tunnel 2.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026112338.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026112436.png)

Copy them and add them to the file.

Next, we will create a new file. At `/etc/ipsec.d/aws-updown` and copy the following code from
your configuration file.

```
sudo vim /etc/ipsec.d/aws-updown.sh
```

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026112657.png)

There is also a minor tweak required on this code. There is a function called `add_route()` 

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026113246.png)

That is my local private ip address on the `on-premise-instance`. After saving the files.

Run `sudo chmod 744 /etc/ipsec.d/aws-updown.sh`

We will need to restart the `ipsec` with the following command.

```
sudo ipsec restart
```

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026113614.png)

Lets check the status of the `ipsec`

```
sudo ipsec status
```

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026113742.png)

All the tunnels are established and installed.

Lets check `ifconfig` but first we need to install `net-tools`

```
sudo apt install net-tools
```
Then execute `ifconfig`

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026114035.png)


You can see, all the tunnels are up.

Lets check our **Site To Site VPN** and confirm that the tunnels are working.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026114139.png)

#### The last part (set static route & ping instance)

Before we ping the private aws-instance using its private ip address, we will need to add the 
`on-premise-instance` private IP address or its VPC CIDR block, in the static routes of the Site
To Site VPN we created.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026115057.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026115424.png)

#### NOTE
You can set the whole range of the on-premise-vpc, or just a specific IP address 
i.e the on-premise-instance.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026115538.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026115424.png)

You can add one of these two.

#### CAUTION
Don't forget to click on it *from the drop-down list* and save changes.

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026120309.png)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026115358.png)

#### NOTE
I have tested it in both ways, from a specific range i.e 10.0.0.0/16 (vpc-on-premise),
>and from a specific private ip address of an instance 10.0.0.87/32 (on-premise-instance).

Ping from 10.0.0.87 (on-premise-instance) to 10.1.0.179 (aws-instance)

![](https://raw.githubusercontent.com/omar0ali/posts/refs/heads/main/Setup-Guides/AWS-SiteToSiteVPN-Guide/screenshots/Pasted%20image%2020241026120010.png)

With a few additional modifications to the AWS VPC, we can add more public and private subnets and
possibly another bastion host or multiple instances. We can also SSH into any AWS instance using 
only its private IP address, but we must enable SSH access in the security groups.

Additionally, it’s important to note that, since we included the entire on-premises CIDR range in
the Site-to-Site VPN configuration *static IPs*, any instance within that on-premises CIDR block 
can securely access resources within the AWS VPC using their private ip addresses, as long as
appropriate routing and security group rules are configured.
