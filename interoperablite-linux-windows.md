# Interoperablite entre Linux et Windows grace a la solution samba-dc 
0. Pré-requis:
- 1 serveur Fedora Linux 36 pour samba.
    - Hostname: dc1.accel.lan
    - Domain: accel.lan
    - IP: 192.168.14.108/24

- 1 serveur Fedora Linux 36 pour le dns.
    - Hostname: dns.accel.lan
    - Domain: accel.lan
    - IP: 192.168.14.78/24

- 1 machine windows client.
    - Hostname: pc1.accel.lan
    - Domain: accel.lan
    - IP: 192.168.14.98/24

1. Installation et Configuration de Samba 

- Configurer le nom d'hôte
```shell
    sudo hostnamectl hostname dc1.accel.lan
```

- Installer Samba et les dépendances
```shell
    dnf install -y samba samba-dc samba-client krb5-workstation
```
- Ouvrir le service Samba dans le pare-feu et Recharger le pare-feu
```shell
    firewall-cmd --permanent --add-service samba-dc && firewall-cmd --reload
```
- Activer les paramètres SELinux pour Samba et Appliquer les contextes de sécurité avec restorecon
```shell
    setsebool -P samba_create_home_dirs=on samba_domain_controller=on samba_enable_home_dirs=on samba_portmapper=on use_samba_home_dirs=on  && restorecon -Rv /
```
- Créer le répertoire pour la configuration DNS personnalisée
```shell
    mkdir -p /etc/systemd/resolved.conf.d/
```
- Créer le fichier /etc/systemd/resolved.conf.d/custom.conf pour la configuration DNS
```yaml
    [Resolve]
    DNSStubListener=no
    Domains=accel.lan
    DNS=192.168.14.78

```
- Redémarrer le service systemd-resolved

```shell
    systemctl restart systemd-resolved
```
- Supprimer le fichier de configuration Samba par défaut
```shell
     rm /etc/samba/smb.conf
```
- Provisonner le domaine Samba
>l'option --domain-sid est a utiliser si l'on souhaite fixer nous meme un sid a notre server samba dans ce cas si nous l'avons utiliser pour le faire correspondre au sid du server ad 
>vu que nous voulons que le client puisse s'auth sur le ad ou le samba. 
```shell
    samba-tool domain provision --server-role=dc --use-rfc2307 --dns-backend=SAMBA_INTERNAL --realm=ACCEL.LAN --domain=ACCEL --adminpass=Passer@123 --domain-sid=$DOMAIN_SID
```
Remplacer le dns forwarder dans le fichier smb.conf qui pointe par defaut sur l'IP de l'hote samba par un dns au choix ici 8.8.8.8 

```shell
    sed -i 's/dns forwarder = 192.168.14.108/dns forwarder = 8.8.8.8/' /etc/samba/smb.conf
```
2. Configuration de Kerberos 
- Editer le fichier /etc/krb5.conf 
```yaml
    [libdefaults]
        default_realm = ACCEL.LAN
        dns_lookup_realm = false
        dns_lookup_kdc = true

    [realms]
    ACCEL.LAN = {
        default_domain = ACCEL
        }

    [domain_realm]
        dc1.accel.lan = ACCEL.LAN

```
- Activer et démarrer Samba

```shell
    systemctl enable --now samba
```

3. Tests
- Connectivité
```shell
    smbclient -L localhost -N
```
>Le résultat de la commande smbclient indique que la connexion a été établie avec succès.

- Testez maintenant la connexion de l'administrateur au partage netlogon a l'invite mettez le mot de passe de l'administrateur creer lors du provisionnement de samba ici Passer@123 :
```shell
    smbclient //localhost/netlogon -UAdministrator -c 'ls'
```
- test DNS:
>Pour vérifier si la résolution de noms fonctionne, exécutez les commandes suivantes :
```shell
    host -t SRV _ldap._tcp.accel.lan.
```
>le resultat de la commande ressemble a ceci:
```shell
_ldap._tcp.accel.lan has SRV record 0 100 389 dc1.accel.lan.
```
```shell
    host -t SRV _kerberos._udp.accel.lan.
```
>le resultat de la commande ressemble a ceci:
```shell
_kerberos._udp.accel.lan has SRV record 0 100 88 dc1.accel.lan.
```

```shell
   host -t A dc1.accel.lan.
```
>le resultat de la commande ressemble a ceci:
```shell
dc1.accel.lan has address 192.168.14.108
```
- test Kerberos 
>Il est important de tester Kerberos car il génère les tickets nécessaires pour permettre aux clients de s'authentifier par cryptage. Il s'appuie fortement sur une heure correcte.

>On ne saurait trop insister sur la nécessité de régler correctement la date et l'heure, d'où l'importance d'un service de synchronisation de l'heure fonctionnant à la fois sur les clients et sur les serveurs.
```shell
    kinit administrator
```

```shell
    klist
```

4. Ajouter un utilisateur au domaine
>samba-tool nous fournit une interface pour exécuter des tâches d'administration de domaine, de sorte que nous pouvons facilement ajouter un utilisateur au domaine.
- Ajout d'un utilisateur linux au domaine :
```shell
    sudo samba-tool user add devops --unix-home=/home/devops --login-shell=/bin/bash --gecos 'Devops Ing.' --given-name=Ing --surname='DevIng' --mail-address='devops@accel.lan'
```

- Ajout d'un utilisateur Windows au domaine :
```shell
    sudo samba-tool user add windev  --gecos 'windev Ing.' --given-name=windev --surname='Ing' --mail-address='windev@accel.lan'
```
- Pour dresser la liste des utilisateurs du domaine :
```shell
sudo samba-tool user list
```