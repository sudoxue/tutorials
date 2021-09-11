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
  - tag `Name: nat`

- Create NAT gateway
  - call it `nat`

- Place it to public subnet

## Create AWS Private Subnets
- Create `private-large` subnet `10.0.16.0/20`

- Create `private-small` subnet `10.0.32.0/24`

- Create `private` route table with default route to nat gateway

- Update route tables for private subnets

## Create Ubuntu EC2 Instance
- Allocate static public IP address `openvpn`

- Create Ubuntu 20.04
  - tag `Name: openvpn`
  - Instance type: `t3.small`
  - SG: `OpenVPN`, add `1194` custom udp from `Anywhere`

- Associate Elastic IP with EC2

## Install OpenVPN on Ubuntu 20.04
- Update permissions on the key
```bash
chmod 400 devops.pem
```
- SSH to the Ubuntu server
```bash
ssh -i devops.pem ubuntu@<ip>
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

## Install easy-rsa on Ubuntu 20.04
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
easyrsa --version
```

## Creating PKI for OpenVPN with easy-rsa
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

## Generate certificate request for OpenVPN server
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
cat ta.key
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

- Configure IP Tables
```bash
sudo iptables -t nat -S
```

- Find out network public network interface
```bash
ip route list default
```

- Configure nat routing
```bash
sudo iptables -t nat -I POSTROUTING -s 10.8.0.0/24 -o ens5 -j MASQUERADE
```

- Save iptables
```bash
sudo apt-get install iptables-persistent
```

- Create config file, leave routes out for now
```bash
sudo vim /etc/openvpn/server/server.conf
```

- Check if you have `nobody` user
```bash
cat /etc/passwd | grep nobody
```

- Check if you have `nogroup`
```bash
cat /etc/group | grep nogroup
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

## Install docker on Ubuntu 20.04
- Set up the repository
```bash
sudo apt install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
- Add Docker’s official GPG key
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
- Set up the stable repository
```bash
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
- Install Docker Engine
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

- Install docker compose
```bash
sudo apt install docker-compose
```
vim docker-compose.yaml
sudo docker-compose up -d
sudo docker ps

## Install MySQL 5.7 on Ubuntu 20.10
- Get repository from [here](https://dev.mysql.com/downloads/repo/apt/)
```bash
cd
wget https://dev.mysql.com/get/mysql-apt-config_0.8.19-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.19-1_all.deb
```

mysql -u root -p -h 127.0.0.1 -P 3306

- Install MySQL server
```bash
sudo apt install mysql-client
```

- Check the status
```bash
sudo systemctl status mysql
```

- Configure access
```bash
sudo mysql_secure_installation
```

- Log in to MySQL as root
```bash
sudo mysql
```

- Create user for gate-sso
```sql
CREATE USER 'gate' IDENTIFIED BY 'devops123';
```

- Grant access to `gate_development` and `gate_test` databases
```sql
GRANT ALL PRIVILEGES ON gate_development.* TO 'gate';
GRANT ALL PRIVILEGES ON gate_test.* TO 'gate';
FLUSH PRIVILEGES;
```

- Log out
```bash
exit
```

## Install ruby on Ubuntu 20.04

- Check ruby version (must be >= 2.4)
```bash
ruby -v
```

- Install rvm
```bash
curl -L https://get.rvm.io | bash -s stable
```

- Import GPG keys
```bash
curl -sSL https://rvm.io/mpapis.asc | gpg --import -
curl -sSL https://rvm.io/pkuczynski.asc | gpg --import -
```

- Run script again
```bash
curl -L https://get.rvm.io | bash -s stable
```

- To start using RVM, load the script environment variables using the source command:
```bash
source ~/.rvm/scripts/rvm
```

- Install ruby `2.4.3` with rvm
```bash
rvm install 2.4.3
```

- Install `bundler` gem
```bash
gem install bundler
```

## Install GATE-SSO
- Clone gate-sso
```bas
cd /opt
sudo git clone https://github.com/gate-sso/gate.git
sudo chown -R ubuntu:ubuntu gate
```

- Install dependencies
```bash
cd gate
bundle install
```

- Install deps
```bash
sudo apt-get install libmysqlclient-dev
```
- Run bundle install again
```bash
bundle install
```



- Run init
```bash
rake app:init
```

- Install nodejs
```bash
sudo apt install nodejs
```

- Run again
```bash
rake app:init
```

- Update env
```bash
vim .env
```

- Run setup
```bash
rake app:setup
```

- Start rails
```bash
rvmsudo rails server --port 80 --binding 0.0.0.0 --daemon
```

ID:
771040318735-n574l3luikrpvcp5qc5gcb1dkr95ujr6.apps.googleusercontent.com
Secret:
fuX-lyhnoIe2ZpNJi-8ZX8ec
lsof -wni tcp:8080

rvmsudo rails server -p 80

redirect_uri: http://gate.devopsbyexample.io/users/auth/google_oauth2/callback

sudo mkdir /opt/vpnkeys/


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
