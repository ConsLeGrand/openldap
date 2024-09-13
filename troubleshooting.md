# Changer le SID d'un serveur
## Vérifier le SID actuel du domaine
>Pour voir le SID actuel du domaine Samba, vous pouvez utiliser la commande suivante :
```shell
    sudo net getdomainsid
```

## Spécifier un SID lors de la configuration initiale de samba
```shell
    sudo samba-tool domain provision --use-rfc2307 --realm=mydomain.local --domain=MYDOMAIN --domain-sid=S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX
```
##  Changer le SID d'un domaine existant
>Changer le SID d'un domaine existant est une opération délicate et n'est généralement pas recommandé sauf si c'est absolument nécessaire (par exemple, après une migration). Voici comment procéder si vous devez le faire :
1. Arrêter les services Samba :

```shell
    sudo systemctl stop samba.service
```
2. Utiliser la commande net setdomainsid pour spécifier un nouveau SID :

```shell
    sudo net setdomainsid S-1-5-21-XXXXXXXXXX-XXXXXXXXXX-XXXXXXXXXX
```
>Si vous devez également changer le SID du serveur local :

```shell
    sudo net setlocalsid S-1-5-21-YYYYYYYYYY-YYYYYYYYYY-YYYYYYYYYY
```

3. Vérifier que le changement a été appliqué :
```shell
    sudo net getdomainsid

```
4. Redémarrer les services Samba :
```shell
    sudo systemctl start samba.service
```
