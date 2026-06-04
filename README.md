# Практическая работа №15 (семестр 2)

## Выполнила: Сорокина К.С., ЭФМО-01-25

## Тема: Деплой приложения на VPS. Настройка systemd

### Цель:

Освоить базовый процесс деплоя Go-приложения на реальный удалённый сервер (VPS), научиться управлять контейнерным стеком через systemd, обеспечить автоматический запуск сервиса при перезагрузке сервера и проверить доступность API с внешней машины.

## Технологии

- **Go** — язык реализации сервиса
- **Docker / Docker Compose** — контейнеризация и оркестрация
- **NGINX** — балансировщик нагрузки (reverse proxy)
- **systemd** — управление сервисом на VPS
- **Ubuntu 24.04 LTS** — операционная система VPS
- **SSH / rsync** — подключение и копирование файлов на сервер

## Вариант деплоя

В данной работе выбран **Вариант Б: контейнерный деплой через Docker Compose**.

systemd используется не для запуска отдельного Go-бинарника, а для управления всем docker-compose стеком как единой службой. При старте systemd выполняет `docker-compose up -d --build`, при остановке — `docker-compose down`.

## Схема деплоя

```
MacBook (разработка)
       │
       │ rsync -av ./ root@VPS:/opt/pr15/
       ▼
VPS Ubuntu 24.04
       │
       ├── /opt/pr15/deploy/lb/docker-compose.yml
       ├── /opt/pr15/services/tasks/Dockerfile
       └── /etc/systemd/system/pr15.service
              │
              │ systemctl start pr15
              ▼
       docker-compose up -d --build
              │
    ┌─────────┼─────────┐
    │         │         │
 tasks_1   tasks_2   nginx_lb
 :8082     :8082     :8080 (публичный)
```

## Структура проекта

```
PR15_sem2/
├── services/
│   └── tasks/
│       ├── Dockerfile
│       ├── go.mod
│       └── cmd/server/main.go
├── deploy/
│   └── lb/
│       ├── docker-compose.yml    ← две реплики tasks + nginx
│       └── nginx.conf
├── docs/                         ← скриншоты
└── README.md
```

---

## 1. Подключение к VPS по SSH

VPS: Ubuntu 24.04.4 LTS, 2 Core, 1024 MB RAM

Подключение с Mac:
```bash
ssh root@<VPS_IP>
```

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/проверка%20системы.png)

## 2. Установка Docker на VPS

```bash
apt update && apt install -y docker.io docker-compose
```

Проверка:
```
Docker version 29.1.3
docker-compose version 1.29.2
```
![](https://github.com/krrristina/PR15_2/blob/main/screenshots/docker%20версия.png)

## 3. Копирование проекта на VPS

С локальной машины (Mac):
```bash
rsync -av --exclude='.git' --exclude='bin' --exclude='screenshots' \
  ./ root@<VPS_IP>:/opt/pr15/
```

Результат — 13 файлов скопированы в `/opt/pr15/`.

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/перенос%20файлов.png)

## 4. Запуск контейнеров

```bash
cd /opt/pr15/deploy/lb
docker-compose up -d --build
```

Результат:
```
Creating tasks_1 ... done
Creating tasks_2 ... done
Creating nginx_lb ... done
```

Проверка:
```bash
docker-compose ps
```

| Name | State | Ports |
|---|---|---|
| nginx_lb | Up | 0.0.0.0:8080->8080/tcp |
| tasks_1 | Up | 8082/tcp |
| tasks_2 | Up | 8082/tcp |

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/проверка%20запуска%20контейнеров.png)

## 5. Systemd unit-файл

Файл `/etc/systemd/system/pr15.service`:

```ini
[Unit]
Description=PR15 Docker Compose stack
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/opt/pr15/deploy/lb
ExecStart=/usr/bin/docker-compose up -d --build
ExecStop=/usr/bin/docker-compose down
ExecReload=/usr/bin/docker-compose up -d --build --remove-orphans
RemainAfterExit=yes
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

### Пояснение параметров:

- `Requires=docker.service` — стек зависит от Docker, запускается только после него
- `After=docker.service network-online.target` — порядок запуска: сначала Docker и сеть
- `WorkingDirectory` — рабочая папка с `docker-compose.yml`
- `ExecStart` — команда запуска: сборка и старт контейнеров в фоне
- `ExecStop` — команда остановки: останавливает и удаляет контейнеры
- `ExecReload` — обновление: пересобирает изменённые образы и перезапускает контейнеры
- `RemainAfterExit=yes` — unit остаётся активным после завершения `docker-compose up -d` (команда завершается сразу, контейнеры продолжают работать)
- `Type=oneshot` — systemd запускает команду один раз и не ожидает длительного процесса

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/содержимое%20файла.png)

## 6. Запуск и управление через systemd

```bash
systemctl daemon-reload
systemctl enable pr15
systemctl start pr15
systemctl status pr15 --no-pager
```

Статус:
```
Active: active (exited) since Wed 2026-06-03 23:30:42 MSK
Process: ExecStart=/usr/bin/docker-compose up -d --build (code=exited, status=0/SUCCESS)
```

`active (exited)` — корректный статус для `Type=oneshot`: команда выполнена успешно, контейнеры запущены в фоне.

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/сервис%20создан%20и%20включен.png)

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/статус%20системы.png)

## 7. Проверка доступности сервиса

С самого VPS:
```bash
curl -i http://localhost:8080/health
```

Ответ:
```
HTTP/1.1 200 OK
X-Instance-Id: tasks-1
{"instance":"tasks-1","status":"ok"}
```

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/проверка%201.png)

С внешней машины (Postman):
- **Method:** `GET`
- **URL:** `http://<VPS_IP>:8080/health`
- **Ответ:** `200 OK` + `{"instance":"tasks-1","status":"ok"}`

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/postman%20проверка.png)

## 8. Обновление версии

Синхронизируем изменённые файлы с Mac на VPS:
```bash
rsync -av --exclude='.git' ./ root@<VPS_IP>:/opt/pr15/
```

Перезапускаем стек через systemd (ExecReload пересобирает образы):
```bash
systemctl reload pr15
systemctl status pr15 --no-pager
```

Результат:
```
Process: ExecReload=/usr/bin/docker-compose up -d --build --remove-orphans (code=exited, status=0/SUCCESS)
```

![](https://github.com/krrristina/PR15_2/blob/main/screenshots/проверка%20обновлений.png)

## 9. Откат

```bash
cd /opt/pr15
git checkout <предыдущий_коммит>
systemctl restart pr15
```

Если проект переносился без Git:
```bash
# восстановить предыдущую версию файлов вручную
systemctl restart pr15
```

---

## Ответы на контрольные вопросы

**Зачем нужен systemd и чем он лучше запуска в screen/tmux?**
systemd обеспечивает автозапуск после перезагрузки VPS, единое управление сервисом (`start/stop/reload/status`), интеграцию с журналом логов (`journalctl`) и корректную остановку. screen и tmux — ручные инструменты для интерактивных сессий, не являются системой управления сервисами.

**Что означает Active: active (exited) для Type=oneshot?**
Это корректный статус. Type=oneshot означает что systemd запускает команду один раз и считает сервис активным после её успешного завершения. `docker-compose up -d` завершается сразу — контейнеры запущены в фоне. RemainAfterExit=yes оставляет unit активным.

**Почему не стоит запускать сервис от root?**
При компрометации приложения злоумышленник получает права процесса. Если сервис работает от root — ущерб максимальный. Принцип минимальных привилегий: сервис должен работать от отдельного пользователя с ограниченными правами.

**Зачем хранить env-конфиг отдельно от кода?**
Переменные окружения содержат пароли, адреса сервисов и настройки окружения. Их нельзя жёстко зашивать в код, потому что код переносится между окружениями и может попасть в публичный репозиторий.

**Как посмотреть логи сервиса?**
Логи systemd unit:
```bash
journalctl -u pr15 -n 50 --no-pager
```
Логи контейнеров:
```bash
cd /opt/pr15/deploy/lb
docker-compose logs --tail=50
```

**Что происходит при перезагрузке VPS?**
systemd автоматически запускает `pr15.service` благодаря `systemctl enable pr15`. Это выполняет `docker-compose up -d --build` и поднимает все контейнеры без ручного вмешательства.

**В чём преимущество Варианта Б перед Вариантом А?**
Вариант Б не требует сборки бинарника под Linux и ручной настройки зависимостей на сервере. Весь стек описан в docker-compose.yml — он одинаково работает на любом сервере где установлен Docker. Достаточно скопировать файлы и запустить.

**Почему Apache нужно было остановить?**
Apache занимал порт 8080, а NGINX в docker-compose также пробрасывает порт 8080. Два процесса не могут одновременно слушать один порт — Docker не мог запустить контейнер с NGINX.

**Что делает rsync и чем он лучше scp для деплоя?**
rsync передаёт только изменённые файлы, а scp копирует всё целиком. При повторном деплое rsync значительно быстрее — он сравнивает файлы и отправляет только дельту изменений.

**Что такое ExecReload и когда он используется?**
ExecReload выполняется при `systemctl reload`. В данной работе он запускает `docker-compose up -d --build --remove-orphans` — пересобирает изменённые образы и перезапускает контейнеры без полной остановки стека.
