# Фильтрация трафика

## Задачи

1. реализовать knocking port

\- centralRouter может попасть на ssh inetrRouter через knock скрипт
пример в материалах

2. добавить inetRouter2, который виден(маршрутизируется (host-only тип сети для виртуалки)) с хоста или форвардится порт через локалхост

3. запустить nginx на centralServer

4. пробросить 80й порт на inetRouter2 8080

5. дефолт в инет оставить через inetRouter

\* реализовать проход на 80й порт без маскарадинга

---

## Выполнение

1. Произвожу настройку согласно примеру в статье <https://wiki.archlinux.org/index.php/Port_knocking#Simple_port_knocking>. Сценарий для iptables подгружается командой iptables-restore из файла сценария, который синхронизируется из каталога с Vagrantfile в директорию /vagrant на виртуалках.

2. Добавляю inetRouter2. Прописываю ему дополнительный интерфейс (eth2 192.168.11.121), чтобы обеспечить доступность с хостовой машины для проверки проброса запросов до nginx на centralServer.

3. Установлен nginx из репозитория epel-release. Модицифирован дефолтный index.html, сервис запущен на прослушивание tcp:80.

4. Проброшен порт 8080 с хоста inetRouter2 на 80 порт хоста centralServer без использования MASQUERADE следующими командами:

```
iptables -A FORWARD -d 192.168.0.2/32 -p tcp -m tcp --dport 80 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -t nat -A PREROUTING -i eth2 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
```

Для того, чтобы centralRouter знал, куда возвращать пакеты при запросах с хостовой машины, необходим дополнительный маршрут:

```
[root@centralRouter ~]# ip ro | grep 192.168.11
192.168.11.0/24 via 192.168.245.1 dev eth2
```

5. Дефолт наружу оставлен через inetRouter


### Проверка

1. Проверяю port-knocking

По дефолту:

```
[root@centralRouter ~]# ssh vagrant@192.168.255.1
ssh: connect to host 192.168.255.1 port 22: Connection timed out
```

Применяю скрипт knock.sh и пробую еще раз:

```
[root@centralRouter ~]# /vagrant/knock.sh 192.168.255.1 8881 7777 9991

Starting Nmap 6.40 ( http://nmap.org ) at 2020-06-13 17:27 MSK
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00034s latency).
PORT     STATE    SERVICE
8881/tcp filtered unknown
MAC Address: 08:00:27:52:22:8F (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.37 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2020-06-13 17:27 MSK
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00026s latency).
PORT     STATE    SERVICE
7777/tcp filtered cbt
MAC Address: 08:00:27:52:22:8F (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.37 seconds

Starting Nmap 6.40 ( http://nmap.org ) at 2020-06-13 17:27 MSK
Warning: 192.168.255.1 giving up on port because retransmission cap hit (0).
Nmap scan report for 192.168.255.1
Host is up (0.00026s latency).
PORT     STATE    SERVICE
9991/tcp filtered issa
MAC Address: 08:00:27:52:22:8F (Cadmus Computer Systems)

Nmap done: 1 IP address (1 host up) scanned in 0.37 seconds
[root@centralRouter ~]# ssh vagrant@192.168.255.1
vagrant@192.168.255.1's password:
Last login: Sat Jun 13 17:05:24 2020 from 192.168.255.2
[vagrant@inetRouter ~]$
```

#### Примечания

* В моем примере окно на пропуск соединения сокращено до 15 секунд
* Комбинация портов использована в качестве примера из статьи без изменений. На проде комбинация должна быть другой!


2. С хостовой машины проверяю проброс до nginx

```
zradeg@homeBook:~$ curl 192.168.11.121:8080
Mapping from inetRouter2:8080 to centralServer:80 is ok!
```

3. Проверяю дефолтный маршрут наружу:

```
[root@centralServer ~]# traceroute -I ya.ru
traceroute to ya.ru (87.250.250.242), 30 hops max, 60 byte packets
 1  gateway (192.168.0.1)  0.219 ms  0.149 ms  0.111 ms
 2  192.168.255.1 (192.168.255.1)  0.479 ms  0.526 ms  0.459 ms
 3  * * *
 4  192.168.88.1 (192.168.88.1)  2.537 ms  2.869 ms  2.832 ms
 5  172.16.253.52 (172.16.253.52)  2.875 ms  2.929 ms  2.880 ms
 6  172.16.253.244 (172.16.253.244)  4.290 ms  2.862 ms  3.140 ms
 7  atlant.asr.intelsc.net (188.191.160.129)  3.099 ms  2.561 ms  2.754 ms
 8  77.94.162.185 (77.94.162.185)  3.520 ms  3.488 ms  3.540 ms
 9  ae1-atlant-mmts9-msk.naukanet.ru (77.94.160.53)  4.392 ms  4.458 ms  4.410 ms
10  styri.yndx.net (195.208.208.116)  5.243 ms  5.170 ms  5.212 ms
11  ya.ru (87.250.250.242)  8.007 ms  8.254 ms  7.503 ms
```
---

### Итоги
1. Port-knocking работает
2. Проброс с inetRouter2:8080 до nginx на centralServer:80 работает без использования MASQUERADE
3. Дефолтный маршрут наружу остался через inetRouter