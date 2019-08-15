# Installation et Configuration des packages bind et nginx sur CentOS 7 pour réseau local

Domain Name System est responsable de la traduction du domaine Internet et des noms d’hôte en adresses IP, et inversement. Les clients DNS enverront les demandes (nom d’hôte / adresses IP) au serveur DNS et attendront une réponse.
-------------------------------------------------------------- 
### Installation du package du serveur DNS BIND
Installez DNS avec toutes ses dépendances.

```bash
# yum install -y bind*
-------------------------------------------------------------- ```
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
DOMAIN=odii.cic
DNS1=192.168.8.2
DNS2=127.0.0.1
PEERDNS=yes
PEERROUTES=yes

```
Trouvez des descriptions des variables de réseau ci-dessus dans votre `/usr/share/doc/initscripts*/sysconfig.txt` fichier.
-------------------------------------------------------------- 
### Attribuer un FQDN (Fully Qualified Domain Name) au serveur

Modifier le`/etc/sysconfig/network` fichier pour ajouter le NOM D'HÔTE de votre choix. Le mien est `host.odii.cic`
** Remarque ** Étant donné que nous configurons un serveur DNS local, le domaine ne doit pas s'arrêter avec .com. Cela doit être évité afin de ne pas neutraliser les serveurs racine avec des requêtes d'un domaine qui n'existent pas sur Internet.
```bash
# vim /etc/sysconfig/network 
NETWORKING=yes
HOSTNAME=host.odii.cic
-------------------------------------------------------------- ```
###  Configurez le `/etc/hosts` fichier en ajoutant une entrée d'hôte
```bash
# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.2    host.odii.cic
```
Remarquez ci-dessus que `192.168.8.2` et` host.odii.cic` ont été ajoutés et qu'ils changeront en fonction de l'adresse IP statique et du nom d'hôte de votre choix.
`127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6 `
Ce qui précède a été généré par défaut lors de l’installation du système d’exploitation.
-------------------------------------------------------------- 
### Configurer `/etc/resolv.conf`
```bash
# vim /etc/resolv.conf
search odii.cic
nameserver 192.168.8.2
```

Remarque: l'étape ci-dessus sera configurée non seulement sur l'hôte du serveur DNS, mais également sur tous les clients DNS du réseau local qui utiliseront le serveur DNS pour la résolution de noms ou d'IP.
-------------------------------------------------------------- 
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
-------------------------------------------------------------- 
### Configurez le fichier `/etc/named.rfc1912.zones` de manière à pointer vers les zones
```bash
# vim /etc/named.rfc1912.zones
...
zone "odii.cic" IN {            # change to your domain name 
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
La zone aval est `odii.cic` et la zone inversée est 8.168.192, qui est l'inverse de l'ID réseau de l'adresse IP attribuée,` 192.168.8 `.
-------------------------------------------------------------- 
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
@	IN SOA	 host.odii.cic.  root.host.odii.cic. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H )	; minimum

@	IN 	NS	host.odii.cic.
host	IN	A	192.168.8.2
ifeka	IN	A	192.168.8.2
www	    IN	A	192.168.8.2
mail	IN	A	192.168.8.2
smtp	IN	A	192.168.8.2
smtps	IN	A	192.168.8.2
pop3	IN	A	192.168.8.2
client1	IN	A	192.168.8.1
client3	IN	A	192.168.8.3
client4	IN	A	192.168.8.4
webmail	IN	CNAME	ifeka
ftp	IN	CNAME	ifeka	
@	IN	A	192.168.8.2
```
configurez le `/var/named/reverse.zone` pour ressembler à ceci
```bash 
$ORIGIN	8.168.192.in-addr.arpa.
$TTL 1D
@	IN 	SOA	host.odii.cic.  root.odii.cic. (
					0	; serial
					1D	; refresh
					1H	; retry
					1W	; expire
					3H ); minimum
@	IN	 NS	host.odii.cic.
@	IN	 PTR	odii.cic.
host	IN	 A	192.168.8.2
client1	IN	 A	192.168.8.1
client3	IN	 A	192.168.8.3
client4	IN	 A	192.168.8.4
1	IN	 PTR	client1.odii.cic.
2	IN	 PTR	host.odii.cic.
3	IN	 PTR	client3.odii.cic.
4	IN	 PTR	client4.odii.cic.
-------------------------------------------------------------- 
### Changer la propriété de groupe des fichiers `forward.zone` et` reverse.zone` de la root à named
```bash
# chgrp named /var/named/forward.zone
# chgrp named /var/named/reverse.zone  
-------------------------------------------------------------- ```
### Redémarrez le DNS named.service
```bash
# systemctl restart named.service
```
Pour vérifier si votre connexion fonctionne, exécutez les commandes `dig` et` host` qui vont rechercher le service named.
```bash
# dig host.odii.cic

; <<>> DiG 9.9.4-RedHat-9.9.4-74.el7_6.2 <<>> host.odii.cic
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 28381
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;host.odii.cic.			IN	A

;; ANSWER SECTION:
host.odii.cic.		86400	IN	A	192.168.8.2

;; AUTHORITY SECTION:
odii.cic.		86400	IN	NS	host.odii.cic.

;; Query time: 26 msec
;; SERVER: 192.168.8.2#53(192.168.8.2)
;; WHEN: Mon Aug 12 02:05:31 GMT 2019
;; MSG SIZE  rcvd: 72

```

Maintenant avec la commande `host`
```bash
# host host.odii.cic
host.odii.cic has address 192.168.8.2
```
Maintenant avec la commande `nslookup`

```bash
# nslookup odii.cic

Server:		192.168.8.2
Address:	192.168.8.2#53

Name:	odii.cic
Address: 192.168.8.2
```

```bash
# nslookup www.odii.cic
Server:		192.168.8.2
Address:	192.168.8.2#53

Name:	www.odii.cic
Address: 192.168.8.2
```
```bash
# nslookup ftp.odii.cic
Server:		192.168.8.2
Address:	192.168.8.2#53

ftp.odii.cic	canonical name = ifeka.odii.cic.
Name:	ifeka.odii.cic
Address: 192.168.8.2
```
```bash
# hostname
odii.cic
```

Les commandes `host` et` dig` ne sont utilisées que pour interroger les serveurs DNS. Elles ne vérifient pas le fichier `nsswitch.conf` pour rechercher d'autres emplacements, tels que le fichier des hôtes locaux. Pour cela, je devrais utiliser le Commande `getent` pour trouver un hôte nommé odii.cic qui a été entré dans mon fichier` /etc/hosts` local
```bash
# getent hosts
127.0.0.1       localhost localhost.localdomain localhost4 localhost4.localdomain4
127.0.0.1       localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.8.2     odii.cic
```


Ce qui précède montre que le DNS est correctement interrogé et qu'il est capable de résoudre les noms en IP.


-------------------------------------------------------------- 
# L'installation et la configuration de Nginx

** Remarque **: Les étapes importantes suivantes doivent être effectuées si Nginx doit être installé sur un CentOS doté du serveur Web par défaut Apache pour la plupart des distributions en cours d'exécution sans avoir à désinstaller Apache avant l'installation de Nginx.
+ Modifiez le port 80 par défaut d'Apache en un port personnalisé.
+ Désactivez Apache à partir du démarrage du système.
+ Masque Apache de jamais courir.
-------------------------------------------------------------- 
#### Changer le port 80 par défaut d'Apache en un port personnalisé
Configurez le service Apache httpd pour qu’il écoute sur un port différent du port 80, par exemple le port 8090 afin de réserver le port 80 pour le service Nginx. Ouvrez le fichier de configuration `/etc/httpd/conf/httpd.conf`, puis changez le port 80 en port 8090:
```bash
# vim /etc/httpd/conf/httpd.conf
...
Listen 8090
...
```
Pour autoriser le port 8090 via un pare-feu, procédez comme suit.
```bash 
$ sudo firewall-cmd --permanent --add-port=8090/tcp
```
Pour appliquer les modifications ci-dessus afin de permettre à Apache de se lier sur le nouveau port 8090, redémarrez le démon httpd et vérifiez la table des sockets locales à l'aide de la commande `netstat`.
Redémarrez le serveur Web Apache
```bash
# systemctl restart httpd.service 
```
Ajouter des règles SeLinux pour le port 8090
```bash
# semanage port -a -t http_port_t -p tcp 8090
# semanage port -m -t http_port_t -p tcp 8090
```

Vérifiez que le nouveau port 8090 lie et écoute le trafic entrant avec la commande `netstat`
```bash
# netstat -tupln | grep httpd
tcp        0      0 :::8090              :::*               LISTEN      14682/httpd 
tcp6       0      0 :::8090             :::*                LISTEN      14682/httpd
```
Ouvrez un navigateur et accédez à l'adresse IP du serveur ou au nom de domaine sur le port 8090 pour vérifier si le nouveau 8090 est accessible sur le réseau. La page par défaut d'Apache doit être affichée dans le navigateur.
```bash
http://192.168.8.2:8090
http://host.odii.cic:8090
-------------------------------------------------------------- ```
### Désactiver Apache à partir du démarrage avec le démon d'initialisation systemd
Pour désactiver Apache à partir du démarrage, exécutez la commande:
```bash 
# syatemctl disable httpd.service
```
Cependant, cela n'arrête pas immédiatement le service. L'option stop de systemctl doit être utilisée pour qu'il s'arrête immédiatement s'il est en cours d'exécution.
```bash
# systemctl stop httpd.service
```
Avec les étapes ci-dessus, Apache a été désactivé et ne démarrera pas au démarrage seul. Parfois, la désactivation d’un service ne suffit pas pour s’assurer qu’il ne s’exécute pas, car si un autre service répertoriait httpd en tant que dépendance, ce service démarrer httpd quand il a commencé

Pour désactiver httpd de manière à l'empêcher de s'exécuter sur le système, utilisez l'option mask. Pour que le service httpd ne s'exécute jamais, tapez ce qui suit:
```bash
# systemctl mask httpd.service
Created symlink from /etc/systemd/system/httpd.service to /dev/null.
```
Comme le montre la sortie, le fichier httpd.service dans / etc est lié à /dev/null.So, même si quelqu'un essayait d'exécuter ce service httpd, rien ne se passerait. Pour pouvoir utiliser à nouveau httpd, tapez
```bash
# systemctl unmask httpd.service
```
Ainsi, avec les étapes ci-dessus, le serveur Web Nginx peut être installé simultanément avec Apache sur le même serveur CentOS sans avoir à désinstaller Apache pour Nginx.

** Remarque ** Les deux serveurs Web peuvent être configurés pour s'exécuter en même temps, l'un servant de proxy pour l'autre.
-------------------------------------------------------------- 
### Installer le serveur Web Nginx
Commencez par mettre à jour les packages logiciels système avec la dernière version:
```bash
# yum -y update
```
 Ensuite, installez le serveur HTTP Nginx à partir du référentiel EPEL en utilisant le gestionnaire de paquets YUM comme suit:
 ```bash
# yum install epel-release
# yum install nginx 
```
Une fois le serveur Web Nginx installé, vous pouvez le démarrer pour la première fois et lui permettre de démarrer automatiquement au démarrage du système, car Apache a été désactivé:
```bash
# systemctl start nginx
# systemctl enable nginx
# systemctl status nginx
```
Par défaut, le pare-feu intégré à CentOS 7 est configuré pour bloquer le trafic Nginx. Pour autoriser le trafic Web sur Nginx, mettez à jour les règles de pare-feu du système pour autoriser les paquets entrants sur HTTP et HTTPS à l'aide des commandes ci-dessous.
```bash
# firewall-cmd --zone=public --permanent --add-service=http
# firewall-cmd --zone=public --permanent --add-service=https
# firewall-cmd --reload
```
Vérifiez que le serveur Nginx fonctionne correctement après l'installation en accédant à l'URL configurée en tant que nom d'hôte ou IP.
```bash
http://host.odii.cic
http://192.168.8.2
```
Une page Nginx par défaut sera affichée.


-------------------------------------------------------------- 
#####                               References 
							
+ Negus, N.  2015: Linux Bible(9th edition). Indianapolis, Indiana: John Wiley & Sons Inc., pp. 187,355-358, 362-375,449-476,576-577,708-713.

+ 2019, [BIND 9 Administrator Reference Manual BIND 9.14.4 (Stable Release](https://downloads.isc.org/isc/bind9/cur/9.14/doc/arm/Bv9ARM.pdf (accessed August 6,2019)) .
+ 2019, [System Administrator's Guide ](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/system_administrators_guide/index (accessed August 7,2019)) .

+ 1987, [DOMAIN NAMES - IMPLEMENTATION AND SPECIFICATION ](https://tools.ietf.org/html/rfc1035 (accessed August 7,2019)) .

+ 1987, [DOMAIN NAMES - CONCEPTS AND FACILITIES ](https://tools.ietf.org/html/rfc1034 (accessed August 7,2019)) .

+ 2013, [Introduction to DNS (Domain Name Services) ](https://www.youtube.com/watch?v=VwpP8PUzqLw (accessed August 6,2019)) .

+ 2018, [Linux Administration Tutorial - Configuring A DNS Server In 10 Simple Steps | Edureka Live  ](https://www.youtube.com/watch?v=0X9em99Vcl0&t=7s (accessed August 6,2019)) .

+ 2014, [Setting Up DNS Server On CentOS 7](https://www.unixmen.com/setting-dns-server-centos-7/ (accessed August 7,2019)) .
 
+ 2018, [How to configure DNS Name Server in Centos7 , Redhat7 (Server and Client Configuration)) ](https://www.youtube.com/watch?v=is-eg2X5ru4&t=2s (accessed August 6,2019)) .

+ 2017, [How to Install Nginx on CentOS 7](https://www.tecmint.com/install-nginx-on-centos-7/ (accessed August 11,2019)) .
 
+ 2016, [How To Change Apache Default Port To A Custom Port ](https://www.ostechnix.com/how-to-change-apache-ftp-and-ssh-default-port-to-a-custom-port-part-1/ (accessed August 10,2019)) .

+ 2012, [Install & Configure BIND DNS Server in CentOS - Part 3 ](https://www.youtube.com/watch?v=70aVLHzbMzw&t=17s (accessed August 10,2019)) .

