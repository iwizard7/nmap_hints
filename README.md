# Nmap hints ⚙️
```
nmap -sL 192.168.10.0/24 - выводит список обнаруженных узлов (IP адреса всех активных узлов в подсети с адресами от 192.168.10.1 до 192.168.10.254).
nmap -p80,443 192.168.10.10-20 - исследует диапазон IP адресов в поисках открытых портов 80 и 443.
nmap -p T:80,8080,6588,800 172.16.0.1/22 - исследует все узлы с адресами от 172.16.0.1 до 172.16.3.254 в поисках открытых TCP портов 80,8080,6588 и 800 
nmap -sP 192.168.10.10,20 - проводит пинг-исследование двух узлов
nmap -PN 192.168.10.0/29 - исследует все узлы в диапазоне адресов от 192.168.10.1 до 192.168.10.6. Иногда межсетевые экраны на серверах не пропускают пинг-запросы, что усложняет процесс. В этих случаях полезен параметр -PN, позволяющий исследовать узлы, считая, что они активны.
nmap -A -F 192.168.10.1 - определяет операционную систему и службы, выполняющиеся на устройстве.
```
## Техника "Idlescan"
```
nmap -v -sI 192.168.10.100 192.168.10.105
```
В данном случае исследуется система с IP адресом 192.168.10.105 с имитацией того, что источником пакетов является другой узел; в логах исследуемой системы будет зафиксирован адрес 192.168.10.100. Данный узел называется зомби-узлом (zombie host).
В контексте сетей, зомби-узлами называются узлы, контролируемые другими узлами в сети. Не все узлы могут быть использованы как зомби, так как для этого необходимо выполнить ряд условий. (Найти зомби-узел в сети можно при помощи таких пакетов, как hping.) Параметр -v позволяет выводить более подробные сообщения в процессе работы программы.

## Техника узлов-ловушек "Decoy host"
```
nmap -sS -P0 -D 192.168.10.201,192.168.10.202,192.168.10.203 192.168.10.50
```
Данная команда особенно полезна во время тестирования IDS/IPS (системы обнаружения и предотвращения вторжений). При помощи параметра -sS производится SYN-исследование устройства. В процессе происходит подмена содержимого пакетов для того, чтобы устройство считало источниками пакетов узлы, заданные параметром -D. Параметры -s1 и -D не могут быть использованы одновременно по понятным причинам.

Теперь небольшое предупреждение: будьте осторожны, чтобы не допустить непреднамеренной атаки отказа в обслуживании при использовании параметра -D. Для понимания того, как это может произойти, необходимо знать принцип действия трехэтапного рукопожатия TCP (TCP handshake). TCP - протокол, обеспечивающий постоянное соединение с гарантированной доставкой пакетов, использует трехэтапное рукопожатие:
- Клиент инициирует соединение, отправляя серверу сегмент SYN
- Сервер отвечает сегментами SYN-ACK
- Клиент отвечает сегментом ACK, после чего соединение считается установленным и может производиться обмен данными

Если параметр -D используется и присутствует работающее устройство по адресу узла-ловушки, то сегменты SYN-ACK достигают устройства с IP-адресом узла-ловушки, а не устройства с запущенной программой Nmap. Поскольку устройство с адресом узла-ловушки не инициировало соединение, оно закрывает его, отправляя сегмент RST. Здесь нет никаких проблем.

Тем не менее, проблема появляется, если устройство с адресом узла-ловушки не активно в сети - сегмент RST не отправляется исследуемому устройству, котораое держит соединение открытым. Так как Nmap продолжает генерировать запросы с адресом узла-ловушки, как источника пакетов, количество открытых соединений в стадии "соединение инициировано" растет. Это увеличивает потребление ресурсов исследуемым устройством и может привести к отказу в обслуживании, в том числе и соединений с другими узлами.

## TCP-исследования
```
nmap -sS 192.168.100.100 Пример командной строки для осуществления SYN-исследования, возвращающего список открытых TCP-портов
nmap -sP -n -oG hostlist 192.168.100.0/24 && cut -d " "-f2 hostlist > iplist команда позволяет получить список активных IP-адресов в диапазоне 192.168.100.0/24 iplist - список всех активных IP-адресов в заданном диапазоне
nmap -sU 192.168.100.100 Пример командной строки для осуществления UDP-исследования, возвращающий список открытых/закрытых/открытых/фильтруемых UDP-портов
nmap -sT 192.168.100.100 Пример TCP-иследования. Недостатком данного типа исследования является то, что он использует больше ресурсов, чем TCP SYN-исследование, так как проводится полная процедура открытия TCP-соединения с его последующим сбросом. Также при использовании данного типа остаются записи в журнале событий исследуемого устройства.
nmap -v -sV 192.168.100.100 получения версий служб, выполняющихся на открытых портах исследуемого устройства. позволяет вывести список открытых портов исследуемого устройства вместе с версиями всех системных служб, работающих на этих портах.
nmap -v -O 192.168.100.100 определить операционную систему, под управлением которой работает устройство.
```
Тип исследования TCP SYN, оставляет много следов в системном журнале исследуемого устройства, позволяя обнаружить адрес узла с запущеной программой Nmap. Некоторые узлы с системами защиты от проникновения (IDS) и межсетевыми экранами следят за SYN-сегментами, отправленными на определенные порты. Как же обойти эту проблему при тестировании на возможность проникновения в систему FIN-исследования!

## Работа со скриптами
```
nmap-script sshv1 -iL IPList.txt -osshv1.txt	Команда для выплнения только одного сценария, sshv1.
nmap-script sniffer-detect -iL IPList.txt -osniffer-detect.txt	Запуск идентификации сниффера.
nmap-script smb-enum-users -iL IPList.txt -osmb-enum-users.txt	Запуск исследования адресов из файла IPList.txt в поисках пользователей SMB

```
## Полезные ссылки 
https://rus-linux.net/MyLDP/admin/nmap/nmap-5-scanning-firewalls.html
https://rus-linux.net/MyLDP/admin/nmap/nmap-6-scanning-firewalls-continued.html
https://rus-linux.net/MyLDP/admin/nmap/nmap-7-script-scanning.html


