# Groupe de pauche_m 986469

## 0 - Setup les machines virtuelles

### Créer un réseau HOST-ONLY

Sur Virtualbox, aller dans "Gestionnaire de sous réseaux" puis modifier ou créer un sous réseau pour convenir à votre range ainsi que votre masque de sous réseau.

### Création de la VM template

Installer l'iso de Debian Bullseye:
https://www.debian.org/releases/bullseye/debian-installer/

Sur Virtualbox créer une VM template. Lui plug une carte réseau en bridge. 

Faire l'installation comme indiqué dans le sujet.

Pour définir le hostname:
```
su root
hostnamectl set-hostname template.res1.local 
```

Faites les installations utiles:
```
apt get install sudo iptables ... -y
```

### Création des autres VM

Refaire les étapes précédantes en changeant le nom de l'hôte.
Pour la VM gateway, ajoutez une carte en bridged. Une seconde carte réseau devra être configurer dans un réseau HOST-ONLY.

Pour les trois autres VM, plug seulement la carte réseau en HOST-ONLY.


## 1 - Accès à Internet

### Activer le forwarding d'ip

```
sudo nano /etc/sysctl.conf
```
Il va falloir décommenter cette ligne
![image_1](https://cdn.discordapp.com/attachments/457484268319539211/1020373818079969381/unknown.png)

### Changement sur l'interface de la VM gateway

Sur votre gateway, configurer votre carte en HOST-ONLY ainsi:
![image_2](https://cdn.discordapp.com/attachments/457484268319539211/1020375238900453376/unknown.png)
En considérant que votre adapter est enp0s3, si ce n'est pas le cas changer le.
Configurez votre addresse dans le sous réseau.

Pour votre deuxième adapter configurez le ainsi:
![image_3](https://cdn.discordapp.com/attachments/457484268319539211/1020377317714972742/unknown.png)
En considérant encore que votre adapter est enp0s8.

Vous pouvez ensuite redemarrer votre vm
```
sudo reboot
```

Vous serez ensuite capable de pouvoir ping google.

### Configurer les iptables 

Nous devons configurer la table nat:
*cette table est consultée lorsqu'on rencontre un paquet qui crée une nouvelle connexion.*

Installer le package iptables:
```
sudo apt install iptables
```

Nous allons configurer la chaîne FORWARDING: *la chaîne intégrée POSTROUTING pour NAT sur le périphérique réseau externe du pare-feu. POSTROUTING permet aux paquets d'être modifiés lorsqu'ils quittent le périphérique externe du pare-feu. La cible est spécifiée pour masquer l'adresse IP privée d'un noeud avec l'adresse IP externe du pare-feu / de la passerelle.*

```
sudo iptables -t nat -A POSTROUTING -o enp0s8 -j MASQUERADE
```

Cependant cette règle ne sera pas sauvegarder lors d'un reboot. Pour pouvoir rendre les règles iptables permanents :
```
sudo apt install iptables-persistent -y
sudo sh -c "iptables-save /etc/iptables/rules.v4"
sudo iptables-restore -n < /etc/iptables/rules.v4
```

### Changer les interfaces des autres VM:

Dans le fichier */etc/network/interfaces/* de chaque VM modifier l'interface ainsi:
![image_4](https://cdn.discordapp.com/attachments/457484268319539211/1020385171394023514/unknown.png)

Modifier l'addresse selon la VM et votre range. L'addresse de la gateway doit correspondre à la votre.

Pour la VM client, vous n'avez pas besoin d'effectuer ces changements.

Vous pouvez effectuer un:
```
sudo reboot
```

Vous pouvez ensuite ping google depuis vos VM dans le réseau.

## 2 - DHCP

### Configurer le server DHCP

```
sudo apt install isc-dhcp-server -y
```

Ensuite configurer le fichier :
```
sudo nano /etc/dhcp/dhcpd.conf
```
![image_5](https://cdn.discordapp.com/attachments/457484268319539211/1020389704748765244/unknown.png)

Définissez votre sous réseau ainsi:
![image_6](https://cdn.discordapp.com/attachments/457484268319539211/1020390446926663760/unknown.png)

Evidemment, définissez les valeurs selon votre schéma et vos adresses et ranges.

Sur votre VM client, votre ip doit correspondre à la première IP de la range (ici 10.242.1.11).

### Setup les clefs RSA pour le SSH

Générer une clé RSA avec 2048 bits

```
ssh-keygen -b 2048
```

ssh-keygen génère automatiquement une clé de type RSA


Il faut créer sur les VM distante le dossier /home/{user} ainsi que le .ssh/ et le fichier authorized_keys si il n'existe pas. (dans le répertoire .ssh/)

Changer le owner du authorized_keys à {user}.

Copier la clé publique de gateway dans le authorized_keys.

Pour pouvoir se connecter sans mot de passe il faut renseigner le PasswordAuthorization dans le sshconfig en no

```
sudo nano /etc/ssh/ssh_config
```
![image_7](https://cdn.discordapp.com/attachments/457484268319539211/1020392662932672542/unknown.png)

Après un reboot, vous pourrez vos connecter à vos autres VMs sans mot de passe.

```
ssh {user}@{address}
```

## 3 - Nginx

### Install package

Installer le package Nginx et UFW
```
sudo apt install nginx ufw -y
```

UFW est un équivalent à iptables, il permet de setup un pare-feu. **La configuration de UFW sera remplacé par iptables.**

### Setup UFW

L'on doit autoriser l'application NGINX à accéder aux ports de notre machine (notamment le port HTTP, HTTPS...).

Pour cela:
```
sudo ufw allow 'Nginx HTTP'
```

Ensuite pour vérifier que le service fonctionne bien.
```
systemctl status nginx
```

### Changer le fichier index.html par défaut

L'on pourra modifier le fichier index.html à cet emplacement:
```
sudo nano /var/www/html/index.nginx-debian.html
```

### Setup les logs

**A faire plus tard**

### Empêcher l'update du package Nginx

Pour empêcher le package nginx de s'update lors d'un apt update, par exemple, il suffit de faire cette commande:
```
sudo apt-mark hold nginx
```
De cette manière, vous pourrez gérer l'update de votre nginx au besoin quand vous le voudrez.

Pour voir les packages hold:
```
sudo apt-mark showhold
```

### 4 - OPENVPN



**© pauche_m && garo_n**