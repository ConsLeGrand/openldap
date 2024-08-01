# Installer Openldap
```shell
 dnf -y install openldap-servers openldap-clients
 systemctl enable --now slapd
 ```
# 	Generer un mot de passe pour le root de ldap
```shell
 slappasswd
```
# Creer un fichier Root
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
#  Ajouter les entrées 
```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f chroot.ldif
```

# Creer le domaine de base
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
# Creer la base
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
#  Ajouter les entrées 
```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f base.ldif
```

#	Importer les  Schemas basic. 
```shell
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
```
# Creer deux user linux
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

# Rechercher les users
```shell
ldapsearch -x -LLL -D "cn=Manager,dc=ldap,dc=lan" -W -b "ou=devops,dc=ldap,dc=lan"
```