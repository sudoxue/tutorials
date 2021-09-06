# 

[YouTube Tutorial]()

## Create AWS VPC
- Create AWS VPC (Virtual Private Cloud)
  - give it a name `main`
  - define IPv4 CIDR block `10.0.0.0/16`

## Create AWS Internet Gateway
- Create AWS Internet Gateway
  - call it `igw`

- Attach Internet Gateway to AWS VPC

## Create AWS Public Subnet
- Create public subnet
  - call it `public`
  - define IPv4 CIDR block `10.0.0.0/22`, it will give you `1,024` IP addresses, witht the last IP - `10.0.3.255`

- Create `public` route table with default route to the internet gateway

- Attach `public` route table to the public subnet

## Create AWS NAT Gateway
- Allocate Elastic IP address for nat

- Create NAT gateway
  - call it `nat`

- Place it to public subnet

## Create AWS Private Subnets
- Create `private-large` subnet `10.0.16.0/20`

- Create `private-small` subnet `10.0.32.0/24`

- Create private route table with default route to nat gateway

- Update route tables for private subnets

## Create Ubuntu EC2 Instance
- Allocate static public IP address `openvpn`

- Create Ubuntu 20.04
  - tag Name: OpenVPN`
  - SG: `OpenVPN`, add `1194` custom udp

- Associate Elastic IP with EC2

## Install OpenVPN Ubuntu 20.04
- Update permissions on the key
```bash
chmod 400 devops.pem
```
- SSH to the Ubuntu server
```bash
ssh -i devops.pem ubuntu@44.197.77.129
```

- Update Ubuntu repositories
```bash
sudo apt update
```
- Check OpenVPN candidate
```bash
apt policy openvpn
```
- Compate verion with the latest release of OpenVPN on [GitHub](https://github.com/OpenVPN/openvpn)

- We would need to run commands as a root, let's temporary use `sudo -s`
```bash
sudo -s
```

- Then import the public GPG key that is used to sign the packages:
```bash
wget -O - https://swupdate.openvpn.net/repos/repo-public.gpg|apt-key add -
```

- Add OpenVPN repo
```bash
echo "deb http://build.openvpn.net/debian/openvpn/stable focal main" > /etc/apt/sources.list.d/openvpn-aptrepo.list
```

- Update repositories again with the new openvpn source list
```bash
apt update
```

- Exit root
```bash
exit
```

- Check version of candidate again
```bash
apt policy openvpn
```

- Install the latest one
```bash
sudo apt install openvpn=2.5.3-focal0
```

## Install easy-rsa Ubuntu 20.04
- Check the candidate verion
```bash
apt policy easy-rsa
```
- Check available verions on [GitHub](https://github.com/OpenVPN/easy-rsa)

- Download `easy-esa` tarball
```bash
wget https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8.tgz
```
- Untar it
```bash
tar -zxf EasyRSA-3.0.8.tgz
```

- Clean UP
```bash
ls
rm EasyRSA-3.0.8.tgz
```

- Move `easy-rsa` to OpenVPN
```bash
sudo mv EasyRSA-3.0.8/ /etc/openvpn/easy-rsa
```

- (Optionally) create soft link
```bash
sudo ln -s /etc/openvpn/easy-rsa/easyrsa /usr/local/bin/
```

- Change directory to home and test cli
```bash
cd
easyrsa --version
```

## Creating a PKI for OpenVPN with easy-rsa
- Change directory to openvpn
```bash
cd /etc/openvpn/easy-rsa
```

- Initialize a PKI CA
```bash
easyrsa init-pki
```

- List directories
```bash
ls
ls pki
```

- Create vars file
```bash
vim vars
```

- Create CA (security or convenience)
```bash
easyrsa build-ca nopass
```

- List files
```bash
ls pki
ls pki/private
```

## Generate certificate for OpenVPN server
- Generate signing request
```bash
easyrsa gen-req openvpn-server nopass
```

- Sign cert
```
easyrsa sign-req server openvpn-server
```

## Configure OpenVPN Cryptographic Material
- Generate the tls-crypt pre-shared key
```bash
openvpn --genkey secret ta.key
```

## Configure OpenVPN server
- Enable IP forwarding
```bash
sudo vim /etc/sysctl.conf
```

- Read the file and load the new values for the current session
```bash
sudo sysctl -p
```

- Disable Source/destination check for openvpn


- Create config file, leave routes out for now
  - `sudo vim /etc/openvpn/server/server.conf`

- Check if you have `nobody` user
```bash
cat /etc/passwd | grep nobody
```

- Check if you have `nogroup`
```bash
cat /etc/group | grep no
```

- Check subnet masks for CIDR [here](https://docs.netgate.com/pfsense/en/latest/network/cidr.html)

- Start OpenVPN
```bash
sudo systemctl start openvpn-server@server
```

- Check status OpenVPN
```bash
sudo systemctl status openvpn-server@server
```
- Enable openvpn-server
```bash
sudo systemctl enable openvpn-server@server
```

- Check logs
```bash
journalctl --no-pager --full -u openvpn-server@server -f
```

## Create client profile .ovpn manually (Example 1)

- Generate key pair
```bash
easyrsa gen-req example-1 nopass
```

- Sign certificate request
```bash
easyrsa sign-req client example-1
```

- Create config
```bash
cd
vim example-1.ovpn
```

- Copy profile to the host machine
```bash
scp -i devops.pem ubuntu@44.197.77.129:~/example-1.ovpn .
```

- Install tunnelblick
```bash
brew install --cask tunnelblick
```

- Open profile

```bash
traceroute 10.0.3.69
```


## Add routes back

```
push "route 10.0.0.0 255.255.252.0"
push "route 10.0.16.0 255.255.240.0"
push "route 10.0.32.0 255.255.255.0"
```








install dns mask
10.8.0.1

- Check routes on mac
netstat -r

## Create Route53 Private Hosted Zone
- Create devops.pvt private hosted zone
- Create `test.devops.pvt` A record `10.10.10.10`
- Try to relove it from `local host`
```bash
dig test.devops.pvt
```

- Try to relove it from `openvpn`
```bash
dig test.devops.pvt
```

- To use private hosted zones, you must set the following Amazon VPC settings to true:
  - enableDnsHostnames
  - enableDnsSupport

## Create EC in private subnet and ssh to it using private ip address


## Install DNSmasq


ens5






## Links
- [Easy-RSA 3 Quickstart README](https://github.com/OpenVPN/easy-rsa/blob/master/README.quickstart.md)
- [OpenVPN - Getting started How-To](https://community.openvpn.net/openvpn/wiki/GettingStartedwithOVPN?__cf_chl_jschl_tk__=pmd_X4JgyIGE5orR7DsfIDms7ZJjTaenJB7TAkouWI00egk-1630687272-0-gqNtZGzNAfujcnBszQgR)
- [SSL stripping]

Custom DNS servers
If you have configured custom DNS servers on Amazon EC2 instances in your VPC, you must configure those DNS servers to route your private DNS queries to the IP address of the Amazon-provided DNS servers for your VPC. This IP address is the IP address at the base of the VPC network range "plus two." For example, if the CIDR range for your VPC is 10.0.0.0/16, the IP address of the DNS server is 10.0.0.2.

- [Certificate Authority (CA)](https://wiki.archlinux.org/title/Easy-RSA)
Starting from OpenVPN 2.4, one can also use elliptic curves for TLS connections (e.g. tls-cipher TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384). Elliptic curve cryptography provides more security and eliminates the need for a Diffie-Hellman parameters file. See [2] and [3].

## Clean UP
- VPC `main`
- SG `OpenVPN`
- Key pair `devops`
- Delete tunnelblick
```bash
brew remove tunnelblick
```