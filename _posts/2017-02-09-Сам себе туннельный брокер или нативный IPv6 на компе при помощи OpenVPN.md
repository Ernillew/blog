---
layout: post
title: Сам себе туннельный брокер или нативный IPv6 на компе при помощи OpenVPN
---

Я — большой сторонник использования IPv6, стараюсь его использовать где это только возможно. Недавно подумав я решил, что на большинстве своих виртуалок переведу ssh на ipv6-only, биндить буду на рандомно выбранный при конфигурации адрес, который потом пропишу в ДНСах для своего удобства. Но возник вопрос с доступом с моего ноутбука к тем кого таким образом настрою. Понятно, что всегда можно ходить через сервер, где у меня IPv6, конечно же есть, обычно я так и делаю, но случаи бывают разные.

 Почесав немного голову я понял, что я же могу взять какую-нибудь /112 из /64 даваемой хостером и раздать по OpenVPN своему ноуту и прочим личным машинам, тем самым получив настоящий ipv6, а не адрес от брокеров.

 Решил, значит надо делать. Выбрал для этого виртуалку от vultr, на которой у меня ничего изначально не было и которая предназначена была для тестов и взялся за настройку.

Vultr выдает виртуалкам сети /64, в нашем примере пусть это будет сеть 2001:NNNN:NNNN:NNNN::/64, из нее мы возьмем «маленький» кусочек /112, который и будем раздавать своим компам, пусть это будет 2001:NNNN:NNNN:NNNN:80::/112. Процедуру генерации ключей для OpenVPN я описывать не буду, она достаточно подробно описана в других руководствах, рассмотрю только конфиг и скрипты, которые будут использоваться для наших целей.

В файле /etc/openvpn/variables мы пропишем сеть и маску которые будем использовать, отсюда у нас это дело заберут скрипты:

    # Subnet
    prefix=2001:NNNN:NNNN:NNNN:80::
    # netmask
    prefixlen=112

Конфиг openvpn-сервера:

    # Listen port
    port 8149
     
    # Protocol
    proto udp
     
    # IP tunnel
    dev tap0
     
    # Master certificate
    ca ca.crt
     
    # Server certificate
    cert server.crt
     
    # Server private key
    key server.key
     
    # Diffie-Hellman parameters
    dh dh2048.pem
     
    # Allow clients to communicate with each other
    client-to-client
     
    # Client config dir
    client-config-dir /etc/openvpn/ccd
     
    # Run client-specific script on connection and disconnection
    script-security 2
    client-connect "/usr/bin/sudo -u root /etc/openvpn/server-clientconnect.sh"
    client-disconnect "/usr/bin/sudo -u root /etc/openvpn/server-clientdisconnect.sh"
     
    # Server mode and client subnets
    server 10.18.0.0 255.255.255.0
    server-ipv6 2001:NNNN:NNNN:NNNN:80::/112
    topology subnet
     
    # IPv6 routes
    push "route-ipv6 2001:NNNN:NNNN:NNNN::/64"
    push "route-ipv6 2000::/3"
    
    persist-key
    persist-tun 
    # Ping every 10s. Timeout of 120s.
    keepalive 10 120
     
    # Enable compression
    comp-lzo
     
    # User and group
    user vpn
    group vpn
     
    # Log a short status
    status openvpn-status.log
    verb 4  
    sndbuf 0
    rcvbuf 0

В конфиге у нас прописаны скрипты, которые будут запускаться при подключении и отключении клиента:

server-clientconnect.sh

    #!/bin/sh
    
    # Check client variables
    if [ -z "$ifconfig_pool_remote_ip" ] || [ -z "$common_name" ]; then
            echo "Missing environment variable."
            exit 1
    fi
    
    # Load server variables
    . /etc/openvpn/variables
    
    ipv6=""
    
    # Find out if there is a specific config with fixed IPv6 for this client
    if [ -f "/etc/openvpn/ccd/$common_name" ]; then
            # Get fixed IPv6 from client config file  
            ipv6=$(sed -nr 's/^.*ifconfig-ipv6-push[ \t]+([0-9a-fA-F\\:]+).*$/\1/p' "/etc/openvpn/ccd/$common_name")
    fi
    
    # Get IPv6 from IPv4
    if [ -z "$ipv6" ]; then
            ipp=$(echo "$ifconfig_pool_remote_ip" | cut -d. -f4)
            if ! [ "$ipp" -ge 2 -a "$ipp" -le 254 ] 2>/dev/null; then
                    echo "Invalid IPv4 part."
                    exit 1
            fi
            hexipp=$(printf '%x' $ipp)
            ipv6="$prefix$hexipp"
    fi
    
    # Create proxy rule
    /sbin/ip -6 neigh add proxy $ipv6 dev eth0

и server-clientdisconnect.sh

    #!/bin/sh
     
    # Check client variables
    if [ -z "$ifconfig_pool_remote_ip" ] || [ -z "$common_name" ]; then
            echo "Missing environment variable."
            exit 1
    fi
     
    # Load server variables
    . /etc/openvpn/variables
     
    ipv6=""
     
    # Find out if there is a specific config with fixed IPv6 for this client
    if [ -f "/etc/openvpn/ccd/$common_name" ]; then
            # Get fixed IPv6 from client config file  
            ipv6=$(sed -nr 's/^.*ifconfig-ipv6-push[ \t]+([0-9a-fA-F\\:]+).*$/\1/p' "/etc/openvpn/ccd/$common_name")
    fi
     
    # Get IPv6 from IPv4
    if [ -z "$ipv6" ]; then
            ipp=$(echo "$ifconfig_pool_remote_ip" | cut -d. -f4)
            if ! [ "$ipp" -ge 2 -a "$ipp" -le 254 ] 2>/dev/null; then
                    echo "Invalid IPv4 part."
                    exit 1
            fi
            hexipp=$(printf '%x' $ipp)
            ipv6="$prefix$hexipp"
    fi
     
    # Delete proxy rule
    /sbin/ip -6 neigh del proxy $ipv6 dev eth0

Как можно было увидеть в конфиге сервера запускать мы его будем под пользоателем vpn, а потому нам надо добавить пользователя

    # useradd vpn

и разрешить этому пользователю sudo на наши скрипты добавив в /etc/sudoers(не правьте его руками открывая напрямую в редакторе, вызывайте visudo, что бы перед сохранением проверить корректность файла!):

    Defaults:vpn env_keep += "ifconfig_pool_remote_ip common_name"
    vpn ALL=NOPASSWD: /etc/openvpn/server-clientconnect.sh
    vpn ALL=NOPASSWD: /etc/openvpn/server-clientdisconnect.sh

Теперь включим ndp(Neighbor Discovery Protocol), который нам необходим для того, что бы наши хосты находили по IPv6 друг друга и что бы они были доступны из интернета по своим адресам, добавив в /etc/sysctl.conf(или в отдельный файл в /etc/sysctl.d/, по вашему желанию) строки:

    net.ipv6.conf.all.forwarding=1
    net.ipv6.conf.all.proxy_ndp=1

и выполнив

    # sysctl -p

Настроим адреса для отдельной машины создав файл с именем хоста который будет подключаться(имя должно совпадать с именем использовавшемся при создании сертификата для машины), пусть это будет abyrvalg-laptop в /etc/openvpn/ccd

/etc/openvpn/ccd/abyrvalg-laptop

    ifconfig-push 10.18.0.101 255.255.255.0
    ifconfig-ipv6-push 2001:NNNN:NNNN:NNNN:80::1001/112 2001:NNNN:NNNN:NNNN:80::1

Первым из IPv6-адресов указан адрес который будет выдаваться хосту, второй адрес его гейта.

Сервер готов, давайте напишем конфиг для клиента:

abyrvalg-laptop.conf

    # Client mode
    client
     
    # IPv6 tunnel
    dev tap
     
    # TCP protocol
    proto udp
     
    # Address/Port of VPN server
    remote SERVER_IP 8149
     
    # Don't bind to local port/address
    nobind
     
    # Don't need to re-read keys and re-create tun at restart
    persist-key
    persist-tun
     
    # User/Group
    ;user nobody
    ;group nobody
     
    # Remote peer must have a signed certificate
    remote-cert-tls server
    ns-cert-type server
     
    # Enable compression
    comp-lzo
    ca ca.crt
    cert abyrvalg-laptop.crt
    key abyrvalg-lapto`enter code here`p.key
    sndbuf 0
    rcvbuf 0

И попробуем в ручном режиме запустить сервер и клиента для тестов. Я исхожу из того, что конфиг-файл сервера и файлы сертификатов лежат у вас в /etc/openvpn на сервере и конфиг-файл для клиента вместе с сертификатами лежат в /etc/openvpn на клиенте, конфиги при этом именуются server.conf на сервере и ipv6.conf на клиенте

На сервере делаем:

    # cd /etc/openvpn
    # openvpn ./server.conf

на клиенте

    # cd /etc/openvpn
    # openvpn ./ipv6.conf

Если все сделано правильно, то на клиенте команда ip -6 a s dev tap0 покажет нам что-то типа

    tap0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 state UNKNOWN qlen 100
        inet6 2001:NNNN:NNNN:NNNN:80::1001/112 scope global

и  `ping6 -c 4 ipv6.google.com`
покажет:
 

    PING ipv6.google.com(lr-in-x8a.1e100.net) 56 data bytes
    64 bytes from lr-in-x8a.1e100.net: icmp_seq=1 ttl=46 time=110 ms
    64 bytes from lr-in-x8a.1e100.net: icmp_seq=2 ttl=46 time=113 ms
    64 bytes from lr-in-x8a.1e100.net: icmp_seq=3 ttl=46 time=110 ms
    64 bytes from lr-in-x8a.1e100.net: icmp_seq=4 ttl=46 time=110 ms
    
    --- ipv6.google.com ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3001ms
    rtt min/avg/max/mdev = 110.586/111.367/113.285/1.183 ms
Вуаля! У нас теперь на нашем ноутбуке или стационарнике есть нормальный IPv6, без туннельных брокеров.

Для добавления старта openvpn при загрузке на ваших системах используйте штатные средства, в зависимости от дистрибутива они могут различаться.

При настройке и написании использована <a href="https://techblog.synagila.com/2016/02/24/build-a-openvpn-server-on-ubuntu-to-provide-a-ipv6-tunnel-over-ipv4/">данная</a> статья, там же вы можете посмотреть на примеры генерации ключей для OpenVPN, если до этого их не генерировали. В отличии от оригинальной статьи я использую udp, а не tcp и tap, а не tun-девайсы.
