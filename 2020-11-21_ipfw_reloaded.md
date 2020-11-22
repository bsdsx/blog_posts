###### 202011211200 ipfw shell
# ipfw: modern FreeBSD (reloaded)

Après plusieurs mois de production, petit retour sur mon script de mur de feu.

## Rappels

Ma configuration générale se trouve dans **/etc/rc.conf.d** et je n'ai pas de **/etc/rc.conf**. La configuration du mur de feu se trouve donc dans le répertoire **/etc/rc.conf.d/ipfw**. Jouer à distance avec un mur de feu peut vite devenir périlleux: une typo, une mauvaise règle et patatra le mur s'écroule, emportant avec lui la connexion **ssh**. Pour éviter ces désagréments:

- pas de règle dynamique pour le **ssh**
- uiliser le set **31**

Plus d'explication [dans mon premier billet](http://blog.bsdsx.fr/2020/01/2020-01-10_ipfw_table.md.html) où on trouvera aussi le pourquoi du **modern FreeBSD**. Attention, ce script ne convient pas à un hôte de type routeur.

## Le script

Après quelques évolutions, j'arrive à ce résultat [http://download.bsdsx.fr/rc.firewall](http://download.bsdsx.fr/202011211200_rc.firewall):

    $ cat -n /etc/rc.conf.d/rc.firewall
     1	set -e
     2	
     3	table_create() {
     4		$fwcmd table $1 create type ${2:-addr} ${3:-}
     5	}
     6	
     7	table_load_addr() {
     8		local table=$1
     9		local files=""
    10		local addrs=""
    11	
    12		for arg in $@; do
    13			case $arg in
    14				/*)
    15					[ -f $arg ] || continue
    16					files="$files $arg"
    17					;;
    18				[0-9a-fA-F]*)
    19					addrs="$addrs $arg";;
    20			esac
    21		done
    22		(echo $addrs; cat ${files:-/dev/null}) | sed -E -n -e 's/#.*//' -e 's/([[:xdigit:]:\.\/]+)/\1 0/gp' | tr ',' "\n" | sort --unique --ignore-leading-blanks | xargs $fwcmd table $table atomic add
    23	}
    24	
    25	set_31() {
    26		[ ${firewall_debug} = 'NO' -a $($fwcmd set 31 list | wc -l) -gt 1 ] && return
    27	
    28		table_create t_pouilleux
    29		table_create t_ssh
    30	
    31		table_load_addr t_ssh ${firewall_ssh6:-} ${firewall_ssh4:-} ${firewall_t_ssh:-}
    32		table_load_addr t_pouilleux ${firewall_t_pouilleux:-}
    33	
    34		$add set 31 check-state :KS
    35		if $fwcmd table t_ssh info | grep -q 'items: 0'; then
    36			$add set 31 deny in lookup src-ip t_pouilleux
    37			$add set 31 pass { dst-port ssh or src-port ssh }
    38		else
    39			$add set 31 pass src-port ssh
    40			$add set 31 pass in dst-port ssh lookup src-ip t_ssh
    41			$add set 31 deny in dst-port ssh
    42			$add set 31 deny in lookup src-ip t_pouilleux
    43		fi
    44		$add set 31 pass { proto ipv6-icmp or proto icmp }
    45	}
    46	. /etc/rc.subr
    47	load_rc_config ipfw
    48	
    49	set -u
    50	
    51	[ $# -eq 1 ] && [ $1 = "-d" ] && firewall_debug=YES
    52	
    53	fwcmd=/sbin/ipfw
    54	case ${firewall_quiet:-} in
    55		[Yy][Ee][Ss]) fwcmd="$fwcmd -q";;
    56	esac
    57	
    58	case ${firewall_debug:-NO} in
    59		[Yy][Ee][Ss]) fwcmd="echo $fwcmd";;
    60		*) firewall_debug='NO';;
    61	esac
    62	
    63	add="$fwcmd add"
    64	
    65	set_31
    66	
    67	KS="keep-state :KS"
    68	
    69	$fwcmd -f flush
    70	
    71	$add pass via lo*
    72	
    73	[ -f $0.$(hostname) ] && . $0.$(hostname)
    74	
    75	$add pass out $KS // after NAT
    76	case ${firewall_logging:-} in
    77	    [Yy][Ee][Ss])
    78			for ports in ${firewall_nologports:-}; do
    79				$add deny in dst-port $ports
    80			done
    81			;;
    82	esac
    83	$add deny log in
    84	
    85	# vim: set sw=4 sts=4 ts=4 ft=sh:

- 7-23: cette fonction me permet de charger une table de type **addr** à partir de variables et/ou de fichiers, les listes pouvant utiliser l'espace et/ou la virgule en tant que séparateur
- 25-45: définition du set **31** qui doit me permettre de garder ma connexion **ssh** en toute circonstance. Si le chargement de la table a échoué (ligne 35) alors l'accès **ssh** n'est pas restreint
- 51-61: debug / verbose
- 67: je précise le nom du **flow** des règles dynamiques, en prévision du **nat**
- 71: les interfaces loopback. A l'écriture de ce billet, je me demande si sa place ne serait pas dans le set **31** (au moins pour **lo0**, je verrais plus tard comment utiliser les jails et le **nat** sur **lo1**)
- 73: les règles de l'hôte. Utiliser le nom d'hôte me permet de centraliser/diffuser/copier sans risque ma configuration
- 75: le traffic sortant initié par l'hôte. Le commentaire a son importance: les futures règles de **nat** devront se placer avant celle-ci
- 76-82: traffic exempt de log uniquement en cas d'utilisation de **syslog** (pour éviter de le saturer)

Avec ce script et sans configuration je ne peux pas perdre l'accès **ssh** mais ce service est ouvert à tous les pouilleux, rabouins, peignes-cul et autres pénibles du nain ternet.

## La configuration de base

    $ cat -n /etc/rc.conf.d/ipfw/ipfw
     1  firewall_enable="YES"
     2  firewall_script="/etc/rc.conf.d/rc.firewall"
     3
     4  # logging == syslogd, logif == tcpdump
     5  #firewall_logging="YES"
     6  firewall_logif="YES"
     7
     8  ip6_dead="2001:db8:dead::/48"
     9  ip6_beef="2001:db8:beef::/48"
    10  ip6_cafe="2001:db8:cafe:cafe::/64"
    11
    12  ip4_maison="192.168.42.0/24 192.168.84.0/24, 10.10.0.0/16"
    13
    14  firewall_ssh6="$ip6_dead, $ip6_beef $ip6_cafe"
    15  firewall_ssh4="$ip4_maison"

Pour tester et vérifier (ne pas hésiter à faire volontairement des typos dans les adresses et observer le résultat):

    $ /bin/sh /etc/rc.conf.d/rc.firewall -d
    /sbin/ipfw table t_pouilleux create type addr
    /sbin/ipfw table t_ssh create type addr
    /sbin/ipfw table t_ssh atomic add 10.10.0.0/16 0 2001:db8:beef::/48 0 2001:db8:cafe:cafe::/64 0 192.168.42.0/24 0 192.168.84.0/24 0 2001:db8:dead::/48 0
    /sbin/ipfw add set 31 check-state :KS
    /sbin/ipfw add set 31 pass src-port ssh
    /sbin/ipfw add set 31 pass in dst-port ssh lookup src-ip t_ssh
    /sbin/ipfw add set 31 deny in dst-port ssh
    /sbin/ipfw add set 31 deny in lookup src-ip t_pouilleux
    /sbin/ipfw add set 31 pass { proto ipv6-icmp or proto icmp }
    /sbin/ipfw -f flush
    /sbin/ipfw add pass via lo*
    /sbin/ipfw add pass out keep-state :KS // after NAT
    /sbin/ipfw add deny log in

## Une configuration xen (non finalisée)

    $ cat /etc/rc.conf.d/rc.firewall.xen.bsdsx.fr
    table_create t_unbound addr or-flush && table_load_addr t_unbound ${firewall_unbound6:-} ${firewall_unbound4:-} ${firewall_t_unbound:-}
    
    $add pass in dst-port domain lookup src-ip t_unbound $KS
    $add pass in dst-port http $KS
    
    $add pass in src-port bootpc via xnb*
    $add pass in dst-port ntp,syslog,546 via xnb*
    $add pass in src-port bootps via vlan22

Cette machine fournit un service dns à accès restreint, un service http non restreint et laisse passer du traffic en provenance/destination des domU (j'y reviendrais certainement dans un autre billet).

## A venir

J'ai abandonné l'idée de mur de feu entièrement pilotable à coup de variables, le code nécessaire pour définir un service était beaucoup trop volumineux. L'utilisation d'un script local à l'hôte me parait plus simple. La prochaine évolution concernera le **nat** et des règles qui, je l'espère, sortiront des sentiers battus.

Commentaires: [https://github.com/bsdsx/blog_posts/issues/6](https://github.com/bsdsx/blog_posts/issues/6)
