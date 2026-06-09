# Replication FreeIPA sur Rocky Linux 9

Documentation pour ajouter un serveur replica FreeIPA au domaine :

```text
ironforge.lab
```

Serveur FreeIPA principal :

```text
ldaps01.ironforge.lab
```

Replica FreeIPA :

```text
ldaps02.ironforge.lab
```

Sources de reference :

- Red Hat IdM RHEL 9 : https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/installing_identity_management/index
- FreeIPA Replica Setup : https://www.freeipa.org/page/V4/Replica_Setup

---

## 1. Objectif

Un replica FreeIPA permet d'avoir un deuxieme serveur capable de repondre aux services d'identite :

- LDAP ;
- Kerberos KDC ;
- DNS FreeIPA, si installe avec `--setup-dns` ;
- CA Dogtag, si installe avec `--setup-ca` ;
- interface web FreeIPA ;
- SSSD/client IPA local.

Avec un replica, les clients Linux peuvent continuer a s'authentifier si le serveur principal est indisponible.

---

## 2. Architecture

```text
                         DNS amont
                         192.0.2.1
                              |
                              v
+-------------------+      +-----------------------------+
| Clients Linux     | DNS  | FreeIPA principal           |
| SSSD + Kerberos   +----->| ldaps01.ironforge.lab       |
| DNS: ldaps01/02   | LDAP | 192.0.2.10                  |
+-------------------+----->| DNS + LDAP + Kerberos + CA  |
          |                +-----------------------------+
          |                            ^
          |                            |
          |                  replication LDAP/CA/DNS
          |                            |
          v                            v
                    +-----------------------------+
                    | FreeIPA replica             |
                    | ldaps02.ironforge.lab       |
                    | 192.0.2.11                  |
                    | DNS + LDAP + Kerberos + CA  |
                    +-----------------------------+
```

---

## 3. Informations utilisees

Les IP sont generiques pour la documentation. Remplacez-les par vos vraies adresses privees dans votre lab.

| Element | Valeur |
|---|---|
| Domaine DNS | `ironforge.lab` |
| Realm Kerberos | `IRONFORGE.LAB` |
| Serveur principal | `ldaps01.ironforge.lab` |
| IP principale generique | `192.0.2.10` |
| Replica | `ldaps02.ironforge.lab` |
| IP replica generique | `192.0.2.11` |
| DNS forwarder generique | `192.0.2.1` |
| Zone reverse generique | `2.0.192.in-addr.arpa.` |

---

## 4. Prerequis

Sur le serveur principal `ldaps01` :

- FreeIPA est deja installe ;
- `kinit admin` fonctionne ;
- DNS FreeIPA fonctionne ;
- les ports FreeIPA sont ouverts ;
- `/root/cacert.p12` est sauvegarde.

Sur le futur replica `ldaps02` :

- Rocky Linux 9 est installe ;
- l'adresse IP est fixe ;
- le hostname est `ldaps02.ironforge.lab` ;
- l'heure est synchronisee ;
- le serveur peut joindre `ldaps01.ironforge.lab` ;
- aucun ancien serveur FreeIPA n'est deja installe.

---

## 5. Preparer le hostname et `/etc/hosts` sur `ldaps02`

Configurer le FQDN :

```bash
sudo hostnamectl set-hostname ldaps02.ironforge.lab
```

Verifier :

```bash
hostname
hostname -f
```

Resultat attendu :

```text
ldaps02.ironforge.lab
```

Editer `/etc/hosts` :

```bash
sudo vi /etc/hosts
```

Exemple generique :

```text
127.0.0.1       localhost localhost.localdomain
192.0.2.10      ldaps01.ironforge.lab ldaps01
192.0.2.11      ldaps02.ironforge.lab ldaps02
```

Verifier :

```bash
getent hosts ldaps01.ironforge.lab
getent hosts ldaps02.ironforge.lab
```

---

## 6. Configurer DNS sur `ldaps02`

Avant l'installation, `ldaps02` doit resoudre le domaine FreeIPA. Le plus simple est de faire pointer temporairement son DNS vers `ldaps01`.

Exemple avec l'interface `ens192` :

```bash
sudo nmcli connection modify ens192 ipv4.dns 192.0.2.10
sudo nmcli connection up ens192
```

Verifier :

```bash
cat /etc/resolv.conf
dig ldaps01.ironforge.lab
dig _ldap._tcp.ironforge.lab SRV
dig _kerberos._tcp.ironforge.lab SRV
```

---

## 7. Preparer firewalld sur `ldaps02`

Ouvrir les ports FreeIPA avant l'installation :

```bash
sudo dnf install freeipa-server freeipa-server-dns freeipa-client -y
sudo firewall-cmd --reload
```

Verifier :

```bash
sudo firewall-cmd --list-services
```

Ports utilises :

| Service | Port | Protocole |
|---|---:|---|
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| LDAP | 389 | TCP/UDP |
| LDAPS | 636 | TCP |
| Kerberos | 88 | TCP/UDP |
| Kerberos password change | 464 | TCP/UDP |
| DNS | 53 | TCP/UDP |
| NTP | 123 | UDP |

---

## 8. Synchroniser l'heure

Kerberos exige une heure coherente entre `ldaps01`, `ldaps02` et les clients.

Sur `ldaps02` :

```bash
sudo systemctl enable --now chronyd
chronyc tracking
timedatectl
```

---

## 9. Installer les paquets sur `ldaps02`

Installer les paquets FreeIPA serveur, DNS et client :

```bash
sudo dnf install -y freeipa-server freeipa-server-dns freeipa-client
```

Alternative si Rocky ne trouve pas les paquets `freeipa-*` :

```bash
sudo dnf install -y ipa-server ipa-server-dns ipa-client
```

---

## 10. Autoriser l'installation du replica

Il existe deux methodes courantes.

### 10.1 Methode simple avec le compte `admin`

Sur `ldaps02`, l'installateur demandera un compte autorise, generalement `admin`.

Cette methode est simple en lab.

### 10.2 Methode avec OTP depuis `ldaps01`

Sur `ldaps01`, obtenir un ticket admin :

```bash
kinit admin
```

Ajouter l'hote replica avec un mot de passe temporaire OTP :

```bash
ipa host-add ldaps02.ironforge.lab --password
```

FreeIPA demande de saisir un OTP. Cet OTP sera utilise pendant l'installation du replica.

Verifier :

```bash
ipa host-show ldaps02.ironforge.lab
```

---

## 11. Installer le client IPA sur `ldaps02`

Cette etape est utile si vous voulez enroler `ldaps02` comme client avant promotion en replica.

Avec le compte admin :

```bash
sudo ipa-client-install \
  --server=ldaps01.ironforge.lab \
  --domain=ironforge.lab \
  --realm=IRONFORGE.LAB \
  --mkhomedir
```

Ou avec OTP :

```bash
sudo ipa-client-install \
  --server=ldaps01.ironforge.lab \
  --domain=ironforge.lab \
  --realm=IRONFORGE.LAB \
  --password='OTP_GENERE_SUR_LDAPS01'
```

Verifier :

```bash
ipa ping
getent hosts ldaps01.ironforge.lab
getent hosts ldaps02.ironforge.lab
```

---

## 12. Installer le replica avec DNS et CA

Sur `ldaps02`, lancer :

```bash
sudo ipa-replica-install \
  --setup-dns \
  --setup-ca \
  --forwarder=192.0.2.1
```

Explication :

| Option | Role |
|---|---|
| `--setup-dns` | installe DNS/BIND sur le replica |
| `--setup-ca` | installe une CA replica Dogtag |
| `--forwarder=192.0.2.1` | configure le DNS amont generique |

Pendant l'installation, verifier les valeurs :

```text
Hostname: ldaps02.ironforge.lab
Domain:   ironforge.lab
Realm:    IRONFORGE.LAB
Server:   ldaps01.ironforge.lab
```

Message attendu :

```text
The ipa-replica-install command was successful
```

---

## 13. Variante sans CA replica

Si vous ne voulez pas repliquer la CA sur `ldaps02` :

```bash
sudo ipa-replica-install \
  --setup-dns \
  --forwarder=192.0.2.1
```

Pour un lab ou une production simple, il est souvent preferable d'avoir au moins deux serveurs avec CA pour la continuite du service certificats.

---

## 14. Verifier les services sur `ldaps02`

```bash
sudo ipactl status
```

Tester Kerberos :

```bash
kinit admin
klist
```

Tester l'API :

```bash
ipa ping
```

Tester DNS :

```bash
dig @127.0.0.1 ldaps02.ironforge.lab
dig @127.0.0.1 ldaps01.ironforge.lab
dig @127.0.0.1 _ldap._tcp.ironforge.lab SRV
dig @127.0.0.1 _kerberos._tcp.ironforge.lab SRV
```

---

## 15. Verifier la topologie depuis `ldaps01` ou `ldaps02`

Obtenir un ticket admin :

```bash
kinit admin
```

Lister les serveurs IPA :

```bash
ipa server-find
```

Afficher les roles :

```bash
ipa server-role-find
```

Verifier les segments de replication du domaine :

```bash
ipa topologysegment-find domain
```

Verifier les segments de replication CA :

```bash
ipa topologysegment-find ca
```

Lister les replicas :

```bash
ipa-replica-manage list
```

---

## 16. Verifier la replication des utilisateurs

Sur `ldaps01` :

```bash
kinit admin
ipa user-add testreplica --first=Test --last=Replica --password
```

Sur `ldaps02` :

```bash
kinit admin
ipa user-show testreplica
getent passwd testreplica
```

Si l'utilisateur apparait sur `ldaps02`, la replication LDAP fonctionne.

---

## 17. Configurer les clients avec deux DNS

Sur les clients Linux, configurez les deux serveurs DNS FreeIPA :

```bash
sudo nmcli connection modify ens192 ipv4.dns "192.0.2.10 192.0.2.11"
sudo nmcli connection up ens192
```

Verifier :

```bash
cat /etc/resolv.conf
dig ldaps01.ironforge.lab
dig ldaps02.ironforge.lab
dig _ldap._tcp.ironforge.lab SRV
```

---

## 18. Tester la bascule

Depuis un client Linux joint au domaine :

```bash
kinit admin
ipa ping
ssh adminsys@client01.ironforge.lab
```

Tester DNS explicitement contre chaque serveur :

```bash
dig @192.0.2.10 ldaps01.ironforge.lab
dig @192.0.2.11 ldaps01.ironforge.lab
```

Pour un test de lab, arretez temporairement les services IPA sur `ldaps01` :

```bash
sudo ipactl stop
```

Puis tester depuis un client :

```bash
kinit admin
ipa ping
dig @192.0.2.11 ldaps01.ironforge.lab
```

Redemarrer `ldaps01` :

```bash
sudo ipactl start
```

Important : ne faites pas de test d'arret en production sans fenetre de maintenance.

---

## 19. Ajouter les enregistrements DNS du replica

Si DNS FreeIPA est actif, l'installation ajoute normalement les enregistrements necessaires.

Verifier :

```bash
ipa dnsrecord-find ironforge.lab ldaps02
ipa dnsrecord-find 2.0.192.in-addr.arpa
```

Si besoin, ajouter les records manuellement :

```bash
ipa dnsrecord-add ironforge.lab ldaps02 --a-rec=192.0.2.11
ipa dnsrecord-add 2.0.192.in-addr.arpa 11 --ptr-rec=ldaps02.ironforge.lab.
```

---

## 20. Logs importants

| Fichier | Role |
|---|---|
| `/var/log/ipareplica-install.log` | log principal d'installation replica |
| `/var/log/ipaclient-install.log` | log d'enrolement client |
| `/var/log/dirsrv/slapd-IRONFORGE-LAB/` | LDAP Directory Server |
| `/var/log/httpd/error_log` | interface web/API |
| `/var/log/krb5kdc.log` | Kerberos KDC |
| `/var/log/messages` | logs systeme |

---

## 21. Depannage

### 21.1 DNS ne resout pas le serveur principal

Sur `ldaps02` :

```bash
dig ldaps01.ironforge.lab
dig _ldap._tcp.ironforge.lab SRV
dig _kerberos._tcp.ironforge.lab SRV
```

Verifier que `ldaps02` utilise `ldaps01` comme DNS avant l'installation.

### 21.2 Erreur Kerberos

Verifier l'heure :

```bash
timedatectl
chronyc tracking
```

Redemarrer Chrony :

```bash
sudo systemctl restart chronyd
```

### 21.3 Erreur de credentials pendant l'installation

Verifier le compte admin :

```bash
kinit admin
klist
```

Si vous utilisez OTP, regenerer l'OTP depuis `ldaps01` :

```bash
ipa host-del ldaps02.ironforge.lab
ipa host-add ldaps02.ironforge.lab --password
```

### 21.4 Nettoyer une installation replica echouee en lab

Sur `ldaps02` :

```bash
sudo ipa-server-install --uninstall
sudo ipa-client-install --uninstall
```

Sur `ldaps01`, supprimer l'hote si necessaire :

```bash
kinit admin
ipa host-del ldaps02.ironforge.lab
```

Attention : ces commandes sont destructrices pour l'installation FreeIPA locale.

---

## 22. Checklist finale

- [ ] `ldaps01.ironforge.lab` fonctionne deja.
- [ ] `ldaps02.ironforge.lab` a un FQDN correct.
- [ ] `/etc/hosts` contient `ldaps01` et `ldaps02`.
- [ ] `ldaps02` resout `_ldap._tcp.ironforge.lab`.
- [ ] `firewalld` est configure sur `ldaps02`.
- [ ] `chronyd` est actif.
- [ ] Les paquets FreeIPA sont installes.
- [ ] `ipa-client-install` fonctionne si utilise avant la promotion.
- [ ] `ipa-replica-install --setup-dns --setup-ca` se termine avec succes.
- [ ] `sudo ipactl status` fonctionne sur `ldaps02`.
- [ ] `ipa server-find` affiche `ldaps01` et `ldaps02`.
- [ ] `ipa topologysegment-find domain` affiche un segment de replication.
- [ ] `ipa topologysegment-find ca` affiche un segment CA si `--setup-ca` est utilise.
- [ ] Les clients ont deux DNS : `ldaps01` et `ldaps02`.
- [ ] Un utilisateur cree sur `ldaps01` est visible sur `ldaps02`.

---

## 23. Resume rapide

Sur `ldaps02` :

```bash
sudo hostnamectl set-hostname ldaps02.ironforge.lab
sudo vi /etc/hosts
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=freeipa-4
sudo firewall-cmd --permanent --add-service=dns
sudo firewall-cmd --permanent --add-service=ntp
sudo firewall-cmd --reload
sudo systemctl enable --now chronyd
sudo dnf install -y freeipa-server freeipa-server-dns freeipa-client
sudo ipa-client-install --server=ldaps01.ironforge.lab --domain=ironforge.lab --realm=IRONFORGE.LAB --mkhomedir
sudo ipa-replica-install --setup-dns --setup-ca --forwarder=192.0.2.1
sudo ipactl status
```

Sur `ldaps01` ou `ldaps02` :

```bash
kinit admin
ipa server-find
ipa server-role-find
ipa topologysegment-find domain
ipa topologysegment-find ca
ipa-replica-manage list
```
