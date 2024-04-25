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
        external_url 'https://10.0.4.200'
        letsencrypt['enable'] = true
        letsencrypt['contact_emails'] = ['***@***']  # Скрыл, так как использовалась настоящая почта организации
    ports:
      - "443:443"
      - "80:80"
      - "22:22"
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
      - /var/run/docker.sock:/var/run/docker.sock  # Проброс директорий
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
bu1ya@gitserver:~/mySSL$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 
-keyout 10.0.4.200.key -out 10.0.4.200.crt -subj 
"/C=RU/ST=Tatarstan/L=Kazan/O=KazanEXPO/CN=10.0.4.200" 
-addext "subjectAltName = IP:10.0.4.200"
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
bu1ya@gitserver:~/docker$ sudo docker-compose up -d
Creating network "docker_net" with driver "bridge"
Pulling gitlab (gitlab/gitlab-ce:latest)...
latest: Pulling from gitlab/gitlab-ce
7021d1b70935: Pull complete
84ce88366b95: Pull complete
e1c92fb77de1: Pull complete
719e91a15767: Pull complete
d680b9b016aa: Pull complete
f0027cb8bf12: Pull complete
8090ecb94c61: Pull complete
2d67bfd969b5: Pull complete
216d686bfe05: Pull complete
Digest: sha256:7c1182ba8d3f30fb82a4c1d2fcc4da35e23929775564642a2310941bc92de1a7
Status: Downloaded newer image for gitlab/gitlab-ce:latest
Creating docker_gitlab_1 ... done
bu1ya@gitserver:~/docker$ sudo docker ps --all
CONTAINER ID   IMAGE                     COMMAND             CREATED          STATUS                             PORTS                                                                                                         NAMES
e673b493d236   gitlab/gitlab-ce:latest   "/assets/wrapper"   13 seconds ago   Up 10 seconds (health: starting)   0.0.0.0:22->22/tcp, :::22->22/tcp, 0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp   docker_gitlab_1
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
administrator@gitlab:~/docker$ sudo docker exec -it e955fb8da51c gitlab-runner register  --url https://10.0.4.200  --token glrt-5-BFMHy8mJbY_yGSy8Ns  --tls-ca-file /etc/10.0.4.200.crt  # Указываем проброшенный сертификат

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
![settings-runner.png](images%2Fsettings-runner.png)
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
  url = "https://10.0.4.200"
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
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    network_mtu = 0
```

---
## Задание 2

### Клонировал репозиторий:
```commandline
bulatahmadeev@MacBook-Pro-Bulat NETOLOGY % git clone git@github.com:netology-code/sdvps-materials.git
Cloning into 'sdvps-materials'...
Enter passphrase for key '/Users/bulatahmadeev/.ssh/id_rsa_github': 
remote: Enumerating objects: 97, done.
remote: Counting objects: 100% (30/30), done.
remote: Compressing objects: 100% (9/9), done.
remote: Total 97 (delta 22), reused 21 (delta 21), pack-reused 67
Receiving objects: 100% (97/97), 28.15 KiB | 4.02 MiB/s, done.
Resolving deltas: 100% (38/38), done.
```

### Создал новую ветку:
```commandline
bulatahmadeev@MacBook-Pro-Bulat sdvps-materials % git checkout -b gitlab
bulatahmadeev@MacBook-Pro-Bulat sdvps-materials % git remote add gitlab git@10.0.4.200:root/netology-test.git
bulatahmadeev@MacBook-Pro-Bulat sdvps-materials % git remote -v                                              
gitlab  git@10.0.4.200:root/netology-test.git (fetch)
gitlab  git@10.0.4.200:root/netology-test.git (push)
origin  git@github.com:netology-code/sdvps-materials.git (fetch)
origin  git@github.com:netology-code/sdvps-materials.git (push)
```

### Создал новый файл <b>.gitlab-ci.yml</b>:
```commandline
stages:
  - test
  - build

test:
  stage: test
  image: golang:1.17
  script: 
   - go test .

build:
  stage: build
  image: docker:latest
  script:
   - docker build .
```

### Запушил в свой GitLab:
```commandline
bulatahmadeev@MacBook-Pro-Bulat sdvps-materials % git push gitlab gitlab                                                   
Enter passphrase for key '/Users/bulatahmadeev/.ssh/id_rsa_KZN_EXPO_GIT_SERVER_ADMINISTRATOR': 
Enumerating objects: 103, done.
Counting objects: 100% (103/103), done.
Delta compression using up to 12 threads
Compressing objects: 100% (56/56), done.
Writing objects: 100% (103/103), 28.71 KiB | 3.59 MiB/s, done.
Total 103 (delta 40), reused 96 (delta 38), pack-reused 0
remote: Resolving deltas: 100% (40/40), done.
To 10.0.4.200:root/netology-test.git
 * [new branch]      gitlab -> gitlab
```

### Как выглядит теперь проект в GitLab с новой веткой:

![push_project.png](images%2Fpush_project.png)

### TEST и BUILD заработали!

![test_pipeline.png](images%2Ftest_pipeline.png)

