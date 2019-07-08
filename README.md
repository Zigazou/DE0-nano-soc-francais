DE0-Nano-SoC et Linux
=====================

Pré-requis
----------

Ce tuto s’adresse à la configuration suivante:

- PC de développement sous Debian (ci-après appelé PC)
- DE0-Nano-SoC (ci-après appelé DE0)

Vous devez disposer des droits root sur le PC.

Configuration du PC
-------------------

### S’assurer que en_US.UTF-8 est installé

Quartus travaille avec la locale en_US.UTF-8 exclusivement.

Suivant votre configuration, il se peut qu’elle ne soit pas installée.

Vous pouvez vous en assurer et corriger la situation avec la commande suivante :

    sudo dpkg-reconfigure locales

### Donner les droits d’accès à l’USB Blaster II

Pour configurer le FPGA (dans le cadre des tests) ou flasher l’EPCS (pour rendre
permanente la configuration du FPGA) depuis le PC via un câble USB, vous devez
indiquer au système les droits d’accès :

Grâce à la commande :
    sudo nano /etc/udev/rules.d/51-altera-usb-blaster.rules

Créer un fichier contenant les lignes suivantes :

    # USB-Blaster
    SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6001", MODE="0666"
    SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6002", MODE="0666"
    SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6003", MODE="0666"

    # USB-Blaster II
    SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6010", MODE="0666"
    SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6810", MODE="0666"

Une fois le fichier créé, assurez-vous que Linux le prenne bien en compte :

    sudo udevadm control --reload

### Installer le support des applications 32 bits pour ModelSim

Quartus Prime tourne normalement en 64 bits mais la version gratuite de ModelSim
(ModelSim ASE ou ModelSim Altera Stater Edition) tourne encore en 32 bits.

L’installation est un peu plus fastidieuse.

Pour installer le support 32 bits nécessaire à ModelSim ASE :

    sudo dpkg --add-architecture i386
    sudo apt update
    sudo apt install lib32z1 lib32stdc++6 lib32gcc1 gcc-multilib g++-multilib
    sudo apt install libx11-6:i386 libxext6:i386 libxft2:i386 libncurses5:i386 libstdc++6:i386
    sudo apt-get build-dep -a i386 libfreetype6

Le reste des opérations considère que ModelSim a été installé dans un
sous-répertoire de ~/intelFPGA (par exemple
`/home/fred/intelFPGA/18.1/modelsim_ase`)

Il faut maintenant compiler une ancienne version de FreeType :

    # Récupération du code source de FreeType 2.4.12
    cd ~/intelFPGA
    wget http://download.savannah.gnu.org/releases/freetype/freetype-2.4.12.tar.bz2
    tar xjvf freetype-2.4.12.tar.bz2
    cd freetype-2.4.12

    # Compilation
    ./configure --build=i686-pc-linux-gnu "CFLAGS=-m32" "CXXFLAGS=-m32" "LDFLAGS=-m32"
    make -j8

### Vérifier la présence de `libpng12`

Les versions les plus récentes de Debian utilisent `libpng16`, mais Quartus
Prime 18.1 utilise `libpng12` et refusera de se lancer si cette bibliothèque
n’est pas présente.

Pour vérifier sa présence :

    ls -l /usr/lib/x86_64/libpng*

Si `libpng12.so.0` n’apparaît pas, il faut l’installer :

    sudo apt install libpng12-0

Si cette commande échoue, il reste la possibilité de télécharger la version
Ubuntu de ce package. Elle devrait s’installer et fonctionner sans problème.

Récupérer le fichier sur la
[page de téléchargement de `libpng12-0_1.2.54-1ubuntu1.1_amd64.deb` pour l'architecture AMD64](https://packages.ubuntu.com/xenial/amd64/libpng12-0/download)

Une fois récupéré, exécuter la commande :

    sudo apt install ./libpng12-0_1.2.54-1ubuntu1.1_amd64.deb

### Désinstaller `brltty`

Pour pouvoir accéder à la console système du DE0 depuis le PC en USB, le port
`/dev/ttyUSB0` doit être libéré de toute application.

Par défaut, Debian installe l’application `brltty` qui surveille ce port et
tente de dialoguer avec un équipement braille, rendant impossible l’utilisation
du port pour une autre utilisation.

Pour désinstaller `brltty` :

    sudo apt remove brltty

### Installer `minicom`

Pour utiliser la console système du DE0, vous devez installer `minicom` sur le
PC :

    sudo apt install minicom

### Installer le compilateur croisé

Le système Linux livré sur la carte MicroSD du DE0 est un système minimaliste :
il utilise BusyBox et TinyLogin qui embarquent des version réduites de la
plupart des commandes Linux.

Il ne contient aucun compilateur.

Pour pouvoir compiler des programmes pour le Linux du DE0, il faut recourir à
un compilateur croisé sur le Linux du PC :

    sudo apt install gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf

Pour pouvoir utiliser ce compilateur croisé, les commandes doivent commencer par 
`arm-linux-gnueabihf-`.

Par exemple, pour utiliser GCC, il faut taper `arm-linux-gnueabihf-gcc`.

### Tester la compilation croisée

Sur le PC, créer le fichier `hello.c` :

    #include <stdio.h>

    int main(int argc, char **argv) {
        printf("Hello, World!\n");

        return 0;
    }

Compiler le fichier :

    arm-linux-gnueabihf-gcc hello.c -o hello

Si votre PC tourne sur processeur x86 ou x86_64 et que vous tentez d’exécuter
`hello`, Linux vous répondra :

    bash: ./hello : impossible d'exécuter le fichier binaire : Erreur de format pour exec()

En utilisant la commande `file hello`, vous devriez voir quelque chose comme
ceci :

    hello: ELF 32-bit LSB shared object, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 3.2.0, BuildID[sha1]=d00ddc43df67088fd82fbcb1d3568e707e43a492, not stripped

## Installer SoC FPGA Embedded Development Suite (EDS)

Pour pouvoir compiler des programmes utilisant les particularités du DE0, il
faut télécharger l’EDS : `SoCEDSSetup-18.1.0.625-linux.run`.

Note : les numéros de version varieront avec le temps.

Rendez-vous sur la page
[Download Center for FPGAs/SoCEDS](http://fpgasoftware.intel.com/soceds).

Sélectionner :

- Select edition = Standard
- Select release = 18.1 (la dernière à l’écriture de ce tuto)
- Operating System = Linux
- Download Method = Direct Download

Télécharger : Intel SoC FPGA Embedded Development Suite Standard Edition.

Lancer `SoCEDSSetup-18.1.0.625-linux.run` :

    chmod 700 SoCEDSSetup-18.1.0.625-linux.run
    ./SoCEDSSetup-18.1.0.625-linux.run

Notes :

- Ne pas installer Quartus Prime Programmer and Tools.
- Ne pas installer DS-5.
- L’installation ne nécessite pas les droits root.
- Les fichiers sont placés dans `~/intelFPGA/18.1/embedded`.
- Les exemples se trouvent dans `~/intelFPGA/18.1/embedded/examples/software`.

### Installer la suite Quartus Prime

Voici le minimum à télécharger pour pouvoir compiler du code pour le FPGA :

- `QuartusLiteSetup-18.1.0.625-linux.run` − la version gratuite
- `cyclonev-18.1.0.625.qdz` − les fichiers spécifiques à la famille de FPGA
  Cyclone V utilisée par le DE0-Nano-SoC.

Note :

- Les numéros de version varieront avec le temps.
- Ce minimum s’élève quand même à **3,4 Go à télécharger**…

Rendez-vous sur la page
[Download Center for FPGAs/Quartus](http://fpgasoftware.intel.com/?edition=lite).

Sélectionner :

- Select edition = Lite
- Select release = 18.1 (la dernière à l’écriture de ce tuto)
- Operating System = Linux
- Download Method = Direct Download
- Onglet = Individual files

Télécharger :

- Quartus Prime Lite Edition (Free) = Quartus Prime (includes Nios II EDS)
- Devices = Cyclone V device support

Lancer `QuartusLiteSetup-18.1.0.625-linux.run` :

    chmod 700 QuartusLiteSetup-18.1.0.625-linux.run
    ./QuartusLiteSetup-18.1.0.625-linux.run

Notes :

- L’installation ne nécessite pas les droits root.
- Les fichiers sont placés dans `~/intelFPGA_lite/18.1/quartus`.
- Pour lancer Quartus Prime : `~/intelFPGA_lite/18.1/quartus/bin/quartus`.

### Patcher ModelSim Altera Starter Edition

Sur le PC, se placer dans le répertoire de ModelSim ASE et donner les droits en
écriture au fichier `vco` :

    cd ~/intelFPGA/18.1/modelsim_ase
    chmod 755 vco

Avec un éditeur de texte, ouvrir le fichier `vco`, repérer la ligne `doatenv=0`
et ajouter les lignes suivantes juste en dessous :

    vco="linux"
    export LD_LIBRARY_PATH=$HOME/intelFPGA/freetype-2.4.12/objs/.libs:$LD_LIBRARY_PATH

Configuration du DE0
--------------------

### Se connecter à la console système du DE0

Relier le PC au DE0 avec le câble USB en le connectant à la prise UART du DE0,
celle juste à côté de la prise Ethernet.

Utiliser `minicom` pour se connecter à la console :

    minicom -D /dev/ttyUSB0

Configurer le port série en effectuant les combinaisons de touches suivantes :

- **CTRL+a, z** − Fait apparaître le menu de commandes
- **o** − Fait apparaître le menu de configuration
- Avec les **touches fléchées** et la touche **Entrée**, sélectionnez
  "Configuration du port série"
- Assurez-vous que l’entrée "Débit/Parité/Bits" indique 115200 8N1
- **f** − Désactivez le "Contrôle de flux matériel"

Si vous ne désactivez pas le contrôle de flux matériel, `minicom` sera en mesure
d’afficher les messages de la console système du DE0 mais vous ne pourrez taper
aucune commande.

Pour vous connecter, indiquez simplement `root` à l’invite `socfpga login`.

Aucun mot de passe n’est nécessaire.

### Ajouter un utilisateur pour les transferts

La pile réseau du DE0 est fonctionnelle mais refuse par défaut l’utilisation du
compte `root` pour toute connexion SSH/SFTP.

Pour faciliter les transferts de fichier, créer un utilisateur simple :

    adduser fred

Note : remplacer `fred` par le nom qu’il vous plaira.

### Repérer l’adresse IP du DE0

Le DE0 obtient son adresse IP par DHCP, celle-ci n’est donc pas fixée une fois
pour toute.

Pour connaître l’adresse IP courante, taper dans la console du DE0 :

    ip addr

### Tester la connexion SFTP

Sur le PC, utiliser Nautilus pour se connecter en SFTP sur le DE0. Faites la
combinaison de touches `CTRL+L` ou cliquer sur l’onglet "Autres emplacements".

Taper l’adresse du serveur suivante :

    sftp://fred@192.168.0.22

Notes :

- `fred` est à remplacer par le nom d’utilisateur que vous avez donné
  précédemment
- `192.168.0.22` est à remplacer avec l’adresse IP courante du DE0
- Nautilus vous demandera de confirmer que vous acceptez la connexion la toute
  première fois

Rendez-vous dans le répertoire `/home/fred` pour pouvoir déposer vos fichiers.

### Tester le programme `hello`

Repérer l’exécutable `hello` compilé sur le PC et le copier dans le répertoire
`/home/fred` sur le DE0.

Dans la console système :

    cd /home/fred
    ./hello

Si tout s’est bien passé, cela devrait afficher (surprise !) :

    Hello, World!

Compiler les exemples EDS
-------------------------

    export SOCEDS_DEST_ROOT=~/intelFPGA/18.1/embedded
    export PATH=$PATH:~/intelFPGA_lite/18.1/quartus/bin

Charger une image binaire depuis le DE0 vers le FPGA
----------------------------------------------------

### Préparation de l’image sur le PC

Dans Quartus Prime, sélectionner le menu **File/Convert Programming Files**…

Configurer les entrées suivantes :

- Programming file type = Raw Binary File (.rbf)
- Mode = 1-bit Passive Serial
- Filename = output_files/image.rbf

Sélectionner l’entrée **SOF Data / Page_0** et cliquer sur **Add File**.

Choisir le fichier *.sof à convertir.

**Important** : cliquer sur **Properties** et cocher la case **Compression**.

Si la compression n’est pas activée, le fichier générera un timeout.

Transférer le fichier `image.rbf` généré sur la carte SD du DE0.

### Chargement de l’image sur le DE0

Avant de booter, s’assurer que les switch MSEL sont sur ON-OFF-ON-OFF-ON-OFF. Si
ce n’est pas le cas, l’image ne pourra pas être dans le FPGA.

Dans la console système, taper la commande :

    dd if=image.rbf of=/dev/fpga0

L’image est alors chargée et immédiatement active.

Erreurs avec QSys
-----------------

Si l’erreur suivante apparaît :

    Error: The auto-constraining script was not able to detect any instance for
    core < hps_sdram_p0 >

Cela veut simplement dire que vous avez oublié d’exécuter le script qui assigne
les broches nécessaires au composant QSys que vous utilisez.

Aller dans le menu Tools → TCL Scripts… et exécuter le script :

    synthesis/submodules/hps_sdram_p0_pin_assignments.tcl

-----

https://forums.intel.com/s/question/0D50P00003yyPoESAU/how-to-include-a-hps-with-block-symbol-file

Ajouter uniquement le fichier *.qip dans les fichiers du projet !

