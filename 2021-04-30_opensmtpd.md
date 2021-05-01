###### 202104302000 opensmtpd
# Pour bien comprendre la configuration initiale de son serveur de courriel

Trouver une configuration pour [opensmtpd](https://opensmtpd.org/) prête à l'emploi n'est pas bien difficile mais j'aime bien partir de zéro et comprendre le pourquoi du comment. Pour se faire, je vais simplement installer **opensmtpd** dans un chroot, le lancer en mode debug et tester l'évolution de mon fichier de configuration.

## Chroot et installation

J'utilise mon outil maison [sjail](http://fossil.bsdsx.fr/sjail) pour initialiser mon [chroot](http://download.bsdsx.fr/chroot.txt) et installer **opensmtpd**:

    $ doas zfs create -o quota=10G zcun/zjails/mail
    $ doas chown -R dsx /zjails/mail
    $ sjail /zjails/mail init
    ...
    $ sjail /zjails/mail pkg install opensmtpd
    ...
    $ sjail /zjails/mail pkg info
    ca_root_nss-3.63               Root certificate bundle from the Mozilla Project
    libasr-1.0.4                   Asynchronous DNS resolver library
    libevent-2.1.12                API for executing callback functions on events or timeouts
    opensmtpd-6.8.0,1              Security- and simplicity-focused SMTP server from OpenBSD

Un grand merci à bapt@ pour pkg et son option **-c chroot**. Pour vérifier le chroot:

    $ doas chroot /zjails/mail /bin/sh
    # echo *
    bin dev etc lib libexec sbin tmp usr var
    # ls
    /bin/sh: ls: not found
    # echo /bin/*
    /bin/sh /bin/sleep
    # echo sbin/*
    sbin/ldconfig
    # /usr/local/sbin/smtpd -h
    version: OpenSMTPD 6.8.0p2
    usage: smtpd [-dFhnv] [-D macro=value] [-f file] [-P system] [-T trace]
    # exit
    $ cat /zjails/mail/etc/passwd
    root:*:0:1001:Privileged user:/var/empty:/bin/sh
    nobody:*:65534:65534:Unprivileged user:/nonexistent:/usr/sbin/nologin
    _smtpd:*:257:257:OpenSMTPD:/var/empty:/usr/sbin/nologin
    _smtpq:*:258:258:OpenSMTPD queue user:/var/empty:/usr/sbin/nologin
    vmail:*:2000:2000:OpenSMTPD virtual user:/var/empty:/usr/sbin/nologin
    $ cat /zjails/mail/etc/group
    wheel:*:1001:
    nobody:*:65534:
    _smtpd:*:257:
    _smtpq:*:258:
    vmail:*:2000:

Les binaires disponibles dans mon chroot étant réduits au strict minimum, le script de post-installation ne peut pas créer les utilisateurs/groupes. On peut s'inspirer de ce petit [hook](http://fossil.bsdsx.fr/sjail/artifact/867ea86bb0313f30) ou de la sortie (un peu pénible à lire) de **doas pkg -c /zjails/mail info --raw opensmtpd**. L'utilisateur/groupe **vmail** trouvera son utilité quand j'aborderai les utilisateurs.

L'emplacement théorique de la configuration se situe (depuis le chroot) dans **/usr/local/etc/mail/** mais:

- je suis dans un chroot donc le distingo base/paquet bofbof
- l'arborescence du chroot m'appartient
- je veux pouvoir modifier mes fichiers sans **doas vim** ou **sudoedit**

Je décide donc de placer ma configuration (depuis le chroot) dans le répertoire **/etc/mail**:

    $ mkdir /zjails/mail/etc/mail

## 0/7 - Minimal

Le plus petit fichier de configuration d'**opensmtpd** tient en une seule ligne de quatre mots:

    $ mkdir /zjails/mail/etc/mail
    $ cat /zjails/mail/etc/mail/smtpd.conf
    match from any reject
    $ doas chroot /zjails/mail /usr/local/sbin/smtpd -f /etc/mail/smtpd.conf -n
    configuration OK
    $ doas chroot /zjails/mail /usr/local/sbin/smtpd -f /etc/mail/smtpd.conf -d
    info: OpenSMTPD 6.8.0p2 starting

Cette configuration est vraiment sécurisée car seule la socket unix de contrôle est en écoute. Pour la beauté du geste, voici comment l'utiliser (depuis un autre shell):

    $ ln /zjails/mail/usr/local/sbin/smtpctl /zjails/mail/tmp/sendmail
    $ doas chgrp 258 /zjails/mail/tmp/sendmail
    $ doas chroot /zjails/mail /tmp/sendmail -h
    sendmail: illegal option -- h
    usage: sendmail [-tv] [-f from] [-F name] to ...
    $ date | doas chroot /zjails/mail /tmp/sendmail root
    sendmail: command failed: 550 Invalid recipient: <root@cun.bsdsx.fr>

Et côté serveur:

    bf23567dd3150185 smtp connected address=local host=cun.bsdsx.fr
    bf23567dd3150185 smtp failed-command command="RCPT TO:<root@cun.bsdsx.fr> " result="550 Invalid recipient: <root@cun.bsdsx.fr>"
    bf23567dd3150185 smtp disconnected reason=disconnect

## 1/7 - Listen local

La directive **listen** accepte de nombreux paramètres mais pour les tests le strict minimum fera l'affaire. Je prend soin de ne pas utiliser le port usuel afin de ne pas perturber l'hôte:

    $ cat /zjails/mail/etc/mail/smtpd.conf
    listen on ::1 port 2525
    
    match from any reject
    $ doas chroot /zjails/mail /usr/local/sbin/smtpd -f /etc/mail/smtpd.conf -d -v
    ...
    debug: smtp: listen on [::1] port 2525 flags 0x400 pki "" ca ""
    ...

et dans un autre shell:

    $ netstat -an -p tcp | grep 2525
    tcp6       0      0 ::1.2525               *.*                    LISTEN

Première connexion:

    C telnet ::1 2525
    S Trying ::1...
    S Connected to localhost.
    S Escape character is '^]'.
    S 220 cun.bsdsx.fr ESMTP OpenSMTPD
    C EHLO localhost
    S 250-cun.bsdsx.fr Hello localhost [::1], pleased to meet you
    S 250-8BITMIME
    S 250-ENHANCEDSTATUSCODES
    S 250-SIZE 36700160
    S 250-DSN
    S 250 HELP
    C MAIL FROM:<>
    S 250 2.0.0 Ok
    C RCPT TO:<root>
    S 550 Invalid recipient: <root@cun.bsdsx.fr>
    C quit
    S 221 2.0.0 Bye
    S Connection closed by foreign host.

On voit que par défaut:

- la taille des messages est limitée à 36700160 octets soit 35 Mo
- les options communément admises sont activées (8BITMIME, ENHANCEDSTATUSCODES et DSN)

## 2/7 - Filtrer les connexions

Mon serveur ne doit pas relayer le pourriel, il faut donc filtrer au plus tôt le bon grain de l'ivraie:

    $ cat /zjails/mail/etc/mail/smtpd.conf
    filter check_rdns   phase connect match !rdns   disconnect "550 no rDNS please fix"
    filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS please fix"
    filter check_dns chain { check_rdns, check_fcrdns }
    
    listen on ::1 port 2525 filter check_dns
    
    match from any reject

Il est rare qu'une adresse lien-local possède un reverse dns:

    $ telnet -s fe80::1%lo0 ::1 2525
    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    550 no rDNS please fix
    Connection closed by foreign host.

Côté serveur:

    a2cc7671f3eb8824 smtp connected address=[fe80::1] host=<unknown>
    a2cc7671f3eb8824 smtp failed-command command="" result="550 no rDNS please fix"
    a2cc7671f3eb8824 smtp disconnected reason=quit

Pour tester le **forward-confirmed reverse DNS**, il me faut une adresse ip qui pointe vers un nom de machine qui pointe vers une autre adresse ip. Mon **local-unbound** vient à la rescousse:

    $ cat ipv6doc.conf
    server:
        local-zone: "ipv6.doc." refuse
            local-data: "un.ipv6.doc.          10800 IN AAAA 2001:db8:666::2"
            local-data: "deux.ipv6.doc.        10800 IN AAAA 2001:db8:666::1"
        local-zone: "0.0.0.0.6.6.6.0.8.b.d.0.1.0.0.2.ip6.arpa." refuse
            local-data-ptr: "2001:db8:666::1 un.ipv6.doc"
            local-data-ptr: "2001:db8:666::2 deux.ipv6.doc"
    $ host un.ipv6.doc
    un.ipv6.doc has IPv6 address 2001:db8:666::2
    $ host 2001:db8:666::2
    2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.6.6.6.0.8.b.d.0.1.0.0.2.ip6.arpa domain name pointer deux.ipv6.doc.

J'ai bien un problème: "2001:db8:666::1" résoud vers "un.ipv6.doc" mais "un.ipv6.doc" a pour adresse "2001:db8:666::2".

    $ doas ifconfig lo0 inet6 2001:db8:666::1 prefixlen 128 alias
    $ telnet -s 2001:DB8:666::1 ::1 2525
    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    550 no FCrDNS please fix
    Connection closed by foreign host.
    $ doas ifconfig lo0 inet6 2001:db8:666::1 -alias

Côté serveur:

    5edc91252297e4fa smtp connected address=[2001:db8:666::1] host=un.ipv6.doc
    5edc91252297e4fa smtp failed-command command="" result="550 no FCrDNS please fix"
    5edc91252297e4fa smtp disconnected reason=quit

Côté serveur avec l'option **-T filters**:

    e3bdf844cf6fb1a3 filters session-begin
    e3bdf844cf6fb1a3 filters protocol phase=connect, resume=n, action=proceed, filter=check_rdns, query=[2001:db8:666::1]
    e3bdf844cf6fb1a3 filters protocol phase=connect, resume=y, action=disconnect, filter=check_fcrdns, query=[2001:db8:666::1], response=550 no FCrDNS please fix

Les moins téméraires remplaceront **disconnect** par **junk** pour ne pas perdre de courriel, [certains en parlent mieux que moi](https://poolp.org/posts/2019-12-23/mettre-en-place-un-serveur-de-mail-avec-opensmtpd-dovecot-et-rspamd/#configurer-opensmtpd).

## 3/7 - Accepter du courriel pour mes utilisateurs

Je ne veux pas utiliser les entrées de **/etc/passwd** pour définir mes utilisateurs. Je verrai plus tard comment intégrer le tout avec **dovecot** et [cette petite manipulation](https://doc.dovecot.org/configuration_manual/authentication/passwd_file/#authentication-passwd-file). Les **uid** et **gid** sont ceux de mon utilisateur **vmail**.

    $ cat /zjails/mail/etc/mail/smtpd.conf
    filter check_rdns   phase connect match !rdns   disconnect "550 no rDNS please fix"
    filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS please fix"
    filter check_dns chain { check_rdns, check_fcrdns }
    
    listen on ::1 port 2525 filter check_dns
    
    table alias { abuse = postmaster, postmaster = root, root = foo }
    table users { foo = "2000:2000:/var/empty", bar = "2000:2000:/var/empty" }
    action "local_mail" maildir userbase <users> alias <alias>
    match from any action "local_mail"

Les seuls adresses valides étant abuse, postmaster, root, foo et bar; je teste:

- les utilisateurs du chroot (seul root doit être valide)
- les adresses valides dans plusieurs déclinaisons (avec et sans domaine)
- un utilisateur externe à mon domaine

    $ telnet ::1 2525
    ...
    mail from:<>
    250 2.0.0 Ok
    rcpt to:<nobody>
    550 Invalid recipient: <nobody@cun.bsdsx.fr>
    rcpt to:<_smtpd>
    550 Invalid recipient: <_smtpd@cun.bsdsx.fr>
    rcpt to:<_smtpq>
    550 Invalid recipient: <_smtpq@cun.bsdsx.fr>
    rcpt to:<vmail>
    550 Invalid recipient: <vmail@cun.bsdsx.fr>
    rcpt to:<abuse>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to:<postmaster>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to:<root>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to:<foo>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to:<bar>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to:<bar@cun.bsdsx.fr>
    250 2.1.5 Destination address valid: Recipient ok
    rcp tto:<baz>
    500 5.5.1 Invalid command: Command unrecognized
    rcpt to:<foo@bsdsx.fr>
    550 Invalid recipient: <foo@bsdsx.fr>
    rcpt to:<foo@example.com>
    550 Invalid recipient: <foo@example.com>
    quit
    221 2.0.0 Bye
    Connection closed by foreign host.

Côté serveur avec l'option **-T expand** on pourra voir l'expansion de:

- abuse en postmaster
- postmaster en root
- root en foo

    ...
    expand: lka_expand: address: abuse@cun.bsdsx.fr [depth=0]
    ...
    expand: lka_expand: username: abuse [depth=1, sameuser=0]
    ...
    expand: lka_expand: username: postmaster [depth=2, sameuser=0]
    ...
    expand: lka_expand: username: root [depth=3, sameuser=0]
    ...
    expand: lka_expand: username: foo [depth=4, sameuser=0]
    expand: no .forward for user foo, just deliver

On remarque aussi que le domaine "bsdsx.fr" n'est pas valide: le seul domaine valide est celui de l'hôte soit **@cun.bsdsx.fr**. Ce domaine étant automatiquement ajouté aux adresses sans domaine **foo** est transformé en **foo@cun.bsdsx.fr**.

## 4/7 - Accepter du courriel pour mon domaine

Si j'ai un domaine, ce n'est pas pour avoir des adresses en **@machine.bsdsx.fr**:

    $ cat /zjails/mail/etc/mail/smtpd.conf
    filter check_rdns   phase connect match !rdns   disconnect "550 no rDNS please fix"
    filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS please fix"
    filter check_dns chain { check_rdns, check_fcrdns }
    
    listen on ::1 port 2525 filter check_dns
    
    table alias { abuse = postmaster, postmaster = root, root = foo }
    table users { foo = "2000:2000:/var/empty", bar = "2000:2000:/var/empty" }
    action "local_mail" maildir userbase <users> alias <alias>
    match from any for domain bsdsx.fr action "local_mail"

Je teste les déclinaisons:

- sans domaine
- avec domaine
- avec machine.domaine

où seule la version "avec domaine" doit être valide:

    $ telnet ::1 2525
    ...
    mail from:<>
    250 2.0.0 Ok
    rcpt to:<foo>
    550 Invalid recipient: <foo@cun.bsdsx.fr>
    rcpt to:<foo@bsdsx.fr>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to:<foo@cun.bsdsx.fr>
    550 Invalid recipient: <foo@cun.bsdsx.fr>
    quit
    221 2.0.0 Bye
    Connection closed by foreign host.

## 5/7 - Accepter du courriel des machines de mon domaine

Je veux centraliser les crontab des machines de mon domaine, je dois donc accepter tous les sous-domaines possibles: 

    $ cat /zjails/mail/etc/mail/smtpd.conf
    filter check_rdns   phase connect match !rdns   disconnect "550 no rDNS please fix"
    filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS please fix"
    filter check_dns chain { check_rdns, check_fcrdns }
    
    listen on ::1 port 2525 filter check_dns
    
    table domains { "bsdsx.fr", "*.bsdsx.fr" }
    table aliases { abuse = postmaster, postmaster = root, root = foo }
    table users { foo = "2000:2000:/var/empty", bar = "2000:2000:/var/empty" }
    action "local_mail" maildir userbase <users> alias <aliases>
    match from any for domain <domains> action "local_mail"

Je vérifie qu'il y a bien une comparaison stricte avec les domaines:

    $ telnet ::1 2525
    ...
    mail from:<>
    250 2.0.0 Ok
    rcpt to:<root@bsdsx.fr>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to:<root@foo.bsdsx.fr>
    250 2.1.5 Destination address valid: Recipient ok
    rcpt to: <root@absdsx.fr>
    550 Invalid recipient: <root@absdsx.fr>
    rcpt to:<root@bsdsxafr>
    550 Invalid recipient: <root@bsdsxafr>
    rcpt to:<root@bsdsx.fr.fr>
    550 Invalid recipient: <root@bsdsx.fr.fr>
    quit
    221 2.0.0 Bye
    Connection closed by foreign host.

Par rapport à la configuration précédente l'adresse **foo** étant automatiquement transformée en **foo@cun.bsdsx.fr** elle redevient une adresse valide de mon domaine.

## 6/7 - Refuser du courriel usurpant mon domaine

Il est temps d'être un peu plus strict concernant le **mail from**. Seules les machines de mon domaine peuvent utiliser un **mail from** en relation avec celui-ci. Pour me faciliter ce test je désactive la vérification dns. On pourrait croire que **myfroms** et **domains** font double emploi mais attention: **domains** ne doit pas contenir le caractère "@" et **myfroms** est une liste d'expressions régulières.

    $ cat /zjails/mail/etc/mail/smtpd.conf
    filter check_rdns   phase connect match !rdns   disconnect "550 no rDNS please fix"
    filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS please fix"
    filter check_dns chain { check_rdns, check_fcrdns }
    
    table trusted { 2001:DB8:666::/64 }
    table myfroms { "@bsdsx\.fr$", "@.*\.bsdsx\.fr$" }
    filter check_trusted phase mail-from match src <trusted> bypass
    filter check_myfroms phase mail-from match mail-from regex <myfroms> disconnect "550 invalid mail from"
    filter check_mail_from chain { check_trusted, check_myfroms }
    
    listen on ::1 port 2525 filter check_mail_from
    
    table domains { "bsdsx.fr", "*.bsdsx.fr" }
    table aliases { abuse = postmaster, postmaster = root, root = foo }
    table users { foo = "2000:2000:/var/empty", bar = "2000:2000:/var/empty" }
    action "local_mail" maildir userbase <users> alias <aliases>
    match from any for domain <domains> action "local_mail"

Ce filtre est une déclinaison personnelle de celui fourni par la documentation:

    match !from src <other-relays> mail-from "@example.com" for any reject

Son principal avantage est qu'il coupe la connexion dès la phase **mail from** alors qu'un **match** va qualifier tous les destinataires comme étant invalides.

Une connexion hors domaine:

    $ telnet ::1 2525
    ...
    mail from:<root@bsdsx.fr>
    550 invalid mail from
    Connection closed by foreign host.
    
    $ telnet ::1 2525
    ...
    mail from:<root@cun.bsdsx.fr>
    550 invalid mail from
    Connection closed by foreign host.
    
    $ telnet ::1 2525
    ...
    mail from:<root@example.com>
    250 2.0.0 Ok
    quit

Une connexion du domaine:

    $ doas ifconfig lo0 inet6 2001:db8:666::1 prefixlen 128 alias
    $ telnet -s 2001:DB8:666::1 ::1 2525
    ...
    mail from:<root@bsdsx.fr>
    250 2.0.0 Ok
    quit
    $ telnet -s 2001:DB8:666::1 ::1 2525
    ...
    mail from:<root@cun.bsdsx.fr>
    250 2.0.0 Ok
    quit
    $ doas ifconfig lo0 inet6 2001:db8:666::1 -alias

Il serait tentant d'exiger d'un **mail-from** qu'il corresponde à une adresse mais [il semblerait que cela soit une FBI](https://serverfault.com/questions/151955/why-an-empty-mail-from-address-can-sent-out-email) (Fausse Bonne Idée).

## 7/7 - Délivrer le courriel de mon domaine

Le **$HOME** de mes utilisateurs étant invalide, les courriels seront stockés dans **/opt/vmail**. J'en profite pour réactiver le filtre **check_dns** en mode **junk**:

    $ mkdir -p /zjails/mail/opt/vmail && doas chown -R 2000 /zjails/mail/opt/vmail
    $ cat /zjails/mail/etc/mail/smtpd.conf
    filter check_rdns   phase connect match !rdns   junk
    filter check_fcrdns phase connect match !fcrdns junk
    filter check_dns chain { check_rdns, check_fcrdns }
    
    table trusted { 2001:DB8:666::/64 }
    table myfroms { "@bsdsx\.fr$", "@.*\.bsdsx\.fr$" }
    filter check_trusted phase mail-from match src <trusted> bypass
    filter check_myfroms phase mail-from match mail-from regex <myfroms> disconnect "550 invalid mail from"
    filter check_mail_from chain { check_trusted, check_myfroms }
    
    filter filters chain { check_rdns, check_fcrdns, check_trusted, check_myfroms }
    listen on ::1 port 2525 filter filters
    
    table domains { "bsdsx.fr", "*.bsdsx.fr" }
    table aliases { abuse = postmaster, postmaster = root, root = foo }
    table users { foo = "2000:2000:/var/empty", bar = "2000:2000:/var/empty" }
    action "local_mail" maildir "/opt/vmail%{rcpt}" junk userbase <users> alias <aliases>
    match from any for domain <domains> action "local_mail"

Mon premier pourriel (l'adresse lien-local n'ayant pas de reverse dns):

    $ telnet -s fe80::1%lo0 ::1 2525
    ...
    mail from:<>
    250 2.0.0 Ok
    rcpt to:<foo>
    250 2.1.5 Destination address valid: Recipient ok
    data
    354 Enter mail, end with "." on a line by itself
    Subject: deliver test but junk
    
    It works !
    Check header for junk.
    
    @+
    .
    250 2.0.0 09e5c671 Message accepted for delivery
    quit
    221 2.0.0 Bye
    Connection closed by foreign host.

Côté serveur:

    580c6c127534bb9f smtp connected address=[fe80::1] host=<unknown>
    580c6c127534bb9f smtp message msgid=09e5c671 size=499 nrcpt=1 proto=ESMTP
    580c6c127534bb9f smtp envelope evpid=09e5c6717c6aa838 from=<> to=<foo@cun.bsdsx.fr>
    580c6c14a21d7761 mda delivery evpid=09e5c6717c6aa838 from=<> to=<foo@cun.bsdsx.fr> rcpt=<foo@cun.bsdsx.fr> user=foo delay=54s result=Ok stat=Delivered
    580c6c127534bb9f smtp disconnected reason=quit

L'arborescence générée:

    $ doas tree -a /zjails/mail/opt/vmail/foo
    /zjails/mail/opt/vmail/foo
    |-- .Junk
    |   |-- cur
    |   |-- new
    |   |   `-- 1619801197.b213c4db.cun.bsdsx.fr
    |   `-- tmp
    |-- cur
    |-- new
    `-- tmp
    
    7 directories, 1 file

Et le fameux pourriel:

    $ doas cat /zjails/mail/opt/vmail/foo/.Junk/new/1619801197.b213c4db.cun.bsdsx.fr
    Delivered-To: foo@cun.bsdsx.fr
    X-Spam: Yes
    Received: from cun (<unknown> [fe80::1])
            by cun.bsdsx.fr (OpenSMTPD) with ESMTP id 09e5c671
            for <foo@cun.bsdsx.fr>;
            Fri, 30 Apr 2021 18:45:46 +0200 (CEST)
    Subject: deliver test but junk
    Date: Fri, 30 Apr 2021 18:45:46 +0200 (CEST)
    Message-ID: <580c6c13692a128a@cun.bsdsx.fr>
    
    It works !
    Check header for junk.
    
    @+

## Pour finir

Ma configuration initiale:

- filtre (ou qualifie) les connexions
- filtre (ou qualifie) le **mail from**
- accepte tout mon domaine
- délivre (et qualifie) des courriels

La prochaine étape abordera l'authentificaton des utilisateurs afin qu'ils puissent lire et envoyer du courriel.

Commentaires: [https://github.com/bsdsx/blog_posts/issues/7](https://github.com/bsdsx/blog_posts/issues/7)
