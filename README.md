# Практика №4 — миграция Mattermost 9.5 + MySQL → 12.0.1 + PostgreSQL 15

Идемпотентный пайплайн GitHub Actions на self-hosted раннере: бэкап → перенос данных MySQL → PostgreSQL
(официальные `migration-assist` + `mattermost/mattermost-pgloader`) → запуск Mattermost 12.0.1 → smoke-тесты.

```
.
├── legacy/
│   ├── docker-compose.yml        # старый стек: Mattermost 9.5 + MySQL 8.0
│   └── .env.example              # пароли legacy-стека (копируется в legacy/.env)
├── docker-compose.prod.yml       # новый стек: Mattermost 12.0.1 + PostgreSQL 15
├── migration/Dockerfile          # образ с утилитой migration-assist
├── .github/workflows/migration.yml
├── scripts/reset-lab.sh          # полный сброс стенда для повторной репетиции
├── .env.example
└── .gitignore
```

---

## 0. Подготовка (один раз)

**Нужно:** Linux, macOS или Windows + **WSL2 (Ubuntu)**. На Windows раннер ставьте *внутрь WSL*,
а не в PowerShell (в Docker Desktop включите *Settings → Resources → WSL integration*).

1. **Docker + Compose v2**: `docker compose version` должна работать без `sudo`.
   На Linux: `sudo usermod -aG docker $USER`, затем перелогиньтесь.
2. **Репозиторий на GitHub.** Создайте пустой репозиторий и залейте туда эти файлы:
   ```bash
   git init && git add . && git commit -m "Practice 4: Mattermost migration"
   git branch -M main
   git remote add origin https://github.com/<you>/<repo>.git
   git push -u origin main
   ```
3. **Self-hosted runner** — на ТОЙ ЖЕ машине, где будет работать Docker со стеком:
   GitHub → *Settings → Actions → Runners → New self-hosted runner* → выполните показанные команды
   (`./config.sh ...`, затем `./run.sh`). Окно с `./run.sh` держите открытым — статус раннера должен быть **Idle**.
4. **Секреты**: *Settings → Secrets and variables → Actions → New repository secret*:

   | Имя | Значение |
   |---|---|
   | `DB_MYSQL_ROOT_PASSWORD` | **тот же**, что `DB_MYSQL_ROOT_PASSWORD` в `legacy/.env` |
   | `DB_POSTGRES_PASSWORD` | любой новый пароль для PostgreSQL |

   Пароли — только буквы и цифры (символы `@ : / ? #` ломают строки подключения).

---

## 1. Запуск legacy-стека и тестовые данные

```bash
cd legacy
cp .env.example .env          # впишите свои пароли
docker compose up -d
docker compose ps             # mattermost и mattermost-mysql — running / healthy
```

Через ~1 минуту откройте **http://localhost:8065** и:

1. Создайте учётную запись (первый пользователь автоматически становится администратором).
2. Создайте команду.
3. **Загрузите аватар** (Профиль → Edit profile picture) — это проверка общего тома.
4. Создайте **приватный канал** и напишите **3–5 сообщений**, в одном из них прикрепите картинку.
5. Сделайте скриншот канала — пригодится для сравнения «до/после».

> Внизу слева виден номер версии (*Main Menu → About Mattermost*): сейчас **9.5.x**.

---

## 2. Запуск миграции (прогон №1)

```bash
cd ..                          # корень репозитория
git checkout -b migration
git push -u origin migration   # push в ветку migration запускает пайплайн
```

GitHub → вкладка **Actions** → *Idempotent Mattermost Migration & Upgrade*. Первый прогон идёт 5–10 минут
(сборка образа migration-assist, скачивание образов). Что происходит:

| Шаг | Что делает |
|---|---|
| Check Migration Status | флага `~/.mattermost_migration_completed` нет → миграция нужна |
| Backup Legacy MySQL | останавливает старый Mattermost, делает `mysqldump` + архив тома с файлами в `~/mattermost_backups/` |
| Start Target PostgreSQL | поднимает только `db-postgres`, ждёт healthcheck |
| Reset target schema (include drop) | `DROP SCHEMA … CASCADE` — повторный запуск после сбоя не дублирует таблицы |
| Prepare migration | `migration-assist mysql` (фиксы), создание схемы PG версии 9.5, генерация `migration.load` |
| Run pgloader | перенос данных образом `mattermost/mattermost-pgloader`, восстановление полнотекстовых индексов |
| Mark Migration as Completed | создаёт флаг-файл |
| Deploy Upgraded Stack | `docker compose up -d --build --remove-orphans` → Mattermost 12.0.1, MySQL удалён |
| Run Health Checks & Verify Data | `/api/v4/system/ping` = 200, версия 12.0.1, драйвер postgres, MySQL отсутствует, число сообщений не уменьшилось, вывод последних сообщений |
| Collect artifacts | **всегда** (даже при ошибке): отчёт по шагам, тесты, данные из PostgreSQL, логи контейнеров, состояние Docker, логи pgloader; пароли вычищаются из файлов |
| Upload … | артефакты `migration-report-runN-attemptM` (30 дней) и `backup-runN` — дамп MySQL + архив тома (7 дней) |

## 3. Прогон №2 (проверка идемпотентности)

Любой из способов:
- на странице прогона **Re-run all jobs**, или
- `git commit --allow-empty -m "re-run" && git push`.

Шаги бэкапа и миграции будут **skipped** (серые), прогон займёт ~30–60 секунд и тоже будет **Success**.

## Артефакты прогона

Внизу страницы прогона, блок **Artifacts**:

| Артефакт | Что внутри |
|---|---|
| `migration-report-runN-attemptM` | `report.md` — итог каждого шага (success / skipped / failure) и тестов; `tests/api-ping.json`; `db/` — число пользователей/каналов/сообщений и последние сообщения из PostgreSQL, число сообщений в MySQL до миграции; `logs/` — логи контейнеров mattermost, postgres (и mysql, если был); `docker/` — `docker ps`, образы, тома, сеть, итоговый compose-конфиг; `migration/` — лог pgloader, его конфиг, список миграций; `migration-flag.txt`; `backups-list.txt`; `secrets-used.txt` (только имена) |
| `backup-runN` | `mattermost_mysql_backup_….sql.gz` (дамп MySQL) и `mattermost_data_….tar.gz` (аватары, вложения). Только в прогоне, где делался бэкап |

Артефакты собираются шагом с `if: always()` — даже если прогон упал, логи для разбора всё равно сохранятся.
Пароли из файлов заменяются на `***` (GitHub прячет секреты только в логе на странице, а не внутри файлов);
если пароль всё-таки остался — шаг падает. Полный текстовый лог прогона: на странице прогона ⋯ → *Download log archive*.

## 4. Ручная проверка

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'
# mattermost           mattermost/mattermost-team-edition:12.0.1
# mattermost-postgres  postgres:15-alpine
# (mattermost-mysql отсутствует)

curl -s http://localhost:8065/api/v4/system/ping        # {"status":"OK",...}
```

Откройте http://localhost:8065, войдите тем же логином/паролем — канал, сообщения, аватар и вложение на месте,
в *About Mattermost* версия **12.0.1**.

---

## 5. Как показывать преподавателю

Подготовьте заранее: открытая вкладка Actions с двумя зелёными прогонами, терминал, браузер с Mattermost.
Идите по критериям — так проще набрать все 10 баллов.

**Критерий 1. Docker и тома (3 б.)**
- `docker ps` — работают `mattermost` (12.0.1) и `mattermost-postgres`, MySQL нет, порт 8065 без конфликтов.
- `docker volume ls | grep mattermost_data` и покажите в обоих compose-файлах строку
  `mattermost_data:/mattermost/data` — том один и тот же. Аватар отображается, значит прав доступа хватает.
- `docker network inspect mattermost_network --format '{{.Driver}}'` → `bridge`. Покажите, что у БД нет
  секции `ports:` — они доступны только внутри сети; migration-assist и pgloader запускаются с
  `--network mattermost_network`.

**Критерий 2. Секреты (2 б.)**
- Покажите *Settings → Secrets*: `DB_MYSQL_ROOT_PASSWORD`, `DB_POSTGRES_PASSWORD`.
- В workflow пароли только через `${{ secrets.… }}`; в compose — через `${DB_POSTGRES_PASSWORD}`; `.env` в `.gitignore`.
- Откройте лог шага Backup/Prepare: паролей нет — они передаются через переменные окружения
  (`MYSQL_PWD`, `-e PG_DSN`), а не в тексте команд, плюс GitHub маскирует их как `***`.

**Критерий 3. Идемпотентность (3 б.)**
- Покажите два прогона: первый долгий, во втором шаги миграции **skipped**, шаг *Check Migration Status*
  пишет «already migrated» и печатает содержимое флаг-файла (`cat ~/.mattermost_migration_completed`).
- Шаг *Reset target schema (include drop)*: перед каждой попыткой переноса схема удаляется целиком, поэтому
  при перезапуске упавшего пайплайна таблицы не дублируются (см. объяснение в п. 6).
- Шаг *Deploy* — `--remove-orphans`, именно он удалил `mattermost-mysql`.

**Артефакты.** Внизу страницы прогона → *Artifacts* → скачайте `migration-report-…`, откройте `report.md`: таблица шагов, результаты тестов, число сообщений до/после. Во втором прогоне шаги миграции там `skipped`.

**Критерий 4. Тестирование (2 б.)**
- Лог шага *Run Health Checks & Verify Data*: `HTTP 200`, версия `12.0.1…`, «running on PostgreSQL»,
  число сообщений до/после и таблица с вашими последними сообщениями прямо из PostgreSQL.
- Браузер: войдите, откройте приватный канал — сообщения, вложение и аватар на месте; *About* → 12.0.1.

---

## 6. Что изменено относительно шаблона и почему (если спросят)

1. **Образ `mattermost/mattermost:9.5` не существует** → `mattermost/mattermost-team-edition:9.5`.
2. **`MM_SQLSETTINGS_DRIVERNAME=github.com/Masterminds/squirrel`** — это SQL-библиотека, а не драйвер → `mysql`.
   Опечатка `MM_TEAMSSETTINGS_…` → `MM_TEAMSETTINGS_…`.
3. **Общее имя compose-проекта `name: mattermost`** в обоих файлах. Без него legacy-стек получил бы имя
   проекта `legacy`, prod — имя папки: тома были бы разными (`legacy_mattermost_data` ≠ `…_mattermost_data`),
   а `--remove-orphans` не удалил бы MySQL из чужого проекта.
4. **Перенос по официальной схеме migration-assist, а не одной командой pgloader.** Одна команда
   `pgloader mysql://… postgresql://…` создаёт таблицы в схеме `mattermost` с типами MySQL; Mattermost 12
   ищет данные в схеме `public`, видит пустую БД и создаёт новую — сообщения «пропадают».
   Поэтому, как в документации Mattermost: схема PG создаётся миграциями той же версии (9.5), pgloader
   переносит только данные (`data only`), затем Mattermost 12 сам обновляет схему до 12.0.1.
5. **`include drop` реализован явным `DROP SCHEMA … CASCADE`.** pgloader не принимает `--with` вместе с
   файлом конфигурации (завершается с FATAL «Option is ignored when using a load file»), а в режиме
   `data only` опция `include drop` ничего не делает — таблицы создаёт не pgloader. Явное удаление схемы
   перед каждой попыткой даёт тот же результат: повторный запуск начинает с чистой схемы.
6. **Флаг-файл в `$HOME` пользователя раннера**, а не жёстко `/home/runner` — у вас пользователь может называться иначе.
7. `sleep` заменён на ожидание healthcheck/цикл опроса; добавлены `concurrency` (нет параллельных прогонов),
   бэкап тома с файлами, сверка числа сообщений.

---

## 7. Если что-то пошло не так

- **Backup: «Legacy MySQL container is not running»** — `cd legacy && docker compose up -d`.
- **Раннер Offline** — запустите `./run.sh` в папке раннера.
- **`permission denied … docker.sock`** — пользователь раннера не в группе `docker`.
- **Упал любой шаг до «Mark Migration as Completed»** — просто перезапустите пайплайн: MySQL не удалялся,
  схема PG будет очищена и перенос пройдёт заново.
- **Откат на старую версию**: `docker compose -f docker-compose.prod.yml down`, затем
  `cd legacy && docker compose up -d` — том `mattermost_mysql_data` не удаляется; дамп лежит в `~/mattermost_backups/`.
- **Начать всё с нуля** (репетиция): `./scripts/reset-lab.sh` — удалит контейнеры, тома и флаг.
- **Apple Silicon**: образ pgloader только amd64, запускается через эмуляцию (`--platform linux/amd64`) — это нормально, просто медленнее.
