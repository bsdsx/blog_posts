###### 202109260800 freebsd arm64 blacklistd
# Un FreeBSD anti-rabouins dans la Freebox

Les Freebox Delta permettent de lancer des [machines virtuelles](https://portail.free.fr/nouveautes/freebox-delta-les-vms-machines-virtuelles-disponibles-pour-tous-les-utilisateurs-et-utilisatrices-dune-freebox-delta-ou-delta-s/). Détail qui a son importance: mode routeur obligatoire et au moins un disque dur dans la box. Bonne nouvelle: les images FreeBSD sont parfaitement compatibles.

## Machine virtuelle

On commence par télécharger l'image depuis l'interface Freebox OS (module 'Téléchargement' puis 'Ajout direct'):

L'url de l'image du 2021-Apr-09 à 10:21:57:

    http://ftp.free.fr/mirrors/ftp.freebsd.org/releases/VM-IMAGES/13.0-RELEASE/aarch64/Latest/FreeBSD-13.0-RELEASE-arm64-aarch64.raw.xz

La vitesse de téléchargement est ... rapide. Une fois le fichier téléchargé direction le module 'Explorateur de fichiers' pour créer un répertoire 'VMs' et y extraire le fichier **FreeBSD-13.0-RELEASE-arm64-aarch64.raw.xz** du répertoire 'Téléchargements' (clic droit, Extraire > Extraire ... et sélectionner 'VMs'). On voit que malgré le ssd, le cpu fait ce qu'il peut. Il reste à configurer la VM depuis le module éponyme:

- nom: freebsd13
- cpu: 2
- ram: 957
- Sélectionner une image de disque virtuel existante (utilisez la touche Tabulation si l'entrée n'est pas visible)

Reste à sélectionner le fichier **VMs/FreeBSD-13.0-RELEASE-arm64-aarch64.raw** et à allumer la machine virtuelle. L'inévitable [dmesg](http://download.bsdsx.fr/arm/freebox_freebsd13_dmesg.txt).

## Configuration

Moins d'1Go de ram, 4 Go de disque (non extensible) avec un moins de 1 Go disponible on ne va pas faire les foufous. Un [DNS menteur](http://blog.bsdsx.fr/2020/03/2020-03-26_unbound_ads.md.html) est parfaitement envisageable mais j'avais envie de jouer avec [blacklistd](http://mdoc.su/f/blacklistd.8) avant de le déployer en production. Je commence par configurer **sshd**:

    $ cat /etc/rc.conf.d/sshd
    sshd_enable="YES"
    sshd_flags="-o UseDNS=yes -o UsePAM=no -o AllowGroups=wheel -o PermitRootLogin=without-password -o UseBlacklist=yes -o Port=22 -o Port=2222"

Ce deuxième port en écoute sera utilisé par **blacklistd** que j'active comme suit:

    $ cat /etc/rc.conf.d/blacklistd
    blacklistd_enable="YES"
    blacklistd_flags="-C /etc/rc.conf.d/rc.firewall.blacklistd -c /etc/rc.conf.d/blacklistd.conf -r"

L'option **-C** précise un fichier script pour gérer l'ajout et le retrait des adresses, l'option **-r** recharge les adresses bloquées. La configuration est simple:

    $ grep -v ^# /etc/rc.conf.d/blacklistd.conf
    [local]
    192.168.0.50:2222   stream  *      *      *     1      10d
    
    [remote]

Une connexion sur le port 2222 qui n'aboutit pas bloque l'adresse ipv4 source pour 10 jours. J'utilise [ipfw](http://mdoc.su/f/ipfw.9) et une table **t_pouilleux** pour laisser les rabouins dehors:

    $ doas ipfw show | grep t_pouilleux
    00500  385   22516 deny in dst-ip lookup src-ip t_pouilleux

Le script **/usr/libexec/blacklistd-helper** n'est pas adapté à mon fonctionnement d'où le **-C /etc/rc.conf.d/rc.firewall.blacklistd**:

    $ cat -n /etc/rc.conf.d/rc.firewall.blacklistd
     1  #!/bin/sh
     2
     3  addr="$4"
     4  mask="$5"
     5  case "$4" in
     6  ::ffff:*.*.*.*)
     7          if [ "$5" = 128 ]; then
     8                  mask=32
     9                  addr=${4#::ffff:}
    10          fi;;
    11  esac
    12
    13  cmd="/sbin/ipfw table t_pouilleux"
    14
    15  case "$1" in
    16  add)
    17          $cmd add "$addr/$mask" $6 2>/dev/null && echo OK
    18          ;;
    19  rem)
    20          $cmd delete "$addr/$mask" 2>/dev/null && echo OK
    21          ;;
    22  flush)
    23          echo OK
    24          ;;
    25  *)
    26          echo "$0: Unknown command '$1'" 1>&2
    27          exit 1
    28          ;;
    29  esac

Ma table **t_pouilleux** contient des entrées statiques et je ne veux pas que **blacklistd** vide ma table, c'est pourquoi **flush** ne fait rien.

## Mise en oeuvre

Une redirection de port (22 de la freebox vers le 2222 de la vm) et au bout de 2 jours:

    $ doas blacklistctl dump -b
            address/ma:port id      nfail   last access
        107.189.8.8/32:2222 added: 107.189.8.8/32 2222      1/1     2021/09/25 11:19:44
      198.98.56.248/32:2222 added: 198.98.56.248/32 2222    1/1     2021/09/26 13:05:33
    205.185.113.225/32:2222 added: 205.185.113.225/32 2222  1/1     2021/09/27 11:21:09
     209.141.53.166/32:2222 added: 209.141.53.166/32 2222   1/1     2021/09/25 06:29:11
     209.141.51.168/32:2222 added: 209.141.51.168/32 2222   1/1     2021/09/25 09:19:23
    185.220.102.246/32:2222 added: 185.220.102.246/32 2222  1/1     2021/09/25 14:24:16
     185.31.175.226/32:2222 added: 185.31.175.226/32 2222   1/1     2021/09/26 10:46:23
       45.61.184.15/32:2222 added: 45.61.184.15/32 2222     1/1     2021/09/26 17:17:26
      198.98.51.254/32:2222 added: 198.98.51.254/32 2222    1/1     2021/09/27 12:02:15
      141.98.10.121/32:2222 added: 141.98.10.121/32 2222    1/1     2021/09/25 07:37:32
        45.133.1.31/32:2222 added: 45.133.1.31/32 2222      1/1     2021/09/25 11:41:19
     171.225.185.69/32:2222 added: 171.225.185.69/32 2222   1/1     2021/09/26 21:23:35
     124.131.184.59/32:2222 added: 124.131.184.59/32 2222   1/1     2021/09/27 09:52:18
     116.105.79.159/32:2222 added: 116.105.79.159/32 2222   1/1     2021/09/26 21:23:22
     171.235.82.139/32:2222 added: 171.235.82.139/32 2222   1/1     2021/09/26 21:23:26
      107.189.3.160/32:2222 added: 107.189.3.160/32 2222    1/1     2021/09/27 01:58:18
    205.185.116.145/32:2222 added: 205.185.116.145/32 2222  1/1     2021/09/27 10:43:34
      141.98.10.179/32:2222 added: 141.98.10.179/32 2222    1/1     2021/09/25 06:25:23
      200.230.71.31/32:2222 added: 200.230.71.31/32 2222    1/1     2021/09/25 08:07:14
      107.189.12.48/32:2222 added: 107.189.12.48/32 2222    1/1     2021/09/26 18:21:16
      198.98.58.250/32:2222 added: 198.98.58.250/32 2222    1/1     2021/09/26 18:23:00
       52.157.97.82/32:2222 added: 52.157.97.82/32 2222     6/1     2021/09/27 01:20:45
     209.141.55.247/32:2222 added: 209.141.55.247/32 2222   1/1     2021/09/27 08:40:01
      104.244.75.62/32:2222 added: 104.244.75.62/32 2222    1/1     2021/09/25 07:52:46
     209.141.55.232/32:2222 added: 209.141.55.232/32 2222   1/1     2021/09/25 12:48:36
       80.49.56.236/32:2222 added: 80.49.56.236/32 2222     1/1     2021/09/25 16:33:55
     45.153.160.135/32:2222 added: 45.153.160.135/32 2222   1/1     2021/09/27 11:01:52
     136.144.49.245/32:2222 added: 136.144.49.245/32 2222   6/1     2021/09/25 06:21:57
     205.185.118.82/32:2222 added: 205.185.118.82/32 2222   1/1     2021/09/25 07:30:43
          5.2.69.50/32:2222 added: 5.2.69.50/32 2222        1/1     2021/09/25 08:45:03
      141.98.10.125/32:2222 added: 141.98.10.125/32 2222    1/1     2021/09/25 12:34:07
    213.164.206.127/32:2222 added: 213.164.206.127/32 2222  1/1     2021/09/25 13:05:53
       107.189.1.85/32:2222 added: 107.189.1.85/32 2222     1/1     2021/09/26 08:05:10
        45.133.1.35/32:2222 added: 45.133.1.35/32 2222      1/1     2021/09/26 18:19:47
     185.31.175.247/32:2222 added: 185.31.175.247/32 2222   1/1     2021/09/26 19:34:27
       45.148.123.3/32:2222 added: 45.148.123.3/32 2222     1/1     2021/09/25 08:54:30
      179.43.175.26/32:2222 added: 179.43.175.26/32 2222    1/1     2021/09/25 13:19:35
     45.153.160.137/32:2222 added: 45.153.160.137/32 2222   1/1     2021/09/26 08:34:28
     60.170.247.162/32:2222 added: 60.170.247.162/32 2222   6/1     2021/09/26 19:01:09
     171.251.27.134/32:2222 added: 171.251.27.134/32 2222   1/1     2021/09/26 21:23:31
     107.189.31.248/32:2222 added: 107.189.31.248/32 2222   1/1     2021/09/27 01:35:41

## Pour approfondir

On ne croule pas sous la littérature mais je conseille (en anglais):

- [blacklistd a new approach to blocking attackers](https://gioarc.me/2017/05/29/blacklistd-a-new-approach-to-blocking-attackers/) dont image d'introduction résume parfaitement l'idée que je me fais de fail2ban ou de sshguard
- [make postfix trigger blacklistd on failed authentication](https://imil.net/blog/posts/2020/make-postfix-trigger-blacklistd-on-failed-authentication) pour protéger son Postfix

Depuis sa version 2.0.21 le [serveur http de votre serviteur](https://fossil.bsdsx.fr/mohawk), disponible dans les ports (www/mohawk), peut dialoguer avec **blacklistd**.

Commentaires: [https://github.com/bsdsx/blog_posts/issues/10](https://github.com/bsdsx/blog_posts/issues/10)
