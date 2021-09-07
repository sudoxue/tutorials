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
  - tag Name: openvpn`
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

<!-- - Disable Source/destination check for openvpn -->


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

<!-- - Create config
```bash
cd
vim example-1.ovpn
``` -->

<!-- - Copy profile to the host machine
```bash
scp -i devops.pem ubuntu@44.197.77.129:~/example-1.ovpn .
``` -->

- Install tunnelblick
```bash
brew install --cask tunnelblick
```
```
cat /etc/openvpn/easy-rsa/pki/ca.crt
cat /etc/openvpn/easy-rsa/pki/issued/example-1.crt
cat /etc/openvpn/easy-rsa/pki/private/example-1.key
cat /etc/openvpn/easy-rsa/ta.key
```

```
nc -vz 10.0.3.144 22
ping 10.0.3.144

nc -vz 10.0.32.234 22
```

- Open profile

netstat -r


```bash
traceroute 10.0.3.69
```


## Add routes back

```
push "route 10.0.0.0 255.255.252.0"
push "route 10.0.16.0 255.255.240.0"
push "route 10.0.32.0 255.255.255.0"
```

## Create EC2 instance in private-small subnet

- Create instance with tag Name: test-1

- test 
```bash
nc -vz 10.0.32.234 22
```

## Update firewall
you need to have ufw

```bash
ip route list default
sudo vim /etc/ufw/before.rules
sudo vim /etc/default/ufw
```

```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.8.0.0/8 -o ens5 -j MASQUERADE
COMMIT
# END OPENVPN RULES
```

```bash
sudo ufw disable
sudo ufw enable
```

nft
```
sudo iptables -S
sudo iptables -L -v -n
sudo iptables -S FORWARD
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

## Revoking Certificates

easyrsa revoke example-1
easyrsa gen-crl
crl-verify /etc/openvpn/easy-rsa/pki/crl.pem


## Generate profiels
```bash
cd /etc/openvpn/
mkdir client-configs
cd client-configs
vim base.ovpn
vim gen_client_profile.sh
chmod +x gen_client_profile.sh
easyrsa gen-req example-2 nopass
easyrsa sign-req client example-2
./gen_client_profile.sh example-2
```


sudo systemctl daemon-reload
sudo systemctl restart systemd-networkd
sudo systemctl restart systemd-resolved

## Create gate-sso

Ensure that ruby is installed (>= 2.4) 
ruby -v
sudo apt install ruby-full
ruby -v

and bundler gem is installed.
gem list
sudo gem install bundler
cd /opt
git clone https://github.com/gate-sso/gate.git
cd gate
bundle install

sudo apt-get install ubuntu-dev-tools ?
sudo apt-get install zlib1g-dev ?
sudo apt-get install libz-dev libiconv-hook1 libiconv-hook-dev ????
gem install atomic ???


sudo apt-get install libmysqlclient-dev  #(mysql development headers) !!!
<!-- sudo gem install mysql2 -- --with-mysql-dir=/etc/mysql/ -->

sudo apt-get install nodejs

rake app:init
rake app:setup

https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04

sudo apt install mysql-server

<!-- gem 'bigdecimal', '1.3.5'
sudo gem install bigdecimal -->


## Install MySQL on Ubuntu 20.10
sudo apt install mysql-server
sudo systemctl status mysql
sudo mysql_secure_installation
sudo mysql
CREATE USER 'gate'@'localhost' IDENTIFIED BY 'devops123User%';
GRANT ALL PRIVILEGES ON gate_development.* TO 'gate'@'localhost';
GRANT ALL PRIVILEGES ON gate_test.* TO 'gate'@'localhost';

FLUSH PRIVILEGES;
exit

Note that this statement also includes WITH GRANT OPTION. This will allow your MySQL user to grant any that it has to other users on the system.

curl -L https://get.rvm.io | bash -s stable
vim /etc/group
rvm:x:1001:linuxfork
rvm list known

rvm install 2.6.6

## Links
- [Easy-RSA 3 Quickstart README](https://github.com/OpenVPN/easy-rsa/blob/master/README.quickstart.md)
- [OpenVPN - Getting started How-To](https://community.openvpn.net/openvpn/wiki/GettingStartedwithOVPN?__cf_chl_jschl_tk__=pmd_X4JgyIGE5orR7DsfIDms7ZJjTaenJB7TAkouWI00egk-1630687272-0-gqNtZGzNAfujcnBszQgR)
- [SSL stripping]

Custom DNS servers
If you have configured custom DNS servers on Amazon EC2 instances in your VPC, you must configure those DNS servers to route your private DNS queries to the IP address of the Amazon-provided DNS servers for your VPC. This IP address is the IP address at the base of the VPC network range "plus two." For example, if the CIDR range for your VPC is 10.0.0.0/16, the IP address of the DNS server is 10.0.0.2.

- [Certificate Authority (CA)](https://wiki.archlinux.org/title/Easy-RSA)
Starting from OpenVPN 2.4, one can also use elliptic curves for TLS connections (e.g. tls-cipher TLS-ECDHE-ECDSA-WITH-AES-256-GCM-SHA384). Elliptic curve cryptography provides more security and eliminates the need for a Diffie-Hellman parameters file. See [2] and [3].

- [TAP vs TUN](https://community.openvpn.net/openvpn/wiki/BridgingAndRouting?__cf_chl_jschl_tk__=pmd_g7hLiK8fPD6lRitFu1jxyHO2yMP9_UrR1KrxCBKE81s-1630949427-0-gqNtZGzNAfujcnBszQnl)

- [​TCP/IP Tutorial and Technical Overview](http://www.redbooks.ibm.com/redbooks/pdfs/gg243376.pdf)

- [AWS DNS](https://d1.awsstatic.com/whitepapers/hybrid-cloud-dns-options-for-vpc.pdf)

## Clean UP
- VPC `main`
- SG `OpenVPN`
- Key pair `devops`
- Delete tunnelblick
```bash
brew remove tunnelblick
```





ip link show
- [create device](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/networking/tuntap.rst?id=HEAD)
- [http://vtun.sourceforge.net/](http://vtun.sourceforge.net/)
- [7.6. ( NAT vs. Proxy ) - How does IP Masquerade differ from Proxy or NAT services?](https://tldp.org/HOWTO/IP-Masquerade-HOWTO/what-is-masq.html)

```
ip route list default
sudo iptables -vnL
sudo iptables -vnL FORWARD
iptables --policy FORWARD ACCEPT (default policy)
sudo iptables -t nat -S

sudo iptables -D FORWARD -i tun0 -o ens5 -m conntrack --ctstate NEW -j ACCEPT


# Allow traffic initiated from VPN to access LAN
iptables -I FORWARD -i tun0 -o ens5 -s 10.8.0.0/24 -d 10.0.0.0/16 -m conntrack --ctstate NEW -j ACCEPT

# Allow traffic initiated from VPN to access "the world"
iptables -I FORWARD -i tun0 -o eth1 -s 10.8.0.0/24 -m conntrack --ctstate NEW -j ACCEPT

# Allow traffic initiated from LAN to access "the world"
iptables -I FORWARD -i eth0 -o eth1 -s 192.168.0.0/24 -m conntrack --ctstate NEW -j ACCEPT

# Allow established traffic to pass back and forth
iptables -I FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# Notice that -I is used, so when listing it (iptables -vxnL) it will be reversed.  This is intentional in this demonstration.

# Masquerade traffic from VPN to "the world" -- done in the nat table
iptables -t nat -I POSTROUTING -o eth1 -s 10.8.0.0/24 -j MASQUERADE

# Masquerade traffic from LAN to "the world"
iptables -t nat -I POSTROUTING -o eth1 -s 192.168.0.0/24 -j MASQUERADE
```



# Allow traffic initiated from VPN to access LAN
iptables -I FORWARD -i tun0 -o ens5 -s 10.8.0.0/24 -d 10.0.0.0/16 -m conntrack --ctstate NEW -j ACCEPT
iptables -I FORWARD -i tun0 -o ens5 -s 10.8.0.5/32 -d 10.0.0.0/16 -m conntrack --ctstate NEW -j ACCEPT

# Masquerade traffic from VPN to "the world" -- done in the nat table
sudo iptables -t nat -I POSTROUTING -s 10.8.0.0/24 -o ens5 -j MASQUERADE


-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE