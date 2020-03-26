###### 202003260800 unbound shell
# Bloquer des domaines avec unbound

J'utilise [https://github.com/StevenBlack/hosts](https://github.com/StevenBlack/hosts) comme source de domaines à bloquer. Comme tout le monde, j'avais un script de conversion hosts vers unbound pour passer de

    0.0.0.0 domaine.tld
à
    local-zone: "domain.tld" redirect
    local-data: "domaine.tld A 0.0.0.0"

Dernièrement, je suis tombé sans trop me faire de mal sur [cet excellent article](https://www.shaftinc.fr/blocage-pubs-unbound.html). Après sa lecture, je me suis précipité sur mes dns menteurs et constaté
la consommation mémoire de mes unbound. J'ai perdu connaissance quelques minutes.
Encore sous le choc, j'ai jeté un oeil [au script](https://framagit.org/Shaft/unbound-adblock/-/blob/master/unbound-adblock). Nouvelle perte de connaissance.

## Ma version personnelle de moi que j'ai fait personnellement moi-même tout seul dans mon coin pour mes besoins que j'ai

Mes besoins:

- télécharger
- transformer
- vérifier
- recharger

L'url de ma prose: [http://download.bsdsx.fr/ads/ads.sh](http://download.bsdsx.fr/ads/ads.sh).

Une centaine de ligne, le "début" du script commence à 70.

## Au résultat

    $ ./bin/ads.sh -h
    ads.sh -u url [-w whitelist.file] [-U /path/to/unbound] [-R] [-d] [-v] [-h]
        -u url               url to StevenBlack hosts
        -w whitelist.file    exclude file (grep format)
        -U /path/to/unbound  default: /usr/sbin/unbound
        -R                   reload unbound
        -d                   mode debug
        -v                   mode verbose
        -h                   this message
    
    Example:
    
    ./bin/ads.sh -u https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn-social/hosts -U /usr/sbin/local-unbound -R
    
    ./bin/ads.sh -u http://sbc.io/hosts/alternates/fakenews-gambling-porn-social/hosts -w whitelist.ads -R -d

Ce script part du principe que votre **unbound.conf** est configuré comme suit:

    $ grep -e directory -e conf.d /chemin/vers/unbound/unbound.conf
            directory: /chemin/vers/unbound
    include: /chemin/vers/unbound/conf.d/*.conf

et que l'utilisateur peut écrire dans **/chemin/vers/unbound/conf.d**

Les utilisateurs de FreeBSD n'hésiteront pas à utiliser l'option **-U**. Ceux de MacOS, DragonFlyBSD, Solaris ... peuvent m'envoyer un patch (ou une machine :) ou se démerder.
Pour établir une liste blanche autant partir de ce qu'on bloque et jouer avec son **$EDITOR**:

    $ grep reddit /chemin/vers/unbound/conf.d/ads.conf > /chemin/vers/ma/liste/blanche

## Au final

Quelques chiffres:

    $ wc -l /var/unbound/conf.d/ads.conf
       76318 /var/unbound/conf.d/ads.conf
    $ top | grep local-unbound
    55175 unbound       1  20    0    50M    38M select   3   0:31   0.00% local-unbound

Un grand merci à shaft pour son article.

Commentaires: [https://github.com/bsdsx/blog_posts/issues/4](https://github.com/bsdsx/blog_posts/issues/4)
