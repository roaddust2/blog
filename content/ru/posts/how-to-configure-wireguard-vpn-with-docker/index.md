---
title: 'Как установить Wireguard VPN на сервер с помощью Docker'
description: Я просто хотел оплатить квартплату
slug: kak-ustanovit-wireguard-vpn-s-pomoshiu-docker

categories: ["Infrastructure"]
tags: ["VPS", "Server", "VPN", "Wireguard", "Docker"]

date: 2024-04-07T17:02:18+07:00
draft: false
---

## Вступление
Небольшое отступление, Wireguard и OpenVPN на текущий момент нестабильны при обходе блокировок внутри России, так как используется DPI (Deep Packet Inspection) для ограничения трафика по протоколу. Рекомендую рассмотреть решения основанные на Shadowsocks, VMESS, VLESS и Trojan.

У меня ситуация строго противоположная, с доступом к зарубежным ресурсам все в порядке, а вот некоторые российские сайты ограничили доступ для зарубежного трафика. Ну что же, прикинемся ~~ветошью~~ москвичем и возьмем сервер дислоцирующийся в Москве. В качестве VPN я выбрал Wireguard, как наиболее современное и производительное решение. Начнем.

## Установка
> Для установки, нам потребуется сервер, о том как его настроить я описал в [этой статье](/ru/posts/2024/03/pervaya-nastroika-vps-posle-pokupki/).

Подобные вещи я предпочитаю устанавливать в контейнерах, поэтому для начала установим Docker Engine, инструкцию по установке можно найти на официальном [сайте](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

Кому лень переходить на сайт, выполняем эти команды:

```bash
# Устанавливаем GPG ключ
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Добавляем репозиторий
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update

# Устанавливаем докер
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Теперь нужно создать docker-compose.yml, обычно я добавляю их куда-нибудь в директорию /var/.

```bash
sudo mkdir -p /var/containers/wireguard/
cd /var/containers/wireguard/
sudo nano docker-compose.yml
```

После того как мы создали файл, необходимо его наполнить. Я использую linuxserver образы, как правило они стабильны и всегда отлично работают.

Наполняем файл следующим содержанием:

```yml
---
services:
  wireguard:
    image: lscr.io/linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Moscow
      - SERVERURL=auto #Или IP сервера
      - SERVERPORT=51820
      - PEERS=5 #Количество конфигураций для клиента
      - PEERDNS=auto
      - INTERNAL_SUBNET=10.13.13.0
      - ALLOWEDIPS=0.0.0.0/0
      - LOG_CONFS=true
    volumes:
      - /path/to/appdata/config:/config
      - /lib/modules:/lib/modules
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

Подробнее о конфигурации compose файла можно почитать [тут](https://docs.linuxserver.io/images/docker-wireguard/#environment-variables-e).

Запускаем в фоновом режиме:

```bash
sudo docker compose up -d
```

## Подключение

Итак VPN запущен, теперь нужно к нему подключиться, для этого необходимо установить клиентское приложение. Скачать его можно на [официальном сайте](https://www.wireguard.com/install/).

После того как мы его скачали, необходимо получить конфигурационный файл, для этого на сервере выполняем команду:

```bash
# Получаем QR код конфигурации
sudo docker exec -it wireguard /app/show-peer 1

# Мы указали пять PEER в компоузе и можем получить их все, например так
sudo docker exec -it wireguard /app/show-peer 1 2 3 4 5
```

Сканируем и вставляем полученную конфигурацию в наше приложение, на мобильном приложении достаточно будет просто отсканировать код. Запускаем VPN и [проверяем локацию](https://whatismyipaddress.com/).