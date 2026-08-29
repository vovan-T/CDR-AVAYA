# INSTALL

Документ для установки CDR AVAYA на сервер.

## Установка версии 2.5

Основной способ установки 2.5 - готовые Docker-образы. На компьютере с
интернетом скачайте и запустите один файл:

```bash
wget https://github.com/vovan-T/CDR-AVAYA/releases/latest/download/install.sh
chmod +x install.sh
sudo ./install.sh
```

Выберите `online`, чтобы скачать проверенный Release-комплект и установить CDR на текущий сервер. Выберите
`offline`, чтобы скачать три файла для переноса на закрытый сервер. Скрипт сам
покажет дальнейшие команды.

Подробности: [Docker](docs/DOCKER.md) и
[PDF-руководство 2.5](frontend/public/docs/cdr-avaya-2.5-manual-ru.pdf).

Описанные ниже архивы относятся к прежней bare-metal линии установки.

## Требования

- Ubuntu Server 24.04.
- Доступ root или sudo.
- Python 3.12+.
- Node.js 18+.
- Docker и Docker Compose, если PostgreSQL ставится локально в контейнере.
- TCP-порт от Avaya CM до сервера CDR.
- SFTP-доступ от сервера CDR до SM/EQ, если используются файловые источники.

## Online-установка

Для установки используется подготовленный комплект одной версии:

```text
dist/online/cdr-avaya-online-<version>.tar.gz
dist/online/install.sh
dist/online/INSTALL.txt
dist/online/uninstall.sh
```

На сервер копируются эти четыре файла в одну папку. Самостоятельно пересобирать
инсталляционный архив для обычного обновления системы не нужно:

```bash
sudo bash install.sh
```

Внешний `install.sh` распакует архив во временную папку и запустит install-процедуру.
На вопрос `Куда ставим? (/opt/cdr):` можно нажать Enter для `/opt/cdr`
или указать другой полный путь установки.

Инсталлятор:

- спрашивает папку установки, по умолчанию `/opt/cdr`;
- создаёт системного пользователя и группу `cdr`;
- создаёт `.env`;
- готовит Python venv;
- готовит Node API/UI;
- поднимает или подключает PostgreSQL;
- применяет базовую схему БД;
- устанавливает systemd-сервисы.

Основной web-интерфейс:

```text
http://SERVER_IP:8010
```

## Offline-установка

Для закрытого сервера используется подготовленный offline-комплект той же версии:

```bash
sudo bash install.sh --offline
```

На сервер копируются три файла в одну папку:

```text
install.sh
uninstaller.sh
cdr-avaya-offline-<version>.tar.gz
```

Offline-режим берёт пакеты из каталога `packages/`.
Python-пакеты ставятся через `python -m pip`.

Подробно:

- `docs/OFFLINE_BUILD.md` - как собрать offline-дистрибутив;
- `docs/OFFLINE_INSTALL.md` - как ставить готовый offline-дистрибутив.

## Параметры установки

Если PostgreSQL уже есть, инсталлятор спросит:

```text
IP/host DB
Порт DB
Имя БД
Логин БД
Пароль БД
```

Если PostgreSQL нет, можно выбрать Docker-вариант.

Файл `.env` создаётся инсталлятором во время установки.
Пользователь не заполняет `.env` вручную: установщик задаёт вопросы, получает значения и записывает итоговый файл сам.
После окончания заполнения ответов инсталлятор выводит путь, куда будут сохранены параметры.

Основные параметры, которые будут записаны:

```text
DB_HOST       <---------- IP/host PostgreSQL
DB_PORT       <---------- порт PostgreSQL, обычно 5432
DB_NAME       <---------- имя базы CDR, обычно avaya_cdr
DB_USER       <---------- пользователь PostgreSQL
DB_PASSWORD   <---------- пароль пользователя PostgreSQL
TZ            <---------- часовой пояс логов и Node API, например Europe/Moscow
API_HOST      <---------- IP, на котором слушает Web UI/API; 0.0.0.0 = все интерфейсы
API_PORT      <---------- порт Web UI/API, обычно 8010
CDR_HOME      <---------- корневая папка CDR, обычно /opt/cdr
```

После заполнения всех параметров итоговый файл хранится здесь:

```text
/opt/cdr/.env
```

## Сервисы

После установки должны быть активны:

```text
cdr-api             Node API + Vue UI
cdr-listener        приём CM CDR по TCP
cdr-processor       обработчик CM raw -> calls
cdr-processor-sm    обработчик SM raw -> calls
cdr-processor-eq    обработчик EQ raw -> calls
cdr-processor-sbce  обработчик SBCE raw -> calls
cdr-processor-other обработчик Other raw -> calls
```

Проверка:

```bash
systemctl status cdr-api cdr-listener cdr-processor cdr-processor-sm cdr-processor-eq cdr-processor-sbce cdr-processor-other --no-pager
curl http://127.0.0.1:8010/api/health
ss -lntp | grep -E ':5001|:8010'
```

## Avaya CM

В CM укажите IP сервера CDR и TCP-порт ACD.
Для базовой ACD `cm1` используется порт:

```text
5001
```

Рекомендуемый custom-набор полей CM:

```text
date
time
calling-num
dialed-num
sec-dur
cond-code
in-trk-code
ucid
seq-num
code-dial
code-used
vdn
feat-flag
in-crt-id
out-crt-id
line-feed
return
```

Для нормальной обработки обязательны `ucid` и `seq-num`.

## SM

SM используется как файловый источник.
Файлы забираются по SFTP пользователем `CDR_User`.

Для SM логин и протокол в UI считаются фиксированными:

```text
login:    CDR_User
protocol: SFTP
port:     22
path:     .
format:   Enhanced XML File
```

CDR_User сразу попадает в специальный CDR-каталог SM
(`/data/home/CDR_User/` на сервере), поэтому в UI используется
фиксированный путь `.`. Парсер CDR AVAYA обрабатывает только
Enhanced XML.

Проверка вручную:

```bash
/opt/cdr/venv/bin/python /opt/cdr/listener/sm_fetch.py --acd cm1
/opt/cdr/venv/bin/python /opt/cdr/listener/processor.py --acd cm1 --source sm --status
```

## EQ

EQ используется как файловый источник XML. На каждой EQMGMT-ноде root-cron
копирует новые и изменённые XML в staging:

```text
/home/CDR_User/data
```

Готовый adjunct: `tools/setup-eq-cdr-staging.sh`.
Подробно: `docs/EQ_CDR_STAGING.md`.

Проверка вручную:

```bash
/opt/cdr/venv/bin/python /opt/cdr/listener/eq_fetch.py --acd cm1
/opt/cdr/venv/bin/python /opt/cdr/listener/processor.py --acd cm1 --source eq --status
```

## SBCE

SBCE сам кладёт файлы на CDR-сервер по SFTP.
Installer создаёт общего пользователя загрузки файлов `CDR_User` и фиксированную
папку первого SBCE `sbce/1`. Пароль вводится один раз в основном опросе и
показывается в итогах установки.
Для этого требуется системный пакет `openssh-server`; online/offline installer
включает его в обязательный набор.

Для второго SBCE выполните
`sudo sh /opt/cdr/bin/add_user_for_files.sh --folder sbce/2`.

В SBCE CDR Adjunct:

```text
Address:  SERVER_IP:22
Username: CDR_User
Путь:     1
```

Забытый пароль не отображается. Задайте новый: `sudo passwd CDR_User`.

Папка на Linux:

```text
/opt/cdr/data/sbce/1
```

Импорт SBCE пропускает свежие файлы младше 10 минут, чтобы не забрать файл во время записи.
Параметр можно изменить через переменную окружения:

```text
SBCE_MIN_FILE_AGE_SEC
```

Подробно: `docs/SBCE_CDR_ADJUNCT.md`.

## Авторизация и системные настройки

После установки выполните вход под `admin` и смените временный пароль.
Пользователи, пароли и права `Чтение`/`Редактирование` для каждой ACD управляются
в `Система -> Пользователи`. В системных настройках также доступны backup и
restore, службы, диагностика, обновления, интерфейс и журнал действий.

## Быстрая проверка после установки

```bash
curl http://127.0.0.1:8010/api/health
tail -n 80 /opt/cdr/logs/api.log
tail -n 80 /opt/cdr/logs/listener.log
tail -n 80 /opt/cdr/logs/processor.log
```

Если что-то не работает, см. `docs/TROUBLESHOOTING.md`.
## Удаление CDR

Штатное удаление запускается из установленной системы:

```bash
sudo /opt/cdr/uninstaller.sh
```

Если CDR установлен в другую папку, используйте путь к `uninstaller.sh` в ней.
Скрипт сначала запрашивает внешнюю папку для backup, создаёт полный системный
архив и проверяет в нём `manifest.env` и `db.dump`. Только после успешной
проверки он останавливает и удаляет CDR services, cron, logrotate, sudoers,
SBCE SSH-конфигурацию, контейнер `cdr-db` с его Compose volume, пользователей
CDR и каталог установки. Docker Engine, Docker images и системные пакеты не
удаляются.

Внешний `uninstaller.sh` из текущего дистрибутива содержит backup helper и
создаёт `CDR_BACKUP_V1`: полный `db.dump`, portable CSV payload каждой ACD,
системные данные и настройки. Перед удалением проверяется наличие portable ACD
payload и его формат в `manifest.env`.

Backup перед удалением можно проверить отдельно, не останавливая службы и не
изменяя CDR:

```bash
sudo ./uninstaller.sh --backup-only
```
