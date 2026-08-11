# Установка CDR AVAYA 2.5

## Требования

- Linux x86_64; основной проверенный вариант — Ubuntu Server 24.04;
- root или `sudo`;
- Docker Engine;
- Docker Compose plugin (`docker compose`);
- свободный TCP-порт `8010` для UI/API;
- TCP-порт `5001` для учебной ACD `CM1`;
- SSH/SFTP `22`, если внешние системы передают CDR-файлы.

Проверка Docker:

```bash
docker --version
docker compose version
sudo docker ps
```

Если Docker отсутствует в Ubuntu:

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
```

## Единый установщик

На компьютере с доступом в интернет выполните:

```bash
wget https://github.com/vovan-T/CDR-AVAYA/releases/latest/download/install.sh
chmod +x install.sh
sudo ./install.sh
```

### Online

Выберите `online`. Установщик:

1. проверит Docker и Docker Compose;
2. запросит Web/API-порт и дважды пароль `CDR_User`;
3. скачает готовые образы CDR AVAYA и PostgreSQL;
4. создаст `/opt/cdr`, базу и пользователя файловой загрузки;
5. дождётся состояния `healthy` и покажет адрес системы.

### Offline

Выберите `offline` и каталог для сохранения. Будут скачаны:

```text
install-offline.sh
cdr-avaya-docker-images-2.5.0.tar.gz
cdr-avaya-docker-images-2.5.0.tar.gz.sha256
```

Перенесите все три файла на закрытый сервер и выполните:

```bash
sha256sum -c cdr-avaya-docker-images-2.5.0.tar.gz.sha256
chmod +x install-offline.sh
sudo ./install-offline.sh
```

Offline installer не обращается к GitHub или GHCR: оба Docker-образа уже
находятся в архиве.

## Первый вход

После установки откройте:

```text
http://SERVER_IP:8010
```

Начальная учётная запись: `admin` / `admin`. При первом входе пароль необходимо
сменить.

Учебная ACD `CM1` содержит примеры полей, словаря и основных типов вызовов.
После создания своей ACD пример можно удалить через контекстное меню.

## Проверка

```bash
cd /opt/cdr
sudo docker compose ps
curl -sS http://127.0.0.1:8010/api/health
sudo docker logs --tail 100 cdr-app
```

Ожидается:

- `cdr-app` — `healthy`;
- `cdr-db` — `healthy`;
- API — `{"status":"ok","db":"ok"}`.

Полное руководство: [cdr-avaya-2.5-manual-ru.pdf](docs/cdr-avaya-2.5-manual-ru.pdf).
