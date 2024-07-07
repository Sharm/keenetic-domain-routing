# Предварительные условия
Вы сделали все, что описано в [Перенаправление TCP трафика на внешний Shadowsocks сервер](SHADOWSOCKS_TCP.md)

# Поехали
Нельзя просто так взять и проксировать UDP, поэтому просто поддержки UDP у Shadowsocks недостаточно. Найти реальный адрес назначения в UDP возможности нет. Ну, почти нет. В линуксе для этого придумали специальный тип сокетов, Transparent Proxy, ключевые статьи для понимания:
- https://lwn.net/Articles/252545/
- https://www.kernel.org/doc/Documentation/networking/tproxy.txt
- https://powerdns.org/tproxydoc/tproxy.md.html

Ключевой момент - мы не меняем на самом деле dst адрес у пакета, а заворачиваем его роутом на localhost. После этого iptables заворачивает все пакеты, идущие на localhost с меткой на порт Transparent Proxy. Выглядит, конечно, как костыль, но работает.

## Установка модуля iptables TPROXY
Модуль нужен для перенаправления на специальные Transparent сокеты. Проверьте содержимое `/lib/modules/$(uname -r)/`, если там нет `xt_TPROXY.ko`, значит нужно доставить модули средствами Keenetic. Для этого нужно из интерфейса установить "Протокол IPv6" и "Модули ядра подсистемы Netfilter". Причем пакет "Модули ядра подсистемы Netfilter" появятся в интерфейсе только после установки "Протокол IPv6", т.е. перезагрузииться придется дважды. Если соберетесь использовать еще более широкие возможности iptables, можно установить еще "Пакет расширения Xtables-addons для Netfilter", появится только после установки первых двух. Но для TPROXY он не нужен.
   
## Настройка shadowsocks клиента на поддержку UDP

Редактируем файл `/opt/etc/shadowsocks.json` месяем параметр `mode`
```json
{
 "mode":"tcp_and_udp",
}
```
И перезагружаем shadowsocks `/etc/init.d/S22shadowsocks restart`

## Настройка роутинга
Создаем файл `/opt/etc/init.d/S55_ip_route_tproxy.sh` (`S` в начале имени файла - важно, означает, что скрипт нужно будет запустить с параметром start)
```bash
#!/bin/sh
[ "$1" != "start" ] && exit 0

ip route add local default dev lo table 100
ip rule add fwmark 1 lookup 100

logger "ip route for TPROXY configured"
```
Разберемся что происходит. Выполняется всего 2 простые команды, первая заворачивает все, направленное на `0.0.0.0/0` на localhost и записывает это правило в таблицу 100. Вторая команда говорит о том, что если пакет помечен `fwmark 1`, то должна использоваться таблица маршрутизации 100. Далее средствами iptables мы будем помечать пакеты.

**Полезное для диагностики:**
- `ip rule` - выведет список всех правил. Правила упорядочены по приоритету. Убедитесь, что таблицы `main` и `default` имеют приоритет ниже, чет таблица 100, в противном случае, несмотря на маркерованный пакет, будет использована таблица `main`.
- `ip route list table 100` - выведет правила пмаршрутизации конкретной таблицы.

## Настройка переадресации iptables
Базовые принципы работы iptables [Описаны тут](https://interface31.ru/tech_it/2020/02/osnovy-iptables-dlya-nachinayushhih-chast-1.html). Хотя есть один нюанс, не описанный в этой статье. После цепочки OUTPUT (локальный трафик) будет произведен re-routing, если были изменения в метках MARK пакета.

Создаем файл `/opt/root/iptables_mangle.sh`
```bash
#!/bin/sh

[ -n "$(iptables-save | grep SS_UDP)" ] && exit 0

[ -z "$(lsmod | grep xt_TPROXY)" ] && \
    insmod /lib/modules/$(uname -r)/xt_TPROXY.ko

iptables -t mangle -N SS_UDP_LOCAL
iptables -t mangle -N SS_UDP_TRANSIT

iptables -t mangle -A SS_UDP_TRANSIT -p udp -m mark --mark 0xffffd01 -m set --match-set unblock dst -j TPROXY --on-port 1080 --tproxy-mark 1
iptables -t mangle -A SS_UDP_TRANSIT -i lo -p udp -d 1.1.1.1 --dport 53 -j TPROXY --on-port 1080 --tproxy-mark 1
iptables -t mangle -A SS_UDP_TRANSIT -i lo -p udp -d 8.8.8.8 --dport 53 -j TPROXY --on-port 1080 --tproxy-mark 1

iptables -t mangle -A SS_UDP_LOCAL -p udp -d 1.1.1.1 --dport 53 -j MARK --set-mark 1
iptables -t mangle -A SS_UDP_LOCAL -p udp -d 8.8.8.8 --dport 53 -j MARK --set-mark 1

iptables -t mangle -A PREROUTING -j SS_UDP_TRANSIT
iptables -t mangle -A OUTPUT -j SS_UDP_LOCAL
```

Давайте разберемся. Меняем таблицу mangle. Скрипт будет запускаться многократно из-за нюансов работы Keenetic, он очень часто сбрасывает iptables таблицы к исходному состоянию по своим внутренним причинам. Поэтому проверяем, существует ли наша цепочка в правилах. Включаем модуль `xt_TPROXY.ko`, если еще не включен.  Далее создаем цепочки SS_UDP_LOCAL и SS_UDP_TRANSIT. **Важно!** в PREROUTING попадают только транзитные пакеты и затем в цепочку SS_UDP_TRANSIT. Сгенерированные на роутере пакеты попадут в OUTPUT и затем в SS_UDP_LOCAL. Транзитные UDP пакеты помечаем и перенаправляем на порт Shadowsocks. Пометка будет использоваться в роутинге на localhost.
Перенаправление происходит по нескольким условиям
1. `-m mark --mark 0xffffd01`
   Это метка, которая ставится роутером в зависимости от нахождения хоста-инициатора пакета в политике доступа в интернет ("приоритеты подключений" в интерфейсе Keenetic). Таким образом, мы можем управлять для каких клиентов включить такое перенаправление. Чтобы узнать метку, сделайте `iptables-save | grep MARK` и посмотрите каким IP адресам какие метки присваиваются, по ним можно будет понять у какой группы какая метка.
2. `-m set --match-set unblock dst`
   IP адрес назначения соответствует списку IP адресов "unblock". Этот ipset будет постоянно обновляться нашим DNS сервером на лету.
   
Перенаправления для 1.1.1.1 и 8.8.8.8 имеют следующий смысл. Мы хотим, чтобы DNS запросы на указанные в dnsmasq домены проходили через прокси. Для этого необходимо 2 вещи:
1. Перенаправить DNS запросы от dnsmasq в сторону upstream серверов на localhost. Так как dnsmasq генерит запросы локально, это цепочка OUTPUT и SS_UDP_LOCAL. Они помечаются, после чего произойдет re-routing таких пакетов.
2. После такого перенаправления пакеты попадут в PREROUTING как транзитные, но с локального хоста. Их мы отправляем в Shadowsocks. Таким образом, локально сгенерированные DNS запросы к 1.1.1.1 и 8.8.8.8 пойдут через прокси, а транзитные (если клиент делает запрос на 1.1.1.1) пойдут по основному соединению.

Теперь этот скрипт можно запустить вручную и проверить работу. Если что-то не так - можно перезагрузить роутер. **В автозагрузку добавлять только после того как все протестировано!** Для добавления в автозагрузку (формально это не совсем автозагрузка, скрипт вызывается роутером в момент очистки и перезагрузки определенной таблицы iptables, таблица nat и mangle перезагружается роутером многократно) создадим файл `/opt/etc/ndm/netfilter.d/iptables_mangle_reload.sh` с содержимым:
```bash
#!/bin/sh

[ "$type" == "ip6tables" ] && exit 0   # check the protocol type in backward-compatible way
[ "$table" == "mangle" ] || exit 0   # check the table name

logger "iptables $table reloaded: load /opt/root/iptables_mangle.sh"

/opt/root/iptables_mangle.sh
```

**Полезное для диагностики:**
- `iptables-save | grep SS_UDP` - покажет все добавленные вами правила
- `nslookup example.com 192.168.0.1`
- `dig @localhost example.com`
- `tcpdump -i lo -p udp`
