# **Введение**

Установка PostgreSQL

# **Подготовка**

Установим Docker

```
root@grc-lin-ub-1:~# curl -fsSL https://get.docker.com -o get-docker.sh
root@grc-lin-ub-1:~# sudo sh get-docker.sh
# Executing docker install script, commit: b2e29ef7a9a89840d2333637f7d1900a83e7153f
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq apt-transport-https ca-certificates curl >/dev/null
+ sh -c mkdir -p /etc/apt/keyrings && chmod -R 0755 /etc/apt/keyrings
+ sh -c curl -fsSL "https://download.docker.com/linux/ubuntu/gpg" | gpg --dearmor --yes -o /etc/apt/keyrings/docker.gpg
+ sh -c chmod a+r /etc/apt/keyrings/docker.gpg
+ sh -c echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu focal stable" > /etc/apt/sources.list.d/docker.list
+ sh -c apt-get update -qq >/dev/null
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq --no-install-recommends docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-scan-plugin >/dev/null
+ version_gte 20.10
+ [ -z  ]
+ return 0
+ sh -c DEBIAN_FRONTEND=noninteractive apt-get install -y -qq docker-ce-rootless-extras >/dev/null
+ sh -c docker version
Client: Docker Engine - Community
 Version:           20.10.17
 API version:       1.41
 Go version:        go1.17.11
 Git commit:        100c701
 Built:             Mon Jun  6 23:02:57 2022
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

Server: Docker Engine - Community
 Engine:
  Version:          20.10.17
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.17.11
  Git commit:       a89b842
  Built:            Mon Jun  6 23:01:03 2022
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.6
  GitCommit:        10c12954828e7c7c9b6e0ea9b0c02b01407d3ae1
 runc:
  Version:          1.1.2
  GitCommit:        v1.1.2-0-ga916309
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

================================================================================

To run Docker as a non-privileged user, consider setting up the
Docker daemon in rootless mode for your user:

    dockerd-rootless-setuptool.sh install

Visit https://docs.docker.com/go/rootless/ to learn about rootless mode.


To run the Docker daemon as a fully privileged service, but granting non-root
users access, refer to https://docs.docker.com/go/daemon-access/

WARNING: Access to the remote API on a privileged Docker daemon is equivalent
         to root access on the host. Refer to the 'Docker daemon attack surface'
         documentation for details: https://docs.docker.com/go/attack-surface/

================================================================================

root@grc-lin-ub-1:~# rm get-docker.sh

```
# **Выполнение**

Сделать каталог /var/lib/postgres
```
root@grc-lin-ub-1:~# mkdir /var/lib/postgres
```
Развернуть контейнер с PostgreSQL 14 смонтировав в него /var/lib/postgres

```
root@grc-lin-ub-1:~# docker network create pg-net
26aa44a29abc795aeadfea56a3346d98e2783343906e4ad0ede6f27b2eab713d
root@grc-lin-ub-1:~# docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5433:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
Unable to find image 'postgres:14' locally
14: Pulling from library/postgres
461246efe0a7: Pull complete
8d6943e62c54: Pull complete
558c55f04e35: Pull complete
186be55594a7: Pull complete
f38240981157: Pull complete
e0699dc58a92: Pull complete
066f440c89a6: Pull complete
ce20e6e2a202: Pull complete
c0f13eb40c44: Pull complete
3d7e9b569f81: Pull complete
2ab91678d745: Pull complete
ffc80af02e8a: Pull complete
f3a57056b036: Pull complete
Digest: sha256:0137ad83edaf18f6b36daf35b7f07f5ac694657178c0087dfa78d71c98774af4
Status: Downloaded newer image for postgres:14

```
Проверим статус
```
root@grc-lin-ub-1:~# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
6e08d6fea054   postgres:14   "docker-entrypoint.s…"   15 seconds ago   Up 14 seconds   0.0.0.0:5433->5432/tcp, :::5433->5432/tcp   pg-docker

```
Развернуть контейнер с клиентом postgres

```
root@grc-lin-ub-1:~# docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
Password for user postgres:
psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

postgres=#
```
Подключимся из контейнера с клиентом к контейнеру с сервером и сделаем
таблицу с парой строк.
```
postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
otus=# insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
(1 row)
```

Подключимся извне
```
root@grc-lin-deb:~#  psql -h x.x.x.x -U postgres -d otus -p 5433
psql (11.15 (Debian 11.15-1.pgdg110+1), сервер 14.4 (Debian 14.4-1.pgdg110+1))
ПРЕДУПРЕЖДЕНИЕ: psql имеет базовую версию 11, а сервер - 14.
                Часть функций psql может не работать.
Введите "help", чтобы получить справку.

otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
(1 строка)

```
Подключились данные видны.

## **Проверим сохранность данных после перезапуска контейнера**
Удалим контейнер и создадим его заново.
```
root@grc-lin-ub-1:~# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
6e08d6fea054   postgres:14   "docker-entrypoint.s…"   20 minutes ago   Up 20 minutes   0.0.0.0:5433->5432/tcp, :::5433->5432/tcp   pg-docker
root@grc-lin-ub-1:~# docker rm -f 6e
6e
root@grc-lin-ub-1:~# docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```
Запуск
```
root@grc-lin-ub-1:~# docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5433:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:14
8fdebcc69c4f9c61ebdb5fbfe93a716a0fcde7793cb81404d0bb22d0305b62df
root@grc-lin-ub-1:~# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
8fdebcc69c4f   postgres:14   "docker-entrypoint.s…"   12 seconds ago   Up 11 seconds   0.0.0.0:5433->5432/tcp, :::5433->5432/tcp   pg-docker
```
Все слои скачаны запуск происходит мгновенно.

Снова подключимся через клиент и проверим данные.
```
root@grc-lin-ub-1:~# docker run -it --rm --network pg-net --name pg-client postgres:14 psql -h pg-docker -U postgres
Password for user postgres:
psql (14.4 (Debian 14.4-1.pgdg110+1))
Type "help" for help.

postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
(1 row)
```
Данные хранятся в смонтированной папке и не удалены.







