# OpenVPN con EasyRSA3

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
* Instalamos openvpn (a fecha de este documento Abril de 2018, la versión es 2.4.5)
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
* Necesitamos easy-rsa 3 (Este documento es con la versión 3.0.4)

Repositorio oficial

[https://github.com/OpenVPN/easy-rsa](https://github.com/OpenVPN/easy-rsa)

Release 3.0.4

[https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz)

* Creamos una carpeta donde descargaremos easy-rsa y nos movemos a esa carpeta
> mkdir /tmp/easy-rsa3.0.4

> cd /tmp/easy-rsa3.0.4
* Descargamos easy-rsa
> wget  [https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz](https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz)
* Extraemos easy-rsa
> tar xvf EasyRSA-3.0.4.tgz
* Creamos la carpeta de easy-rsa en openvpn
> mkdir /etc/openvpn/easy-rsa
* Copiamos los ficheros extraidos de easy-rsa 3.0.4
> cp -rf /tmp/easy-rsa3.0.4/EasyRSA-3.0.4/* /etc/openvpn/easy-rsa
* Copiamos el fichero de ejemplo de configuración de server para openvpn
> cp /usr/share/doc/openvpn-2.4.5/sample/sample-config-files/server.conf /etc/openvpn/server.conf
* En este fichero debemos quitar el comentario a las siguientes líneas
> push "redirect-gateway def1 bypass-dhcp"

> user nobody

> group nobody

> log openvpn.log

> log-append openvpn.log

* Descomentamos y editamos las dns (En este caso he usado las de google)
> push "dhcp-option DNS 8.8.8.8"

> push "dhcp-option DNS 8.8.4.4"

* Nos movemos a la carpeta de easy-rsa
> cd /etc/openvpn/easy-rsa
* Configuración inicial
> ./easyrsa init-pki
* Generamos el fichero de autoridad certificadora y su par de claves

Debemos agregar un pass-phrase al fichero PEM, debemos utilizarlo posteriormente para poder firmar las claves de servidor y de los clientes.
> ./easyrsa build-ca
* Generamos la clave, el certificado y la solicitud para el servidor, en este caso libre de contraseña ya que arrancaremos el servidor de manera automática y sin requisito de contraseña para su inicio.
> ./easyrsa gen-req server nopass
* Generamos la clave, el certificado y la solicitud para los clientes, también libre de contraseña
> ./easyrsa gen-req client1 nopass
> ...
> ./easyrsa gen-req clientN nopass
* Firmamos las claves para los clientes

Debemos contestar "yes" ya que inicialmente nos muestra los requisitos que se han marcado antes y si estamos de acuerdo para continuar con la firma de las claves

Se nos solicitará la pass-phrase que habíamos incluido al fichero PEM de la autoridad certificadora

> ./easyrsa sign-req client client1
> ...
> ./easyrsa sign-req client clientN

* Firmamos las claves del lado servidor

Debemos contestar "yes" ya que inicialmente nos muestra los requisitos que se han marcado antes y si estamos de acuerdo para continuar con la firma de las claves

Se nos solicitará la pass-phrase que habíamos incluido al fichero PEM de la autoridad certificadora

> ./easyrsa sign-req server server

* Generamos el fichero de intercambio (Diffie-Hellman)

Dependiendo de la potencia del servidor este proceso puede tardar algunos minutos
> ./easyrsa gen-dh
* Nos movemos a la carpeta de OpenVPN
> cd /etc/openvpn
* Generamos la clave ta.key
> openvpn --genkey --secret ta.key
* Copiamos el par de claves del servidor, el CA y el fichero de Diffie-Hellman a la raiz de openvpn
> cp /etc/openvpn/easy-rsa/pki/issued/server.crt /etc/openvpn/server.crt
> cp /etc/openvpn/easy-rsa/pki/private/server.key /etc/openvpn/server.key
> cp /etc/openvpn/easy-rsa/pki/ca.crt /etc/openvpn/ca.crt
> cp /etc/openvpn/easy-rsa/pki/dh.pem /etc/openvpn/dh2048.pem
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
> ip address show

En este caso eth0

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
