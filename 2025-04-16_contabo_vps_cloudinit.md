###### 202504160800 cloudinit contabo vps

Pour commencer:

- je ne suis pas affilié à Contabo
- je ne suis pas un gourou de python
- je ne suis pas un ardent défenseur de l'informatique nuagique

## Virtual Private Server

Multi-coeurs, gavés de RAM, un **vps** d'aujourd'hui n'a rien à envier à un serveur dédié. Pour peu que l'on sache être économe sur l'espace de stockage, un **vps** peut rendre de multiples services:

- DNS secondaire
- MX secondaire
- web statique
- repo git

et j'en passe. Mes pré-requis concernant un serveur n'ont pas changé d'un iota: FreeBSD et ipv6. FreeBSD parce que je le vaux bien et l'ipv6 parce que l'ipv4 c'est pour les gueux. Il ne doit pas être simple de trouver un founisseur de vps qui ne propose pas de connectivité ipv6 par contre, question BSD, c'est pas gagné. C'est pourquoi j'ai choisi il y a quelques temps déjà [Contabo](https://contabo.com) qui malheureusement ne propose plus l'option FreeBSD. Mais tout n'est pas perdu.

## API

Le principe est simple: commander un vps sans se soucier du système d'exploitation et le réinstaller à l'aide de l'api. La [documentation](https://api.contabo.com/) est claire et une fois mes identifiants récupérés j'ai pû commencer l'écriture de quelques lignes de code Python. On peut trouver sur les nains ternets moultes implémentations autrement plus complètes que mon code mais mes besoins étant simples j'ai préféré "faire à ma sauce" (qui a dit NIH ?).

## Le script

Le script **contabo.py** utilise le module **api_contabo.py** qui permet de:

- lister les images
- gérer les instances
- gérer les secrets

Je suis un adepte du "--run", lancer le script sans aucun argument revient à afficher l'aide:

    $ python contabo.py
    usage: contabo.py [-h] [--debug] [--run] [--config CONFIG] methode [args ...]
    
    Contabo - api
    
    positional arguments:
      methode
      args
    
    options:
      -h, --help       show this help message and exit
      --debug          debug requests
      --run            run this script
      --config CONFIG  .ini config file
    
    Exemple:
    
    contabo.py --run [--config {config.ini}] images [query]
    contabo.py --run [--config {config.ini}] image {imageId}
    
    contabo.py --run [--config {config.ini}] instances
    contabo.py --run [--config {config.ini}] instance {instanceId}
    contabo.py --run  --config {config.ini}  reset_instance {instanceId} user_data.yml [{imageId} [secretId [secretId...]]
    
    contabo.py --run [--config {config.ini}] secrets
    contabo.py --run [--config {config.ini}] secret {secretId}
    contabo.py --run [--config {config.ini}] create_secret {secret} {name}
    contabo.py --run [--config {config.ini}] delete_secret {secretId}
    
    contabo.py --run [--config {config.ini}] token

## Réinstaller un vps

Avant de réinstaller un vps, il faut commencer par s'occuper du "secret": clef ssh plutôt que maux de passe (maux, pas mot). Pour se faire, placer les identifiants dans des variables d'environnement ou dans un fichier ".ini" (on peut se baser sur le fichier **example.ini** fourni) :

    $ cat contabo.ini
    [Api]
    CLIENT_ID     =
    CLIENT_SECRET = 
    API_USERNAME  = 
    API_PASSWORD  = 

L'accès à l'api requiert un token:

    $ python --run --config contabo.ini token
    [ snip token ]

Pour ne pas générer un token à chaque appel, je l'enregistre dans une variable **API_CONTABO_TOKEN**. Je crée une clef ssh:

    $ ssh-keygen -t ed25519 -N '' -f contabo_ed25519

et j'ajoute mon secret (le nom du secret permet de l'identifier plus facilement) :

    $ python contabo.py --run --config contabo.ini create_secret "`cat contabo_ed25519.pub|tr -d '\n'`" nom_de_mon_secret
    <Response [201]>

Je peux maintenant lister mes secrets:

    $ python contabo.py --run --config contabo.ini secrets

et afficher un secret

    $ python contabo.py --run --config contabo.ini secret 123456

Je liste les images disponibles (FreeBSD par défaut) :

    $ python contabo.py --run --config contabo.ini images

La version la plus récente proposée (14.0) n'est pas vraiment la plus à jour. Si cela pose problème il reste l'option (payante) 'custom image' qui propose un espace de 25 Go pour stocker ses images "aux petits oignons".

Il est temps de lister les instances:

    $ python contabo.py --run --config contabo.ini instances

Pour me simplifier la vie, je rajoute une section 'DEFAULT' à mon fichier ".ini"

    [DEFAULT]
    imageId   = 67a63682-6817-41a1-9b87-c8adb4072f27 # freebsd-14.0
    sshKeys   = 123456                               # contabo_ed25519

et une section pour mon instance:

[123456789]
productId = V45 # VPS 1 SSD (no setup)

et je réinstalle mon vps:

    $ python3 reset_instance {instanceId} user_data.yml

## cloudinit

Il n'y a pas très longtemps que j'ai compris l'usage de ce machin nuagique pour enfin y trouver un intéret. Pour les ceussent qui sachent, je ne vais rien vous apprendre. Pour les autres, un exemple de fichier que j'utilise:

    #cloud-config
    ssh_pwauth: false
    user: CHANGE_THIS
    packages:
      - net/rsync
      - security/doas
      - sysutils/tmux
    write_files:
      - path: /etc/rc.conf.d/background_fsck
        content: |
          background_fsck_delay="10"
      - path: /etc/rc.conf.d/hostname
        content: |
          hostname="CHANGE_THIS"
      - path: /etc/rc.conf.d/local_unbound
        content: |
          local_unbound_enable="YES"
      - path: /etc/rc.conf.d/network
        content: |
          ifconfig_vtnet0='inet CHANGE_THIS netmask 255.255.255.0'
          ifconfig_vtnet0_ipv6="inet6 CHANGE_THIS prefixlen 64"
      - path: /etc/rc.conf.d/ntpd
        content: |
          ntpd_enable="YES"
          ntpd_sync_on_start="YES"
          ntpd_flags="--novirtualips"
          ntp_leapfile_sources="https://hpiers.obspm.fr/iers/bul/bulc/ntp/leap-seconds.list"
      - path: /etc/rc.conf.d/routing
        content: |
          defaultrouter=CHANGE_THIS
          ipv6_defaultrouter=fe80::1%vtnet0
      - path: /etc/rc.conf.d/sshd
        content: |
          sshd_enable="YES"
          sshd_flags="-6 -o UseDNS=yes -o UsePAM=no -o AllowGroups=wheel -o PermitRootLogin=without-password"
      - path: /etc/rc.conf.d/syslogd
        content: |
          syslogd_flags="-ss"
      - path: /usr/local/etc/doas.conf
        content: |
          permit nopass setenv { -ENV PS1=$DOAS_PS1 SSH_AUTH_SOCK } :wheel
    runcmd:
      - pw groupmod unbound -m CHANGE_THIS
      - chmod -R g+w /etc/rc.conf.d /etc/unbound/conf.d
      - rm /etc/rc.conf
      - rm -rf /usr/lib/debug /usr/lib32
      - chflags -R 0 /usr/lib32
      - rm -rf /usr/lib32
    power_state:
      mode: reboot

Et parce que l'ipv6 n'est pas un long fleuve tranquille, on peut [avoir besoin de rajouter](https://forums.freebsd.org/threads/cant-get-ipv6-working-reliably-on-contabo-vps-freebsd-13-1.87611/):

      - path: /etc/sysctl.conf.local
        content: |
          net.inet6.icmp6.nd6_onlink_ns_rfc4861=1 # because fe80::1 don't respond to neighbor solicitation
      - path: /etc/rc.conf.d/netwait
        content: |
          netwait_enable="YES"
          netwait_ip="2a02:c207::1:53"
          netwait_timeout="5"

## Conclusion

Je suis très satisfait de mes 3 vps et si la version (pas très fraiche) de FreeBSD peut en rebuter certains, je suis persuadé que les images **BASIC-CLOUDINIT** (un énorme merci à bapt@ pour [nuageinit](https://github.com/freebsd/freebsd-src/blob/main/libexec/rc/rc.d/nuageinit) ) seront bientôt disponibles.

Commentaires: [https://github.com/bsdsx/blog_posts/issues/22](https://github.com/bsdsx/blog_posts/issues/22)
