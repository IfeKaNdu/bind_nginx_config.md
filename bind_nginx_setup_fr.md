# Installation et configuration des packages bind et nginx sur CentOS 7 pour réseau local

Domain Name System est responsable de la traduction du domaine Internet et des noms d’hôte en adresses IP, et inversement. Les clients DNS enverront les demandes (nom d’hôte / adresses IP) au serveur DNS et attendront une réponse.

### Installation du package du serveur DNS BIND
Installez DNS avec toutes ses dépendances.

```bash
# yum install -y bind*
```
### Attribuer une adresse IP statique au serveur DNS

###### REMARQUE: L'adresse IP doit être statique et non dynamique pour que l'adresse IP reste inchangée à chaque démarrage ou redémarrage de votre ordinateur.

Configurez le `/etc/sysconfig/network-script/ifcfg-* ` fichier en fonction de l'interface réseau que vous souhaitez utiliser. J'ai utilisé une interface sans fil connectée à un modem sans fil HUAWEI DSL.

```bash 
$ sudo vim /etc/sysconfig/network-scripts/ifcfg-HUAWEI-E5336-E71A

HWADDR=**:**:**:**:**:**
ESSID=HUAWEI-E5336-E71A
MODE=Managed
KEY_MGMT=WPA-PSK
MAC_ADDRESS_RANDOMIZATION=default
TYPE=Wireless
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=HUAWEI-E5336-E71A
UUID=c134a4f1-b498-47db-b071-4e7f7679aebd
ONBOOT=yes
IPADDR=192.168.8.2
PREFIX=24
GATEWAY=192.168.8.1
DNS1=192.168.8.2
PEERDNS=no
PEERROUTES=no

```
Trouvez des descriptions des variables de réseau ci-dessus dans votre `/usr/share/doc/initscripts*/sysconfig.txt` fichier.

### Attribuer un FQDN (Fully Qualified Domain Name) au serveur

Modifier le`/etc/sysconfig/network` fichier pour ajouter le NOM D'HÔTE de votre choix. Le mien est `odii.cic.com`
```bash
# vim /etc/sysconfig/network 
NETWORKING=yes
HOSTNAME=odii.cic.com
```
###  Configurez le `/etc/hosts` fichier en ajoutant une entrée d'hôte
```bash
# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.2    odii.cic.com
```
Remarquez ci-dessus que `192.168.8.2` et` odii.cic.com` ont été ajoutés et qu'ils changeront en fonction de l'adresse IP statique et du nom d'hôte de votre choix.
`127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6 `
Ce qui précède a été généré par défaut lors de l’installation du système d’exploitation.

### Configurer `/etc/resolv.conf`
```bash
# vim /etc/resolv.conf

search cic.com
nameserver 192.168.8.2
```

Remarque: l'étape ci-dessus sera configurée non seulement sur l'hôte du serveur DNS, mais également sur tous les clients DNS du réseau local qui utiliseront le serveur DNS pour la résolution de noms ou d'IP.
### Configurez le fichier `/etc/named.conf` 
```bash
# vim /etc/named.conf
...
options {
	listen-on port 53 { 192.168.8.2; }; # Remplacez par votre IP
#	listen-on-v6 port 53 { ::1; };      # Commentez cette ligne comme cela a été fait ici
	directory 	"/var/named";
	dump-file 	"/var/named/data/cache_dump.db";
	statistics-file "/var/named/data/named_stats.txt";
	memstatistics-file "/var/named/data/named_mem_stats.txt";
	recursing-file  "/var/named/data/named.recursing";
	secroots-file   "/var/named/data/named.secroots";
	allow-query     { any; };             # Changer ceci de `none` en `any` comme ce fut le cas ici
...
```
Modifiez les informations ci-dessus en fonction de votre adresse IP, commentez la ligne IPv6 et modifiez l'autorisation de requête de **none** en **any** .
### Configurez le fichier `/etc/named.rfc1912.zones` de manière à pointer vers les zones
```bash
# vim /etc/named.rfc1912.zones
...
zone "cic.com" IN {            # change to your domain name 
	type master;
	file "forward.zone";	   # change the file `forward.zone`
	allow-update { none; };
};

zone "localhost" IN {
	type master;
	file "named.localhost";
	allow-update { none; };
};

zone "1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa" IN {
	type master;
	file "named.loopback";
	allow-update { none; };
};

zone "8.168.192.in-addr.arpa" IN {       # changer à l'inverse de l'ID de réseau de votre adresse IP
	type master;
	file "reverse.zone";             # appelé le fichier `reverse.zone` le vôtre peut être n'importe quoi
	allow-update { none; };
};

zone "0.in-addr.arpa" IN {
	type master;
	file "named.empty";
	allow-update { none; };
```
Le fichier ci-dessus vous permet de définir vos zones, c’est-à-dire vos zones de recherche directe et inverse.
La zone aval est `cic.com` et la zone inversée est 8.168.192, qui est l'inverse de l'ID réseau de l'adresse IP attribuée,` 192.168.8 `.

### Créer et configurer le fichier pour les zones forward et reverse

Il y a tout d'abord un fichier de modèle appelé `named.localhost` fourni avec le paquet bind lors de l'installation, utilisez-le comme modèle pour configurer les fichiers forward.zone et reverse.zone.
``` bash 
#  ls /var/named/
chroot	    data     dyndb-ldap    named.ca	named.localhost  reverse.zone
chroot_sdb  dynamic  forward.zone  named.empty	named.loopback	 slaves
```
Maintenant, copiez le fichier `named.localhost` dans les fichiers `forward.zone` et` reverse.zone` avec l'option archive (-a) afin de conserver les tampons date / heure et les autorisations.
```bash 
# cp -a /var/named/named.localhost  /var/named/forward.zone
# cp -a /var/named/named.localhost  /var/named/reverse.zone
```
Maintenant, éditez le `/var/named/forward.zone` pour ressembler à ceci

```bash
# vim /var/named/forward.zone
$TTL 1D
@	IN SOA	 odii.cic.com.  root.odii.cic.com. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
	IN NS	odii.cic.com.
odii	IN A	192.168.8.2
```
configurez le `/var/named/reverse.zone` pour ressembler à ceci
```bash 
$TTL 1D
@	IN SOA	odii.cic.com.  root.odii.cic.com. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum
	IN NS	odii.cic.com.
2	IN PTR	odii.cic.com.		# le 2 signifie l'identifiant de l'hôte de l'adresse IP attribuée
```

### Changer la propriété de groupe des fichiers `forward.zone` et` reverse.zone` de la root à named
```bash
# chgrp named /var/named/forward.zone
# chgrp named /var/named/reverse.zone  
```
### Redémarrez le DNS named.service
```bash
# systemctl restart named.service
```
Pour vérifier si votre connexion fonctionne, exécutez les commandes `dig` et` host` qui vont rechercher le service named.
```bash
$ dig odii.cic.com

; <<>> DiG 9.9.4-RedHat-9.9.4-74.el7_6.2 <<>> odii.cic.com
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 55935
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;odii.cic.com.			IN	A

;; ANSWER SECTION:
odii.cic.com.		86400	IN	A	192.168.8.2

;; AUTHORITY SECTION:
cic.com.		86400	IN	NS	odii.cic.com.

;; Query time: 1 msec
;; SERVER: 192.168.8.2#53(192.168.8.2)
;; WHEN: Fri Aug 09 01:53:42 GMT 2019
;; MSG SIZE  rcvd: 71
```
et aussi il peut être vérifié en utilisant la commande `host`
```bash
$ host odii.cic.com
odii.cic.com has address 192.168.8.2
```

### *à suivre*...
#####                               Références 
							
+ Negus, N.  2015: Linux Bible(9th edition). Indianapolis, Indiana: John Wiley & Sons Inc., pp. 187,355-358, 362-375,449-476,576-577,708-713.

+ 2019, [BIND 9 Administrator Reference Manual BIND 9.14.4 (Stable Release](https://downloads.isc.org/isc/bind9/cur/9.14/doc/arm/Bv9ARM.pdf (accessed August 6,2019)) .
+ 2019, [System Administrator's Guide ](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/index (accessed August 7,2019)) .

+ 1987, [DOMAIN NAMES - IMPLEMENTATION AND SPECIFICATION ](https://tools.ietf.org/html/rfc1035 (accessed August 7,2019)) .

+ 1987, [DOMAIN NAMES - CONCEPTS AND FACILITIES ](https://tools.ietf.org/html/rfc1034 (accessed August 7,2019)) .

+ 2013, [Introduction to DNS (Domain Name Services) ](https://www.youtube.com/watch?v=VwpP8PUzqLw (accessed August 6,2019)) .

+ 2018, [Linux Administration Tutorial - Configuring A DNS Server In 10 Simple Steps | Edureka Live  ](https://www.youtube.com/watch?v=0X9em99Vcl0&t=7s (accessed August 6,2019)) .

+ 2014, [Setting Up DNS Server On CentOS 7](https://www.unixmen.com/setting-dns-server-centos-7/ (accessed August 7,2019)) .
 
+ 2018, [How to configure DNS Name Server in Centos7 , Redhat7 (Server and Client Configuration)) ](https://www.youtube.com/watch?v=is-eg2X5ru4&t=2s (accessed August 6,2019)) .



