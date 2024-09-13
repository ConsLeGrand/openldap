# 1 Installer le server Openldap 

## installer les paquets suivants
```shell
 dnf -y install openldap-servers openldap-clients
 systemctl enable --now slapd
 ```
## 	Generer un mot de passe pour le root de ldap
```shell
 slappasswd
```
## Creer un fichier Root pour l'administrtreur de ldap (rootDN)
```shell
 vi chrootpw.ldif
```
```yaml
dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ldap,dc=lan

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=ldap,dc=lan

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: <mot de passe root>
```
##  Ajouter les entrées 
```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f chroot.ldif
```

## Accorder les  privileges au rootDN
```shell
 vi domain.ldif
```
```yaml
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth"
  read by dn.base="cn=Manager,dc=ldap,dc=lan" read by * none

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ldap,dc=lan

dn: olcDatabase={2}mdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=ldap,dc=lan

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}4KaG8zyKB5RWVetm1I3/Rk77FGMK/tmF

dn: olcDatabase={2}mdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by
  dn="cn=Manager,dc=ldap,dc=lan" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=ldap,dc=lan" write by * read

```
```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f domain.ldif
```
## Creer la base et deux OU
```shell
 vi base.ldif
```
```yaml
dn: dc=ldap,dc=lan
objectClass: organization
objectClass: dcObject
o: ldap
dc: ldap

dn: ou=devops,dc=ldap,dc=lan
objectClass: organizationalUnit
ou: devops

dn: ou=accel,dc=ldap,dc=lan
objectClass: organizationalUnit
ou: accel

```
##  Ajouter les entrées 
```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f base.ldif
```

##	Importer les  Schemas basic. 
```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```
## Creer deux user linux
```shell
 vi useers.ldif
```
```yaml
dn: uid=sams,ou=devops,dc=ldap,dc=lan
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
cn: sams
sn: sams
uid: sams
userPassword: passer
uidNumber: 5000
gidNumber: 5000
loginshell: /bin/bash
homeDirectory: /home/sams
mail: sams@ldap.lan
givenName: sams

dn: uid=cons,ou=devops,dc=ldap,dc=lan
objectClass: posixAccount
objectClass: shadowAccount
objectClass: inetOrgPerson
cn: cons
sn: cons
uid: cons
userPassword: passer
uidNumber: 5001
gidNumber: 5001
loginshell: /bin/bash
homeDirectory: /home/cons
mail: cons@ldap.lan
givenName: cons
```
```shell
ldapadd -x -D "cn=Manager,dc=ldap,dc=lan" -w passer123 -f users.ldif
```

## Rechercher les users
```shell
ldapsearch -x -LLL -D "cn=Manager,dc=ldap,dc=lan" -W -b "ou=devops,dc=ldap,dc=lan"
```
## Afficher les schemas et autres conf
```shell
 ldapsearch -H ldapi:/// -Y EXTERNAL -b "cn=schema,cn=config"
```

# 2. configurer ssl pour ldap

## generer sur le server ldap les certs
```shell
cd /etc/pki/tls/certs
openssl genrsa -aes128 2048 > server.key
# Enter PEM pass phrase:                  # set passphrase
# Verifying - Enter PEM pass phrase:      # confirm
```

## supprimer la passphrase de la cle privee
```shell
openssl rsa -in server.key -out server.key
# Enter pass phrase for server.key:   # input passphrase
# writing RSA key
```
## Vous pouvez laisser les champs par defaut sauf le champs common  name ou vous devez priciser le FQDN du server ldap obligatoirement
```shell
openssl req -utf8 -new -key server.key -out server.csr
# Common Name (eg, your name or your server's hostname) []:dlp.srv.world   # server's FQDN
```

## creer un certificat avec 10 ans date expiration 
```shell
openssl x509 -in server.csr -out server.crt -req -signkey server.key -days 3650
chmod 600 server.key
```

 ## Config a faire cote serveur ldap
 ```shell
 cp /etc/pki/tls/certs/server.key \
/etc/pki/tls/certs/server.crt \
/etc/pki/tls/certs/ca-bundle.crt \
/etc/openldap/certs/
```

```shell
chown ldap. /etc/openldap/certs/server.key \
/etc/openldap/certs/server.crt \
/etc/openldap/certs/ca-bundle.crt
update-ca-trust extracted
```

```shell
vi mod_ssl.ldif
```
```file
dn: cn=config
changetype: modify
add: olcTLSCACertificateFile
olcTLSCACertificateFile: /etc/openldap/certs/ca-bundle.crt
-
replace: olcTLSCertificateFile
olcTLSCertificateFile: /etc/openldap/certs/server.crt
-
replace: olcTLSCertificateKeyFile
olcTLSCertificateKeyFile: /etc/openldap/certs/server.key
```

```shell
ldapmodify -Y EXTERNAL -H ldapi:/// -f mod_ssl.ldif
```
```shell
 systemctl restart slapd
 firewall-cmd --add-service={ldap,ldaps}
 firewall-cmd --runtime-to-permanent
 ```
## tester avec la commande 
```shell
ldapsearch -x -H  ldaps://ldapserver.lan -D 'cn=Manager,dc=ldap,dc=lan'  -w passer123 -b 'dc=ldap,dc=lan' -v -Z
```
## troubleshooting 
en cas d'erreur avec le certificat penser a copier les certificat  dans /etc/pki/ca-trust/source/anchors
```shell
cp /etc/openldap/certs/server.crt /etc/pki/ca-trust/source/anchors
cp /etc/openldap/certs/ca-bundle.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extracted
 ```

 # 3. configurer le client ldap (rhel9) comme suite si vous avez realiser l'etape 2 (ssl activer sur le server ldap)
```shell
dnf -y install openldap-clients sssd sssd-ldap oddjob-mkhomedir
```
## switch authentication provider to sssd
```shell
authselect select sssd with-mkhomedir --force
vi /etc/openldap/ldap.conf
```
```file
URI ldap://ldapserver.lan/
BASE dc=ldap,dc=lan
```

```shell
 vi /etc/sssd/sssd.conf
 ```

```file
# replacez [ldap_uri], [ldap_search_base]  a votre environment 
[domain/default]
id_provider = ldap
autofs_provider = ldap
auth_provider = ldap
chpass_provider = ldap
ldap_uri = ldap://ldapserver.lan/
ldap_search_base = dc=ldap,dc=lan
ldap_id_use_start_tls = True
ldap_tls_cacertdir = /etc/openldap/certs
cache_credentials = True
ldap_tls_reqcert = allow

[sssd]
services = nss, pam, autofs
domains = default

[nss]
homedir_substring = /home

```
```shell
 chmod 600 /etc/sssd/sssd.conf
 systemctl enable --now sssd oddjobd
 ```
 ## Tester

```shell
id utilisateur_ldap
su - utilisateur_ldap
```

# 4. configurer le client ldap (rhel9) comme suite si vous avez saute l'etape 2 (ssl activer sur le server ldap)

```shell
  sudo dnf install sssd sssd-client oddjob oddjob-mkhomedir authselect-compat
  sudo authselect select sssd with-mkhomedir --force
```

## Editer le fichier de conf sssd
```shell
 vi /etc/sssd/sssd.conf
```

```yaml
[sssd]
services = nss, pam
config_file_version = 2
domains = LDAP

[domain/LDAP]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://ldap.example.com
ldap_search_base = dc=example,dc=com
ldap_default_bind_dn = cn=admin,dc=example,dc=com
ldap_default_authtok_type = password
ldap_default_authtok = <password>

```
```shell
sudo chown root:root /etc/sssd/sssd.conf
sudo chmod 600 /etc/sssd/sssd.conf
sudo systemctl enable --now sssd
sudo systemctl status sssd
```
## Tester

```shell
id utilisateur_ldap
su - utilisateur_ldap
```