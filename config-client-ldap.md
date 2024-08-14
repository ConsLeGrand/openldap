# 1. configurer le client ldap (rhel9) avec ssl activer sur le server ldap
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

# 2. configurer le client ldap (rhel9) ssl desactiver sur le server ldap
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