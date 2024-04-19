# Домашнее задание к занятию «GitLab»

---
## Ахмадеев Булат Наилевич

---

## Задание 1

Так как я решил делать через <b>docker-compose</b>, создал файл docker-compose.yml, предварительно установив все
необходимые пакеты <b>Docker</b> и <b>Docker-Compose</b> на локальную машину.

Машину поднял уже на работе, чтобы его использовать дальше в работе.

### Содержимое docker-compose.yml:

```commandline
version: '3.8'

services:

  gitlab:
    image: gitlab/gitlab-ce:latest
    hostname: gitlab
    restart: unless-stopped
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        gitlab_rails['gitlab_shell_ssh_port'] = 22
        external_url 'https://gitlab'
        letsencrypt['enable'] = true
        letsencrypt['contact_emails'] = ['***@***']  # Скрыл, так как использовалась настоящая почта организации
    ports:
      - "443:443"
      - "80:80"
      - "7778:22"
    volumes:
      - /srv/gitlab/etc/gitlab:/etc/gitlab  # Проброс директорий
      - /srv/gitlab/var/opt/gitlab:/var/opt/gitlab  # Проброс директорий
      - /srv/gitlab/var/log/gitlab:/var/log/gitlab  # Проброс директорий
      - /home/administrator/mySSL/gitlab.key:/etc/gitlab/ssl/gitlab.key  # Чуть ниже объясню зачем
      - /home/administrator/mySSL/gitlab.crt:/etc/gitlab/ssl/gitlab.crt  # Чуть ниже объясню зачем
    networks:
      - gitlab_net

  gitlab-runner:
    image: gitlab/gitlab-runner:latest
    restart: unless-stopped
    depends_on:
      - gitlab
    volumes:
      - /home/administrator/mySSL/gitlab.crt:/etc/gitlab.crt  # Чуть ниже объясню зачем
      - /home/administrator/mySSL/gitlab.key:/etc/gitlab.key  # Чуть ниже объясню зачем
      - /srv/gitlab-runner/etc/gitlab-runner:/etc/gitlab-runner  # Проброс директорий
      - /srv/gitlab-runner/var/run/docker.sock:/var/run/docker.sock  # Проброс директорий
    networks:
      - gitlab_net

# Закидываю их в одну сеть
networks:
  gitlab_net:
    driver: bridge
```

Как поясняет официальное руководство GitLab.com, использование <b>регистрационных токенов</b>, начиная с версии gitlab:15.0, 
уже является невозможным. Начиная с версии gitlab:15.0 есть возможность использовать <b>токены аутентификации</b>.

Поэтому необходимо выполнить дополнительные настройки:
1) Создать свой самоподписывающий сертификат;
2) Пробрасывать этот сертификат в контейнеры еще на стадии сборки.

### Создал сертификат командой:
```commandline
administrator@gitlab:~/mySSL$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout gitlab.key 
-out gitlab.crt -subj "/C=RU/ST=Tatarstan/L=***/O=***/CN=gitlab" -addext "subjectAltName = DNS:gitlab"

[sudo] password for administrator: 

..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+.....+.+.....+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.........+.+.....+......+...................+..+....+...+..+......+...+......+.............+............+.........+......+...+...+..............+....+.....+.+.........+...........................+........+.......+..+...............+......+....+...............+...........+.+.....+.+..+...+...+............+...+......+....+.....+....+............+........+..........+......+.........+...+...............+.........+...+..+.........+..........+..+...+............+....+...+..................+..+......+.+......+...+......+......+........+............+.......+..+.........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
....+.......+..+.......+.........+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+..+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*...+.............+.....+.+..............+......+.+..............+.+...+.....+....+............+...+..+...+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
-----

administrator@gitlab:~/mySSL$ ls
gitlab.crt  gitlab.key
```
### Далее уже пробросил их в контейнеры:
```commandline
      ...
      - /home/administrator/mySSL/gitlab.key:/etc/gitlab/ssl/gitlab.key  # Чуть ниже объясню зачем
      - /home/administrator/mySSL/gitlab.crt:/etc/gitlab/ssl/gitlab.crt  # Чуть ниже объясню зачем
      ...
      - /home/administrator/mySSL/gitlab.crt:/etc/gitlab.crt  # Чуть ниже объясню зачем
      - /home/administrator/mySSL/gitlab.key:/etc/gitlab.key  # Чуть ниже объясню зачем
      ...
```

### Запустил docker-compose.yml:
```commandline
administrator@gitlab:~/docker$ sudo docker compose up -d && sudo docker ps --all

WARN[0000] /home/administrator/docker/docker-compose.yml: `version` is obsolete
[+] Running 2/2
 ✔ Container docker-gitlab-1         Running    0.0s
 ✔ Container docker-gitlab-runner-1  Started    0.3s
CONTAINER ID   IMAGE                         COMMAND                  CREATED         STATUS                   PORTS                                                                                                             NAMES
e955fb8da51c   gitlab/gitlab-runner:latest   "/usr/bin/dumb-init …"   1 second ago    Up Less than a second                                                                                                                      docker-gitlab-runner-1
8446f5f10740   gitlab/gitlab-ce:latest       "/assets/wrapper"        7 minutes ago   Up 4 minutes (healthy)   0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp, 0.0.0.0:7778->22/tcp, :::7778->22/tcp   docker-gitlab-1
```
### Далее пришлось узнать временный пароль root и создать новый пароль:
![root-password.png](images%2Froot-password.png)

### Создаю новый проект:
![create-project.png](images%2Fcreate-project.png)
![new-project.png](images%2Fnew-project.png)

### Регистрация Runner
Чтобы зарегистрировать runner в новых версиях gitlab, теперь нужно сделать следующее:
1) В панели администратора зайти в CI/CD -> Runners;
2) Создать новый Runner:
![create-runner.png](images%2Fcreate-runner.png)
3) Копируем предоставленную команду для регистрации Runner;
4) Вводим эту команду с дополнением где развернут Runner:
```commandline
administrator@gitlab:~/docker$ sudo docker exec -it e955fb8da51c 
gitlab-runner register  --url https://gitlab  --token glrt-5-BFMHy8mJbY_yGSy8Ns  
--tls-ca-file /etc/gitlab.crt  # Указываем проброшенный сертификат

Runtime platform                                    arch=amd64 os=linux pid=17 revision=91a27b2a version=16.11.0
Running in system-mode.                            
                                                   
Enter the GitLab instance URL (for example, https://gitlab.com/):
[https://gitlab]: 
Verifying runner... is valid                        runner=5-BFMHy8m
Enter a name for the runner. This is stored only in the local config.toml file:
[e955fb8da51c]: 
Enter an executor: docker-autoscaler, instance, virtualbox, docker-windows, docker+machine, kubernetes, docker, custom, shell, ssh, parallels:
docker
Enter the default Docker image (for example, ruby:2.7):
test-netology
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
 
Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
```
### И вот Runner зарегистрирован!
![runner-online.png](images%2Frunner-online.png)
### Конфигурация Runner'а:
```commandline
administrator@gitlab:~/docker$ sudo docker exec -it e955fb8da51c cat /etc/gitlab-runner/config.toml
concurrent = 1
check_interval = 0
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "e955fb8da51c"
  url = "https://gitlab"
  id = 10
  token = "glrt-5-BFMHy8mJbY_yGSy8Ns"
  token_obtained_at = 2024-04-19T00:17:41Z
  token_expires_at = 0001-01-01T00:00:00Z
  tls-ca-file = "/etc/gitlab.crt"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    MaxUploadedArchiveSize = 0
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "test-netology"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
    network_mtu = 0
```

---

