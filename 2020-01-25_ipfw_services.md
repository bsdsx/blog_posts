###### 202001251100 ipfw shell
# ipfw: modern FreeBSD, services

Pour illustrer mon propos, il me faut un service. Un truc simple qui fasse udp/tcp et ipv6/ipv4. J'aurais pu utiliser **nc -k -l 1234** mais il n'est pas possible de faire tcp/udp/ipv6/ipv4 en même temps (sauf à lancer 4 nc). J'ai donc jeter mon dévolu sur **daytime** fourni par **inetd**:

    $ sed -n '/#daytime/s/^.//p' /etc/inetd.conf > /tmp/inetd.conf
    $ sudo /usr/sbin/inetd -d -l /tmp/inetd.conf
    [ snip du debug ]

Depuis la même machine, je peux m'y connecter en tcp:

    $ nc -d -6 localhost 13
    $ nc -d -4 localhost 13

et vérifier l'udp:

    $ nc -z -u -6 localhost 13
    $ nc -z -u -4 localhost 13

On doit voir une trace de ces connexions dans la console où est lancé **inetd** (je ne sais pas pourquoi l'udp génère 4 lignes de debug). Toute tentative depuis une machine distante se soldera par un échec car l'accès au port "13" n'est pas autorisé.

## Service

J'aime assez le concept de mur de feu piloté par des variables. Mon service **ssh** est géré par les variables suivantes:

- firewall_ssh6
- firewall_ssh4
- firewall_t_ssh

Le nommage de ces variables suit cette règle: 'firewall' + nom_du_service + 4|6 ou 'firewall' + 't' + nom_du_service. Cette règle à (au moins) 2 défauts:

- pas de distinction udp/tcp
- les variables utilisent le nom du service (**ssh**) alors que le script utilise le numéro de port associé (**22**)

Pour simplifier mes règles, je pars du principe qu'un service est à la fois **tcp** et **udp**. Pour les noms/numéros de port je peux utiliser les noms dans le script mais comment définir un service qui écoute sur le port **3128** ? Ou pire, sur plusieurs ports ? Voici ce que je voudrais pouvoir faire:

     1	firewall_services="proxy http,dns ,foo bar, baz ,, ,  ,   mail_int daytime"
     2	
     3	firewall_proxy_port=3128
     4	firewall_proxy4="192.168.0.0/24 172.16.0.0/24, 172.16.2.0/24"
     5	
     6	firewall_dns_ports=",53,853, 8853"
     7	
     8	firewall_foo_ports=9000
     9	firewall_t_foo=t_bar
    10	
    11	firewall_bar_port="9001 9002"
    12	firewall_t_bar=/etc/rc.conf.d/bar.*
    13	
    14	firewall_baz_port=9003
    15	firewall_t_baz=t_bar
    16	
    17	firewall_mail_int_ports="smtp 587 imap 993 4190"
    18	firewall_t_mail_int=t_ssh


- 1: je définis une liste de service (on portera une attention particulière aux services "http" et "daytime")
- 3-4: un service "proxy" écoute sur le port "3128" en mode "restreint" à partir d'une liste (qui sera transformée en table)
- 6: un service "dns" écoute sur les ports "53", "853" et "8853" en mode "ouvert aux 4 vents"
- 8-9: un service "foo" écoute sur le port "9000" en mode "restreint" en utilisant la table "t_bar" qui sera définie plus tard
- 11-12: un service "bar" écoute sur les ports "9001" et "9002" en mode "restreint" en utilisant sa table "t_bar" qui sera chargée à partir de fichiers
- 14-15: un service "baz" écoute sur le port "9003" en mode "restreint" en utilisant la table "t_bar" définie plus tôt
- 17-18: un service "mail_int" écoute sur des ports qui semblent concerner le courriel (smtp, submission, imap, imaps et sieve) en mode "restreint" en utilisant la table "t_ssh"

Sans configuration particulière, les services "http" et "daytime" écoutent sur leur port respectif en mode "ouvert aux 4 vents".

Je veux aussi pouvoir traiter les gros doigts ("port" vs "ports") et les listes ("1 2 3" vs "a,b, c ,, d").

## Code

    60	set_31
    61	$fwcmd -f flush
    62	
    63	$add pass via lo0
    64	$add pass out keep-state
    65	
    66	for svc in $(echo ${firewall_services:-} | tr ',' ' '); do
    67		eval "port=\"\${firewall_${svc}_ports:-\${firewall_${svc}_port:-$svc}}\""
    68		[ "$port" = $svc ] || port=$(echo $port | sed -E -e 's/[ ,]+/,/g' -e 's/(^,|,$)//')
    69	
    70		eval "table=\"\${firewall_t_${svc}:-}\""
    71		case "$table" in
    72			t_*) [ $table = "t_$svc" ] || $add pass in dst-port $port lookup src-ip $table keep-state; continue;;
    73		esac
    74		
    75		eval "table_src=\"\${firewall_${svc}6:-} \${firewall_${svc}4:-} \${firewall_t_${svc}:-}\""
    76		if [ "$table_src" = "  " ]; then
    77			$add pass in dst-port $port keep-state
    78			continue
    79		fi
    80	
    81		$fwcmd table t_$svc create type addr or-flush
    82		load_table_addr t_$svc $table_src
    83		$add pass in dst-port $port lookup src-ip t_$svc keep-state
    84	done
    85	
    86	$add pass { proto ipv6-icmp or proto icmp }
    87	$add deny log in

- 66: je normalise le contenu de la variable **firewall_services** à l'aide de **tr**
- 67: l'échappement de nombreux caractères ne facilite pas la lecture: définir une variable **port** à partir de **{firewall_svc_ports:-{firewall_svc_port:-svc}}**
- 68: je normalise le contenu de **port** s'il est différent de **svc**
- 70-73: si mon service utilise une table et que le nom de cette table ne correspond pas à t_service alors j'ajoute une règle en mode "restreint" + règle dynamique et je passe au service suivant
- 75-79: si mon service n'est pas en mode "restreint" (les variables **firewall_svc(6|4)** et **firewall_t_svc** sont vides) alors j'ajoute une règle en mode "ouvert aux 4 vents" + règle dynamique et je passe au service suivant
- 81-83: je crée une table correspondant au service (ou je la vide si elle existe déjà: **or-flush**), je la charge et j'ajoute une règle en mode "restreint" + règle dynamique

## Au final

     1	$ sudo service ipfw restart
     2	Flushed all rules.
     3	00500 allow via lo0
     4	00000 allow out keep-state :default
     5	added: 172.16.2.0/24 0
     6	added: 192.168.0.0/24 0
     7	added: 172.16.0.0/24 0
     8	00000 allow in dst-port 3128 dst-ip lookup src-ip t_proxy keep-state :default
     9	00000 allow in dst-port 80 keep-state :default
    10	00000 allow in dst-port 53,853,8853 keep-state :default
    11	00000 allow in dst-port 9000 dst-ip lookup src-ip t_bar keep-state :default
    12	00000 allow in dst-port 9001,9002 dst-ip lookup src-ip t_bar keep-state :default
    13	00000 allow in dst-port 9003 dst-ip lookup src-ip t_bar keep-state :default
    14	00000 allow in dst-port 25,587,143,993,4190 dst-ip lookup src-ip t_ssh keep-state :default
    15	00000 allow in dst-port 13 keep-state :default
    16	01500 allow { proto ipv6-icmp or proto icmp }
    17	01600 deny log in
    18	Firewall rules loaded.
    19	Firewall logging enabled.

On retrouve bien les règles définies au paragraphe "Service". Je pense que 'dst-ip' n'est qu'un bug d'affichage.

Si depuis une machine distante je teste le port "13" en udp ("nc -z -u -4 ip.de.la.machine 13") je peux voir qu'une règle dynamique a bien été ajoutée:

    $ sudo ipfw -d -S show | grep ' 13 '
    01400    8    332 set 0 allow in dst-port 13 keep-state :default
    01400    8    332 (9s) STATE udp 172.16.200.193 10930 <-> 172.16.100.50 13 :default

C'est un peu plus compliqué avec le tcp car la règle est supprimée dès que la connexion est terminée (et elle se termine très vite :).

    01400    7     406 (1s) STATE tcp 172.16.200.193 24467 <-> 172.16.100.50 13 :default

Commentaires: [https://github.com/bsdsx/blog_posts/issues/3](https://github.com/bsdsx/blog_posts/issues/3)
