# Immich (Ansible + Docker Compose)

[Immich](https://immich.app/) — фотоархив на своём сервере вместо Google Фото и iCloud — на двух узлах, разворачивается одним плейбуком. В комплекте фоторамки [ImmichFrame](https://immichframe.online/), Telegram-бот загрузки фото и автоматические альбомы по лицам.

- Immich на двух узлах, оба активны: состояние в общих PostgreSQL и Redis, библиотека на общем хранилище
- Обновление без простоя: узлы перезапускаются по очереди, каждый проверяется по API
- Фоторамки ImmichFrame: сколько угодно, настройки раскладываются плейбуком
- [Telegram-бот](https://github.com/apnagaev/telegram-immich-bot) на обоих узлах: active/passive через файл блокировки на общем хранилище
- Альбомы по лицам ([immich-face-to-album](https://hub.docker.com/r/rbrucker/immich-face-to-album)): фото человека автоматически попадают в его альбом
- Firewall на nftables: порты Immich доступны только узлам, reverse proxy и внутренней сети
- Журналы контейнеров с ротацией, секреты в отдельных файлах (ansible-vault)

```
          браузеры, приложения Immich, рамки
                         │ https
                 ┌───────▼───────┐  vIP
                 │ reverse proxy │
                 └───┬───────┬───┘
               :2283 │       │ :2283
           ┌─────────▼┐     ┌▼─────────┐
           │  node1   │     │  node2   │   immich-server, machine-learning,
           │          │     │          │   фоторамки (28091…), Telegram-бот
           └──┬────┬──┘     └──┬────┬──┘
              │    └────┬──────┘    │
         ┌────▼─────┐ ┌─▼─────┐ ┌───▼──────────┐
         │PostgreSQL│ │ Redis │ │ библиотека   │
         │(VectorCh)│ │       │ │ (NFS и т. п.)│
         └──────────┘ └───────┘ └──────────────┘
```

## Структура

```
install_immich.yaml             # плейбук: firewall, файлы, бот, поочерёдный перезапуск, альбомы, очистка
inventory.example.yml           # пример inventory (скопировать в inventory.yml)
vars/immich.example.yml         # переменные: порты, firewall, рамки, бот, альбомы по лицам
tasks/immich-healthcheck.yaml   # ожидание готовности узла (/api/server/ping, /ping)
files/immich/
  immich.yaml                   # docker-compose.yaml
  immich.nft.j2                 # правила nftables
  .env.example                  # версии и подстановки для compose
  immich.env.example            # переменные Immich: база, Redis, машинное обучение
  telebot.env.example           # секреты Telegram-бота
  frame.example.yml             # пример настроек фоторамки (Settings.yml)
```

## Требования

- Два хоста Debian/Ubuntu с Docker Engine и плагином `docker compose`
- Ansible и коллекция `community.docker` (`ansible-galaxy collection install community.docker`)
- PostgreSQL с расширением [VectorChord](https://immich.app/docs/administration/postgres-standalone) и Redis, доступные обоим узлам
- Общее хранилище для библиотеки (например, NFS), смонтированное на обоих узлах в `UPLOAD_LOCATION`
- Reverse proxy с TLS перед узлами (nginx/OpenResty)

## Настройка

```bash
cp inventory.example.yml inventory.yml
cp vars/immich.example.yml vars/immich.yml
cp files/immich/.env.example files/immich/.env
cp files/immich/immich.env.example files/immich/immich.env
cp files/immich/telebot.env.example files/immich/telebot.env
cp files/immich/frame.example.yml files/immich/frameap.yml     # по файлу на каждую рамку
```

1. **inventory.yml** — адреса узлов и `node` (1 или 2, порядок перезапуска). Узел 1 при обновлении первым применяет миграции базы.
2. **files/immich/.env** — версии (`IMMICH_VERSION`, `IMMICHFRAME_VERSION`), путь к библиотеке, адрес Immich, пароль админки рамок, DNS для контейнеров (`DNS_SERVER`, обязательно) и подсеть Docker-сети (`IMMICH_SUBNET`). **Закрепите версию Immich** (например `v2.2.0`) вместо `release`: новые версии мигрируют базу, откатиться без бэкапа нельзя.
3. **files/immich/immich.env** — `DB_URL`, `REDIS_HOSTNAME`, `REDIS_PORT`, `IMMICH_TRUSTED_PROXIES` (адреса reverse proxy).
4. **files/immich/telebot.env** — если нужен бот (иначе уберите сервис `telegram-immich-bot` из compose). Файл сессии Telethon создаётся на каждом узле отдельно скриптом `telethon_session.sh` из репозитория бота и лежит рядом с `docker-compose.yaml`; путь — `TELEGRAM_SESSION_FILE` в `.env`. Плейбук его не копирует. Бот работает на обоих узлах: активен тот, кто держит файл блокировки `LOCK_FILE_PATH` на общем хранилище, второй ждёт и подхватывает работу, если первый пропал.
5. **Рамки** — по `Settings.yml` на рамку и список `frames` в `vars/immich.yml` (каталог, исходный файл, имя сервиса). Сервисы рамок в compose создаются по образцу `immichframeap`.
6. **vars/immich.yml** — `trusted_nets` (сети reverse proxy и мониторинга), `immich_api_key` и `face_albums` для альбомов по лицам.

Секреты зашифруйте:
```bash
ansible-vault encrypt vars/immich.yml files/immich/.env files/immich/immich.env files/immich/telebot.env files/immich/frame*.yml
```

## Развертывание

```bash
ansible-playbook -i inventory.yml install_immich.yaml --ask-vault-pass
```

Плейбук на каждом узле по очереди:
1. Применяет правила nftables (`/etc/nftables.d/immich.nft`, с проверкой синтаксиса).
2. Раскладывает compose, секреты (`0600`) и настройки рамок.
3. Собирает образ Telegram-бота — только если изменился его код или образа нет.
4. Скачивает образы и выполняет `docker compose up -d`: пересоздаются только изменившиеся контейнеры.
5. Ждёт ответа `/api/server/ping` и `/ping` машинного обучения, и только потом переходит к следующему узлу.
6. На узле 1 раскладывает фото по альбомам лиц.
7. Удаляет неиспользуемые образы (тома не трогает).

## Firewall

Порты из `immich_ports` (сервер, машинное обучение, рамки) открыты только для `lo`, Docker-сетей, узлов кластера и `trusted_nets`. Правила фильтруют в `prerouting` с приоритетом −150, до DNAT Docker, поэтому видят настоящий порт и адрес клиента. Не делайте `systemctl restart nftables`: `/etc/nftables.conf` начинается с `flush ruleset` и удалит правила Docker. Отключить: `firewall: false`.

## Альбомы по лицам

ID лица — из адреса страницы человека в Immich (`/people/<id>`), ID альбома — из адреса альбома (`/albums/<id>`). Ключ API создаётся в «Настройки аккаунта → API-ключи». Задачу можно вынести в cron, чтобы альбомы пополнялись без запуска плейбука.

## Проверка

```bash
docker ps --format '{{.Names}}\t{{.Status}}'            # immich_server: healthy
curl -s http://127.0.0.1:2283/api/server/ping           # {"res":"pong"}
curl -s http://127.0.0.1:3003/ping                      # pong
nft list table inet immich
```

## Обновление

Поменяйте `IMMICH_VERSION` в `files/immich/.env` и прогоните плейбук. Перед обновлением сделайте бэкап базы и прочитайте [заметки к релизу](https://github.com/immich-app/immich/releases): у Immich бывают ломающие изменения.
