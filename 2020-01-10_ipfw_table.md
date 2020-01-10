###### 202001161600 ipfw shell
# ipfw: modern FreeBSD, les tables

Les tables permettent de faire des listes de différents types:

- adresse (ipv6/ipv4 et masque de réseau optionnel)
- nombre (port, uid/gid, jail id)
- nom d'interface
- flow

C'est l'outil idéal pour bloquer ce que je nomme pudiquement **les pouilleux**:

    $fwcmd table t_pouilleux create type addr
    ...
    $fwcmd add deny in lookup src-ip t_pouilleux

Pour ajouter/retirer une adresse, rien de plus simple:

    # ipfw table t_pouilleux add 8.8.8.8
    added: 8.8.8.8/32 0
    # ipfw table t_pouilleux delete 8.8.8.8
    deleted: 8.8.8.8/32 0

On peut aussi utiliser des noms d'hôtes qui seront résolus lors de l'ajout. Mais attention, si cette résolution retourne plusieurs adresses (ipv4 et/ou ipv6) elles ne seront pas toutes intégrées. Je pars donc du principe que ce n'est pas une bonne idée et qu'il faut utiliser uniquement des adresses.

## La purge

Par défaut, chaque entrée possède une valeur (de type **legacy**) qui vaut **0**. On peut donc ajouter une information temporelle dans l'objectif de purger régulièrement la table:

    # ipfw table t_pouilleux flush
    # ipfw table t_pouilleux add 8.8.8.8 `date '+%s'`
    added: 8.8.8.8/32 1577471400
    # ipfw table t_pouilleux add 8.8.4.4 1577385000
    added: 8.8.4.4/32 1577385000

Le script suivant supprime les entrées d'une table dont la valeur est inférieure à un nombre de secondes:

     1  #!/bin/sh
     2  
     3  set -eu
     4  
     5  nb_days=15
     6  table=t_pouilleux
     7  dbg=
     8  quiet=
     9  
    10  usage() {
    11  echo "Usage: ${0##*/} [-n nb_days] [-t table] [-d] [-q] [-h]
    12          -n nb_days default $nb_days
    13          -t table   default $table
    14          -d         debug mode
    15          -q         quiet mode
    16          -h         this message
    17  "
    18  }
    19  
    20  show_error() { echo $msg_error'. Exit'; }
    21  
    22  args=$(getopt dqhn:t: $*)
    23  
    24  if [ $? -ne 0 ]; then
    25          usage
    26          exit 2
    27  fi
    28  set -- $args
    29  while :; do
    30          case "$1" in
    31          -d) dbg=echo; shift;;
    32          -q) quiet=$1; shift;;
    33          -n) nb_days=$2; shift; shift;;
    34          -t) table=$2; shift; shift;;
    35          -h) usage; exit 0;;
    36          --) shift; break;;
    37          esac
    38  done
    39  
    40  trap show_error 0
    41  msg_error="table '$table': not found"
    42  ipfw table all list | grep --quiet "table($table)"
    43  msg_error="table '$table': not type addr"
    44  ipfw table $table info | grep --quiet 'type: addr'
    45  trap "" 0
    46  
    47  max_seconds=$(($(date '+%s') - 86400 * $nb_days))
    48  
    49  ipfw table $table list | while read addr seconds; do
    50          [ ${seconds:-0} -ne 0 -a $seconds -lt $max_seconds ] && $dbg ipfw $quiet table $table delete $addr
    51  done

- 1-38: variables, options, usage (soit les 3/4 du script)
- 40-44: l'utilisation combiné de **set -e** et d'un pipe m'oblige à intercepter les erreurs (**trap**). Il faut définir le message d'erreur **avant** que l'éventuelle erreur se produise.
- 45: fin d'interception des erreurs
- 49-51: le coeur du script

## Table ou pas table ?

On peut imaginer remplacer les variables **firewall_ssh6** et **firewall_ssh4** (voir mon [billet précédent](http://blog.bsdsx.fr/2019/12/2019-12-21_ipfw.md.html)) par une table.

Pour:

- une seule règle au lieu de 2
- ajout d'entrées à la demande (mais ce point peut être discutable: dans la majorité des cas, les accès ssh sont définis une bonne fois pour toute) 
- simplification dans le cas de liste à rallonge

Contre:

- code supplémentaire pour charger la table

Si mes adresses sont dans un (ou des) fichier(s) avec la syntaxe suivante:

    # commentaire
    8.8.8.8
    
      # autre commentaire avec des espaces inutiles
    2001:4860:4860::8888
          2002::/8 # espaces inutiles et commentaire
    1.1.1.1, 2.2.2.2/2   3.3.3.3/32, 4.4.4.4/24 #plusieurs adresses par ligne avec ou sans virgule

je dois, pour que ces données soient compréhensibles par **ipfw**:

- supprimer les commentaires
- afficher ce qui ressemble à une adresse en la faisant suivre par ' 0,'
- transformer les ',' en "\n"

Ce qui peut donner (les améliorations dans les commentaires sont les bienvenues :):

    # sed -E -n -e 's/#.*//' -e 's/([[:xdigit:]:\.\/]+)/\1 0,/gp' fichier(s) | tr ',' "\n" | xargs ipfw table t_ssh add

Je laisse le soin aux gourous de **sed** d'affiner mon motif qui doit leur paraitre bien naïf. Quant aux furieux d'**ipfw**, ils n'hésiteront pas à ajouter l'option **atomic** dans un pur esprit "tout ou rien".
Et cerise sur le gâteau, **xargs** a le bon goût de ne rien faire si aucun argument n'est fourni.

J'enrobe le tout dans une fonction **load_table_addr** et ma table **t_ssh** est chargée depuis un ou des fichiers. Et pourquoi pas charger **t_pouilleux** avec la liste des [bogons](https://en.wikipedia.org/wiki/Bogon_filtering) ? Mais il va se poser 2 problèmes:

- **sed** génère une erreur s'il ne peut pas accéder à un fichier
- **ipfw** sans **-q** génère une erreur en cas de doublon d'adresse

Ces erreurs seront interceptées par **set -e** et le script se terminera prématurément, laissant notre mur de feu dans un état proche de l'Ohio (dixit Isabelle A.).

Le cas "sed" peut se gérer avec une boucle:

    for file in $files; do
            [ -f $file ] || continue
            sed ... $file | tr ',' "\n" | xargs $fwcmd ...
    done

mais on exécute la commande pour chaque fichier. Autant faire une liste de fichier existant (et au passage tester cette liste):

    local files_ok=""
    for file in $files; do
            [ -f $file ] || continue
            $files_ok="$files_ok $file"
    done
    [ -n "$files_ok" ] || return
    sed ... $files_ok | tr ',' "\n" | xargs $fwcmd ...

Le cas des doublons d'adresse est tout aussi simple:

    sed ... $files_ok | tr ',' "\n" | sort --unique --ignore-leading-blanks | xargs $fwcmd ...

## Une fonction pour tous les charger

     1  load_table_addr() {
     2          local table=$1; shift
     3          local files=""
     4          for file in $@; do
     5                  [ -f $file ] || continue
     6                  files="$files $file"
     7          done
     8          [ -n "$files" ] || return
     9          sed -E -n -e 's/#.*//g' -e 's/([[:xdigit:]:\.\/]+)/\1 0,/gp' $files | tr ',' "\n" | sort --unique --ignore-leading-blanks | xargs $fwcmd table $table add
    10  }

Les tests des lignes 5 et 8 méritent sans doute un message d'alerte.

## Fichiers ou variables ?

Si je veux gérer les doublons tout en utilisant le contenu de variables *ET* de fichiers je dois ruser. 

     1	load_table_addr() {
     2	        [ $# -gt 1 ] || return
     3	        local table=$1
     4	        local files=""
     5	        local addrs=""
     6	
     7	        for arg in $@; do
     8	                case $arg in
     9	                /*)
    10	                        [ -f $arg ] || continue
    11	                        files="$files $arg"
    12	                        ;;
    13	                [0-9a-fA-F]*)
    14	                        addrs="$addrs $args";;
    15	                esac
    16	        done
    17	        (echo $addrs; cat ${files:-/dev/null}) | sed -E -n -e 's/#.*//g' -e 's/([[:xdigit:]:\.\/]+).*/\1 0/gp' | tr ',' "\n" | sort --unique --ignore-leading-blanks | xargs $fwcmd table $table atomic add
    18	}

- 2: pas la peine de continuer si aucun autre argument que le nom de la table
- 4-5: deux listes: fichiers et adresses
- 9-12: l'argument correspond à un nom de fichier (nom absolu donc commence par '/')
- 13-15: l'argument correspond à une adresse (grosso merdo)
- 17: j'affiche **addrs** puis le contenu des fichiers correspondant à **files** (qui au pire vaut **/dev/null**) le tout dans un subshell dont la sortie est redirigée vers le reste de mes commandes. Y'a pas à dire: le shell, c'est bô.

Alors fichiers ou variables ? Les deux mon Général !

## Pour finir

Le chargement de mes tables est grandement simplifié:

    load_table_addr t_ssh ${firewall_ssh6:-} ${firewall_ssh4:-} ${firewall_t_ssh:-}
    load_table_addr t_pouilleux ${firewall_t_pouilleux:-}

Tout comme les règles concernant le **ssh**: je charge la table **t_ssh** et si elle est vide je passe en mode "ouvert aux 4 vents".

    $ cat -n /etc/rc.conf.d/rc.firewall
     1  set -e
     2  
     3  load_table_addr() {
     4          local table=$1
     5          local files=""
     6          local addrs=""
     7  
     8          for arg in $@; do
     9                  case $arg in
    10                  /*)
    11                                 [ -f $arg ] || continue
    12                                 files="$files $arg"
    13                          ;;
    14                  [0-9a-fA-F]*)
    15                          addrs="$addrs $arg";;
    16                  esac
    17          done
    18          (echo $addrs; cat ${files:-/dev/null}) | sed -E -n -e 's/#.*//' -e 's/([[:xdigit:]:\.\/]+)/\1 0/gp' | tr ',' "\n" | sort --unique --ignore-leading-blanks | xargs $fwcmd table $table atomic add
    19  }
    20  
    21  set_31() {
    22          [ ${firewall_debug} = 'NO' -a $($fwcmd set 31 list | wc -l) -gt 1 ] && return
    23  
    24          $fwcmd table t_pouilleux create type addr
    25          $fwcmd table t_ssh create type addr
    26  
    27          load_table_addr t_ssh ${firewall_ssh6:-} ${firewall_ssh4:-} ${firewall_t_ssh:-}
    28          load_table_addr t_pouilleux ${firewall_t_pouilleux:-}
    29  
    30          $add set 31 check-state
    31          if $fwcmd table t_ssh info | grep -q 'items: 0'; then
    32                  $add set 31 deny in lookup src-ip t_pouilleux
    33                  $add set 31 pass { dst-port 22 or src-port 22 }
    34          else
    35                  $add set 31 pass src-port 22
    36                  $add set 31 pass in dst-port 22 lookup src-ip t_ssh
    37                  $add set 31 deny in lookup src-ip t_pouilleux
    38          fi
    39  }
    40  
    41  . /etc/rc.subr
    42  load_rc_config ipfw
    43  
    44  set -u
    45  
    46  [ $# -eq 1 ] && [ $1 = "-d" ] && firewall_debug=YES
    47  
    48  fwcmd=/sbin/ipfw
    49  case ${firewall_quiet:-} in
    50          [Yy][Ee][Ss]) fwcmd="$fwcmd -q";;
    51  esac
    52  
    53  case ${firewall_debug:-NO} in
    54          [Yy][Ee][Ss]) fwcmd="echo $fwcmd";;
    55                     *) firewall_debug='NO';;
    56  esac
    57  
    58  add="$fwcmd add"
    59  
    60  set_31
    61  $fwcmd -f flush
    62  
    63  $add pass via lo0
    64  $add pass out keep-state
    65  $add pass { proto ipv6-icmp or proto icmp }
    66  $add deny log in

Quelques remarques:

- en mode debug le test de la ligne 31 sera toujours faux
- en mode "ouvert aux 4 vents" le trafic "ssh" ne génèrera aucune règle dynamique (ligne 33). Ce n'est pas le cas en mode "ssh restreint": le premier paquet d'une connexion **initiée** par la machine (out, dst-port 22) ne correspond pas aux lignes 35-36 mais à la ligne 64 et génère une règle dynamique. Je ne pense pas que ce soit un problème car quand je me connecte sur une machine et que je joue avec les règles du mur de feu, c'est *MA* connexion qui ne doit pas être coupée et la susdite machine va rarement initier du ssh dans mon dos (aka cron) en même temps.
- le contenu de mes règles suit l'ordre suivant: drapeau - comparaison - recherche

Commentaires: [https://github.com/bsdsx/blog_posts/issues/2](https://github.com/bsdsx/blog_posts/issues/2)
