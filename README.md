# VPN

## Задание

1. Настроить **VPN** между двумя виртуальными машинами в **tun/tap** режимах, замерить скорость в туннелях, сделать вывод об отличающихся показателях.
2. Поднять **RAS** на базе **OpenVPN** с клиентскими сертификатами.
3. Самостоятельно изучить и настроить **ocserv**.

## Реализация

Задание сделано на **rockylinux/9** версии **v4.0.0**. Для автоматизации процесса написан **Ansible Playbook** [playbook.yml](playbook.yml) который последовательно запускает следующие роли:

- **epel_release** - добавляет репозиторий **Extra Packages for Enterprise Linux (EPEL)**.
- **iperf3** - устанавливает **iperf3**.
- **openvpn_key** - генерит секретный ключ **static.key** для **OpenVPN** и разливает его на сервера (сам ключ сохраняется в директории [passwords](passwords)).
- **tls_ca** - генерит корневой сертификат центра сертификации по переменным в [all.yml](group_vars/all.yml), который используется для подписи всех остальных сертификатов, используемых для подключения к **VPN** (сертификат и ключ сохраняются в директории [passwords](passwords)).
- **tls_cert** - генерит сертификат для сервера и клиента согласно заданным в [all.yml](group_vars/all.yml), [client.yml](host_vars/client.yml), [server.yml](host_vars/server.yml) переменным.
- **dhparams** - генерит параметры **OpenSSL Diffie-Hellman** для **OpenVPN** и **OCServ**.
- **openvpn** - устанавливает и настраивает **openvpn** сервер и клиент по переменным указанным в файлах [client.yml](host_vars/client.yml), [server.yml](host_vars/server.yml) и шаблонам конфигурации в директории [templates](roles/openvpn/templates) роли.
- **ocserv** - устанавливает и настраивает **ocserv** по переменным указанным в [server.yml](host_vars/server.yml) и шаблону [ocserv.conf](roles/ocserv/templates/ocserv.conf).
- **openconnect** - устанавливает и настраивает **openconnect** по переменным указанным в [client.yml](host_vars/client.yml) и шаблону [default.conf](roles/openconnect/templates/default.conf), шаблон сервиса задан в [openconnect.service](roles/openconnect/templates/openconnect.service).

**OpenVPN** по умолчанию использует шифр **AES-256-GCM**, которые не может использоваться для соединений с общим секретным ключом в профилях **tun** и **tap**, поэтому в них он заменён на **AES-256-CBC**. Также из настроек **OpenVPN** исключено сжатие, которое и так отключено в последних версиях из соображений безопасности (и вызывает только лишние предупреждения при запуске сервиса). Также исключена директива **route**, так как сеть **192.168.56.0/24** доступна локально для хоста и всех виртуальных машин.

## Запуск

На узле, где запускается **Ansible** (хосте) необходимо установить пакет **python3-cryptography**.

Необходимо скачать **VagrantBox** для **rockylinux/9** версии **v4.0.0** и добавить его в **Vagrant** под именем **rockylinux/9/v4.0.0**. Сделать это можно командами:

```shell
curl -OL https://app.vagrantup.com/rockylinux/boxes/9/versions/4.0.0/providers/virtualbox/amd64/vagrant.box
vagrant box add vagrant.box --name "rockylinux/9/v4.0.0"
rm vagrant.box
```

Для того, чтобы **vagrant 2.3.7** работал с **VirtualBox 7.1.0** необходимо добавить эту версию в **driver_map** в файле **/usr/share/vagrant/gems/gems/vagrant-2.3.7/plugins/providers/virtualbox/driver/meta.rb**:

```ruby
          driver_map   = {
            "4.0" => Version_4_0,
            "4.1" => Version_4_1,
            "4.2" => Version_4_2,
            "4.3" => Version_4_3,
            "5.0" => Version_5_0,
            "5.1" => Version_5_1,
            "5.2" => Version_5_2,
            "6.0" => Version_6_0,
            "6.1" => Version_6_1,
            "7.0" => Version_7_0,
            "7.1" => Version_7_0,
          }
```

После этого нужно сделать **vagrant up**.

Протестировано в **OpenSUSE Tumbleweed**:

- **Vagrant 2.3.7**
- **VirtualBox 7.1.4_SUSE r165100**
- **Ansible 2.17.5**
- **Python 3.11.10**
- **Jinja2 3.1.4**

## Проверка

Подключимся у узлу **server** и проверим интерфейсы и слушаемые порты:

```text
[vagrant@server ~]$ ip -brief add
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             10.0.2.15/24 fd00::a00:27ff:fefc:e996/64 fe80::a00:27ff:fefc:e996/64
eth1             UP             192.168.56.10/24 fe80::a00:27ff:fe9b:cb9d/64
tap0             UNKNOWN        10.10.10.1/24 fe80::ccc6:21ff:fe93:a17f/64
tun0             UNKNOWN        10.10.11.1/24 fe80::1a2b:532f:4bd:7a30/64
tun1             UNKNOWN        10.10.12.1/24 fe80::84:fe81:e56:4b57/64
vpns1            UNKNOWN        10.10.13.1 peer 10.10.13.176/32 fe80::ce7a:d836:3448:7c05/64

[vagrant@server ~]$ sudo ss -tupln
Netid   State    Recv-Q   Send-Q     Local Address:Port      Peer Address:Port   Process
udp     UNCONN   0        0                0.0.0.0:111            0.0.0.0:*       users:(("rpcbind",pid=533,fd=5),("systemd",pid=1,fd=58))
udp     UNCONN   0        0                0.0.0.0:1194           0.0.0.0:*       users:(("openvpn",pid=30562,fd=5))
udp     UNCONN   0        0                0.0.0.0:1195           0.0.0.0:*       users:(("openvpn",pid=30724,fd=5))
udp     UNCONN   0        0                0.0.0.0:1207           0.0.0.0:*       users:(("openvpn",pid=30880,fd=6))
udp     UNCONN   0        0              127.0.0.1:323            0.0.0.0:*       users:(("chronyd",pid=595,fd=5))
udp     UNCONN   0        0                0.0.0.0:443            0.0.0.0:*       users:(("ocserv-main",pid=30419,fd=8))
udp     UNCONN   0        0                   [::]:111               [::]:*       users:(("rpcbind",pid=533,fd=7),("systemd",pid=1,fd=60))
udp     UNCONN   0        0                  [::1]:323               [::]:*       users:(("chronyd",pid=595,fd=6))
udp     UNCONN   0        0                   [::]:443               [::]:*       users:(("ocserv-main",pid=30419,fd=9))
tcp     LISTEN   0        128              0.0.0.0:22             0.0.0.0:*       users:(("sshd",pid=751,fd=3))
tcp     LISTEN   0        4096             0.0.0.0:111            0.0.0.0:*       users:(("rpcbind",pid=533,fd=4),("systemd",pid=1,fd=56))
tcp     LISTEN   0        1024             0.0.0.0:443            0.0.0.0:*       users:(("ocserv-main",pid=30419,fd=3))
tcp     LISTEN   0        128                 [::]:22                [::]:*       users:(("sshd",pid=751,fd=4))
tcp     LISTEN   0        4096                [::]:111               [::]:*       users:(("rpcbind",pid=533,fd=6),("systemd",pid=1,fd=59))
tcp     LISTEN   0        1024                [::]:443               [::]:*       users:(("ocserv-main",pid=30419,fd=7))
```

Тоже самое сделаем на клиенте:

```text
[vagrant@client ~]$ ip -brief add
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             10.0.2.15/24 fd00::a00:27ff:fefc:e996/64 fe80::a00:27ff:fefc:e996/64
eth1             UP             192.168.56.20/24 fe80::a00:27ff:fea6:49d9/64
tap0             UNKNOWN        10.10.10.2/24 fe80::94ba:7aff:fe97:8daf/64
tun0             UNKNOWN        10.10.11.2/24 fe80::c086:ecb9:1e8f:3d1e/64
tun1             UNKNOWN        10.10.12.2/24 fe80::2921:3971:a1b3:fe6e/64
tun2             UNKNOWN        10.10.13.176/32 fe80::f6d1:c711:e7f0:2178/64

[vagrant@client ~]$ sudo ss -tupln
Netid   State    Recv-Q   Send-Q     Local Address:Port      Peer Address:Port  Process
udp     UNCONN   0        0                0.0.0.0:50171          0.0.0.0:*      users:(("openvpn",pid=30104,fd=4))
udp     UNCONN   0        0                0.0.0.0:55309          0.0.0.0:*      users:(("openvpn",pid=29946,fd=4))
udp     UNCONN   0        0                0.0.0.0:43092          0.0.0.0:*      users:(("openvpn",pid=30261,fd=3))
udp     UNCONN   0        0                0.0.0.0:111            0.0.0.0:*      users:(("rpcbind",pid=533,fd=5),("systemd",pid=1,fd=100))
udp     UNCONN   0        0              127.0.0.1:323            0.0.0.0:*      users:(("chronyd",pid=595,fd=5))
udp     UNCONN   0        0                   [::]:111               [::]:*      users:(("rpcbind",pid=533,fd=7),("systemd",pid=1,fd=102))
udp     UNCONN   0        0                  [::1]:323               [::]:*      users:(("chronyd",pid=595,fd=6))
tcp     LISTEN   0        4096             0.0.0.0:111            0.0.0.0:*      users:(("rpcbind",pid=533,fd=4),("systemd",pid=1,fd=96))
tcp     LISTEN   0        128              0.0.0.0:22             0.0.0.0:*      users:(("sshd",pid=691,fd=3))
tcp     LISTEN   0        4096                [::]:111               [::]:*      users:(("rpcbind",pid=533,fd=6),("systemd",pid=1,fd=101))
tcp     LISTEN   0        128                 [::]:22                [::]:*      users:(("sshd",pid=691,fd=4))
```

Видно, что **OpenVPN Server** слушает **UDP** порты **1194**, **1195** и **1207**, а **ocserv** слушает **TCP** и **UDP** порты **443**. Также видно, что на интерфейсах **tap0**, **tun0**, **tun1**, **tun2**, **vpns1** назначены верные **IP** адреса.

Дополнительно можно проверить сессии с помощью **occtl**:

```text
[vagrant@server ~]$ sudo occtl show sessions all
session     user    vhost             ip               user agent  created   status
5Xa02l   client  default  192.168.56.20 Open AnyConnect VPN Agen  46m:15s authenticated
```

## Замер скорости в различных тунелях

### Профиль tap

```text
[vagrant@client ~]$ iperf3 -c 10.10.10.1 -t 40 -i 5
Connecting to host 10.10.10.1, port 5201
[  5] local 10.10.10.2 port 57546 connected to 10.10.10.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   148 MBytes   249 Mbits/sec   61    112 KBytes
[  5]   5.00-10.00  sec   152 MBytes   255 Mbits/sec   41    119 KBytes
[  5]  10.00-15.00  sec   150 MBytes   251 Mbits/sec   50    130 KBytes
[  5]  15.00-20.00  sec   150 MBytes   251 Mbits/sec   58    129 KBytes
[  5]  20.00-25.00  sec   148 MBytes   249 Mbits/sec   78    135 KBytes
[  5]  25.00-30.00  sec   149 MBytes   250 Mbits/sec   46    119 KBytes
[  5]  30.00-35.00  sec   156 MBytes   261 Mbits/sec   36    191 KBytes
[  5]  35.00-40.00  sec   153 MBytes   257 Mbits/sec  118    135 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec  1.18 GBytes   253 Mbits/sec  488             sender
[  5]   0.00-38.31  sec  1.18 GBytes   264 Mbits/sec                  receiver

iperf Done.
```

### Профиль tun

```text
[vagrant@client ~]$ iperf3 -c 10.10.11.1 -t 40 -i 5
Connecting to host 10.10.11.1, port 5201
[  5] local 10.10.11.2 port 44944 connected to 10.10.11.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   151 MBytes   253 Mbits/sec   65    141 KBytes
[  5]   5.00-10.00  sec   153 MBytes   257 Mbits/sec   53    114 KBytes
[  5]  10.00-15.00  sec   154 MBytes   258 Mbits/sec   56    128 KBytes
[  5]  15.00-20.00  sec   150 MBytes   252 Mbits/sec   67    112 KBytes
[  5]  20.00-25.00  sec   152 MBytes   255 Mbits/sec   65    146 KBytes
[  5]  25.00-30.00  sec   154 MBytes   258 Mbits/sec   75    188 KBytes
[  5]  30.00-35.00  sec   156 MBytes   262 Mbits/sec   81    127 KBytes
[  5]  35.00-40.00  sec   154 MBytes   259 Mbits/sec   59    159 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec  1.20 GBytes   257 Mbits/sec  521             sender
[  5]   0.00-38.31  sec  1.19 GBytes   268 Mbits/sec                  receiver

iperf Done.
```

### Профиль ras

```text
[vagrant@client ~]$ iperf3 -c 10.10.12.1 -t 40 -i 5
Connecting to host 10.10.12.1, port 5201
[  5] local 10.10.12.2 port 48218 connected to 10.10.12.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   162 MBytes   272 Mbits/sec   83    165 KBytes
[  5]   5.00-10.00  sec   160 MBytes   269 Mbits/sec  113    134 KBytes
[  5]  10.00-15.00  sec   158 MBytes   265 Mbits/sec   72    285 KBytes
[  5]  15.00-20.00  sec   158 MBytes   265 Mbits/sec   76    142 KBytes
[  5]  20.00-25.00  sec   157 MBytes   264 Mbits/sec   63    276 KBytes
[  5]  25.00-30.00  sec   166 MBytes   279 Mbits/sec  126    164 KBytes
[  5]  30.00-35.00  sec   173 MBytes   290 Mbits/sec   67    136 KBytes
[  5]  35.00-40.00  sec   158 MBytes   265 Mbits/sec   81    133 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec  1.26 GBytes   271 Mbits/sec  681             sender
[  5]   0.00-38.31  sec  1.26 GBytes   283 Mbits/sec                  receiver

iperf Done.
```

### OpenConnect

```text
[vagrant@client ~]$ iperf3 -c 10.10.13.1 -t 40 -i 5
Connecting to host 10.10.13.1, port 5201
[  5] local 10.10.13.176 port 39968 connected to 10.10.13.1 port 5201
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-5.00   sec   202 MBytes   339 Mbits/sec  141   94.5 KBytes
[  5]   5.00-10.00  sec   200 MBytes   336 Mbits/sec   97    119 KBytes
[  5]  10.00-15.00  sec   208 MBytes   349 Mbits/sec  113    116 KBytes
[  5]  15.00-20.00  sec   211 MBytes   354 Mbits/sec  115   99.9 KBytes
[  5]  20.00-25.00  sec   205 MBytes   344 Mbits/sec  100   85.0 KBytes
[  5]  25.00-30.00  sec   220 MBytes   369 Mbits/sec  119   86.4 KBytes
[  5]  30.00-35.00  sec   222 MBytes   372 Mbits/sec   98   98.5 KBytes
[  5]  35.00-40.00  sec   222 MBytes   372 Mbits/sec   94   86.4 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-40.00  sec  1.65 GBytes   354 Mbits/sec  877             sender
[  5]   0.00-38.31  sec  1.65 GBytes   370 Mbits/sec                  receiver

iperf Done.
```

### Вывод

**OpenConnect** демонстрирует на **30%** большую производительность. **tun** интерфейс **OpenVPN** работает незначительно быстрее, чем **tap** интерфейс.
