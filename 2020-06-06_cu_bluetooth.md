###### 202006060800 cu bluetooth
# Configurer un module HC-05 (série/bluetooth)

Ce genre de module se configure à l'aide d'un adaptateur usb / série et des commandes [AT](https://fr.wikipedia.org/wiki/Commandes_Hayes). Mes premiers essais avec **cu** n'ont pas été concluant:

    $ cu -l /dev/cuaU0 -s 38400
    can't open log file /var/log/aculog.
    Connected
    OK
    OK
    OK
    OK
    ...
    ~.
    [EOT]

La séquence "AT + Enter" ne s'affiche pas et provoque une boucle sans fin.

Il y a une petite subtilité (écrit en très gros dans la documentation) : les commandes doivent se terminer par **\r\n**. J'aurais pû demander à FreeBSD, tty, xterm, tmux, tcsh ou cu de modifier le comportement de la touche **Enter** mais j'ai vite abandonné. Dixit la page de manuel:

     ~$      Pipe the output from a local UNIX process to the remote host.
             The command string sent to the local UNIX system is processed by
             the shell.

J'ai dégainé mon ami **printf**:

    $ cu -l /dev/cuaU0 -s 38400
    can't open log file /var/log/aculog.
    Connected
    ~$Local command? printf "AT\r\n"
    OK
    ~$Local command? printf "AT+VERSION\r\n"
    +VERSION:2.0-20100601
    OK
    ~.
    [EOT]

Pour obtenir la configuration complète du module:

    $ cat at.sh
    #!/bin/sh
    at() {
        printf "AT\r\n"
        sleep 1
    }
    
    get() {
        printf "AT+$1\r\n"
        sleep 1
    }
    
    at
    at
    for i in NAME VERSION CLASS UART PSWD ADDR ROLE IAC INQM CMODE POLAR MPIO IPSCAN SNIFF SENM; do
        get $i
    done
    $ cu -l /dev/cuaU0 -s 38400
    can't open log file /var/log/aculog.
    Connected
    ~$Local command? ./at.sh

Après quelques secondes de patience (il faut laisser le temps au module de répondre, d'où les **sleep 1**):

    OK
    OK
    +VERSION:2.0-20100601
    OK
    +UART:9600,0,0
    OK
    +PSWD:1234
    OK
    +ADDR:14:3:60605
    OK
    +ROLE:0
    OK
    +IAC:9e8b33
    OK
    +INQM:1,1,48
    OK
    +CMOD:1
    OK
    +POLAR:1,1
    OK
    +MPIO:0
    OK
    +IPSCAN:1024,512,1024,512
    OK
    +SNIFF:0,0,0,0
    OK
    +SENM:0,0
    OK
    ~.
    [EOT]

Commentaires: [https://github.com/bsdsx/blog_posts/issues/5](https://github.com/bsdsx/blog_posts/issues/5)
