# Борьба с приоритетом IPv6 в Docker

###  Описание проблемы (Issue)
При выполнении команды `docker pull` или обращении контейнеров к внешним хостам (`registry-1.docker.io`) возникали ошибки:
* `no such host`
* `cannot assign requested address`

 Проблема заключалась в том, что Docker продолжал использовать IPv6, даже если они были некорректно настроены или были отключены на уровне ядра

---

### Полное решение

#### 1. Отключение IPv6 на уровне ядра (ОС)
Для начала принудительно отключим IPv6 для всех интерфейсов в текущей сессии:
```bash
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.default.disable_ipv6=1
sudo sysctl -w net.ipv6.conf.lo.disable_ipv6=1
```


#### 2. Очистка сетевых интерфейсов Docker'а
Даже при отключении на уровне ядра, у мостов Докера могут оставаться активные IPv6-адреса. Чистим их вручную:
```bash
sudo ip -6 addr flush dev docker0
# Чекнуть ID моста можно через самый обычный 'ip a' 
sudo ip -6 addr flush dev br-"ID__МОСТА"
```

#### 3. Настройка daemon.json
Указываем Докеру использовать корректные DNS и выключаем поддержку IPv6 уже на его уровне:
```json
{
  "dns": ["8.8.8.8", "1.1.1.1"],
  "ipv6": false
}
```
После изменения конфига, не забудь рестартнуть ```sudo systemctl restart docker```.


#### 4. Сохраняем настройки на постоянную основу.
Чтоб после перезагрузки настройки не слетели, добавляем их в sysctl.conf и качаем iptables-persistent для блокировки трафика на уровне Фаерволла: 
```bash
sudo ip6tables -A OUTPUT -j DROP
sudo ip6tables -A FORWARD -j DROP
sudo ip6tables -A INPUT -j DROP

sudo apt install iptables-persistent
sudo netfilter-persistent save # Сохранит все правила
```

#### 5. Правка в /etc/hosts
Обычно верхних пунктов хватает, но иногда DNS продолжает отдавать IPv6 приоритет, тогда можно прописать IPv4 адрес вручную:
```bash
echo "52.20.100.68 registry-1.docker.io" | sudo tee -a /etc/hosts
```

#### 6. Финал
Сделай ```docker network prune -f``` и перезапусти все контейнеры, чтоб проблема исчезла 


