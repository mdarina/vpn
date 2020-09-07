# Installation et configuration d'un serveur OpenVPN sur UBUNTU
=============================================================

On va faire installation et configuration OpenVPN serveur Ubuntu en 2 étapes.

 Dans le premier on va installer et configurer OpenVPN server créer 1 client avec script et utiliser premiere client pour connecter à VPN .

Dans le deuxième  on va créer deuxieme  client manuellement et connecter ordinateur avec Linux dans le réseau VPN en utilisant linge de commande.
Aussi, parce que 2 ordinateurs (Windows et linux) va être dans le même
réseau virtuel on va connecter  de Windows à Linux en ssh.

Aujourd'hui je  vais faire première étape.

Tout d’abord je vais installer le serveur OpenVPN sur [VPS](https://serverum.com/virtual-servers) avec 1 GB RAM, 1 CPU et 10 GB HDD et seulement 10 Mbit réseau  qui coûte 
seulement  1\$ par mois. Je vais créer 3 utilisateur VPN. Parce que c’est seulement projet pour test, je vais faire login comme root.

### L’installation les logicieles : 

Il faut installer:

-   openvpn
-   openssl
-   easy-rsa iptables et
-   bash-completion

apt-get install openvpn openssl easy-rsa iptables bash-completion -y

<img src="/images/image25.png">

### Préparation les  variables pour génération les clés: 

Avant création, il faut configurer  CA Directory: copier easy-rsa dans
\~/openvpn-ca

make-cadir \~/openvpn-ca

<img src="/images/image10.png">

après on va aller dans ce dossier:

cd \~/openvpn-ca

<img src="/images/image33.png"> 

Vérification \~/openvpn-ca

ls -l

<img src="/images/image11.png">

On commence la rédaction vars, il faut ouvrir vars et remplir les champs
suivants:

vim vars

<img src="/images/image37.png">

### Génération des certificats et clés du serveur: 

Après il faut exécuter:

source vars

<img src="/images/image2.png">

Et puis il faut supprimer tous les anciens certificats  :

./clean-all

<img src="/images/image45.png">

On on va commencer crée les nouveaux certificats . Premierement on va
creer   build-ca:

./build-ca

Il faut confirme tout parce que tout les info utilisent de file
précédent

<img src="/images/image20.png">

Après on va créer key-server :

./build-key-server server

<img src="/images/image29.png">

Ensuite, nous pouvons générer une signature HMAC pour renforcer les
capacités de vérification d’intégrité TLS du serveur:

openvpn --genkey --secret keys/ta.key

<img src="/images/image31.png">

* * * * *

### Générez un certificat client et une paire de clés: 

./build-key client1  no pass

./build-key client2 no pass

./build-key client3 no pass

<img src="/images/image28.png">

generate keys, génération peux prendre le temps

./build-dh

<img src="/images/image30.png">

Le serveur a également besoin d'un fichier de paramètres DH. Cela peut
être créé en utilisant OpenSSL:

openssl dhparam -out dh2048.pem 2048

<img src="/images/image3.png">

### Configurez le service OpenVPN. 

Il faut  copier les fichiers dans / etc / openvpn

cd \~/openvpn-ca/keys

<img src="/images/image14.png">

cp ca.crt server.crt server.key ta.key dh2048.pem /etc/openvpn

gunzip -c
/usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz |
sudo tee /etc/openvpn/server.conf

<img src="/images/image15.png">

### Configurer la configuration OpenVPN 

 vim /etc/openvpn/server.conf

Il faut supprimer  “;” Pour activer la ligne tls-auth:

tls-auth ta.key 0 \# This file is secret

cipher AES-128-CBC

user nobody

group nogroup

Sous “cipher AES-128-CBC” il faut ajouter:

auth SHA256

<img src="/images/image35.png">

Aussi il faut supprimer “;” Pour décommenter la ligne:

push "redirect-gateway def1 bypass-dhcp"

<img src="/images/image38.png">

### Configurer la configuration réseau de votre serveur 

Autorisation de transfert IP:

Configuration

vim /etc/sysctl.conf

Activer: net.ipv4.ip\_forward=1

<img src="/images/image17.png">

Pour appliquer changement :

sudo sysctl -p

<img src="/images/image21.png">

Il faut activer forwarding:

echo 1 \>\> /proc/sys/net/ipv4/conf/all/forwarding

<img src="/images/image32.png">

### Configurer les règles UFW pour masquer les connexions client 

Vérification carte réseau:

ip route | grep default

<img src="/images/image43.png">

On ajouter dans:

vim /etc/ufw/before.rules

\# START OPENVPN RULES

\# NAT table rules

\*nat

:POSTROUTING ACCEPT [0:0]

\# Allow traffic from OpenVPN client to ens3

-A POSTROUTING -s 10.8.0.0/24 -o ens3 -j MASQUERADE

COMMIT

\# END OPENVPN RULES

<img src="/images/image12.png">

Pour changer les règles Firewall il faut corriger:

/etc/default/ufw

vim /etc/default/ufw

Il faut change:

DEFAULT\_FORWARD\_POLICY="DROP" to

DEFAULT\_FORWARD\_POLICY="ACCEPT"

<img src="/images/image24.png">

On va ouvrir les ports 1194/udp et OpenSSH (je vais utiliser port par
défaut)

ufw allow 1194/udp && ufw allow OpenSSH

<img src="/images/image27.png">

Pour appliquer les changement il faut redémarrer FW:

ufw disable && ufw enable

<img src="/images/image18.png">

Verification status firewall:

ufw status

<img src="/images/image5.png">

Vérification NAT:

iptables -L -t nat

<img src="/images/image46.png"> 

###  

### Activer le service OpenVPN 

Si vérification passé bien on va démarrer openVPN:

systemctl start openvpn@server

<img src="/images/image22.png">

Pour vérifier si il démarre bien il faut exécuter:

systemctl status openvpn@server

<img src="/images/image36.png">

Pour vérification tun0 disponibilité de l'interface OpenVPN il faut
faire:

 ip addr show tun0

<img src="/images/image7.png">

SI openvpn ne marche pas essayer de faire restart openvpn ou faire
restart serveur au complet.

service openvpn restart

<img src="/images/image9.png">

Si il y a le problème il faut regarder log:

tail -f /var/log/syslog

Si tout fonctionne bien, passez à l'étape suivante :)

### Créer une configuration client 

Premièrement on va faire création dossier ccd/files pour clients:

mkdir -p \~/ccd/files

<img src="/images/image40.png">

Changer permissions:

chmod 700 \~/ccd/files

<img src="/images/image39.png">

Copier example file dans dossier de client:

cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf
\~/ccd/base.conf

<img src="/images/image42.png">

On va changer base.conf:

vim \~/ccd/base.conf

Il faut mettre IP publique de votre serveur et port (comme pendant
configuration serveur):

<img src="/images/image23.png">

On utilise protocol UDP pour activer ce protocol il faut supprimer “;”:

<img src="/images/image26.png">

Pour le client Windows on active user nobody et group nogroup (il faut
supprimer “;”), mais si vous utiliser comme client openvpn  Linux, il
faut laisser cela inactive:

<img src="/images/image6.png">

Il faut  désactiver prochaines lignes (ajouter pour “;”):

<img src="/images/image16.png">

Aussi on va ajouter prochaines lignes:

<img src="/images/image44.png">

et key-direction 1 (parce que c’est client, pour serveur on a utilisé 0)

<img src="/images/image1.png">

Pour faire configuration client on peut utiliser  script ou faire
fichier manuellement.Je vais montrer deux possibilité :)

Pour configuration client avec script vous devez devez suivre ces
étapes:

1.  Création d'un script de génération de configuration (vous pouvez
    aussi télécharger ce fichier
    [ici](hhttps://github.com/olexdziuba/vpn/blob/master/make_config.sh&sa=D&ust=1599441183299000&usg=AOvVaw2oN2kFmCdveUgLUvUL5rM2))

vim \~/ccd/make\_config.sh

À l'intérieur, collez le script suivant:

 

\#!/bin/bash

\# First argument: Client identifier

KEY\_DIR=\~/openvpn-ca/keys

OUTPUT\_DIR=\~/ccd/files

BASE\_CONFIG=\~/ccd/base.conf

cat \${BASE\_CONFIG} \\

    \<(echo -e '\<ca\>') \\

    \${KEY\_DIR}/ca.crt \\

    \<(echo -e '\</ca\>\\n\<cert\>') \\

    \${KEY\_DIR}/\${1}.crt \\

    \<(echo -e '\</cert\>\\n\<key\>') \\

    ./make\_config.sh client

    \<(echo -e '\</key\>\\n\<tls-auth\>') \\

    \${KEY\_DIR}/ta.key \\

    \<(echo -e '\</tls-auth\>') \\

    \> \${OUTPUT\_DIR}/\${1}.ovpn

<img src="/images/image4.png">

2.  Changer permission du fichier:

chmod 700 \~/ccd/make\_config.sh

<img src="/images/image8.png">

​3. Générer un fichier de configuration client

cd \~/ccd

./make\_config.sh client1

<img src="/images/image13.png">

On verifier:

ls \~/ccd/files

<img src="/images/image19.png">

Il faut vérifier ce file, il faut ajouter remote avant adresse IP:

<img src="/images/image34.png">

Après on copier ce file sur ordinateur de client . Sur ordinateur avec
Windows il faut installer
[OpenVPN](https://openvpn.net/community-downloads/&sa=D&ust=1599441183308000&usg=AOvVaw39D_ZVuJ_mmEa__34jTmCE),
ajouter nouveau client. Aussi il faut exécuter openvpn comme
administrateut ("run as admin"). Si il connection bloquer, c’est bonne
idée de lire logfile :)

Et voilà, on a connecté. Je mesure vitesse internet avec speedtest’ ping
grand et vitesse 10 Mbps upload et download, mais c’est correct, parce
que le serveur a ces restrictions.

\
<img src="/images/image41.png">