# OpenVPN con EasyRSA2

Partimos de una versión de CentOS 7 minimal x64
Descargado de la web oficial
[http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso](http://isoredirect.centos.org/centos/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso)
Por ejemplo, en este caso la versión 1708
[http://ftp.uma.es/mirror/CentOS/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso](http://ftp.uma.es/mirror/CentOS/7/isos/x86_64/CentOS-7-x86_64-Minimal-1708.iso)

* Comenzamos conectando vía ssh

##### **NOTA:**  Todos los comandos a partir de este punto se ejecutan desde root, y en el caso del uso de yum se fuerza el yes (-y) a la instalación para no estar pendiente de las preguntas.
----------
* Actualizamos el sistema
> yum -y update
* Instalamos el paquete epel-release
> yum -y install epel-release
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
Ruta del fichero: /etc/sysctl.conf
> net.ipv4.ip_forward = 1
* Reiniciamos los servicios de red
> systemctl restart network
* Necesitamos easy-rsa 2 (Este documento es con la versión 2.2.2)
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
Tenemos todas las configuraciones por "default" que se usarán al crear cualquier clave y certificado, podemos cambiar todos los datos que queramos, pero en este ejemplo sólo se cambia el nombre de la clave.
> export KEY_NAME="server"
export KEY_CN="VPN-Server"
* Copiamos el fichero openssl-1.0.0.cnf como openssl.cnf a nuestra carpeta de easy-rsa
> cp /etc/openvpn/easy-rsa/openssl-1.0.0.cnf /etc/openvpn/easy-rsa/openssl.cnf
* Nos movemos a la carpeta de easy-rsa
/etc/openvpn/easy-rsa
Recargamos las modificaciones del fichero vars
> source ./vars
* Ejecutamos clean all para eliminar todas las claves y certificados generados con anterioridad
> ./clean-all
* Generamos el fichero de autoridad certificadora
Respondemos a las configuraciones que hemos ajustado previamente en el fichero vars, en este punto también se pueden usar otras respuestas diferentes a las del fichero vars, por eso, tampoco era estrictamente necesario modificarlo antes.
> ./build-ca
* Generamos la clave y el certificado para el servidor
Tenemos que contestar que sí, tanto a la firma del cerfiticado como al commit.
> ./build-key-server server
* Generamos el fichero de intercambio (Diffie-Hellman)
Dependiendo de la potencia del servidor este proceso puede tardar algunos minutos
> ./build-dh
* Generamos las claves para los clientes
A las preguntas de firmar certificado y aplicar commit contestamos que sí.
Todas las claves se han generado en /etc/openvpn/easy-rsa/keys
> ./build-key client1
...
./build-key clientN
* Nos movemos a la carpeta de las claves
> cd /etc/openvpn/easy-rsa/keys
* Copiamos la clave y el certificado del servidor, el de la autoridad certificadora y el de intercambio diffie-hellman
> cp dh2048.pem ca.crt server.crt server.key /etc/openvpn
* Nos movemos a la carpeta de openvpn
> cd /etc/openvpn
* Generamos la clave ta.key
> openvpn --genkey --secret ta.key
* Desactivamos y retiramos del arranque firewalld e instalamos iptables
> systemctl mask firewalld
* Instalamos iptables
> yum -y install iptables-services
* Habilitamos iptables para el arranque
> systemctl enable iptables
* Paramos firewalld
> systemctl stop firewalld
* Arrancamos iptables
> systemctl start iptables
* Eliminamos todas las normas que puedan tener las iptables
> iptables --flush
* Necesitamos conocer nuestra interfaz, la consultamos:
En este caso eth0
> ip address show
* Agregamos masquerade al tráfico generado desde el servidor vpn, para poder navegar por internet
Se agrega la red configurada en el servidor  10.8.0.0/24
> iptables -t nat -A POSTROUTING -s  10.8.0.0/24  -o eth0 -j MASQUERADE
* Guardamos a fichero las iptables, para convertir las normas en persistentes al reinicio
> iptables-save > /etc/sysconfig/iptables
* Habilitamos el servicio de openvpn
> systemctl enable openvpn@server.service
* Reiniciamos y al reconectar una vez encendido comprobamos que el servicio está funcionando.
> systemctl status openvpn@server.service
* El lado servidor ya estaría configurado
