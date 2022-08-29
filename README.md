# otus-linux-day15
Ansible

# **Prerequisite**
- Host OS: Debian 11.3.0
- Guest OS: CentOS 7.8.2003
- VirtualBox: 6.1.36
- Vagrant: 2.2.19
- Ansible: 2.12.4

# **Содержание ДЗ**

* На сервере, используя Ansible необходимо развернуть nginx со следующими условиями:
  - необходимо использовать модуль yum/apt;
  - конфигурационные файлы должны быть взяты из шаблона jinja2 с переменными;
  - после установки nginx должен быть в режиме enabled в systemd;
  - должен быть использован notify для старта nginx после установки;
  - сайт должен слушать на нестандартном порту - 8080, для этого использоватьпеременные в Ansible;
* Выполнить те же дейcтвия с использованием роли Ansible

# **Выполнение**

## Ansible
Структура проекта с использованием Ansible:
```sh
root@nas:/study/day15# tree
.
├── ansible.cfg
├── nginx.yml
├── staging
│   └── hosts
├── templates
│   └── nginx.conf.j2
└── Vagrantfile
```
Конфигурационный файл Ansible содержит параметры подключения:
```sh
root@nas:/study/day15# cat ansible.cfg
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```

Inventory с единственным хостом:
```sh
root@nas:/study/day15# cat staging/hosts
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```
Шаблон с конфигурацией Nginx:
```sh
root@nas:/study/day15# cat templates/nginx.conf.j2
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
```
Playbook содержит задания установки репозитория EPEL, Nginx из него и создание конфиг-файла сервиса из шаблона, используются обработчики событий (handlers):
```yml
root@nas:/study/day15# cat nginx.yml
---
- name: NGINX | Install and configure NGINX
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080

  tasks:
    - name: NGINX | Install EPEL Repo package from standart repo
      yum:
        name: epel-release
        state: present
      tags:
        - epel-package
        - packages

    - name: NGINX | Install NGINX package from EPEL Repo
      yum:
        name: nginx
        state: latest
      notify:
        - restart nginx
      tags:
        - nginx-package
        - packages

    - name: NGINX | Create NGINX config file from template
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - reload nginx
      tags:
        - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```

## Ansible role

Создание роли:
```sh
root@nas:/study/day15_role/roles# ansible-galaxy init role-01
- Role role-01 was created successfully
```
Выполнен перенос в роль переменных `vars`:
```yml
root@nas:/study/day15_role/roles# cat role-01/vars/main.yml
---
# vars file for role-01
vars:
  nginx_listen_port: 8080
```
Выполнен перенос в роль обработчиков `handlers`:
```yml
root@nas:/study/day15_role/roles# cat role-01/handlers/main.yml
---
# handlers file for role-01
- name: restart nginx
  systemd:
    name: nginx
    state: restarted
    enabled: yes
- name: reload nginx
  systemd:
    name: nginx
    state: reloaded
```
Выполнен перенос в роль шаблона `templates`:
```yml
root@nas:/study/day15_role/roles# cat role-01/templates/nginx.conf.j2
# {{ ansible_managed }}
events {
    worker_connections 1024;
}

http {
    server {
        listen       {{ nginx_listen_port }} default_server;
        server_name  default_server;
        root         /usr/share/nginx/html;

        location / {
        }
    }
}
```

Выполнен перенос в роль заданий `tasks`:
```yml
root@nas:/study/day15_role/roles# cat role-01/tasks/main.yml
---
# tasks file for role-01
- name: NGINX | Install EPEL Repo package from standart repo
  yum:
    name: epel-release
    state: present
  tags:
    - epel-package
    - packages

- name: NGINX | Install NGINX package from EPEL Repo
  yum:
    name: nginx
    state: latest
  notify:
     - restart nginx
  tags:
    - nginx-package
    - packages

- name: NGINX | Create NGINX config file from template
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify:
    - reload nginx
  tags:
    - nginx-configuration
```

Структура проекта с использованием роли Ansible:
```sh
root@nas:/study/day15# tree
.
├── ansible.cfg
├── nginx-role.yml
├── roles
│   └── role-01
│       ├── defaults
│       │   └── main.yml
│       ├── files
│       ├── handlers
│       │   └── main.yml
│       ├── meta
│       │   └── main.yml
│       ├── README.md
│       ├── tasks
│       │   └── main.yml
│       ├── templates
│       │   └── nginx.conf.j2
│       ├── tests
│       │   ├── inventory
│       │   └── test.yml
│       └── vars
│           └── main.yml
├── staging
│   └── hosts
└── Vagrantfile
```

```
root@nas:/study/day15# ansible-playbook nginx-role.yml

PLAY [NGINX-role | Install and configure NGINX using role] *********************

TASK [Gathering Facts] *********************************************************
ok: [nginx]

TASK [role-01 : NGINX | Install EPEL Repo package from standart repo] **********
changed: [nginx]

TASK [role-01 : NGINX | Install NGINX package from EPEL Repo] ******************
changed: [nginx]

TASK [role-01 : NGINX | Create NGINX config file from template] ****************
changed: [nginx]

RUNNING HANDLER [role-01 : restart nginx] **************************************
changed: [nginx]

RUNNING HANDLER [role-01 : reload nginx] ***************************************
changed: [nginx]

PLAY RECAP *********************************************************************
nginx                      : ok=6    changed=5    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Nginx прослушивает порт 8080/tcp и отдаёт данные:
```sh
[vagrant@nginx ~]$ sudo -i
[root@nginx ~]# ss -tnlp
State       Recv-Q Send-Q                                 Local Address:Port                                                Peer Address:Port
LISTEN      0      128                                                *:111                                                            *:*                   users:(("rpcbind",pid=345,fd=8))
LISTEN      0      128                                                *:8080                                                           *:*                   users:(("nginx",pid=3905,fd=6),("nginx",pid=3828,fd=6))
LISTEN      0      128                                                *:22                                                             *:*                   users:(("sshd",pid=659,fd=3))
LISTEN      0      100                                        127.0.0.1:25                                                             *:*                   users:(("master",pid=903,fd=13))
LISTEN      0      128                                             [::]:111                                                         [::]:*                   users:(("rpcbind",pid=345,fd=11))
LISTEN      0      128                                             [::]:22                                                          [::]:*                   users:(("sshd",pid=659,fd=4))
LISTEN      0      100                                            [::1]:25                                                          [::]:*                   users:(("master",pid=903,fd=14))
[root@nginx ~]# curl -I http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Sat, 13 Aug 2022 15:07:17 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes


```

# **Результаты**

Выполнено развёртывание nginx на стенде с использованием Ansible и ролей.
Полученный в ходе работы `Vagrantfile` с настроенным Ansible provisioner помещен в публичный репозиторий.
При поднятии ВМ выполняется установка и настройка Nginx с использованием роли Ansible

- **GitHub** - https://github.com/jimidini77/otus-linux-day15
