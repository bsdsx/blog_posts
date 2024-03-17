###### 202402290800 bhyve wifi openwrt

J'ai pendant très longtemps utilisé un [mini routeur sous OpenWrt](https://openwrt.org/toh/gl.inet/gl-mt300a) car le wifi de [mon ancien nuc](https://www.intel.com/content/www/us/en/products/sku/95062/intel-nuc-kit-nuc6cayh/specifications.html) n'était ni stable ni performant. Les ressources de ma nouvelle machine de travail (AMD Ryzen 7 3750H) étant bien supérieures, l'idée m'est venue de mimer [FreeBSD wifibox](https://github.com/pgj/freebsd-wifibox) à la sauce OpenWrt. Un portable Lenovo T450 en a aussi fait les frais.

## wifibox

Le principe est assez simple: si la plateforme matérielle le permet, attribuer le périphérique wifi à une machine virtuelle dont le pilote a une meilleure prise en charge que celui de FreeBSD. Le projet a fait le choix l'Alpine Linux.

## wifi et bhyve

Je commence par identifier le périphérique:

    $ pciconf -lv
    ...
    iwm0@pci0:2:0:0:        class=0x028000 rev=0x59 hdr=0x00 vendor=0x8086 device=0x095a subvendor=0x8086 subdevice=0x5010
        vendor     = 'Intel Corporation'
        device     = 'Wireless 7265'
        class      = network
    ...

Il ne doit plus être pris en charge par le pilote **iwm** mais par le **Pci PassThru** (ppt). Je dois aussi charger le module **vmm** et parce que j'ai un processeur AMD activer un **sysctl** comme indiqué sur [le wiki FreeBSD](https://wiki.freebsd.org/bhyve/pci_passthru))

    $ cat /boot/loader.conf.d/vmm.conf
    vmm_load="YES"
    pptdevs="2/0/0" # à adapter suivant pciconf -lv
    hw.vmm.amdvi.enable=1

Un redémarrage plus tard, on obtient:

    $ pciconf -lv
    ...
    ppt0@pci0:2:0:0:	class=0x028000 rev=0x59 hdr=0x00 vendor=0x8086 device=0x095a subvendor=0x8086 subdevice=0x5010
        vendor     = 'Intel Corporation'
        device     = 'Wireless 7265'
        class      = network
    ...

## Machine virtuelle

Je récupère la dernière image d'OpenWrt:

    $ fetch https://downloads.openwrt.org/releases/23.05.2/targets/x86/64/openwrt-23.05.2-x86-64-generic-ext4-combined-efi.img.gz
    $ gunzip openwrt-23.05.2-x86-64-generic-ext4-combined-efi.img.gz
    gunzip: openwrt-23.05.2-x86-64-generic-ext4-combined-efi.img.gz: trailing garbage ignored

Pour que **bhyve** puisse utiliser une image **efi** je dois ajouter le paquet suivant:

    $ doas pkg install bhyve-firmware

et je vérifie que la machine virtuelle démarre bien:

    $ doas bhyve -H -P -m 192M -s 0:0,hostbridge -s 1:0,lpc -s 2:0,virtio-net,tap0 -s 3:0,virtio-blk,./openwrt-23.05.2-x86-64-generic-ext4-combined-efi.img -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd -l com1,stdio -W vm0

Un **halt** plus tard je peux détruire cette machine virtuelle:

    $ doas bhyvectl --destroy --vm=vm0

Utile quand on joue au plus malin et qu'avec 128 Mo de mémoire on obtient:

    error: out of memory.
    error: out of memory.
    
    
    Press any key to continue...Press any key to continue...

## OpenWrt

Léger problème avec l'image fournie: elle ne contient absolument rien (module, pilote, outil ...) concernant le wifi (c'est voulu par le projet pour la plate-forme x86). Je prépare un disque à ajouter à mon image avec le contenu nécessaire:

    $ truncate -s 5M pkgs.img # oui, 5 Mo c'est bien assez
    $ newfs_msdos ./pkgs.img
    newfs_msdos: warning, ./pkgs.img is not a character device
    ./pkgs.img: 10192 sectors in 1274 FAT12 clusters (4096 bytes/cluster)
    BytesPerSec=512 SecPerClust=8 ResSectors=1 FATs=2 RootDirEnts=512 Sectors=10240 Media=0xf0 FATsecs=4 SecPerTrack=63 Heads=255 HiddenSecs=0
    $ doas mdconfig ./pkgs.img 
    md0
    $ doas mount -t msdosfs /dev/md0 /media

Depuis **https://downloads.openwrt.org/releases/23.05.2/packages/x86_64/base/** je récupère les paquets:

    hostapd-common_2023-09-08-e5ccbfc6-6_x86_64.ipk
    iw_4.14-1_x86_64.ipk
    iwinfo_2023-07-01-ca79f641-1_x86_64.ipk
    iwlwifi-firmware-iwl7265d_20230804-1_x86_64.ipk
    libnl-tiny1_2023-07-27-bc92a280-1_x86_64.ipk
    ucode-mod-nl80211_2023-11-07-a6e75e02-1_x86_64.ipk
    ucode-mod-rtnl_2023-11-07-a6e75e02-1_x86_64.ipk
    ucode-mod-uloop_2023-11-07-a6e75e02-1_x86_64.ipk
    wireless-regdb_2024.01.23-1_all.ipk
    wpa-cli_2023-09-08-e5ccbfc6-6_x86_64.ipk
    wpa-supplicant-mini_2023-09-08-e5ccbfc6-6_x86_64.ipk

et depuis **https://downloads.openwrt.org/releases/23.05.2/targets/x86/64/packages/** les modules:

    kmod-cfg80211_5.15.137+6.1.24-3_x86_64.ipk
    kmod-crypto-aead_5.15.137-1_x86_64.ipk
    kmod-crypto-cbc_5.15.137-1_x86_64.ipk
    kmod-crypto-ccm_5.15.137-1_x86_64.ipk
    kmod-crypto-cmac_5.15.137-1_x86_64.ipk
    kmod-crypto-ctr_5.15.137-1_x86_64.ipk
    kmod-crypto-gcm_5.15.137-1_x86_64.ipk
    kmod-crypto-gf128_5.15.137-1_x86_64.ipk
    kmod-crypto-ghash_5.15.137-1_x86_64.ipk
    kmod-crypto-hmac_5.15.137-1_x86_64.ipk
    kmod-crypto-manager_5.15.137-1_x86_64.ipk
    kmod-crypto-null_5.15.137-1_x86_64.ipk
    kmod-crypto-rng_5.15.137-1_x86_64.ipk
    kmod-crypto-seqiv_5.15.137-1_x86_64.ipk
    kmod-crypto-sha512_5.15.137-1_x86_64.ipk
    kmod-iwlwifi_5.15.137+6.1.24-3_x86_64.ipk
    kmod-mac80211-hwsim_5.15.137+6.1.24-3_x86_64.ipk
    kmod-mac80211_5.15.137+6.1.24-3_x86_64.ipk

Je démarre la machine virtuelle avec ce nouveau disque:

    $ doas umount /media
    $ doas mdconfig -du md0
    $ doas bhyve -H -P -m 192M -s 0:0,hostbridge -s 1:0,lpc -s 2:0,virtio-net,tap0 -s 3:0,virtio-blk,./openwrt-23.05.2-x86-64-generic-ext4-combined-efi.img -s 4:0,virtio-blk,./pkgs.img -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd -l com1,stdio -W vm0

et j'installe les paquets:

    root@OpenWrt:/# mount -t vfat /dev/vdb /mnt/
    root@OpenWrt:/# opkg install /mnt/libnl-tiny1_2023-07-27-bc92a280-1_x86_64.ipk
    Package libnl-tiny1 (2023-07-27-bc92a280-1) installed in root is up to date.
    root@OpenWrt:/# ln /usr/lib/libnl-tiny.so.1 /usr/lib/libnl-tiny.so # sinon iw va râler
    
    root@OpenWrt:/# opkg install /mnt/hostapd-common_2023-09-08-e5ccbfc6-6_x86_64.ipk /mnt/iw_4.14-1_x86_64.ipk /mnt/iwinfo_2023-07-01-ca79f641-1_x86_64.ipk /mnt/iwlwifi-firmware-iwl7265d_20230804-1_x86_64.ipk /mnt/ucode-mod-* /mnt/wireless-regdb_2024.01.23-1_all.ipk /mnt/wpa-cli_2023-09-08-e5ccbfc6-6_x86_64.ipk /mnt/wpa-supplicant-mini_2023-09-08-e5ccbfc6-6_x86_64.ipk
    Installing hostapd-common (2023-09-08-e5ccbfc6-6) to root...
    Installing iw (4.14-1) to root...
    Installing iwinfo (2023-07-01-ca79f641-1) to root...
    Installing iwlwifi-firmware-iwl7265d (20230804-1) to root...
    Installing ucode-mod-nl80211 (2023-11-07-a6e75e02-1) to root...
    Installing ucode-mod-rtnl (2023-11-07-a6e75e02-1) to root...
    Installing ucode-mod-uloop (2023-11-07-a6e75e02-1) to root...
    Installing wireless-regdb (2024.01.23-1) to root...
    Installing wpa-cli (2023-09-08-e5ccbfc6-6) to root...
    Installing wpa-supplicant-mini (2023-09-08-e5ccbfc6-6) to root...
    Configuring iwinfo.
    Configuring iw.
    Configuring ucode-mod-uloop.
    Configuring hostapd-common.
    Configuring ucode-mod-nl80211.
    Configuring ucode-mod-rtnl.
    Configuring wpa-supplicant-mini.
    Configuring iwlwifi-firmware-iwl7265d.
    Configuring wpa-cli.
    Configuring wireless-regdb.
    root@OpenWrt:/# opkg install /mnt/kmod-*
    ...
    root@OpenWrt:/# umount /mnt
    root@OpenWrt:/# halt

Il est temps d'affecter le périphérique wifi à la machine virtuelle (adapter **2/0/0** au **pciconf -lv** et ne pas oublier l'option **-S**):

    $ doas bhyve -H -P -S -m 192M -s 0:0,hostbridge -s 1:0,lpc -s 2:0,virtio-net,tap0 -s 3:0,virtio-blk,./openwrt-23.05.2-x86-64-generic-ext4-combined-efi.img -s 7:0,passthru,2/0/0 -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd -l com1,stdio -W vm0

## Configuration OpenWrt

Pas besoin du délai de **grub**:

    root@OpenWrt:/# grep timeout /boot/grub/grub.cfg 
    set timeout="0"

Quelques modules sont chargés au démarrage, je ne garde que le minimum (et encore, je pense que seul **30-fs-vfat** est utile pour récupérer la configuration de **grub**):

    root@OpenWrt:/# ls -1 /etc/modules-boot.d/
    02-crypto-hash
    04-crypto-crc32c
    09-crypto-aead
    09-crypto-manager
    30-fs-vfat

Pareil pour les modules chargés après le démarrage (je doit pourvoir encore en supprimer quelques uns):

    root@OpenWrt:/# ls -1 /etc/modules.d/
    02-crypto-hash
    04-crypto-crc32c
    09-crypto-acompress
    09-crypto-aead
    09-crypto-cbc
    09-crypto-ccm
    09-crypto-cmac
    09-crypto-ctr
    09-crypto-gcm
    09-crypto-gf128
    09-crypto-ghash
    09-crypto-hmac
    09-crypto-manager
    09-crypto-null
    09-crypto-rng
    09-crypto-seqiv
    09-crypto-sha512
    25-nls-cp437
    25-nls-iso8859-1
    25-nls-utf8
    30-fs-vfat
    iwlwifi
    lib-crc-ccitt
    lib-crc32c
    lib-lzo
    mac80211-hwsim

Je désactive les services inutiles:

    root@OpenWrt:/# /etc/init.d/dnsmasq     stop && /etc/init.d/dnsmasq     disable
    root@OpenWrt:/# /etc/init.d/dropbear    stop && /etc/init.d/dropbear    disable
    root@OpenWrt:/# /etc/init.d/firewall    stop && /etc/init.d/firewall    disable
    root@OpenWrt:/# /etc/init.d/gpio_switch stop && /etc/init.d/gpio_switch disable
    root@OpenWrt:/# /etc/init.d/led         stop && /etc/init.d/led         disable
    root@OpenWrt:/# /etc/init.d/sysntpd     stop && /etc/init.d/sysntpd     disable
    root@OpenWrt:/# /etc/init.d/uhttpd      stop && /etc/init.d/uhttpd      disable

Je configure le wifi:

    root@OpenWrt:/# cat /etc/config/wireless
    ...
    config wifi-device 'radio2'
    	option type 'mac80211'
    	option path 'pci0000:00/0000:00:07.0'
    	option channel '36'
    	option band '5g'
    	option htmode 'VHT80'
    	option disabled '0'
    
    config wifi-iface 'default_radio2'
    	option device 'radio2'
    	option network 'wan wan6'
    	option mode 'sta'
    	option ssid 'mon_super_ssid'
    	option encryption 'psk2'
    	option key 'ma_super_key'

et le réseau:

    root@OpenWrt:/# cat /etc/config/network
    config interface 'loopback'
    	option device 'lo'
    	option proto 'static'
    	option ipaddr '127.0.0.1'
    	option netmask '255.0.0.0'
    
    config interface 'lan'
    	option device 'eth0'
    	option proto 'static'
    	option ipaddr '172.20.99.1'
    	option netmask '255.255.255.0'
    	option ip6addr '2db8:xxxx:xxxx:xx3:172:20:99:1/112'
    
    config interface 'wan'
    	option proto 'static'
    	option ipaddr '172.20.62.99'
    	option netmask '255.255.255.0'
    	option gateway '172.20.62.1'
    	list dns '172.30.63.20'
    	list dns '172.30.63.10'
    
    config interface 'wan6'
    	option proto 'static'
    	option ip6addr '2db8:xxxx:xxxx:xx2::99/64'
    	option ip6gw '2db8:xxxx:xxxx:xx2::1'
    	list dns '2db8:xxxx:xxxx:xx1:172:30:63:20'
    	list dns '2db8:xxxx:xxxx:xx1:172:30:63:10'

## Configuration FreeBSD

Le **bridge**:

    $ cat /etc/rc.conf.d/bridge
    autobridge_interfaces="bridge0"
    autobridge_bridge0="tap0"

Le réseau:

    $ cat /etc/rc.conf.d/network
    cloned_interfaces="bridge0 tap0"
    
    ifconfig_bridge0="inet 172.20.99.10 netmask 255.255.255.0"
    ifconfig_bridge0_ipv6="inet6 auto_linklocal"
    ifconfig_bridge0_alias0="inet6 2db8:xxxx:xxxx:xx3:172:20:99:10 prefixlen 112"

    $ cat /etc/rc.conf.d/routing
    defaultrouter="172.20.99.1"
    ipv6_defaultrouter="2db8:xxxx:xxxx:xx3:172:20:99:1"

L'accès à la console de la machine virtuelle à l'aide de **nmdm**:

    $ cat /etc/rc.conf.d/kld
    kld_list="/boot/modules/amdgpu.ko nmdm"

Le démarrage de la machine virtuelle:

    $ cat /etc/rc.local
    /usr/sbin/bhyve -H -P -S -m 192M -s 0:0,hostbridge -s 1:0,lpc -s 3:0,virtio-blk,/chemin/complet/vers/openwrt-23.05.2-x86-64-generic-ext4-combined-efi.img -s 7:0,passthru,2/0/0 -l bootrom,/usr/local/share/uefi-firmware/BHYVE_UEFI.fd -l com1,/dev/nmdm0A vm0 &

L'accès à la console de la machine virtuelle se fera à l'aide de **cu**:

    $ doas cu -l /dev/nmdm0B # oui "B", man 4 nmdm

## Mon réseau

La configuration réseau présentée est loin d'être la plus commune (pas de **dhcp** ni de **nat**, **ipv6** sans **autoconf**). Pour une version plus "traditionnelle" (**dhcp**, pas d'**ipv6**), la machine virtuelle devra:

- faire du **dhcp** sur l'interface **wan**
- activer le **firewall** pour faire du **nat**

Ce qui devrait peu ou prou correspondre à (pas encore testé):

    root@OpenWrt:/# cat /etc/config/network
    ....
    config interface 'wan'
        option proto 'dhcp'
    
    root@OpenWrt:/# /etc/init.d/firewall enable && /etc/init.d/firewall start

Au pire la documentation d'OpenWrt [https://openwrt.org/docs/guide-user/network/start](https://openwrt.org/docs/guide-user/network/start) est plus que fournie.

## Pour conclure

Le **iperf** est sans appel:

    $ iperf3 -6 -c proxy
    ...
    [ ID] Interval           Transfer     Bitrate         Retr
    [  5]   0.00-10.00  sec   226 MBytes   190 Mbits/sec   19             sender
    [  5]   0.00-10.01  sec   226 MBytes   189 Mbits/sec                  receiver
    
    $ iperf3 -6 -c proxy
    ...
    [  5]   0.00-10.00  sec   263 MBytes   221 Mbits/sec    0             sender
    [  5]   0.00-10.02  sec   263 MBytes   220 Mbits/sec                  receiver

C'est loin d'être parfait au vu de la configuration du réseau:

    freebox delta [ vm proxy sous FreeBSD ] <-- rj45 --> ap wifi Mikrotik hAP ac^2 <- -> nuc

mais c'est toujours mieux que la dizaine de Mbits/sec (dont je me contentais très bien) de mon ancienne installation.

Le mini routeur qui trônait fièrement au dessus du nuc devrait finir au fin fond d'un tirroir. Mais comme il possède un lien série, j'en ferais bien un Frankenstein à base de bluetooth ...

Commentaires: [https://github.com/bsdsx/blog_posts/issues/20](https://github.com/bsdsx/blog_posts/issues/20)

## Correction 2024-03-17

J'ai oublié '-s 2:0,virtio-net,tap0' à chaque commande **bhyve**.
