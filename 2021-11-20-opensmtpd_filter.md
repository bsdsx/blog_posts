###### 202111200800 opensmtpd filter
# Les filtres avec OpenSMTPD

Pour protéger les ressources de mon serveur de courriel j'utilise ces filtres:

    filter check_rdns   phase connect match !rdns   disconnect "550 no rDNS please fix"
    filter check_fcrdns phase connect match !fcrdns disconnect "550 no FCrDNS please fix"

qui sont les pendants DNS des célèbres "pas d'bras, pas d'chocolat" et "T'as des baskets ? Bah ! Tu rentres pôôô" (chercher "sketch Maxime 'le videur'" pour la référence). Pour efficaces qu'ils soient, ces filtres ne peuvent pas grand chose face au côté répétitif des pénibles:

    $ grep --count 'smtp connected address=193.56.29.192 host=<unknown>' /var/log/mail.log
    109

Sachant qu'une seule connexion de ce genre génère 3 lignes de log:

    Nov  2 00:01:33 backup_mx smtpd[60503]: 88166f7f21c951f4 smtp connected address=193.56.29.192 host=<unknown>
    Nov  2 00:01:33 backup_mx smtpd[60503]: 88166f7f21c951f4 smtp failed-command command="" result="550 no rDNS please fix"
    Nov  2 00:01:33 backup_mx smtpd[60503]: 88166f7f21c951f4 smtp disconnected reason=quit

Il est temps de bouter ces mécréants hors du royaume.

## Table et mur de feu 

J'ai déjà évoqué [blacklistd](http://blog.bsdsx.fr/2021/09/2021-09-26_freebsd_freebox.md.html) qui, à mon sens, est la bonne réponse au problème mais OpenSMTPD ne le supporte pas (encore ?). Reste que le principe est le bon: placer l'adresse ip du rabouin dans une table (ipfw, npf, pf), un pool (ipf) ou un set (iptables/ipset/nftables/truc/je m'y perds dans ces linuxeries) et bloquer tout ce qui provient des adresses de cette table. Voyons comment faire avec un filtre.

## Filtre

Un filtre OpenSMTPD se déclare de la façon suivante:

    $ grep ^filter /etc/mail/smtpd.conf
    filter nom_du_filtre proc-exec "/chemin/vers/mon/filtre arguments..."

Sans plus de précision ce filtre sera exécuté (froidement) en tant qu'utilisateur **_smtpd** du groupe **_smtpd**. Si les droits de cet utilisateur ne sont pas suffisants, on peut en spécifier un autre:

    filter nom_du_filtre proc-exec "/chemin/vers/mon/filtre arguments..." user root group wheel

Un filtre s'active depuis la directive **listen**:

    $ grep ^listen /etc/mail/smtpd.conf
    listen on lo1 port 2525 filter nom_du_filtre

Un filtre de doit rien faire d'autre que lire depuis son entrée standard et écrire vers sa sortie standard (aka UNIX way of life). je ne vais pas paraphraser [cet ancien billet](https://poolp.org/posts/2018-11-03/opensmtpd-released-and-upcoming-filters-preview/) mais en gros un bête script shell fait parfaitement l'affaire:

    #!/bin/sh
    set -eu
    while true; do
        if read line; then
        ....
        fi
    done

Le script devra commencer par lire les lignes suivantes:

    config|smtpd-version|6.8.0p2
    config|smtp-session-timeout|300
    config|subsystem|smtp-in
    config|admd|nom.de.la.machine
    config|ready

Cette dernière ligne devra provoquer une réponse sur la sortie standard:

    register|filter|smtp-in|connect
    register|ready

Le filtre s'étant enregistré sur la phase **connect** du subsystem **smtp-in**, les prochaines lignes à lire auront cette forme:

    filter|0.6|1636278882.100084|smtp-in|connect|b5c1acf410d9dbac|bdf7b27e9c960f99|<unknown>|192.168.192.90
    filter|0.6|1636280533.458048|smtp-in|connect|3954af28cda90f23|f4a93480796395c4|foo.example.com|172.30.63.10

que l'on décompose en:

- commande
- numéro de version du protocol de filtre
- timestamp
- subsystem
- phase
- session ($6)
- token ($7)
- reverse dns
- adresse ip

Le filtre devra répondre comme suit:

    filter-result|$6|$7|$resultat

où **$resultat** devra obligatoirement prendre une des valeurs suivantes:

- proceed
- junk
- rewrite|param
- reject|error
- disconnect|error

## Dehors les romanos

     1	    FWCMD="${1:-true}"
     2	    TIMESTAMP=${2:-1} # 0 no timestamp, 1 timestamp (default), 2 timestamp without nano seconds
     3	    
     4	    while true; do
     5	        if read line; then
     6	            case $line in
     7	                'config|ready') break;;
     8	            esac
     9	        fi
    10	    done
    11	    echo 'register|filter|smtp-in|connect'
    12	    echo 'register|ready'
    13	
    14	    IFS='|'
    15	    while true; do
    16	        if read line; then
    17	            set -- $line
    18	            case $1 in
    19	                filter);;
    20	                *) continue;;
    21	            esac
    22	            [ $# -ne 9 ] && echo "filter-result|$6|$7|proceed" && continue
    23	            [ $8 != '<unknown>' ] && echo "filter-result|$6|$7|proceed" && continue
    24	            echo "filter-result|$6|$7|disconnect|550 go away"
    25	            ts=$3
    26	            case $TIMESTAMP in
    27	                0) ts='';;
    28	                2) ts=${3%%.*};;
    29	            esac
    30	            ${FWCMD} $9 $ts || continue
    31	        fi
    32	    done

- 1: la commande à éxécuter
- 2: timestamp à passer à la commande
- 4-12: lecture des lignes de config et enregistrement du filtre. A partir de ce moment chaque ligne lue sur l'entrée standard devra faire l'objet d'une réponse sur la sortie standard
- 14: nouveau séparateur de champs
- 17: découpage de la ligne suivant le séparateur de champs
- 18-21: seules les lignes 'filter' sont à traiter (il n'y a pas de raison que notre script reçoive autre chose mais on ne sait jamais)
- 22: vérification du bon nombre de champs de la ligne (et réponse OK)
- 23: reverse dns valide, réponse OK
- 24: reverse dns invalide, réponse KO
- 25-30: éxécution de la commande

## ipfw car je le vaux bien

J'ai déjà parlé d'[ipfw, de tables et de timestamp](http://blog.bsdsx.fr/2020/01/2020-01-10_ipfw_table.md.html), ma commande sera la suivante:

    /sbin/ipfw table t_pouilleux add

et le **smtpd.conf** devient:

    filter nom_du_filtre proc-exec "/chemin/vers/mon/filtre '/sbin/ipfw table t_pouilleux add' 2" user root group wheel

## jail ou chroot

Dans le cas où le filtre ne peut pas accéder directement à la commande du mur de feu, on peut imaginer un système à base de fifo:

    ...
    fifo=/chemin/vers/fifo
    ...
    echo "filter-result|$6|$7|disconnect|550 go away"
    ...
    [ -p $fifo ] || continue
    echo $9 $ts > $fifo || continue

et un autre script se chargera de la bonne commande:

    $ cat fifo2fw.sh
    #!/bin/sh
    
    set -eu
    
    echo $$ > $1
    
    fifo=${2}
    
    [ -p $fifo ] || mkfifo $fifo
    
    while true; do
        if read ip timestamp < $fifo; then
            /sbin/ipfw table t_pouilleux add $ip $timestamp
        fi
    done

Exemple d'une jail:

    ...
    $fifo = "/chemin/complet/vers/fifo";
    $pid  = "/var/run/fifo2fw.pid";
    ...
    exec.prestart += "/chemin/vers/fifo2fw.sh $pid $fifo &";
    exec.start += "/usr/local/sbin/smtpd -f /etc/mail/smtpd.conf";
    exec.release += "pkill -9 -F $pid";
    exec.release += "rm -f $fifo $pid";

## La documentation

- la [source](https://github.com/OpenSMTPD/OpenSMTPD/blob/master/usr.sbin/smtpd/smtpd-filters.7)
- premier résultat de [man smtpd-filters](https://www.man7.org/linux/man-pages/man7/smtpd-filters.7.html)
- un [module Perl](https://github.com/afresh1/OpenSMTPd-Filter/blob/blead/lib/OpenSMTPd/Filter.pm)
- un [script awk](https://github.com/jirutka/opensmtpd-filter-rewrite-from/blob/master/filter-rewrite-from)

Commentaires: [https://github.com/bsdsx/blog_posts/issues/11](https://github.com/bsdsx/blog_posts/issues/11)
