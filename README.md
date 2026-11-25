# Partie 2 : utilisateurs, groupes et privilèges

Fait partie du projet [Linux Administration](../../tree/master). Cette branche met en place une politique d'accès sur le serveur `srv-fz` (Ubuntu Server 26.04.1 LTS) : chaque rôle du projet reçoit un compte, et chaque compte ne dispose que des privilèges nécessaires à son rôle (principe du **moindre privilège**).

## Politique d'accès

| Compte | Rôle | Groupe principal | Groupes secondaires | Droits `sudo` | Connexion |
|---|---|---|---|---|---|
| `admin` | Administrateur système | `admin` | `sudo` | Tous | Oui |
| `developer` | Développe l'application | `developer` | `developers` | Aucun | Oui |
| `operator` | Exploite le service de l'application | `operators` | `systemd-journal` | Démarrer, arrêter et redémarrer `mon-app.service` uniquement | Oui |
| `appuser` | Compte qui exécute l'application | `appuser` | — | Aucun | Non (`/usr/sbin/nologin`) |

- `developer` n'a aucun droit d'administration. Ses droits d'écriture sur le code de l'application sont donnés par le groupe `developers` (Partie 3, permissions).
- `operator` peut piloter le service de l'application sans être administrateur. Le groupe `systemd-journal` lui permet de lire les journaux sans `sudo`.
- `appuser` est un compte système : pas de mot de passe, pas de dossier personnel, pas de shell. Personne ne peut ouvrir de session avec ce compte ; il sert uniquement à exécuter l'application.
- Le compte `elhanafi`, créé à l'installation, reste le compte d'administration personnel.

## 1. Créer les groupes et les comptes

```bash
# Groupes métier
sudo groupadd developers
sudo groupadd operators

# Comptes humains (-m : dossier personnel, -s : shell, -c : description)
sudo useradd -m -s /bin/bash -c "Administrateur systeme" admin
sudo useradd -m -s /bin/bash -c "Developpeur" -G developers developer
sudo useradd -m -s /bin/bash -c "Operateur exploitation" -g operators -G systemd-journal operator

# Droits d'administration pour admin uniquement
sudo usermod -aG sudo admin

# Compte de service : compte système, sans dossier personnel ni connexion
sudo useradd --system --no-create-home --home-dir /opt/mon-projet \
  --shell /usr/sbin/nologin -c "Compte de service application" appuser

# Mots de passe des comptes humains (appuser n'en a pas, volontairement)
sudo passwd admin
sudo passwd developer
sudo passwd operator
```

`-g` fixe le **groupe principal** d'un compte, `-G` ajoute des **groupes secondaires**.

## 2. Droits limités pour le groupe `operators`

La règle `sudo` du groupe `operators` est placée dans un fichier à part, `/etc/sudoers.d/operators`, plutôt que dans `/etc/sudoers`.

Une erreur de syntaxe dans un fichier `sudoers` peut bloquer complètement `sudo`. Le fichier est donc écrit dans `/tmp`, vérifié avec `visudo -cf`, puis installé seulement si la vérification réussit :

```bash
echo '%operators ALL=(root) /usr/bin/systemctl start mon-app.service, /usr/bin/systemctl stop mon-app.service, /usr/bin/systemctl restart mon-app.service' > /tmp/operators

visudo -cf /tmp/operators            # doit afficher : parsed OK
sudo install -o root -g root -m 440 /tmp/operators /etc/sudoers.d/operators
sudo visudo -c
rm /tmp/operators
```

Le fichier appartient à `root` avec les droits `440` (lecture seule pour `root` et son groupe), comme l'exige `sudo`.

Le service `mon-app.service` sera créé à la Partie 5. La règle est posée dès maintenant pour que la politique d'accès soit complète.

## 3. Vérification

### Identité et groupes

```bash
id admin; id developer; id operator; id appuser
```

```
uid=1001(admin) gid=1003(admin) groups=1003(admin),27(sudo)
uid=1002(developer) gid=1004(developer) groups=1004(developer),1001(developers)
uid=1003(operator) gid=1002(operators) groups=1002(operators),999(systemd-journal)
uid=999(appuser) gid=984(appuser) groups=984(appuser)
```

`appuser` a un UID inférieur à 1000 : c'est un compte système, et non un compte humain.

### Droits `sudo` de chaque compte

```bash
sudo -l -U admin
sudo -l -U developer
sudo -l -U operator
```

```
User admin may run the following commands on srv-fz:
    (ALL) ALL
    (ALL : ALL) ALL

User developer is not allowed to run sudo on srv-fz.

User operator may run the following commands on srv-fz:
    (root) /usr/bin/systemctl start mon-app.service, /usr/bin/systemctl stop mon-app.service, /usr/bin/systemctl restart mon-app.service
```

### Le compte de service ne peut pas se connecter

```bash
sudo su - appuser
```

```
This account is currently not available.
```

### Refus concret pour le développeur

```bash
su - developer
sudo apt update
exit
```

```
developer@srv-fz:~$ sudo apt update
sudo: I'm sorry developer. I'm afraid I can't do that
```

La commande est refusée : `developer` n'a aucun droit `sudo`. Ce message est propre à sudo-rs ; le `sudo` classique affiche « developer is not in the sudoers file ».

## Bilan

| Compte | Attendu | Obtenu |
|---|---|---|
| `admin` | Administration complète | Conforme |
| `developer` | Aucun droit `sudo` | Conforme |
| `operator` | Uniquement la gestion de `mon-app.service` | Conforme |
| `appuser` | Aucune connexion possible | Conforme |

## Remarques

- **Deux règles pour `admin`.** `sudo -l -U admin` affiche deux lignes. La seconde vient du groupe `sudo`. La première vient d'une ancienne règle toujours présente dans `/etc/sudoers` :

  ```
  %admin ALL=(ALL) ALL
  %sudo  ALL=(ALL:ALL) ALL
  ```

  Le compte `admin` en profite parce que son groupe principal s'appelle `admin`. Ce n'est pas gênant ici, mais cela montre qu'un simple nom de groupe peut donner des privilèges : il faut vérifier les règles existantes avant de nommer un groupe.
- **sudo-rs.** Depuis Ubuntu 25.10, la commande `sudo` est fournie par sudo-rs, une réécriture en Rust. Ses messages diffèrent un peu de ceux du `sudo` classique (par exemple l'invite `[sudo: authenticate] Password:`), et son `visudo -c` ne liste pas les fichiers de `/etc/sudoers.d/`. La règle des opérateurs est bien prise en compte, comme le montre `sudo -l -U operator`.

## Problèmes rencontrés

| Problème | Cause | Solution |
|---|---|---|
| `useradd: group operator exists` à la création du compte `operator` | Ubuntu possède déjà un groupe système `operator`, et `useradd` veut créer un groupe portant le nom du nouvel utilisateur | Donner `operators` comme groupe principal avec `-g operators` |
| `visudo` : `syntax error: expected host name` | Le collage dans l'éditeur a échoué et inséré des caractères parasites | Écrire la règle dans `/tmp` avec `echo`, la vérifier avec `visudo -cf`, puis l'installer avec `install -m 440` |
| `passwd: Authentication token manipulation error` | Les deux saisies du mot de passe ne correspondaient pas | Relancer `sudo passwd admin` |

## Environnement

| Élément | Valeur |
|---|---|
| Serveur | `srv-fz`, Ubuntu Server 26.04.1 LTS, noyau 7.0.0-44, arm64 |
| Virtualisation | UTM (QEMU) sur Mac Apple Silicon |
| Adresse | 192.168.64.50 (statique) |

## Auteur

**El Hanafi Fatima Ezahra**, ENSAM Meknès
