

  

  Exercici 1
Configura les diferents interfaces del servidor per a que tinguin les IP estàtiques indicades a la imatge. Veure configuració d’interfaces a través de netplan (fitxer /etc/netplan/*.yaml).
Primero debemos saber el nombre de nuestras interfaces de red, por lo tanto, utilizamos el comando 

ip a

Accedemos al archivo con sudo nano y lo modificamos de la siguiente manera para establecer las IPs fijas
Después de modificar el archivo Ctrl + O, Enter y Ctrl + X para guardar cambios y realizamos el siguiente comando sudo netplan apply para aplicar los cambios guardados.
Verificamos que se han aplicado correctamente con ip a

Exercici 2
Configura el paràmetre IPv4.forwarding a “1” per a que el servidor redirigeixi les sol·licituds IP com un router.
Accedemos al archivo 
sudo nano /etc/sysctl.conf 
descomenta la línea o añade en caso de que no esté 

net.ipv4.ip_forward=1

Después de modificar el archivo Ctrl + O, Enter y Ctrl + X para guardar cambios y 
aplica los cambios con 

sudo sysctl -p

Part 2: Configura una xarxa
Exercici 1
Crea un segon servidor linux que funcioni com a router i configura dos clients a les xarxes Personal 1 i 3 tal i com s’indica a la imatge. Per donar IP’s als clients pots configurar-les manualment o configurar un servidor DHCP.

Haremos igual que en la práctica anterior con las interfaces de red
ip a 
Y configuramos el archivo yaml
Después de modificar el archivo Ctrl + O, Enter y Ctrl + X para guardar cambios y realizamos el siguiente comando sudo netplan apply para aplicar los cambios guardados.
Verificamos que se han aplicado correctamente con ip a

Accedemos al archivo 

sudo nano /etc/sysctl.conf 

descomenta la línea o añade en caso de que no esté 
net.ipv4.ip_forward=1

Después de modificar el archivo Ctrl + O, Enter y Ctrl + X para guardar cambios y 
aplica los cambios con 

sudo sysctl -p

Ahora creamos dos máquinas clientes y cada una tendrá su interfaz de red y le asignaremos una IP manualmente.
Una vez configurado los clientes, tendremos que configurar las rutas estáticas de los servidores tanto del 1 como del 2.

En Server 1, habilitamos el reenvio de IP 

sudo sysctl -w net.ipv4.ip_forward=1

se añadirá una ruta para que todo el tráfico dirigido a la subred 192.168.3.0/24 pase por 192.168.2.2 que es el otro servidor

sudo ip route add 192.168.3.0/24 via 192.168.2.2 dev enp3s0

Ahora en Server 2 haremos exactamente lo mismo, pero al revés una ruta para que todo el tráfico dirigido a la subred 192.168.1.0/24 pase por 192.168.2.1

sudo ip route add 192.168.1.0/24 via 192.168.2.1 dev enp2s0

Configurado ya esto, verificamos la conectividad mediante ping entre las máquinas a ver si se ven.
Al router1, configura a netplan una ruta estàtica per a que conegui com arribar a la xarxa Personal3.

ethernets:
  enp1s0:
    dhcp4: true
  enp2s0 :
    dhcp4: false
    addresses:
      - 192.168.1.1/24
  enp3s0 :
    dhcp4: false
    addresses :
      - 192.168.2.1/24
    routes:
      - to: 192.168.3.0/24
        via: 192.168.2.2
        on-link: true
version: 2

Exercici 5
Configura el NAT al router 1 amb nftables de manera que s’emmascari la xarxa Personal1 quan es surt cap a Personal2.

etc/nftables.conf

table inet nat{
    chain prerouting {
        type nat hook prerouting priority 0; policy accept;
    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        ip saddr 192.168.1.0/24 oif "enp3so" masquerade

Aplicar las reglas:
sudo nft -f /etc/nftables.conf
        
Guardamos el archivo con CTRL + O y ENTER,  CTRL + X para salir del archivo. Ahora aplicamos las reglas con el comando: sudo nft -f /etc/nftables.conf

SSH

Instalamos SSH en caso de que no lo tengamos instalado.
sudo apt update
sudo apt install openssh-server openssh-client
En Cliente 2 (192.168.3.10), configuramos el servidor SSH para que escuche en el puerto 1001.
Edita el archivo de configuración del SSH para descomentar la línea Port 22 a Port 1001

/etc/ssh/sshd_config

Y reiniciamos el servicio SSH para que se apliquen los cambios sudo systemctl restart ssh

En Cliente 1 (192.168.1.10), que será el cliente, genera las claves SSH para la autenticación.
ssh-keygen -t rsa -b 2048
Se puede poner una contraseña, en mi caso para esta práctica no la pondre, se recomendó poner contraseña para más seguridad en un entorno real
Lo siguiente por hacer es copiar la clave pública generada en Cliente 1 hacia Cliente 2 para permitir la autenticación mediante certificados.

sudo ssh-copy-id -p 1001 user@ipdestino

Ahora, en Cliente 1 (192.168.1.10), puedes conectarte a Cliente 2 (192.168.3.10) a través del puerto 1001 usando el siguiente comando:
 sudo ssh -p 1001 user@ipdestino

 Part 3: Configura una zona desmilitaritzada (DMZ)
 Afegeix la xarxa Personal4 al Server1 per a crear una xarxa DMZ i canvia de xarxa el Client1 per a que estigui en aquesta xarxa.
network :
  ethernets:
    enp1s0 :
      dhcp4 :true
    enp2s0 :
      dhcp4 : false
      addresses :
        - 192. 168.1.1/24
  enp3s0: #Personal 2
    dhcp4: false
    addresses :
      - 192. 168.2.1/24
    routes:
      - to: 192.168.3.0/24
      via: 192.168.2.2
      on-link: true
  enp4s0: #Personal 4 (DMZ)
    dhcp4: false
    addresses :
      - 192.168.4.1/24
version: 2


Si fa falta, configura la NAT del Server1 per a que emmascari també la xarxa Personal4 quan es surt cap a Personal2. Treu també de la taula de rutes del Server1 la xarxa Personal3 (fitxer Netplan). Ara simula “Personal3” una xarxa privada, i les xarxes privades no estan a les taules de rutes d’Internet.

Hace falta añadir la regla NAT de la Personal 4 hacia la Personal 2

table ip nat {
    chain prerouting {
       type nat hook prerouting priority -100; policy accept;
    chain postrouting {
       type nat hook postrouting priority 100; policy accept;
       oifname "enp3s0" ip saddr 192.168.4.0/24 masquerade


Aplicar las reglas:
sudo nft -f /etc/nftables.conf

Eliminar la red Personal3 de la tabla de rutas en Server 1
network :
  ethernets:
    enp1s0 :
      dhcp4 :true
    enp2s0 :
      dhcp4 : false
      addresses :
        - 192. 168.1.1/24
  enp3s0: #Personal 2
    dhcp4: false
    addresses :
      - 192. 168.2.1/24
  enp4s0: #Personal 4 (DMZ)
    dhcp4: false
    addresses :
      - 192.168.4.1/24
version: 2

Para verificar que la NAT está funcionando y que Personal3 no está en la tabla de rutas, verificamo las reglas nftables y verificamos la rutas actuales

ip route


Exercici 10
Configura una NAT pel Server2 per a que emmascari la xarxa Personal3 quan es surt cap a Personal2.
Configurar NAT para Personal3 en Server 2
Anadimos una configuracion el ser Server 2
table ip nat {
    chain prerouting {
       type nat hook prerouting priority -100; policy accept;
    chain postrouting {
       type nat hook postrouting priority 100; policy accept;
       oifname "enp2s0" ip saddr 192.168.3.0/24 masquerade

Aplicar las reglas:
sudo nft -f /etc/nftables.conf

Exercici 11
Configura el reenviament de ports del Server1 per a que les connexions entrants per Personal 2 pel port 22 siguin redirigides al Client1 pel port 1001.
En el archivo de configuracion de nftables debemos anadir la sigueinte configuracion para el reenvio de puertos

table ip filter {
    chain input {
       type filter hook in ut priority 0; policy accept;
       tcp dport 22 accept
    chain forward {
       type filter hook forward priority 0; policy accept;
    chain prerouting {
       type nat hook prerouting priority -100; policy accept;
       tcp dport 22 dnat to 192.168.4.10:1001


