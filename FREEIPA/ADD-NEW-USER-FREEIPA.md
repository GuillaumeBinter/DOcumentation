# Ajouter des utilisateurs FreeIPA avec acces SSH

Cette documentation explique comment creer :

- un utilisateur administrateur ;
- un utilisateur standard ;
- l'acces SSH pour ces utilisateurs ;
- la creation automatique du home directory a la premiere connexion.

Contexte FreeIPA :

```text
Serveur IPA : ldaps01.ironforge.lab
Domaine DNS : ironforge.lab
Realm       : IRONFORGE.LAB
```

---

## 1. Prerequis

Sur le serveur FreeIPA, obtenir un ticket Kerberos administrateur :

```bash
kinit admin
```

Verifier le ticket :

```bash
klist
```

Verifier que la commande `ipa` repond :

```bash
ipa ping
```

---

## 2. Creer un groupe administrateur

Creer un groupe FreeIPA pour les administrateurs Linux :

```bash
ipa group-add admins-linux --desc="Administrateurs Linux"
```

Verifier :

```bash
ipa group-show admins-linux
```

---

## 3. Creer un utilisateur administrateur

Exemple avec l'utilisateur `adminsys`.

```bash
ipa user-add adminsys \
  --first=Admin \
  --last=System \
  --shell=/bin/bash \
  --password
```

FreeIPA demande ensuite de saisir un mot de passe temporaire.

Ajouter l'utilisateur au groupe administrateur :

```bash
ipa group-add-member admins-linux --users=adminsys
```

Verifier :

```bash
ipa user-show adminsys
ipa group-show admins-linux
```

---

## 4. Creer un utilisateur standard

Exemple avec l'utilisateur `user01`.

```bash
ipa user-add user01 \
  --first=User \
  --last=Standard \
  --shell=/bin/bash \
  --password
```

Verifier :

```bash
ipa user-show user01
```

---

## 5. Autoriser les administrateurs a utiliser sudo

Creer une regle sudo pour le groupe `admins-linux` :

```bash
ipa sudorule-add sudo-admins-linux
```

Autoriser le groupe `admins-linux` :

```bash
ipa sudorule-add-user sudo-admins-linux --groups=admins-linux
```

Appliquer la regle a toutes les machines inscrites dans FreeIPA :

```bash
ipa sudorule-mod sudo-admins-linux --hostcat=all
```

Autoriser toutes les commandes :

```bash
ipa sudorule-mod sudo-admins-linux --cmdcat=all
```

Verifier :

```bash
ipa sudorule-show sudo-admins-linux
```

Resultat attendu :

- `adminsys` peut se connecter en SSH ;
- `adminsys` peut utiliser `sudo` ;
- `user01` peut se connecter en SSH ;
- `user01` ne peut pas utiliser `sudo`, sauf si une autre regle lui donne ce droit.

### 5.1 Ajouter les trois administrateurs au groupe FreeIPA `admins`

Pour ajouter directement les trois comptes administrateurs au groupe FreeIPA `admins` :

```bash
kinit admin

ipa group-add-member admins \
  --users=tnguyen \
  --users=dphillipeau \
  --users=aabellan
```

Verifier :

```bash
ipa group-show admins
```

Si ces administrateurs doivent aussi utiliser `sudo` sur les machines Linux, creer une regle sudo pour le groupe `admins` :

```bash
ipa sudorule-add sudo-freeipa-admins
ipa sudorule-add-user sudo-freeipa-admins --groups=admins
ipa sudorule-mod sudo-freeipa-admins --hostcat=all
ipa sudorule-mod sudo-freeipa-admins --cmdcat=all
ipa sudorule-show sudo-freeipa-admins
```

---

## 6. Autoriser l'acces SSH

Par defaut, un utilisateur FreeIPA peut se connecter en SSH sur un client Linux si :

- le client est joint au domaine FreeIPA ;
- SSSD fonctionne ;
- SSH autorise l'authentification PAM ;
- aucune regle HBAC ne bloque l'acces.

Important : pour une configuration propre, n'autorisez pas tout le monde en SSH. Creez une regle HBAC limitee aux utilisateurs ou groupes qui doivent vraiment se connecter.

Verifier les regles HBAC :

```bash
kinit admin
ipa hbacrule-find
```

FreeIPA cree souvent une regle par defaut `allow_all`. Si elle existe et est activee, tous les utilisateurs peuvent se connecter aux hosts inscrits.

Verifier :

```bash
ipa hbacrule-show allow_all
```

### 6.1 Validation rapide en lab

Si le mot de passe est correct mais SSH refuse encore, vous pouvez valider rapidement le lab avec `allow_all` :

```bash
ipa hbacrule-enable allow_all
```

Puis tester :

```bash
ssh tnguyen@ldaps01.ironforge.lab
```

Cette methode est pratique pour confirmer que le probleme vient bien de HBAC. Pour une configuration plus stricte, utilisez la section suivante.

### 6.2 Autoriser seulement les utilisateurs administrateurs

Exemple : autoriser uniquement le groupe FreeIPA `admins` a se connecter en SSH.

Creer la regle HBAC :

```bash
ipa hbacrule-add allow-ssh-admins
```

Autoriser uniquement le groupe `admins` :

```bash
ipa hbacrule-add-user allow-ssh-admins --groups=admins
```

Autoriser l'acces sur tous les hosts inscrits dans FreeIPA :

```bash
ipa hbacrule-mod allow-ssh-admins --hostcat=all
```

Autoriser uniquement le service SSH :

```bash
ipa hbacrule-add-service allow-ssh-admins --hbacsvcs=sshd
```

Verifier :

```bash
ipa hbacrule-show allow-ssh-admins
```

Si vous voulez desactiver la regle ouverte `allow_all` apres validation :

```bash
ipa hbacrule-disable allow_all
```

### 6.3 Autoriser seulement des utilisateurs precis

Si vous ne voulez pas passer par un groupe, vous pouvez autoriser seulement certains comptes :

```bash
ipa hbacrule-add allow-ssh-users
ipa hbacrule-add-user allow-ssh-users \
  --users=tnguyen \
  --users=dphillipeau \
  --users=aabellan
ipa hbacrule-mod allow-ssh-users --hostcat=all
ipa hbacrule-add-service allow-ssh-users --hbacsvcs=sshd
```

Verifier :

```bash
ipa hbacrule-show allow-ssh-users
```

---

## 7. Activer la creation automatique du home directory

Cette partie se fait sur chaque client Rocky/RHEL joint au domaine FreeIPA.

### 7.1 Installer les paquets necessaires

Installer `oddjob` et le module de creation automatique du home :

```bash
sudo dnf install -y oddjob oddjob-mkhomedir
```

Si les paquets sont deja installes, DNF ne les reinstallera pas inutilement.

### 7.2 Activer `with-mkhomedir`

Sur chaque client Rocky/RHEL joint au domaine FreeIPA, appliquer le profil `sssd` avec creation automatique du home :

```bash
sudo authselect select sssd with-mkhomedir
sudo systemctl enable --now oddjobd
sudo systemctl restart sssd sshd
```

Cette commande selectionne le profil `authselect` `sssd` et active `with-mkhomedir`. Le service `oddjobd` cree ensuite le repertoire `/home/<utilisateur>` lors de la premiere connexion.

Verifier :

```bash
authselect current
systemctl status oddjobd
systemctl status sssd
```

---

## 8. Verifier la configuration SSH sur le client

Sur le client Linux :

```bash
sudo grep -E "UsePAM|PasswordAuthentication|GSSAPIAuthentication" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf
```

Valeurs attendues au minimum :

```text
UsePAM yes
```

Si besoin, editer la configuration SSH :

```bash
sudo vi /etc/ssh/sshd_config
```

Verifier ou ajouter :

```text
UsePAM yes
PasswordAuthentication yes
```

Redemarrer SSH :

```bash
sudo systemctl restart sshd
```

Note : `PasswordAuthentication yes` permet le test simple par mot de passe. En production, vous pouvez ensuite renforcer la securite avec des cles SSH, MFA ou HBAC plus strict.

---

## 9. Tester la resolution des utilisateurs sur le client

Sur un client joint a FreeIPA :

```bash
getent passwd adminsys
getent passwd user01
```

Verifier les groupes :

```bash
id adminsys
id user01
```

Verifier que `adminsys` appartient bien a `admins-linux`.

### 9.1 Verifier que le serveur voit un utilisateur via SSSD

Sur `ldaps01`, tester par exemple l'utilisateur `tnguyen` :

```bash
getent passwd tnguyen
id tnguyen
```

Si rien ne remonte, vider le cache SSSD et redemarrer le service :

```bash
sudo sss_cache -E
sudo systemctl restart sssd
getent passwd tnguyen
id tnguyen
```

Si `getent passwd tnguyen` et `id tnguyen` retournent des informations, le serveur voit bien l'utilisateur FreeIPA via SSSD.

---

## 10. Tester la connexion SSH

Depuis une autre machine :

```bash
ssh adminsys@client01.ironforge.lab
```

Au premier login, FreeIPA demande souvent de changer le mot de passe temporaire.

Verifier que le home est cree :

```bash
pwd
ls -ld /home/adminsys
```

Tester sudo pour l'administrateur :

```bash
sudo whoami
```

Resultat attendu :

```text
root
```

Tester l'utilisateur standard :

```bash
ssh user01@client01.ironforge.lab
pwd
ls -ld /home/user01
```

Tester que l'utilisateur standard n'a pas sudo :

```bash
sudo whoami
```

Si aucune regle sudo ne lui donne acces, la commande doit etre refusee.

### 10.1 Tester un utilisateur specifique sur le serveur FreeIPA

Tester l'acces SSH de `tnguyen` vers `ldaps01` :

```bash
ssh tnguyen@ldaps01.ironforge.lab
```

Si le mot de passe est correct mais que SSH refuse encore :

```bash
kinit admin
ipa hbacrule-find
ipa hbacrule-show allow_all
```

Pour valider rapidement le lab :

```bash
ipa hbacrule-enable allow_all
```

Puis retester :

```bash
ssh tnguyen@ldaps01.ironforge.lab
```

Apres validation, preferez une regle HBAC limitee aux utilisateurs specifiques ou au groupe `admins`, comme indique dans la section `6. Autoriser l'acces SSH`.

---

## 11. Ajouter une cle SSH a un utilisateur FreeIPA

Sur le poste de l'utilisateur, afficher la cle publique :

```bash
cat ~/.ssh/id_ed25519.pub
```

Sur le serveur FreeIPA, ajouter la cle a l'utilisateur :

```bash
ipa user-mod adminsys --sshpubkey="ssh-ed25519 AAAA... utilisateur@poste"
```

Verifier :

```bash
ipa user-show adminsys --all | grep -i ssh
```

Tester :

```bash
ssh adminsys@client01.ironforge.lab
```

---

## 12. Commandes rapides

Ajouter trois administrateurs au groupe FreeIPA `admins` :

```bash
kinit admin

ipa group-add-member admins \
  --users=tnguyen \
  --users=dphillipeau \
  --users=aabellan
```

Verifier :

```bash
ipa group-show admins
```

Creation admin :

```bash
kinit admin
ipa group-add admins-linux --desc="Administrateurs Linux"
ipa user-add adminsys --first=Admin --last=System --shell=/bin/bash --password
ipa group-add-member admins-linux --users=adminsys
ipa sudorule-add sudo-admins-linux
ipa sudorule-add-user sudo-admins-linux --groups=admins-linux
ipa sudorule-mod sudo-admins-linux --hostcat=all
ipa sudorule-mod sudo-admins-linux --cmdcat=all
```

Creation utilisateur standard :

```bash
kinit admin
ipa user-add user01 --first=User --last=Standard --shell=/bin/bash --password
```

Activation home automatique sur un client :

```bash
sudo dnf install -y oddjob oddjob-mkhomedir
sudo authselect select sssd with-mkhomedir
sudo systemctl enable --now oddjobd
sudo systemctl restart sssd sshd
```

Tests :

```bash
getent passwd adminsys
getent passwd user01
ssh adminsys@client01.ironforge.lab
ssh user01@client01.ironforge.lab
```

---

## 13. Checklist

- [ ] `kinit admin` fonctionne.
- [ ] Le groupe `admins-linux` existe.
- [ ] L'utilisateur admin `adminsys` existe.
- [ ] L'utilisateur standard `user01` existe.
- [ ] `adminsys` est membre de `admins-linux`.
- [ ] La regle sudo `sudo-admins-linux` existe.
- [ ] Les comptes `tnguyen`, `dphillipeau` et `aabellan` sont membres du groupe `admins` si ce sont les administrateurs FreeIPA.
- [ ] Une regle HBAC limitee existe pour SSH : `allow-ssh-admins` ou `allow-ssh-users`.
- [ ] `allow_all` est seulement utilise pour valider le lab, ou desactive apres test.
- [ ] `getent passwd tnguyen` fonctionne sur `ldaps01`.
- [ ] `id tnguyen` fonctionne sur `ldaps01`.
- [ ] Le client Linux est joint au domaine FreeIPA.
- [ ] `oddjobd` est actif sur le client.
- [ ] `authselect` contient `with-mkhomedir`.
- [ ] `getent passwd adminsys` fonctionne sur le client.
- [ ] `getent passwd user01` fonctionne sur le client.
- [ ] `ssh adminsys@client01.ironforge.lab` fonctionne.
- [ ] `/home/adminsys` est cree automatiquement.
- [ ] `ssh user01@client01.ironforge.lab` fonctionne.
- [ ] `/home/user01` est cree automatiquement.
