# configurer ssl pour ldap
```shell
cd /etc/pki/tls/certs
# openssl genrsa -aes128 2048 > server.key
Enter PEM pass phrase:                  # set passphrase
Verifying - Enter PEM pass phrase:      # confirm
```

# supprimer la passphrase de la cle privee
```shell
# openssl rsa -in server.key -out server.key
Enter pass phrase for server.key:   # input passphrase
writing RSA key
```
# Vous pouvez laisser les champs par defaut sauf le champs common  name ou vous devez priciser le FQDN du server ldap obligatoirement
```shell
# openssl req -utf8 -new -key server.key -out server.csr
Common Name (eg, your name or your server's hostname) []:dlp.srv.world   # server's FQDN
```

# creer un certificat avec 10 ans date expiration 
```shell
openssl x509 -in server.csr -out server.crt -req -signkey server.key -days 3650
chmod 600 server.key
```
# tester avec la commande 
```shell
ldapsearch -x -H  ldaps://ldapserver.lan -D 'cn=Manager,dc=ldap,dc=lan'  -w passer123 -b 'dc=ldap,dc=lan' -v -Z
```
# troubleshooting 
en cas d'erreur avec le sertificat penser a copier les certificat 
```shell
cp /etc/openldap/certs/server.crt /etc/pki/ca-trust/source/anchors
cp /etc/openldap/certs/ca-bundle.crt /etc/pki/ca-trust/source/anchors/
 ```