# Предварительные условия
Вы поняли и попробовали все, что описано в [Перенаправление TCP и UDP трафика на внешний Shadowsocks сервер](SHADOWSOCKS_TCP_UDP.md). Инструкция будет на основе этой статьи и очевидные вещи повторяться не будут.

# Идея
Перенаправлять весь трафик с одной из Policy Group Keenetic в SS + локальный трафик самого роутера. Так, чтобы не было утечек мимо SS. Таким образом можно перенаправить только TCP и UDP трафик, остальное будет обтрасываться. Это может не подойти в некоторых ситуациях, так что, конечно, не заменяет VPN. Зато работает максимально быстро и покрывает 99% потребностей обычных пользователей, которые не используют более специализированные протоколы.

# Поехали
Настраиваем Shadowsocks клиент в режиме ss-redir, TCP and UDP как описано в [Перенаправление TCP и UDP трафика на внешний Shadowsocks сервер](SHADOWSOCKS_TCP_UDP.md), можно опустить все что связано с dnsmasq, т.к. пока перенаправление в зависимости от доменного имени использовать не будем. А также можно опустить правила iptables, они будут другие.

[Настраиваем роутинг для TPROXY UDP](https://github.com/Sharm/keenetic-domain-routing/blob/master/SHADOWSOCKS_TCP_UDP.md#%D0%BD%D0%B0%D1%81%D1%82%D1%80%D0%BE%D0%B9%D0%BA%D0%B0-%D1%80%D0%BE%D1%83%D1%82%D0%B8%D0%BD%D0%B3%D0%B0)

## Настройка переадресации iptables
Создаем файл `/opt/root/iptables_nat.sh`. После создания конечно даем права на исполнение, про это не буду далее напоминать.
```bash
#!/bin/sh

[ -n "$(iptables-save | grep SS_TCP)" ] && exit 0

# Create new chain
iptables -t nat -N SS_TCP
iptables -t nat -N SS_TCP_LOCAL

# Anything else should be redirected to shadowsocks's local port
iptables -t nat -A SS_TCP -p tcp -m mark --mark 0xffffd01 -j REDIRECT --to-ports 1080

# Ignore your shadowsocks server's addresses
iptables -t nat -A SS_TCP_LOCAL -p tcp -d xx.xx.xx.xx --dport 443 -j RETURN

iptables -t nat -A SS_TCP_LOCAL -o eth3 -p tcp -j REDIRECT --to-ports 1080

# Apply the rules to transit packets
iptables -t nat -A PREROUTING -p tcp -j SS_TCP
iptables -t nat -A OUTPUT -p tcp -j SS_TCP_LOCAL
```
В отличие от прошлых правил, появилась цепочка SS_TCP_LOCAL из OUTPUT. Теперь мы гоним в SS не только транзитный трафик, но и со всех процессов самого роутера. Под эти правила будет попадать в том числе трафик из самого ss-redir, поэтому важно игнорировать SS сервер. Замените `xx.xx.xx.xx` на адрес вашего сервера. Также обратите внимение на `-o eth3`, это интерфейс провайдера, возможно вам следует заменить на другое имя. Важно, чтобы правило не применялось для пакетов, идущих между локальными сегментами сети.

Создаем файл `/opt/root/iptables_mangle.sh`
```bash
#!/bin/sh

[ -n "$(iptables-save | grep SS_UDP)" ] && exit 0

[ -z "$(lsmod | grep xt_TPROXY)" ] && \
    insmod /lib/modules/$(uname -r)/xt_TPROXY.ko

iptables -t mangle -N SS_UDP_LOCAL
iptables -t mangle -N SS_UDP_TRANSIT

iptables -t mangle -A SS_UDP_TRANSIT -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A SS_UDP_TRANSIT -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A SS_UDP_TRANSIT -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A SS_UDP_TRANSIT -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A SS_UDP_TRANSIT -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A SS_UDP_TRANSIT -d 240.0.0.0/4 -j RETURN

iptables -t mangle -A SS_UDP_TRANSIT -p udp -m mark --mark 0xffffd01 -j TPROXY --on-port 1080 --tproxy-mark 1
iptables -t mangle -A SS_UDP_TRANSIT -i lo -p udp -m mark --mark 1 -j TPROXY --on-port 1080 --tproxy-mark 1

iptables -t mangle -A SS_UDP_LOCAL -p udp -d xx.xx.xx.xx --dport 443 -j RETURN

# Important to select provider interface (eth3). Otherwise VPN shouldn't work.
iptables -t mangle -A SS_UDP_LOCAL -o eth3 -p udp -j MARK --set-mark 1

iptables -t mangle -A PREROUTING -j SS_UDP_TRANSIT
iptables -t mangle -A OUTPUT -j SS_UDP_LOCAL
```
Тоже самое, в отличие от прошлый правил, гоним **весь** локальный UDP через SS, а не только DNS. `-o eth3` - важно, чтобы не испортить VPN трафик между другими интерфейсами, а также локальный. В SS_UDP_TRANSIT вынуждены локальный трафик игнорировать явно, потмоу что в PREROUTING еще нет информации о выходном интерфейсе, это станет понятно только после роутинга.

Добавляем оба скрипта в автозагрузку `/opt/etc/ndm/netfilter.d/`, как было описано ранее.

## Настройка переадресации iptables - защита от утечек
Мы разобрались с UDP и TCP трафиком, но что с остальными? Нужно явно фильтровать его.

Создаем файл `/opt/root/iptables_filter.sh`
```bash
#!/bin/sh

[ -n "$(iptables-save | grep SS_FLT)" ] && exit 0

iptables -t filter -N SS_FLT

# Other VPN connections and lan should work
iptables -t filter -A OUTPUT -o eth3 -j SS_FLT
iptables -t filter -I FORWARD -o eth3 -m mark --mark 0xffffd01 -j SS_FLT

iptables -t filter -A SS_FLT -p udp -d xx.xx.xx.xx --dport 443 -j RETURN
iptables -t filter -A SS_FLT -p tcp -d xx.xx.xx.xx --dport 443 -j RETURN
iptables -t filter -A SS_FLT -m mark --mark 1 -j RETURN
iptables -t filter -A SS_FLT -p tcp -m tcp --dport 1080 -j RETURN # accept local generated packets to outside, ex wget ya.ru
iptables -t filter -A SS_FLT -p all -j REJECT
```
Явно разрешаем только соединения на SS сервер и помеченные пакеты. Помеченные - это UDP трафик, идущий через TPROXY, у них будет оригинальный destination и по IP такие пакеты поймать невозможно. Все остальное REJECT. Обратите внимание на `iptables -t filter -I FORWARD` -I добавит правило в начало цепочки. Обычно на Keenetic FORWARD цепочка имеет policy DROP и содержит правила на ACCEPT, после срабатывания которых дальше уже проверка не пойдет. Поэтому надо нашу цепочку поставить **до** стандартных правил.

Теперь этот скрипт можно запустить вручную и проверить работу. Если что-то не так - можно перезагрузить роутер. **В автозагрузку добавлять только после того как все протестировано!** Для добавления в автозагрузку создадим файл `/opt/etc/ndm/netfilter.d/iptables_filter_reload.sh` с содержимым:
```bash
#!/bin/sh

[ "$type" == "ip6tables" ] && exit 0   # check the protocol type in backward-compatible way
[ "$table" == "filter" ] || exit 0   # check the table name

logger "iptables $table reloaded: load /opt/root/iptables_filter.sh"

/opt/root/iptables_filter.sh
```

## Проверка
Как минимум теперь нужно проверить, что `ping` до интернет узлов не работат. Как с локальных устройств, так и с самого роутера.
