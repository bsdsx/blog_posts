###### 202105080800 dovecot opensmtpd
# Authentification des utilisateurs

Ce billet fait suite à la [configuration initiale de mon serveur opensmtpd](http://blog.bsdsx.fr/2021/04/2021-04-30_opensmtpd.md.html). Petit rappel des faits:

- un chroot minimal où opensmtpd est installé
- les utilisateurs valides ne sont pas dans **/etc/passwd**
- les courriels sont stockés à partir de **/opt/vmail**

Pour authentifier un utilisateur, il me faut un fichier de type **passwd**:

    user:hash du mot de passe:uid:gid:...

D'après [la documentation](https://man.openbsd.org/OpenBSD-6.8/table.5) d'**opensmtpd** il me faudrait un fichier avec ce format:

    user  hash du mot de passe

D'après [la documentation](https://wiki.dovecot.org/HowTo/SimpleVirtualInstall) de **dovecot** (que je dois installer sinon mes utilisateurs vont avoir du mal à lire leurs courriels) il peut avoir cette forme:

    user:{nom de la fonction de hachage}hash du mot de passe

Je vais vérifier qu'un fichier

    user:hash du mot de passe

est bien compatible avec **opensmtpd** et **dovecot** en commençant par installer ce dernier.

## Installation

En utilisant mon outil maison [sjail](http://fossil.bsdsx.fr/sjail):

    $ sjail /zjails/mail pkg install dovecot
    ...

ou en mode barbare:

    $ doas pkg -c /zjails/mail install --no-scripts --yes --no-repo-update dovecot
    ...
    $ doas pw -R /zjails/mail groupadd dovecot  -g 143
    $ doas pw -R /zjails/mail groupadd dovenull -g 144
    $ doas pw -R /zjails/mail useradd  dovecot  -u 143 -g 143 -c "Dovecot User" -d /var/empty -s /usr/sbin/nologin
    $ doas pw -R /zjails/mail useradd  dovenull -u 144 -g 144 -c "Dovecot login User" -d /var/empty -s /usr/sbin/nologin
    $ cp /lib/libm.so.5 /zjails/mail/lib/
    $ doas chroot /zjails/mail /sbin/ldconfig -elf /usr/local/lib /usr/local/lib/dovecot

Test initial de **dovecot**:

    $ doas chroot /zjails/mail /usr/local/sbin/dovecot --version
    Fatal: open(/dev/null) failed: No such file or directory

Rien de méchant mais il ne faudra pas oublier de démonter **/zjails/mail/dev**:

    $ doas mount -t devfs devfs /zjails/mail/dev
    $ doas devfs -m /zjails/mail/dev rule -s 4 applyset
    $ doas chroot /zjails/mail /usr/local/sbin/dovecot --version
    2.3.13 (89f716dc2)
    $ doas umount /zjails/mail/dev

Je n'évoquerai plus les commandes concernant **/dev**. Pour obtenir la (longue) configuration par défaut:

    $ doas chroot /zjails/mail /usr/local/sbin/dovecot -a -c /dev/null
    ...
    [ plus de 1000 lignes ]

## 0 - Minimal

La configuration de dovecot se base sur plusieurs fichiers situés dans **/usr/local/etc/dovecot/example-config/conf.d**. Pour ma configuration de base je veux:

- ajouter du debug
- activer uniquement le pop3
- limiter les sockets en écoute
- écouter sur un port particulier

    $ cat /zjail/mail/etc/mail/dovecot.conf
    log_path = /dev/stderr
    protocols = pop3
    listen = ::1
    
    service pop3-login {
      inet_listener pop3 {
        port = 1110
      }
    }
    $ doas chroot /zjails/mail /usr/local/sbin/dovecot -F -c /etc/mail/dovecot.conf
    May 01 10:07:37 master: Info: Dovecot v2.3.13 (89f716dc2) starting up for imap

Et dans un autre shell:

    $ netstat -an -f inet6 | grep 1110
    tcp6       0      0 ::1.1110               *.*                    LISTEN

## 1 - Utilisateurs et mots de passe - dovecot

Pour obtenir le hash d'un mot de passe je peux utiliser **smtpctl encrypt** ou **doveadm pw** (ne pas oublier de garder un trace du mot de pase en clair avec **|tee**):

    $ touch /zjails/mail/etc/mail/passwd
    $ openssl rand -base64 12 | tee /zjails/mail/.foo | doas chroot /zjails/mail/ /usr/local/sbin/smtpctl encrypt | sed -e 's/^/foo:/' >> /zjails/mail/etc/mail/passwd
    $ doas chroot /zjails/mail/ /usr/local/bin/doveadm -c /dev/null pw -s SHA512-CRYPT -p `openssl rand -base64 12 | tee /zjails/mail/.bar` | sed -e 's/^{SHA512\-CRYPT}/bar:/' >> /zjails/mail/etc/mail/passwd
    $ cat /zjail/mail/etc/mail/dovecot.conf
    log_path = /dev/stderr
    protocols = pop3
    listen = ::1
    
    service pop3-login {
      inet_listener pop3 {
        port = 1110
      }
    }
    
    passdb {
      driver = passwd-file
      args = scheme=SHA512-CRYPT username_format=%n /etc/mail/passwd
    }
    
    userdb {
      driver = static
      args = uid=vmail gid=vmail home=/opt/vmail/%n
    }

Premier essai:

    $ telnet ::1 1110
    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    +OK Dovecot ready.
    user foo
    +OK
    pass KsFgKsTTyHdVu40W
    +OK Logged in.
    Connection closed by foreign host.

Côté serveur:

    May 06 07:42:00 pop3-login: Info: Login: user=<foo>, method=PLAIN, rip=::1, lip=::1, mpid=2987, secured, session=<qkgqxaLBVHIAAAAAAAAAAAAAAAAAAAAB>
    May 06 07:42:00 pop3(foo)<2987><qkgqxaLBVHIAAAAAAAAAAAAAAAAAAAAB>: Error: mail_location not set and autodetection failed: Mail storage autodetection failed with home=/opt/vmail/foo
    May 06 07:42:00 pop3(foo)<2987><qkgqxaLBVHIAAAAAAAAAAAAAAAAAAAAB>: Info: mail_location not set and autodetection failed: Mail storage autodetection failed with home=/opt/vmail/foo top=0/0, retr=0/0, del=0/0, size=0

Je dois renseigner **mail_location**:

    $ cat /zjail/mail/etc/mail/dovecot.conf
    log_path = /dev/stderr
    protocols = pop3
    listen = ::1
    mail_location = maildir:~/
    
    service pop3-login {
      inet_listener pop3 {
        port = 1110
      }
    }
    
    passdb {
      driver = passwd-file
      args = scheme=SHA512-CRYPT username_format=%n /etc/mail/passwd
    }
    
    userdb {
      driver = static
      args = uid=vmail gid=vmail home=/opt/vmail/%n
    }

Deuxième essai:

    $ telnet ::1 1110
    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    +OK Dovecot ready.
    user foo
    +OK
    pass KsFgKsTTyHdVu40W
    +OK Logged in.
    quit
    +OK Logging out.
    Connection closed by foreign host.

## 2 - Utilisateurs et mots de passe - opensmtpd

Je modifie en conséquence la configuration d'**opensmtpd** pour tester l'authentification:

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
    
    table users file:/etc/mail/passwd
    listen on ::1 port 5877 auth <users>
    
    table domains { "bsdsx.fr", "*.bsdsx.fr" }
    table aliases { abuse = postmaster, postmaster = root, root = foo }
    action "local_mail" maildir "/opt/vmail/%{dest.user}" junk userbase <users> alias <aliases>
    match from any for domain <domains> action "local_mail"

Mais le résultat est sans appel:

    $ doas chroot /zjails/mail /usr/local/sbin/smtpd -f /etc/mail/smtpd.conf -d
    smtpd: invalid listen option: auth requires tls/smtps

Un peu de pki:

    $ openssl genrsa -out /zjails/mail/etc/mail/key.key 4096
    $ openssl req -new -x509 -key /zjails/mail/etc/mail/key.key -subj "/CN=mail/C=FR/ST=IdF/L=Paris/O=Pouet inc/OU=Pouet" -out /zjails/mail/etc/mail/crt.crt -days 365
    $ doas chown root /zjails/mail/etc/mail/key.key /zjails/mail/etc/mail/crt.crt

J'ajoute l'option **-T lookup**:

    $ doas chroot /zjails/mail /usr/local/sbin/smtpd -f /etc/mail/smtpd.conf -d -T lookup

L'authentification se fait avec du base64:

    $ printf 'foo' | openssl base64
    Zm9v

Le truc à ne pas faire:

    $ openssl base64 -in /zjails/mail/.foo
    S3NGZ0tzVFR5SGRWdTQwVwo=

car on se retrouve avec un '\n' en trop:

    $ cat /zjails/mail/.foo | tr -d '\n' | openssl base64
    S3NGZ0tzVFR5SGRWdTQwVw==

Et à l'aide d'**openssl s_client**, le telnet du tls:

    $ openssl s_client -6 -starttls smtp -connect localhost:5877
    [ snip tout plein de trucs ]
    ---
    read R BLOCK
    ehlo localhost
    ...
    auth login
    334 VXNlcm5hbWU6
    Zm9v
    334 UGFzc3dvcmQ6
    S3NGZ0tzVFR5SGRWdTQwVw==
    235 2.0.0 Authentication succeeded
    quit
    221 2.0.0 Bye

Je teste mon utilisateur "bar":

    $ printf 'bar' | openssl base64
    YmFy
    $ cat /zjails/mail/.bar | tr -d '\n' | openssl base64
    aXpNNkRSenFydEw5M3g3Mw==
    $ openssl s_client -6 -starttls smtp -connect localhost:5877
    ...
    auth login
    334 VXNlcm5hbWU6
    YmFy
    334 UGFzc3dvcmQ6
    aXpNNkRSenFydEw5M3g3Mw==
    235 2.0.0 Authentication succeeded
    quit
    221 2.0.0 Bye

Côté serveur:

    f7a2e6aa567530c9 smtp connected address=[::1] host=localhost
    f7a2e6aa567530c9 smtp tls ciphers=TLSv1.3:TLS_AES_256_GCM_SHA384:256
    lookup: lookup "bar" as CREDENTIALS in table static:users -> "$6$.BEf4XECFaoMAlDu$x5RRN3/j58n7ke.LfJEwtF7hwWfiSqaxDM5s9Jc1B32UZiCiNfMKH9VueT6ey7wtO4YaQ/y.56PmJ.sHFokVr."
    f7a2e6aa567530c9 smtp authentication user=bar result=ok
    f7a2e6aa567530c9 smtp disconnected reason=quit

## Pour finir

Ce fichier **passwd** termine la configuration initiale de mon serveur de courrier. Chaque étape de la réception du courriel a été testée et l'authentification des utilisateurs est fonctionnelle. Ce qu'il me reste à faire:

- lmtp + sieve + imaps
- relay + dkim

Mais pour se faire je ne saurais trop recommander la lecture de:

- https://poolp.org/posts/2019-12-23/mettre-en-place-un-serveur-de-mail-avec-opensmtpd-dovecot-et-rspamd
- https://rodolphe.breard.tf/en/article/how-to-deploy-a-personal-email-server/

Commentaires: [https://github.com/bsdsx/blog_posts/issues/8](https://github.com/bsdsx/blog_posts/issues/8)
