# OpenVPN with EasyRSA3

We are going to use a CentOS 7 minimal x64 version

From the original source

[http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso)

In this repo, the version 1708

[http://ftp.uma.es/mirror/CentOS/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso](http://ftp.uma.es/mirror/CentOS/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso)

* Start, connecting via ssh

##### **NOTE:**  At this point, all the commands are going to execute with root privileges, in case of yum we are going to force yes (-y).
----------
* Update the system
> yum -y update
* Install epel-release package
> yum -y install epel-release
* Update one more time
> yum -y update
* Install openvpn (at this time, April 2018, the openvpn version is: 2.4.5)
> yum -y install openvpn
* Install wget
> yum -y install wget
* Disabled selinux

The configuration file is: /etc/sysconfig/selinux

Change to:

> SELINUX=disabled
* In sysctl file we enable the ip4 forward

File location: /etc/sysctl.conf

> net.ipv4.ip_forward = 1
* Restart the network services
> systemctl restart network
* We need the easy-rsa3 (In this document the version is: 3.0.4)

Oficial repo

[https://github.com/OpenVPN/easy-rsa](https://github.com/OpenVPN/easy-rsa)

Release 3.0.4

[https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz)

* Make a folder where download the easy-rsa release, and we move to it
> mkdir /tmp/easy-rsa3.0.4

> cd /tmp/easy-rsa3.0.4

* Download easy-rsa
> wget  [https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz)
* Extract easy-rsa
> tar xvf EasyRSA-3.0.4.tgz
* Make a easy-rsa folder at openvpn directory
> mkdir /etc/openvpn/easy-rsa
* Copy the extradted files to easy-rsa folder
> cp -rf /tmp/easy-rsa3.0.4/EasyRSA-3.0.4/* /etc/openvpn/easy-rsa
* Copy the sample config file for the openvpn server
> cp /usr/share/doc/openvpn-2.4.5/sample/sample-config-files/server.conf /etc/openvpn/server.conf
* Uncomment the following lines

> push "redirect-gateway def1 bypass-dhcp"

> user nobody

> group nobody

> log openvpn.log

> log-append openvpn.log

* Uncomment and edit the dns (In this example i used the google ones)
> push "dhcp-option DNS 8.8.8.8"

> push "dhcp-option DNS 8.8.4.4"

* Move to the easy-rsa folder
> cd /etc/openvpn/easy-rsa
* Innitial config
> ./easyrsa init-pki
* Generate your Certificate Authority file

We need to add a pass-phrase to the PEM file, we need to use it to sign the keys of the server and the clients

> ./easyrsa build-ca

* Generate the key and the certificate for the server, with no password requirements for the auto start of the server.
> ./easyrsa gen-req server nopass

* Generate the key and the certificate for the clients, with no password requirements
> ./easyrsa gen-req client1 nopass
> ...
> ./easyrsa gen-req clientN nopass

* Sign the clients keys

We need to answer "yes" for the requirements show, you accept the situation.

Insert the CA pass-phrase

> ./easyrsa sign-req client client1
> ...
> ./easyrsa sign-req client clientN
* Sign the server keys

We need to answer "yes" for the requirements show, you accept the situation.

Insert the CA pass-phrase

> ./easyrsa sign-req server server

* Time to generate the Diffie-Hellman file

This process depend the CPU of your server, can take a few minutes

> ./easyrsa gen-dh
* Move to the openvpn folder
> cd /etc/openvpn
* Generate the ta key, ta.key
> openvpn --genkey --secret ta.key
* Copy the server keys, the CA and the Diffie-Hellman file to the openvpn folder
> cp /etc/openvpn/easy-rsa/pki/issued/server.crt /etc/openvpn/server.crt
> cp /etc/openvpn/easy-rsa/pki/private/server.key /etc/openvpn/server.key
> cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/ca.crt
> cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/dh2048.pem
* Deactivate and disable for autostart the firewalld and install iptables
> systemctl mask firewalld
* Install iptables
> yum -y install iptables-services
* Enable iptables for autostart
> systemctl enable iptables
* Stop firewalld
> systemctl stop firewalld
* Start iptables service
> systemctl start iptables
* Flush al the iptables rules
> iptables --flush
* We need to know the server network interface, check it:

In this exaple: eth0

> ip address show
* We need to add masquerade rule for the postrouting to enable the outbound traffic

We add for the configured network at openvpn server, in this repo the subnet is: 10.8.0.0/24

> iptables -t nat -A POSTROUTING -s  10.8.0.0/24  -o eth0 -j MASQUERADE
* Save the iptables rules to a file for persistent iptables
> iptables-save > /etc/sysconfig/iptables
* Enable the openvpn service
> systemctl enable openvpn@server.service
* Reboot the server, you can check the openvpn server status running:
> systemctl status openvpn@server.service
* The server side it's ready
