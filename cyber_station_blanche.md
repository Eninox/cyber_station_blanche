# <center>Station blanche</center>

# <center> Objectif 1 </center>

    Utilitaires : 
    - VirtualBox 7
    - VM Debian 10 station blanche
    - VM Debian 10 SAS
    - Clé USB avec 2 fichiers sains et 1 fichier infecté
    - Paquet Clamscan (outil de scanning des signatures virales pour des dossiers/fichiers, à l'aide d'un catalogue d'anti-virus)
    - Règle udev (observateur d'événement)
    - GnuPG (chiffrement, déchiffrement)
    - Scripts bash
    - Document écrit avec l'extension markdown (.md) pour faciliter la lecture des commandes

## 1. Installation de la VM  

Dans VirtualBox, importer une machine virtuelle (OVA) ou créer une machine (ISO).  
Depuis l'onget Configuration/Réseau, configurer :   
> 1 carte réseau de type NAT  
> 1 carte réseau de type réseau privé hôte  

Depuis l'onglet  Configuration/Système, définir :  
> 2 CPU  
> 2 Go de RAM

Lancer la machine  
Modifier le fichier de conf réseau, définir une ip "static" ou "dhcp" :
```bash
$nano /etc/network/interfaces
$ifup enp0s8
$systemctl restart networking
``` 

## 2. Installation de clamav  

Installer le paquet clamav :
```bash
$apt update 
$apt install clamscan
``` 
Lecture de la doc de clamscan afin de lancer un premier scan manuel, comprendre le comportement et se projeter sur l'automatisation  
> Piste d'utiliser clamav-daemon pour gérer l'automatisme -> **non suivie**

---

# <center> Objectif 2 </center>

## 1. Découverte mécanisme & script manuel  

Compréhension du mécanisme d'ajout d'un périphérique avec Linux :
```bash
$lsblk
    NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda      8:0    0    8G  0 disk
    ├─sda1   8:1    0    7G  0 part /
    ├─sda2   8:2    0    1K  0 part
    └─sda5   8:5    0  975M  0 part [SWAP]
    sdb      8:16   1  3,8G  0 disk
    └─sdb1   8:17   1  3,8G  0 part
    sr0     11:0    1 1024M  0 rom
``` 
Suite à la commande, visibilité d'une clé USB avec syntaxe "sdb1" dans le dossier /dev  
Montage de cette clé USB et lancement d'un clamscan manuel de son contenu :

```bash
$mount /dev/sdb1 /media/usb
$clamscan /media/usb
``` 
Mécanisme manuel fonctionnel ! Début de création d'un script :
```bash
$mkdir /usr/local/usb
$nano /usr/local/usb/scanusb.sh
```
Screenshot du script **V0** : <center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/script0.jpg) </center> 

Ajout du droit d'exécution au script, 2 façons :
```bash
$chmod +x /usr/local/usb/scanusb.sh
$chmod 751 /usr/local/usb/scanusb.sh
```
* "751" correspond à 
    * lecture (4), écriture (2), exécution (1) pour root (user propriétaire)
    * lecture (4) et exécution (1) pour le groupe (group root)
    * exécution (1) pour les autres (others) -> le script bénéficiera de ce droit

Lancement du script  

## 2. Observation d'événement & script reproductible

En l'état le script est fonctionnel mais trop peu reproductible car sdb1 en dur posera problème pour les prochaines détections de clé, le nom semi-aléatoire est attribué lors du branchement (sdc2, sdb2, ...) mais commence par "sd"  

> Piste de faire tourner le script en cron tab, avec modification du script pour détecter le dossier /dev/sd* -> **Non suivie**

> Piste d'utiliser une règle UDEV au détour d'échanges avec d'autres étudiants puis d'un aiguillage par Quentin -> **Suivie**  

Recherche de l'arborescence udev et création d'une règle pour détecter automatiquement une clé USB :
```bash
$whereis udev
    udev: /usr/lib/udev /etc/udev /usr/share/man/man7/udev.7.gz
$nano /etc/udev/rules.d/scan_usb.rules
    ACTION=="add", KERNEL=="sd[b-z][1-9]", RUN+="/usr/local/usb/scanusb.sh"
```
Screenshot du fichier scan_usb.rules : <center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/udev0rule.png) </center>

* La règle lance le script (run) lorsque le noyau/système (kernel) détecte une action d'ajout de périphérique (add)  
* Syntaxe adaptative avec les crochets pour que le système détecte la clé USB nommée "sd", suivie par un caractère allant de b à z, suivie par un nombre entier de 1 à 9  
* La variable globale ```$name``` prendra la valeur de la clé branchée dans le script

Screenshot du script **V1** : <center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/script1.png) </center> 

* La variable ```$1``` prend la valeur de la clé branchée et est visée par ```$name``` lors de l'exécution du script  
* Ajout de commentaires pour faciliter la lecture de la séquence de tâches
* En l'état les fichiers et logs sont utilisés mais la version finale du script ne retiendra pas ces processus 

Rechargement du service udev pour prise en compte de la règle :
```bash
$udevadm control --reload
```

Passage du journal en mode debug pour analyser la séquence de tâches et valider/adapter le script :
```bash
$udevadm control --log-priority=debug
```
Lancement de la lecture dynamique du journal d'événements 
```bash
$journalctl -f
```

Règle et script fonctionnels ! Toutefois des évolutions sont à prévoir pour répondre aux objectifs suivants.

---

# <center> Objectif 3 </center>

Pour la séparation des fichiers sains et infectés, je fais le choix de les isoler premièrement dans des dossiers temporaires sur la machine, puis de les déplacer sur la clé afin de continuer vers la machine SAS.   
Pour cela j'inclus au script la création des dossiers temporaires : 
```bash
$mkdir /tmp/ok /tmp/quarantaine /tmp/rapports
$touch /tmp/rapports/scan.log
```

Et leur suppression en fin de script :
```bash
$rm -drf /tmp/ok /tmp/quarantaine /tmp/rapports
```

## 1. Isolement des fichiers infectés et création rapport clamav

Utilisation du paramètre ```--move``` de la commande ```clamscan``` pour isoler les fichiers infectés dès la fin du scan.  
Ils sont déplacés premièrement dans le dossier `/tmp/quarantaine` préalablement créé dans le script :
```bash
$clamscan --recursive --move=/tmp/quarantaine /media/usb >> /tmp/rapports/scan.log
```

Puis de retour sur la clé en même temps que les fichiers sains et le rapport clamav grâce à une copie récursive : 
```bash
$cp -r /tmp/quarantaine /media/usb/quarantaine
```
* Mon choix s'est tourné vers une copie récursive `$cp -r` plutôt qu'un déplacement (`$mv`), permettant de bien scinder les actions sur les fichiers/dossiers et facilitant l'analyse durant les tests

## 2. Isolement des fichiers sains 

Après le scan et après avoir déplacé les fichiers injectés et le rapport clamav, les fichiers restants (sains) sont déplacés dans le dossier `/tmp/ok` préalablement créé dans le script :
```bash
$mv /media/usb/* /tmp/ok
```
* Dans la description de l'objectif 4, je vais compléter avec le chiffrement de ce dossier puis son déplacement sur la clé USB.

Screenshot du résultat à ce stade sur la clé USB après un scann clamav : <center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/r%C3%A9sultatClamav1.png) </center> 

Un screenshot final est présenté dans la description de l'objectif 4, reprenant les éléments décrits ci-dessus, complétés par la partie chiffrement. 

---

# <center> Objectif 4 </center>

## 1. Rapport clamav
Dans le script, lors du scan clamav, un fichier de sortie `scan.log` décrit l'analyse des fichiers. Il est d'abord placé dans le dossier `/tmp/rapports` puis déplacé sur la clé USB en même temps que les autres dossiers, permettant d'avoir une cohérence des actions effectuées.  

Screenshot d'un rapport :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/clamavRapport.png) </center> 


## 2. Génération des clés publique/privée

Génération du couple clés publique/privée sur la "station blanche", avec l'interlocuteur WalterWhite :
```bash
$apt update && apt install gnupg2
$gpg --gen-key
    Nom réel : WalterWhite
    Adresse électronique : ww@tp.fr
    Vous avez sélectionné cette identité :
        « WalterWhite <ww@tp.fr> »
```

Screenshot de la génération :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/keyWW.png) </center> 

    Pour la machine SAS, j'ai également créé des clés avec l'interlocuteur GusFring, toutefois elles ne sont finalement pas utilisées pour le TP, je chiffre et déchiffre avec le couple de WalterWhite 

Export clé publique dans un fichier transférable pour l'envoyer à la machine SAS :
```bash
$gpg2 --output /tmp/walter.gpg --export ww@tp.fr
```

Utilisation de `$scp` pour le transfert sécurisé du fichier contenant la clé publique de la station blanche vers la machine distante (SAS) 
```bash
$scp -v /tmp/walter.gpg user@192.168.56.105:/home/user
```
* La machine SAS reçoit la clé publique de WalterWhite

Import de la clé de la station blanche sur le SAS après l'avoir déplacée sur un dossier /tmp :
```bash
$gpg2 --import /tmp/walter.gpg
```
Screenshot :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/keyImportWWsurGF.png) </center> 

## 2. Chiffrement du dossier sain "ok"

A partir des éléments du script déjà fonctionnels, intégration du mécanisme de chiffrement du dossier "ok", contenant les éléments identifiés comme sains après le scann clamav :

```bash
$gpgtar --encrypt --gpg-args='--batch --quiet' --output /tmp/ok.gpg -r WalterWhite ok
```
* `gpgtar` est utilisé pour chiffer/déchiffrer le dossier, il est un complément à `gpg`, qui lui est prévu pour chiffrer/déchiffrer des fichiers 
* `--encrypt` correspond à l'action de chiffrement de l'archive (dossier) 
* `--gpg-args=--batch --quiet` indique des arguments supplémentaires de gpg, ici il s'agit de procéder à l'opération sans intervention humaine (après une première saisie de la passphrase antérieurement)
* `--output` indique le fichier de sortie désiré
* `-r WalterWhite` pour "recipient" correspond à l'interlocuteur ciblé pour le chiffrement, on utilise sa clé publique précédemment importée dans le trousseau de la machine

Génération de la signature du fichier, elle est volontairement détachée pour être interrogeable et vérifiée lors du déchiffrement : 
```bash
$gpg --output /tmp/ok.sig --sign ok.gpg
```
* `--ouput` pour le fichier de sortie désiré
* `--sign` correspond à l'extraction de la signature du fichier chiffré

Screenshot final **V3** du script, incluant toutes les fonctionnalités nécessaires au scann du contenu de la clé, ainsi qu'à l'isolement des fichiers et le chiffrement de ceux qui sont sains :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/scriptFinalStationBlanche.png)</center>  
* Les étapes du script ont été décrites précédemment

Screenshot du déroulé du script complet via le `journalctl` et le mode debug d'udev :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/udev1.png)</center>

Screenshot de vérification du contenu de la clé USB à l'issue du script :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/udev1verifCl%C3%A9.png)</center>

## 3. Déchiffrement du dossier sain "ok" sur le SAS

Prenant en considération les éléments déjà décrits pour chiffrer, j'utilise le même scénario d'événement pour déchiffrer, qui se déroule de la façon suivante :
1. Branchement de la clé USB
2. Activation d'une règle udev `/etc/udev/rules.d/import_usb.rules` qui exécute le script `/usr/local/usb/importusb.sh`
3. Le script effectue successivement :
    * Montage de la clé
    * Attribution des droits pour la réalisation des actions suivantes
    * Copie récursive (facilitant les tests) de l'archive chiffrée et de sa signature dans le dossier local `/partage`
    * Condition qui vérifie que la signature est correcte, puis qui déchiffre grâce au couple de clés de WalterWhite
    * Si la signature n'est pas bonne, information donnée mais pas de déchiffrement
    * Suppression de l'archive chiffrée et de sa signature du dossier local `/partage`, laissant seulement le dossier accessible 

Screenshot du script de déchiffrement :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/scriptFinalSAS.png)</center>

Screenshot du déroulé du script complet via le `journalctl` et le mode debug d'udev :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/udev2.png)</center>

Screenshot de vérification du contenu du dossier `/partage` à l'issue du script :<center>
![alt text](https://github.com/Eninox/cyber_station_blanche/blob/main/media/udev2verifPartage.png)</center>


---