# Sommaire
- I. [Système](#Système)
	- 1. [SELinux](#selinux)
	- 2. [Login Lock](#login-lock)
	- 3. [Gestion des privilèges](#gestion-des-privileges)
		- a. [sudo](#sudo)
		- b. [Empêcher le login sur root](#empecher-le-login-sur-root)
		- c. [Vérifier les UID suspects](#verifier-les-uid-suspects)
- II. [Réseau](#reseau)
	- 1. [IPv6](#desactiver-les-connexions-en-ipv6)
	- 2. [Fail2Ban](#fail2ban)
	- 3. [Port SSH](#port-ssh)

---

# Système
## SELinux
Security-Enhanced Linux est un LSM qui met en place une politique d'access-control aux éléments d'un système linux
Installer le paquet SELinux si pas déjà fait
```SHELL
  # Debian/Ubuntu
  sudo apt install selinux-utils
  
  # Arch
  git clone https://github.com/archlinuxhardened/selinux.git
  cd selinux
  ./recv_gpg_keys.sh
  ./build_and_install_all.sh
```
Vérifier l'état de SELinux
```SHELL
  getenforce
```
Si la commande renvoie *Disabled*, activer en utilisant cette commande
```SHELL
  setenforce 1
```
## Login lock
Pour éviter les attaques de mots de passe en bruteforce ou par dictionnaire, il est possible de setup un lock d'user au bout de n attemps pour x seconds
Pour ça on va utiliser faillog
```SHELL
  # Tester faillog
  sudo faillog
  
  # Si la commande du dessus renvoie 
  # faillog: Cannot open /var/log/faillog: No such file or directory
  sudo touch /var/log/faillog
  sudo chown root:root /var/log/faillog
  sudo chmod 600 /var/log/faillog
  
  # Set un nombre d'essaie avant de lock
  sudo faillog -m 5
  
  # Set le temps de lock en cas d'échec (en secondes)
  sudo faillog -l 1800
  
  # Si besoin d'unlock un user
  sudo faillog -r -u user
```
## Gestion des privilèges
### sudo
Si besoin d'accéder a de hauts privilèges pour une raison ou une autre toujours passer par sudo
Pour qu'un user puisse utiliser sudo il doit être dans le groupe qui correspond
```SHELL
  # Debian/Ubuntu
  sudo usermod -aG sudo user
  
  # RHEL/Arch
  sudo usermod -aG wheel user
  
  # Passer par visudo pour éditer le fichier des sudoers directement
  sudo visudo
```
### Empêcher le login sur root
Editer le fichier passwd pour empêcher le login root sur cli
```SHELL
  sudo vi /etc/passwd
```
```DIFF
  - root:x:0:0:root:/root:/bin/bash
  + root:x:0:0:root:/root:/usr/sbin/nologin
  ```
Empêcher le login root par ssh
```SHELL
  sudo vi /etc/ssh/sshd_config
```
```DIFF
  - PermitRootLogin yes
  + PermitRootLogin no
  ```
```SHELL
  sudo systemctl restart sshd
```
Restreindre l'accès root via PAM
```SHELL
  sudo vi /etc/pam.d/login
```

```DIFF
  + auth    required       pam_listfile.so onerr=succeed  item=user  sense=deny  file=/etc/ssh/deniedusers
```

```SHELL
  # Créer le fichier des denied users
  sudo vi /etc/ssh/deniedusers
```

```DIFF
  + root
```

```SHELL
  # Mettre à jour les droits du fichier
  sudo chmod 600 /etc/ssh/deniedusers
```
### Vérifier les UID suspects
Chaque user à un ID unique et l'ID 0 correspond à root. Si un utilisateur qui n'est pas root possède l'ID 0 il pourrait tromper le système

```SHELL
  # La commande ne doit renvoyer qu'une ligne correspondant au root
  awk -F: '($3 == "0") {print}' /etc/passwd
```
### Vérifier les users avec un mot de passe vide
Un utilisateur avec un mot de passe vide est une faille évidente. En best practice il faut ne pas en avoir ou lock s'il en existe
```SHELL
  # Lister les user avec mot de passe vide
  sudo awk -F: '($2 == "") {print}' /etc/shadow
  
  # Lock un user
  sudo passwd -l user
  ```
# Réseau
## Désactiver les connexions en IPv6
```SHELL
  sudo vi /etc/sysctl.conf
```
```DIFF
  + net.ipv6.conf.all.disable_ipv6 = 1
  + net.ipv6.conf.default.disable_ipv6 = 1
  + net.ipv6.conf.lo.disable_ipv6 = 1
```
```SHELL
  # Appliquer les modifications
  sudo sysctl -p
```
## Fail2Ban
Fail2Ban permet de ban IP les adresses qui font trop d'échecs de connexion évitant les attaques de bots ou d'individus par brute force
```SHELL
  # Debian/Ubuntu
  sudo apt install fail2ban
  
  # RHEL/Centos
  sudo yum install fail2ban
  
  # Arch
  sudo pacman -S fail2ban
  
  # Generic
  git clone https://github.com/fail2ban/fail2ban.git
  cd fail2ban
  sudo python setup.py install
```
```SHELL
  # Démarrer et activer fail2ban
  # Ici on va utiliser systemd
  sudo systemctl start fail2ban
  sudo systemctl enable fail2ban
  
  # Check si le client fonctionne
  sudo fail2ban-client version
```
```SHELL
  # Pour changer la conf par défaut
  sudo vi /etc/fail2ban/jail.conf
  
  # Pour changer les services à surveiller par fail2ban
  sudo vi /etc/fail2ban/jail.d/defaults-<nom de distribution>.conf
  
  #####################################################################
  
  # Aussi possible d'utiliser le client pour effectuer des actions
  
  # Pour afficher les jails actives
  sudo fail2ban-client status
  
  # Agir sur une prison précise (stop/start/status)
  sudo fail2ban-client stop sshd
```
## Port SSH
Le port SSH par défaut est le 22 et est connu, ce qui fait qu'il est régulièrement tester automatiquement pour essayer introduire des systèmes mal sécurisés. Le changer empêche ces attaques automatiques et rend difficile la tentative de connexion pour quelqu'un qui ne connaîtrait pas le port utilisé
```SHELL
  sudo vi /etc/ssh/sshd_config
  ```
```DIFF
  - #Port 22
  + Port <port entre 1024 et 65536>
```
