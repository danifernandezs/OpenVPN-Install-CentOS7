
# Fichero de Cliente/Client File - EJEMPLO/EXAMPLE
````
client
dev tun
proto udp
remote **IP_OR_DOMAIN** 1194
resolv-retry infinite
nobind
auth-nocache
remote-cert-tls server
tls-auth ta.key 1
persist-key
persist-tun
cipher AES-256-CBC
verb 3
ca ca.crt
cert client1.crt
key client1.key
````
