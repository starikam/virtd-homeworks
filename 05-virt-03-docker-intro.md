# Домашнее задание к занятию 4 «Оркестрация группой Docker контейнеров на примере Docker Compose»

## Задача 1

Docker и docker compose plugin установлены, Docker Hub доступен (зеркала не понадобились).

Скачан образ:

```console
$ docker pull nginx:1.29.0
...
Digest: sha256:3ab4ed065a1437cbbd45e65617b1285bdf6523c6bf56a121e00df41720e09a89
Status: Downloaded newer image for nginx:1.29.0
```

`index.html`:

```html
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
```

`Dockerfile`:

```dockerfile
FROM nginx:1.29.0
COPY index.html /usr/share/nginx/html/index.html
```

Сборка и публикация в публичный репозиторий Docker Hub `bokkote11/custom-nginx`:

```console
$ docker build -t custom-nginx:1.0.0 .
...
#7 naming to docker.io/library/custom-nginx:1.0.0 done

$ docker login -u bokkote11
Login Succeeded

$ docker tag custom-nginx:1.0.0 bokkote11/custom-nginx:1.0.0
$ docker push bokkote11/custom-nginx:1.0.0
fea7cebc499c: Mounted from library/nginx
856c000ad0ec: Mounted from library/nginx
27cc0cc68aef: Pushed
1.0.0: digest: sha256:005dc2690855f5734c6d584c59a32bc6539daa48ec1b327fcccab00991d42106 size: 856
```

Ссылка на репозиторий: **https://hub.docker.com/r/bokkote11/custom-nginx**

---

## Задача 2

Запуск контейнера с требуемым именем, в фоне, с публикацией на `127.0.0.1:8080`, затем переименование без удаления:

![Запуск и переименование контейнера](dz01-shots/t2-01-run-rename.png)

Выполните команду `date +"%d-%m-%Y %T.%N %Z" ; sleep 0.150 ; docker ps ; ss -tlpn | grep 127.0.0.1:8080  ; docker logs custom-nginx-t2 -n1 ; docker exec -it custom-nginx-t2 base64 /usr/share/nginx/html/index.html`:

![Проверочная команда](dz01-shots/t2-02-checks.png)

Индекс-страница доступна:

![curl к 127.0.0.1:8080](dz01-shots/t2-03-curl.png)

---

## Задача 3

**1–3.** Подключиться к стандартным потокам ввода/вывода/ошибок запущенного контейнера позволяет команда **`docker attach`**. Подключаемся и нажимаем `Ctrl-C`:

![docker attach и Ctrl-C](dz01-shots/t3-01-attach-ctrlc.png)

`docker attach` по умолчанию работает с `--sig-proxy=true`, то есть проксирует сигналы с терминала в главный процесс контейнера (PID 1). Нажатие `Ctrl-C` отправило `SIGINT` не docker-клиенту, а самому nginx внутри контейнера - это видно прямо в логе: `signal 2 (SIGINT) received, exiting`. Для nginx `SIGINT` - это команда немедленного завершения.

**4–6.** Перезапускаем контейнер, заходим в интерактивный bash и ставим редактор:

![docker start, docker exec -it bash, установка nano](dz01-shots/t3-02-exec-nano.png)

**7.** Открываем `/etc/nginx/conf.d/default.conf` в nano:

![default.conf в nano](dz01-shots/t3-03-nano-open.png)

Меняем `listen 80` на `listen 81`:

![listen 80 исправлен на listen 81](dz01-shots/t3-04-nano-edited.png)

**8.** `nginx -s reload` перечитал конфиг: 80-й порт внутри контейнера больше не слушается, страницу теперь отдаёт 81-й.

![nginx -s reload и curl :80 / :81](dz01-shots/t3-05-reload-curl.png)

**9–10.** Выходим из контейнера (`exit`) и проверяем снаружи:

![ss, docker port, curl 8080](dz01-shots/t3-06-broken.png)

Публикация порта — это свойство контейнера, а не nginx. Правило проброса было зафиксировано при `docker run` и живёт независимо от того, что происходит внутри. На хосте `127.0.0.1:8080` по-прежнему слушается, но приложение внутри переехало на 81. Трафик исправно доставляется на 80-й порт контейнера, где его уже никто не ждёт.

**12.** Удаление работающего контейнера без остановки — `docker rm -f`:

![docker rm -f](dz01-shots/t3-10-rm.png)

---

## Задача 4

Запускаем два контейнера, примонтировав в оба один и тот же каталог хоста в `/data`:

![centos и debian с общим каталогом](dz01-shots/t4-01-run.png)

Создаём файл из первого контейнера и добавляем ещё один с хоста:

![файл из centos + файл с хоста](dz01-shots/t4-02-create.png)

Смотрим из второго контейнера — видны оба файла и их содержимое:

![листинг и содержимое из debian](dz01-shots/t4-03-debian.png)

Bind mount — это один и тот же каталог на хосте, поэтому изменения из любого источника (хост, любой из контейнеров) сразу видны всем остальным.

---

## Задача 5

**1.** Создаём каталог и два манифеста:

![два манифеста в одном каталоге](dz01-shots/t5-01-files.png)

```console
$ docker compose up -d
```

![запустился только compose.yaml](dz01-shots/t5-02-up.png)

**Запустился `compose.yaml`** — поднялся только portainer. Compose ищет манифест по списку имён в фиксированном порядке приоритета, и канонические `compose.yaml`/`compose.yml` стоят выше исторических `docker-compose.yaml`/`docker-compose.yml`.

**2.** Чтобы поднялись оба, подключаем второй манифест через `include`:

```yaml
include:
  - docker-compose.yaml

services:
  portainer:
    network_mode: host
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

![include подключил второй манифест](dz01-shots/t5-03-include.png)

**3.** Заливаем образ в локальный registry.

![push в локальный registry](dz01-shots/t5-04-push.png)

**4.** Начальная настройка Portainer выполнена на `https://127.0.0.1:9443` — задан пароль администратора:

```console
$ docker logs task5-portainer-1 2>&1 | grep -i token
... no administrator account configured; admin initialization and backup restore
require this setup token in the X-Setup-Token header. | setup_token=eb74be505028...
```

![Создание администратора Portainer](dz01-shots/portainer-01-init-admin.png)

![Environment Wizard после создания администратора](dz01-shots/portainer-02-wizard.png)

**5.** В `Home → local → Stacks → Add stack → Web editor` задеплоен компоуз:

```yaml
version: '3'

services:
  nginx:
    image: 127.0.0.1:5000/custom-nginx
    ports:
      - "9090:80"
```

![Компоуз в web editor](dz01-shots/portainer-03-web-editor.png)

![Стек custom-nginx-stack задеплоен](dz01-shots/portainer-04-stack-deployed.png)

Образ подтянулся из локального registry, контейнер работает и отдаёт страницу:

```console
$ docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
NAMES                        IMAGE                          PORTS
custom-nginx-stack-nginx-1   127.0.0.1:5000/custom-nginx    0.0.0.0:9090->80/tcp, [::]:9090->80/tcp
task5-registry-1             registry:2                     0.0.0.0:5000->5000/tcp, [::]:5000->5000/tcp
task5-portainer-1            portainer/portainer-ce:latest

$ curl -s http://127.0.0.1:9090 | head -3
<html>
<head>
Hey, Netology
```

![Список контейнеров в Portainer](dz01-shots/portainer-05-containers.png)

**6.** `Containers → custom-nginx-stack-nginx-1 → Inspect`, представление `<> Tree`, развёрнутое поле `Config` по заданию:

![Inspect контейнера, Tree, от AppArmorProfile до Driver](dz01-shots/portainer-06-inspect-config.png)

**7.** Удаляем один из манифестов и поднимаем проект заново:

![два предупреждения после удаления compose.yaml](dz01-shots/t5-05-warnings.png)

**Суть предупреждений** (их два):

1. `the attribute version is obsolete` — ключ `version` был нужен во времена Compose file format v1/v2/v3, когда номер схемы определял набор доступных возможностей. В Compose Specification (Compose V2) версионирование схемы упразднено: набор полей определяется версией самого `docker compose` Предложенное действие — удалить строку из манифеста:

![ключ version удалён, предупреждение ушло](dz01-shots/t5-06-version-removed.png)

2. `Found orphan containers (task5-portainer-1)` — вместе с `compose.yaml` из проекта пропало описание сервиса `portainer`, но сам контейнер остался запущен и по-прежнему помечен меткой проекта `task5`. Compose видит контейнер, которому больше не соответствует ни один сервис в манифесте, и предупреждает, что не тронет его без явного разрешения. Предложенное действие — флаг `--remove-orphans`.

Гашение всего compose-проекта одной командой:

![down --remove-orphans гасит всё](dz01-shots/t5-08-down-orphans.png)
