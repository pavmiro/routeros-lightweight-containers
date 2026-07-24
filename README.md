## Маленькие, лёгкие контейнеры для RouterOS 7.20 и 7.23+

- [DNSCrypt-Proxy](#dnscrypt) - легковесный DNS-прокси с шифрованием, занимает минимум места и полностью готов к работе сразу после запуска.  
 `https://github.com/dnscrypt/dnscrypt-proxy`
- [PyKMS](#pykms) - KMS-сервер на базе PyKMS, корректно работает с записью ресурсов службы (SRV) в системе доменных имен (DNS), в результате клиенты могут автоматически обнаруживать узел KMS и активировать его без необходимости настройки.
  `https://github.com/itsarts1/PyKMS`
- [AmneziaWG](#amneziawg) - альтернативная версия версия протокола Wireguard, использующая современные методы обфускации.  
  `https://github.com/amnezia-vpn/amneziawg-tools.git`  
  `https://github.com/amnezia-vpn/amneziawg-go`

# dnscrypt

### 🏷️ Список обновлений  
`13-jul-2026, alpine linux 3.24.1, dnscrypt-proxy 2.1.16-r0`

### 🛠️ Особенности dockerfile, изменения в конфигурации
При сборке в файл `/etc/dnscrypt-proxy/dnscrypt-proxy.toml` внесены следующие правки:

1. Используется лишь сервер Google DNS, работа на любом интерфейсе
```bash
server_names = ['google']
listen_addresses = ['0.0.0.0:53']
```
2. Включён веб-интерфейс мониторинга, работа на любом интерфейсе, порт 8080, без авторизации
```bash
[monitoring_ui]
enabled = true
listen_addresses = ['0.0.0.0:8080']
username = "admin"
password = ""
```
Все остальные настройки оставлены по умолчанию.  
Если хотите добавить другие сервера, то воспользуйтесь списком: `https://dnscrypt.info/public-servers/`

### 🚀 Пример использования
```routeros
/interface veth
add address=10.0.0.2/30 gateway=10.0.0.1 name=veth1
/ip address
add address=10.0.0.1/30 interface=veth1 
/container
add name=dnscrypt file=docker-dnscrypt-arm64.tar root-dir=dnscrypt interface=veth1 start-on-boot=yes

/ip dns
set servers=10.0.0.2
```
# pykms

Используется оригинальный код PyKMS из репозитория `https://github.com/itsarts1/PyKMS`  
В `server.py` внесены изменения, позволяющие контейнеру корректно останавливаться по сигналу SIGTERM.

#### 📦 /PyKMS/server.py
```python
import signal
import sys

def graceful_shutdown(signum, frame):
    sys.exit(0)

signal.signal(signal.SIGTERM, graceful_shutdown)
..
```

### 🚀 Пример использования
```routeros
/interface veth
add address=10.0.0.2/30 gateway=10.0.0.1 name=veth1
/ip address
add address=10.0.0.1/30 interface=veth1 
/container
add name=pykms file=docker-pykms-arm64.tar root-dir=pykms interface=veth1 start-on-boot=yes

/ip dns static
add name=_vlmcs._tcp.home.arpa srv-port=1688 srv-priority=0 srv-target=10.0.0.2 srv-weight=0 type=SRV

/ip dhcp-server network
set domain=home.arpa. [find where address=192.168.88.0/24]
```
# amneziawg

### 🏷️ Список обновлений  
`13-jul-2026, alpine linux 3.24.1, amneziawg-go 0.2.19`  
`24-jul-2026, alpine linux 3.24.1, amneziawg-go 3.0.1`

### 🛠️ Особенности dockerfile и конфигурации
В dockerfile производится клонирование и сборка:  
`https://github.com/amnezia-vpn/amneziawg-tools` `https://github.com/amnezia-vpn/amneziawg-go`

После запуска контейнера, запускается `entrypoint.sh`
```bash
#!/bin/bash

shutdown() {
    awg-quick down /awg.conf 2>/dev/null || true
    exit 0
}
trap shutdown SIGTERM SIGINT SIGQUIT

if [ -f "/awg.conf" ]; then
    awg-quick up /awg.conf || { exit 1; }
else
    echo "Configuration file /awg.conf not found."
fi

sleep infinity &
wait $!
```
Файл конфигурации располагается в `/awg.conf`, который вы можете или создать самостоятельно `vi awg.conf` или добавить в `mounts`

### 🚀 Пример использования
```routeros
/interface veth
add address=10.0.0.2/30 gateway=10.0.0.1 name=veth1
/ip address
add address=10.0.0.1/30 interface=veth1 
/container
add name=amneziawg file=docker-amneziawg-arm64.tar root-dir=amneziawg interface=veth1

# RouterOS 7.23
/container
set memory-high=256.0MiB mount=/awg.conf:/awg.conf:rw [find where name=amneziawg]

# RouterOS 7.20
/container mounts
add name=amneziawg src=/awg.conf dst=/awg.conf
/container
set mounts=amneziawg [find where name=amneziawg]
```
### 📝 Пожалуйста обратите внимание!  
Если необходимо указать правила для фаервола, то это можно сделать в файле `awg.conf`, например:

```
[Interface]
PostUp = iptables -t nat -A POSTROUTING -o awg -j MASQUERADE
```
или
```
[Interface]
PostUp = iptables -t nat -A PREROUTING -p udp --dport 53 -j DNAT --to-destination 10.0.0.1:53 ; iptables -t nat -A POSTROUTING -o veth1 -j MASQUERADE
```
Правила в PostDown указывать нет необходимости, т.к. правила iptables, после перезагрузки, не сохраняются.

❤️ pavmiro@mail.ru
