# OpenVPN con EasyRSA2

Partimos de una versión de CentOS 7 minimal x64
Descargado de la web oficial
[http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso)
Por ejemplo, la versión 1708
[http://ftp.uma.es/mirror/CentOS/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso](http://ftp.uma.es/mirror/CentOS/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso)

* Comenzamos conectando vía ssh

##### **NOTA:**  Todos los comandos a partir de este punto se ejecutan desde root, y en el caso del uso de yum se fuerza el yes (-y) a la instalación para no estar pendiente de las preguntas.
----------
* Actualizamos el sistema
> yum -y update
* Instalamos el paquete epel-release
> yum -y install peel-release
* Actualizamos nuevamente
> yum -y update
* Instalamos openvpn (a fecha de este documento Marzo de 2018, la versión es 2.4.5)
> yum -y install openvpn
* Instalamos wget
> yum -y install wget
* Desactivamos selinux
El fichero de configuración es: /etc/sysconfig/selinux
Lo cambiamos a:
> SELINUX=disabled
* En el fichero sysctl habilitamos el forward para ip4
Ruta del ficher: /etc/sysconfig/sysctl.conf
> net.ipv4.ip_forward = 1
* Reiniciamos los servicios de red
> systemctl restart network
* Necesitamos easyrsa 2 (Este documento es con la versión 2.2.2
Repositorio oficial
[https://github.com/OpenVPN/easy-rsa](https://github.com/OpenVPN/easy-rsa)
Release 2.2.2
[https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz)
* Creamos una carpeta donde descargaremos easy-rsa y nos movemos a esa carpeta
> mkdir /tmp/easy-rsa2.2.2
cd /tmp/easy-rsa2.2.2
* Descargamos easy-rsa
> wget  [https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/2.2.2/EasyRSA-2.2.2.tgz)
* Extraemos easy-rsa
> tar xvf EasyRSA-2.2.2.tgz
* Creamos la carpeta de easy-rsa en openvpn
> mkdir /etc/openvpn/easy-rsa
* Copiamos los ficheros extraidos de easy-rsa 2.2.2
> cp -rf /tmp/easy-rsa2.2.2/EasyRSA-2.2.2/* /etc/openvpn/easy-rsa
* Creamos una carpeta donde almacenaremos las claves que generemos
> mkdir /etc/openvpn/easy-rsa/keys
* Copiamos el fichero de ejemplo de configuración de server para openvpn
> cp /usr/share/doc/openvpn-2.4.5/sample/sample-config-files/server.conf /etc/openvpn/server.conf
* En este fichero debemos quitar el comentario a las siguientes líneas
> push "redirect-gateway def1 bypass-dhcp"
user nobody
group nobody
log openvpn.log
log-append openvpn.log
* Descomentamos y editamos las dns (En este caso he usado las de google)
> push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
* Editamos el fichero vars de easy-rsa
/etc/openvpn/easy-rsa/vars
export KEY_NAME="server"
\# export KEY_CN="VPN-Server"
* Cambiamos el nombre a key\_name y descontamos key\_cn y cambiamos el nombre
Copiamos el fichero openssl-1.0.0.cnf como openssl.cnf
cp /etc/openvpn/easy-rsa/openssl-1.0.0.cnf /etc/openvpn/easy-rsa/openssl.cnf
* Nos movemos a la carpeta de easy-rsa
Planchamos las modificaciones del fichero vars
source ./vars
Ejecutamos clean all para eliminar todas las claves y certificados generados con anterioridad
./clean-all
Generamos el fichero de autoridad certificadora
./build-ca
respondemos a las configuraciones que hemos ajustado previamente en el fichero vars
Una vez generados todos los certificados, es recomendado guardar este fichero offline
Generamos la clave y el certificado para el servidor
./build-key-server server
Cuando nos pregunte para firmar el certificado indicamos que sí (y)
Generamos el fichero de intercambio (Diffie-Hellman)
./build-dh
Dependiendo de la potencia del servidor este proceso puede tardar algunos minutos
Generamos las claves para los clientes
./build-key client
./build-key client2
.
.
.
./build-key clients
a la pregunta para firmar la clave contestamos si
Todas las claves se han generado en /etc/openvpn/easy-rsa/keys
Nos movemos a esta carpeta
cd /etc/openvpn/easy-rsa/keys
Copiamos la clave y el certificado del servidor, el de la autoridad certificadora y el de intercambio diffie-hellman
cp dh2048.pem ca.crt server.crt server.key /etc/openvpn
Generamos la clave ta.key
cd /etc/openvpn
openvpn —genkey —secret ta.key
Retiramos firewalld e instalamos iptables
systemctl mask firewalld
yum -y install iptables-services
systemctl enable iptables
systemctl stop firewalld
systemctl start iptables
Eliminamos todas las normas que puedan tener las iptables
iptables - -flush
Necesitamos conocer nuestra interfaz, en este ejemplo eth0
ip address show
Agregamos masquerade al tráfico generado desde el servidor vpv, para poder navegar por internet
Se agrega a la red configurada en el servidor  [10.8.0.0/24](http://10.8.0.0/24)
iptables -t nat -A POSTROUTING -s  [10.8.0.0/24](http://10.8.0.0/24)  -o enp0s3 -j MASQUERADE
Guardamos a fichero las iptables, para convertir las normas en persistentes al reinicio
iptables-save > /etc/sysconfig/iptables
Habilitamos el servicio de openvpn
systemctl enable openvpn@server.service
Reiniciamos y al reincidir una vez encendido comprobamos que el servicio está funcionando y el lado servidor ya estaría configurado
