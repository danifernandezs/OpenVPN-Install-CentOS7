# OpenVPN with EasyRSA2

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
* Install openvpn (at this time, March 2018, the openvpn version is: 2.4.5)
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
* We need the easy-rsa2 (In this document the version is: 2.2.2)

Oficial repo

[https://github.com/OpenVPN/easy-rsa](https://github.com/OpenVPN/easy-rsa)

Release 2.2.2

[https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz)

* Make a folder where download the easy-rsa release, and we move to it
> mkdir /tmp/easy-rsa2.2.2

> cd /tmp/easy-rsa2.2.2

* Download easy-rsa
> wget  [https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz)
* Extract easy-rsa
> tar xvf EasyRSA-2.2.2.tgz
* Make a easy-rsa folder at openvpn directory
> mkdir /etc/openvpn/easy-rsa
* Copy the extradted files to easy-rsa folder
> cp -rf /tmp/easy-rsa2.2.2/EasyRSA-2.2.2/* /etc/openvpn/easy-rsa
* Make a folder to save generated keys
> mkdir /etc/openvpn/easy-rsa/keys
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

* Edit de vars file

/etc/openvpn/easy-rsa/vars

This file have all the default configs, you can change all you want, but for this repo we only change the key name.

> export KEY_NAME="server"

> export KEY_CN="VPN-Server"

* Copy the openssl-1.0.0.cnf file to openssl.cnf at easy-rsa folder.
> cp /etc/openvpn/easy-rsa/openssl-1.0.0.cnf /etc/openvpn/easy-rsa/openssl.cnf
* Move to the easy-rsa folder

/etc/openvpn/easy-rsa

Reload the source vars from the file

> source ./vars
* Execute clean all to delete all the old keys an certificates
> ./clean-all
* Generate your Certificate Authority file

Answer for the configurated questions of the vars file. you can change the answers at this point.

> ./build-ca
* Generate the key and the certificate for the server

For the sign and the commit, we need to answer "yes"

> ./build-key-server server
* Generate the Diffie-Hellman file

This process depend the CPU of your server, can take a few minutes

> ./build-dh
* Generate the keys for the clients

For the sign and the commit, we need to answer "yes"

All the generated keys are saved at: /etc/openvpn/easy-rsa/keys

> ./build-key client1

> ...

> ./build-key clientN

* Move to the keys folder
> cd /etc/openvpn/easy-rsa/keys
* Copy for the openvpn folder the server key, server certificate, Certificate Authority file and the Diffie-Hellman
> cp dh2048.pem ca.crt server.crt server.key /etc/openvpn
* Move to the openvpn folder
> cd /etc/openvpn
* Generate the ta key, ta.key
> openvpn --genkey --secret ta.key
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
