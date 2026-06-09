# Installation de FreeIPA sur Rocky Linux 9

Documentation basee sur l'installation reelle du serveur :

```text
ldaps01.ironforge.lab
```

Cette installation configure un serveur FreeIPA complet avec :

- autorite de certification Dogtag ;
- LDAP Directory Server ;
- Kerberos KDC ;
- interface web Apache ;
- DNS integre BIND ;
- generation SID ;
- PKINIT ;
- client IPA local sur le serveur.

---

## 1. Informations de l'installation

| Element | Valeur |
|---|---|
| Serveur FreeIPA | `ldaps01.ironforge.lab` |
| Nom court | `ldaps01` |
| Adresse IP | `192.0.2.10` |
| Domaine DNS | `ironforge.lab` |
| Realm Kerberos | `IRONFORGE.LAB` |
| NetBIOS domain | `IRONFORGE` |
| DNS forwarder | `192.0.2.1` |
| Zone reverse | `2.0.192.in-addr.arpa.` |
| Version FreeIPA | `4.13.1` |
| Log d'installation | `/var/log/ipaserver-install.log` |

---

## 2. Schema

```text
                         DNS amont
                         192.0.2.1
                              |
                              v
+-------------------+   DNS/LDAP/Kerberos   +-----------------------------+
| Clients Linux     +----------------------->| ldaps01.ironforge.lab       |
| SSSD + Kerberos   |                        | 192.0.2.10                  |
+-------------------+                        | FreeIPA + DNS + CA          |
                                             +-----------------------------+
```

Le serveur FreeIPA gere le domaine `ironforge.lab` et transfere les requetes externes vers `192.0.2.1`.

---

## 3. Configurer le hostname et `/etc/hosts`

Avant de lancer FreeIPA, le serveur doit avoir un FQDN stable et resolvable localement.

Configurer le hostname :

```bash
sudo hostnamectl set-hostname ldaps01.ironforge.lab
```

Verifier :

```bash
hostname
hostname -f
```

Le resultat attendu est :

```text
ldaps01.ironforge.lab
```

Editer `/etc/hosts` :

```bash
sudo vi /etc/hosts
```

Exemple generique :

```text
127.0.0.1       localhost localhost.localdomain
192.0.2.10      ldaps01.ironforge.lab ldaps01
```

Verifier la resolution locale :

```bash
getent hosts ldaps01.ironforge.lab
getent hosts ldaps01
```

Le serveur doit retourner l'adresse IP generique de documentation :

```text
192.0.2.10
```

---

## 4. Preparer firewalld avant l'installation

Avant de lancer `ipa-server-install`, ouvrez les ports FreeIPA dans `firewalld`. Cela evite d'installer un serveur correct mais inaccessible depuis les clients.

Activez et demarrez `firewalld` :

```bash
sudo systemctl enable --now firewalld
```

Ouvrez les services FreeIPA, DNS et NTP :

```bash
sudo firewall-cmd --permanent --add-service=freeipa-4
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload
```

Verifiez :

```bash
sudo firewall-cmd --list-services
```

Les ports concernes sont :

| Service | Port | Protocole |
|---|---:|---|
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| LDAP | 389 | TCP/UDP |
| LDAPS | 636 | TCP |
| Kerberos | 88 | TCP/UDP |
| Kerberos password change | 464 | TCP/UDP |
| DNS / BIND | 53 | TCP/UDP |
| NTP | 123 | UDP |

---

## 5. Commande lancee

Depuis le serveur :

```bash
sudo ipa-server-install --setup-dns
```

La commande lance une installation interactive de FreeIPA avec DNS integre.

---

## 6. Reponses donnees pendant l'installation

### 6.1 Nom du serveur

```text
Server host name [ldaps01.ironforge.lab]: ldaps01.ironforge.lab
```

Le nom FQDN du serveur est donc :

```text
ldaps01.ironforge.lab
```

### 6.2 Domaine DNS

```text
Please confirm the domain name [ironforge.lab]: ironforge.lab
```

Domaine FreeIPA :

```text
ironforge.lab
```

### 6.3 Realm Kerberos

```text
Please provide a realm name [IRONFORGE.LAB]: IRONFORGE.LAB
```

Realm Kerberos :

```text
IRONFORGE.LAB
```

### 6.4 Mots de passe

Deux mots de passe sont demandes :

```text
Directory Manager password:
IPA admin password:
```

Roles :

- `Directory Manager` : administrateur technique LDAP.
- `admin` IPA : compte d'administration FreeIPA utilise avec l'interface web et la commande `ipa`.

### 6.5 DNS forwarder

FreeIPA a detecte le serveur DNS suivant dans `/etc/resolv.conf` :

```text
192.0.2.1
```

Il a ete ajoute comme forwarder :

```text
DNS forwarders: 192.0.2.1
```

Note observee pendant l'installation :

```text
DNS server 192.0.2.1 does not support DNSSEC
WARNING: DNSSEC validation will be disabled
```

Cela signifie que le forwarder `192.0.2.1` ne fournit pas de signatures DNSSEC pour la requete testee. FreeIPA continue l'installation, mais desactive la validation DNSSEC.

### 6.6 Reverse zone

FreeIPA a cherche les zones reverse manquantes :

```text
Do you want to search for missing reverse zones? [yes]: yes
Do you want to create reverse zone for IP 192.0.2.10 [yes]:
Please specify the reverse zone name [2.0.192.in-addr.arpa.]:
```

Zone reverse utilisee :

```text
2.0.192.in-addr.arpa.
```

### 6.7 NetBIOS

```text
NetBIOS domain name [IRONFORGE]: IRONFORGE
```

Nom NetBIOS :

```text
IRONFORGE
```

### 6.8 Chrony

```text
Do you want to configure chrony with NTP server or pool address? [no]:
```

Aucun serveur NTP specifique n'a ete fourni. FreeIPA a utilise la configuration Chrony par defaut.

---

## 7. Resume confirme par l'installeur

Avant de lancer les modifications, l'installeur a affiche :

```text
The IPA Master Server will be configured with:
Hostname:       ldaps01.ironforge.lab
IP address(es): 192.0.2.10
Domain name:    ironforge.lab
Realm name:     IRONFORGE.LAB

The CA will be configured with:
Subject DN:   CN=Certificate Authority,O=IRONFORGE.LAB
Subject base: O=IRONFORGE.LAB
Chaining:     self-signed

BIND DNS server will be configured to serve IPA domain with:
Forwarders:       192.0.2.1
Forward policy:   only
Reverse zone(s):  2.0.192.in-addr.arpa.
```

Validation donnee :

```text
Continue to configure the system with these values? [no]: yes
```

---

## 8. Services configures

L'installation a configure les composants suivants.

### 8.1 Directory Server LDAP

```text
Configuring directory server (dirsrv)
Create database backend: dc=ironforge,dc=lab
Done configuring directory server (dirsrv).
```

Base DN LDAP :

```text
dc=ironforge,dc=lab
```

### 8.2 Kerberos KDC

```text
Configuring Kerberos KDC (krb5kdc)
Done configuring Kerberos KDC (krb5kdc).
```

Le KDC Kerberos gere le realm :

```text
IRONFORGE.LAB
```

### 8.3 Kadmin

```text
Configuring kadmin
Done configuring kadmin.
```

`kadmin` sert a l'administration Kerberos.

### 8.4 Custodia

```text
Configuring ipa-custodia
Done configuring ipa-custodia.
```

Custodia est utilise par FreeIPA pour la gestion et le partage securise de secrets internes, notamment dans les topologies avec replicas.

### 8.5 Autorite de certification Dogtag

```text
Configuring certificate server (pki-tomcatd). Estimated time: 3 minutes
Done configuring certificate server (pki-tomcatd).
```

La CA est auto-signee :

```text
CN=Certificate Authority,O=IRONFORGE.LAB
```

### 8.6 Interface web Apache

```text
Configuring the web interface (httpd)
Done configuring the web interface (httpd).
```

L'interface web sera disponible ici :

```text
https://ldaps01.ironforge.lab
```

### 8.7 DNS BIND

```text
Configuring DNS (named)
created new /etc/named.conf
created named user config '/etc/named/ipa-ext.conf'
created named user config '/etc/named/ipa-options-ext.conf'
created named user config '/etc/named/ipa-logging-ext.conf'
Done configuring DNS (named).
```

FreeIPA a aussi modifie `resolv.conf` pour que le serveur pointe vers lui-meme :

```text
[13/13]: changing resolv.conf to point to ourselves
```

### 8.8 DNS key synchronization

```text
Configuring DNS key synchronization service (ipa-dnskeysyncd)
Done configuring DNS key synchronization service (ipa-dnskeysyncd).
```

### 8.9 SID generation

```text
Configuring SID generation
Done.
```

Cette etape prepare les identifiants SID, utiles notamment pour certaines integrations et fonctions d'identite avancees.

### 8.10 Client IPA local

L'installation configure aussi le serveur comme client FreeIPA :

```text
This program will set up IPA client.
Client hostname: ldaps01.ironforge.lab
Realm: IRONFORGE.LAB
DNS Domain: ironforge.lab
IPA Server: ldaps01.ironforge.lab
BaseDN: dc=ironforge,dc=lab
```

Fichiers/services configures :

```text
Configured /etc/sssd/sssd.conf
Configured /etc/openldap/ldap.conf
Configured /etc/ssh/ssh_config
Configured /etc/ssh/sshd_config.d/04-ipa.conf
SSSD enabled
```

Fin de configuration client :

```text
The ipa-client-install command was successful
```

---

## 9. Fin de l'installation

Message final :

```text
Setup complete
The ipa-server-install command was successful
```

L'installation de FreeIPA est terminee avec succes.

---

## 10. Controle des ports apres installation

L'installeur rappelle les ports reseau necessaires. Ils doivent deja avoir ete ouverts avant l'installation dans la section `4. Preparer firewalld avant l'installation`.

### 10.1 TCP

| Port | Service |
|---:|---|
| 80 | HTTP |
| 443 | HTTPS |
| 389 | LDAP |
| 636 | LDAPS |
| 88 | Kerberos |
| 464 | Kerberos password change |
| 53 | DNS / BIND |

### 10.2 UDP

| Port | Service |
|---:|---|
| 88 | Kerberos |
| 464 | Kerberos password change |
| 53 | DNS / BIND |
| 123 | NTP |

Controle `firewalld` :

```bash
sudo firewall-cmd --list-services
```

Si les services ne sont pas presents, appliquez la configuration :

```bash
sudo firewall-cmd --permanent --add-service=freeipa-4
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload
```

---

## 11. Etapes apres installation

### 11.1 Obtenir un ticket Kerberos

Commande indiquee par l'installeur :

```bash
kinit admin
```

Verifier le ticket :

```bash
klist
```

Le ticket doit afficher :

```text
IRONFORGE.LAB
```

### 11.2 Tester les services FreeIPA

```bash
sudo ipactl status
```

### 11.3 Tester l'API FreeIPA

```bash
ipa ping
```

### 11.4 Tester DNS

```bash
dig ldaps01.ironforge.lab
dig @127.0.0.1 ldaps01.ironforge.lab
dig @127.0.0.1 -x 192.0.2.10
dig @127.0.0.1 _ldap._tcp.ironforge.lab SRV
dig @127.0.0.1 _kerberos._tcp.ironforge.lab SRV
```

### 11.5 Acceder a l'interface web

Depuis un navigateur :

```text
https://ldaps01.ironforge.lab
```

Connexion :

```text
Utilisateur : admin
Mot de passe : mot de passe IPA admin defini pendant l'installation
```

---

## 12. Certificat CA a sauvegarder

L'installeur indique :

```text
Be sure to back up the CA certificates stored in /root/cacert.p12
These files are required to create replicas. The password for these
files is the Directory Manager password
```

Action recommandee :

```bash
sudo ls -lh /root/cacert.p12
```

Copiez ce fichier dans un emplacement de sauvegarde securise.

Important :

- ce fichier est necessaire pour creer des replicas ;
- son mot de passe est celui du `Directory Manager` ;
- il ne doit pas etre stocke dans un depot Git.

---

## 13. Creer un utilisateur de test

Obtenir un ticket admin :

```bash
kinit admin
```

Creer un utilisateur :

```bash
ipa user-add thomas --first=Thomas --last=Admin --password
```

Verifier :

```bash
ipa user-find thomas
```

---

## 14. Ajouter un client Rocky Linux au domaine

Sur un client, configurez d'abord le DNS pour pointer vers le serveur FreeIPA :

```bash
sudo nmcli connection modify ens192 ipv4.dns 192.0.2.10
sudo nmcli connection up ens192
```

Tester la resolution :

```bash
dig ldaps01.ironforge.lab
dig _ldap._tcp.ironforge.lab SRV
```

Installer le client :

```bash
sudo dnf install -y freeipa-client
```

Alternative si le paquet `freeipa-client` n'est pas trouve :

```bash
sudo dnf install -y ipa-client
```

Enroler le client :

```bash
sudo ipa-client-install \
  --server=ldaps01.ironforge.lab \
  --domain=ironforge.lab \
  --realm=IRONFORGE.LAB \
  --mkhomedir
```

Tester un utilisateur FreeIPA :

```bash
getent passwd thomas
su - thomas
```

---

## 15. Commandes de controle utiles

```bash
hostname -f
ip addr
cat /etc/resolv.conf
timedatectl
chronyc tracking
sudo ipactl status
sudo systemctl status named
sudo systemctl status httpd
sudo systemctl status krb5kdc
sudo systemctl status dirsrv@IRONFORGE-LAB
kinit admin
klist
ipa ping
ipa user-find
ipa dnszone-find
```

---

## 16. Logs importants

| Fichier | Role |
|---|---|
| `/var/log/ipaserver-install.log` | Log complet de l'installation serveur |
| `/var/log/ipaclient-install.log` | Log de la partie client IPA |
| `/var/log/httpd/error_log` | Interface web/API |
| `/var/log/krb5kdc.log` | Kerberos KDC |
| `/var/log/dirsrv/slapd-IRONFORGE-LAB/` | LDAP Directory Server |
| `/var/log/messages` | Logs systeme |

---

## 17. Depannage rapide

### 17.1 DNS forwarder sans DNSSEC

Message observe :

```text
DNS server 192.0.2.1 does not support DNSSEC
WARNING: DNSSEC validation will be disabled
```

Ce n'est pas bloquant pour l'installation. Pour corriger, utilisez un forwarder compatible DNSSEC ou gardez DNSSEC desactive dans ce lab.

### 17.2 Probleme de resolution DNS

Tester :

```bash
dig ldaps01.ironforge.lab
dig @192.0.2.10 ldaps01.ironforge.lab
dig @192.0.2.10 _ldap._tcp.ironforge.lab SRV
```

Verifier `/etc/resolv.conf` :

```bash
cat /etc/resolv.conf
```

Le serveur FreeIPA doit utiliser `127.0.0.1` ou `192.0.2.10` comme DNS.

### 17.3 Probleme Kerberos

Tester :

```bash
kinit admin
klist
```

Si l'erreur indique un decalage horaire :

```bash
chronyc tracking
sudo systemctl restart chronyd
```

### 17.4 Relancer tous les services FreeIPA

```bash
sudo ipactl restart
sudo ipactl status
```

---

## 18. Checklist finale

- [ ] `ipa-server-install --setup-dns` s'est termine avec succes.
- [ ] Le serveur est `ldaps01.ironforge.lab`.
- [ ] L'IP du serveur est `192.0.2.10`.
- [ ] `hostname -f` retourne `ldaps01.ironforge.lab`.
- [ ] `/etc/hosts` contient `192.0.2.10 ldaps01.ironforge.lab ldaps01`.
- [ ] `getent hosts ldaps01.ironforge.lab` fonctionne.
- [ ] Le domaine est `ironforge.lab`.
- [ ] Le realm est `IRONFORGE.LAB`.
- [ ] Le forwarder DNS est `192.0.2.1`.
- [ ] La reverse zone est `2.0.192.in-addr.arpa.`.
- [ ] Les ports FreeIPA sont ouverts.
- [ ] `kinit admin` fonctionne.
- [ ] `ipa ping` fonctionne.
- [ ] `dig @192.0.2.10 ldaps01.ironforge.lab` fonctionne.
- [ ] `https://ldaps01.ironforge.lab` est accessible.
- [ ] `/root/cacert.p12` est sauvegarde hors du serveur.

---

## 19. Transcript d'installation fourni

Extrait principal du transcript :

```text
[root@ldaps01 deploy]# sudo ipa-server-install --setup-dns

The log file for this installation can be found in /var/log/ipaserver-install.log
==============================================================================
This program will set up the IPA Server.
Version 4.13.1

This includes:
  * Configure a stand-alone CA (dogtag) for certificate management
  * Configure the NTP client (chronyd)
  * Create and configure an instance of Directory Server
  * Create and configure a Kerberos Key Distribution Center (KDC)
  * Configure Apache (httpd)
  * Configure DNS (bind)
  * Configure SID generation
  * Configure the KDC to enable PKINIT

Server host name [ldaps01.ironforge.lab]: ldaps01.ironforge.lab
Please confirm the domain name [ironforge.lab]: ironforge.lab
Please provide a realm name [IRONFORGE.LAB]: IRONFORGE.LAB

DNS forwarders: 192.0.2.1
Using reverse zone(s) 2.0.192.in-addr.arpa.
NetBIOS domain name [IRONFORGE]: IRONFORGE

The IPA Master Server will be configured with:
Hostname:       ldaps01.ironforge.lab
IP address(es): 192.0.2.10
Domain name:    ironforge.lab
Realm name:     IRONFORGE.LAB

BIND DNS server will be configured to serve IPA domain with:
Forwarders:       192.0.2.1
Forward policy:   only
Reverse zone(s):  2.0.192.in-addr.arpa.

Continue to configure the system with these values? [no]: yes

Client hostname: ldaps01.ironforge.lab
Realm: IRONFORGE.LAB
DNS Domain: ironforge.lab
IPA Server: ldaps01.ironforge.lab
BaseDN: dc=ironforge,dc=lab

Setup complete
The ipa-server-install command was successful
```
