# otus-linux-day24
## *Rsyslog*

# **Prerequisite**
- Host OS: Debian 11.3.0
- Guest OS: CentOS 7.8.2003
- VirtualBox: 6.1.36
- Vagrant: 2.2.19
- Ansible: 2.12.4

# **Содержание ДЗ**

* На сервере, используя Ansible, необходимо развернуть 2 машины (web, log) со следующими условиями:
  - на web настраивается nginx;
  - на log настраивается центральный лог сервер на rsyslog;
  - настраивается аудит, следящий за изменением конфигов nginx;
  - все критичные логи с web должны собираться и локально и удаленно;
  - все логи с nginx должны уходить на удаленный сервер (локально только критичные);
  - логи аудита должны также уходить на удаленную систему;

# **Выполнение**
Далее приведены выдержки содержимого playbook.yml

На обоих хостах настраивается одинаковое время:
```yml
- name: CHRONYD | Tuning
  hosts: all
  become: true
  tasks:
    - name: CHRONYD | copy config file
      copy:
        src: /usr/share/zoneinfo/Europe/Moscow
        dest: /etc/localtime
        remote_src: true
      notify:
        - restart chronyd
      tags:
        - chronyd

  handlers:
    - name: restart chronyd
      systemd:
        name: chronyd
        state: restarted
        enabled: yes
```
На web устанавливается nginx:
```yml
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
```
Порт tcp\80 прослушивается и отдаёт данные:
```sh
[root@web vagrant]# ss -tlpn | grep 80
LISTEN     0      128          *:80                       *:*                   users:(("nginx",pid=4656,fd=6),("nginx",pid=4655,fd=6))
LISTEN     0      128       [::]:80                    [::]:*                   users:(("nginx",pid=4656,fd=7),("nginx",pid=4655,fd=7))
[root@web vagrant]# curl -I http://localhost
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Sat, 03 Sep 2022 09:52:29 GMT
Content-Type: text/html
Content-Length: 4833
Last-Modified: Fri, 16 May 2014 15:12:48 GMT
Connection: keep-alive
ETag: "53762af0-12e1"
Accept-Ranges: bytes
```

### Настройка центрального сервера сбора логов

Правка конфигов rsyslog:
```yml
- name: LOG | Configuration
  hosts: log
  become: true
  tasks:
    - name: RSYSLOG | Edit config
      replace:
        path: /etc/rsyslog.conf
        regexp: "{{ item }}"
        replace: '\1'
      loop:
        - '^\s*#*\s*(\$ModLoad imudp.*)'
        - '^\s*#*\s*(\$UDPServerRun 514.*)'
        - '^\s*#*\s*(\$ModLoad imtcp.*)'
        - '^\s*#*\s*(\$InputTCPServerRun 514.*)'
      notify:
        - restart rsyslog
      tags:
        - rsyslog

    - name: RSYSLOG | Append config
      blockinfile:
        path: /etc/rsyslog.conf
        block: |
          $template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
          *.* ?RemoteLogs
          & ~
      notify:
        - restart rsyslog
      tags:
        - rsyslog
```
Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. Например, Access-логи nginx от сервера web, будут идти в файл `/var/log/rsyslog/web/nginx_access.log`

Порты прослушивает rsyslog:
```sh
[root@log vagrant]# ss -tunlp | grep :514
udp    UNCONN     0      0         *:514                   *:*                   users:(("rsyslogd",pid=3981,fd=3))
udp    UNCONN     0      0      [::]:514                [::]:*                   users:(("rsyslogd",pid=3981,fd=4))
tcp    LISTEN     0      25        *:514                   *:*                   users:(("rsyslogd",pid=3981,fd=5))
tcp    LISTEN     0      25     [::]:514                [::]:*                   users:(("rsyslogd",pid=3981,fd=6))
```

### Настройка отправки логов с web-сервера

Для access-логов указывается удаленный сервер и уровень логов, которые нужно отправлять. Для error-логов добавляется удаленный сервер. 
```yml
    - name: NGINX | configure logging
      lineinfile:
        path: /etc/nginx/nginx.conf
        insertafter: "{{ item.search }}"
        line: "{{ item.after }}"
      with_items: 
        - { search: '\s*error_log.*', after: 'error_log syslog:server={{ log_vm_ip }}:514,tag=nginx_error;' }
        - { search: '\s*access_log.*', after: 'access_log syslog:server={{ log_vm_ip }}:514,tag=nginx_access,severity=info combined;' }
      notify:
        - restart nginx
      tags:
        - nginx-package
        - nginx-conf
```

Для провеки, что логи ошибок также попадают на удаленный сервер log, можно удалить картинку, к которой будет обращаться nginx во время открытия веб-страницы 
```
rm /usr/share/nginx/html/img/header-background.png
```
После выполнения запроса к веб-серверу nginx_access.log и nginx_error.log содержат записи:
```sh
[root@log vagrant]# cat /var/log/rsyslog/web/nginx_error.log
Sep  3 10:23:43 web nginx_error: 2022/09/03 10:23:43 [error] 4656#4656: *3 open() "/usr/share/nginx/html/img/header-background.png" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /img/header-background.png HTTP/1.1", host: "192.168.56.10"
[root@log vagrant]#
[root@log vagrant]#
[root@log vagrant]# cat /var/log/rsyslog/web/nginx_access.log
Sep  3 09:26:09 web nginx_access: 192.168.56.1 - - [03/Sep/2022:09:26:09 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.84.0"
Sep  3 09:52:29 web nginx_access: ::1 - - [03/Sep/2022:09:52:29 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.29.0"
Sep  3 10:23:43 web nginx_access: 192.168.56.1 - - [03/Sep/2022:10:23:43 +0000] "GET /img/header-background.png HTTP/1.1" 404 3650 "-" "Wget/1.21.3"
Sep  3 10:32:55 web nginx_access: 192.168.56.1 - - [03/Sep/2022:10:32:55 +0000] "GET / HTTP/1.1" 200 4833 "-" "curl/7.84.0"
[root@log vagrant]# client_loop: send disconnect: Connection reset
```

### Настройка аудита, контролирующего изменения конфигурации nginx
Добавляется правило, которое будет отслеживать изменения в конфигурации nginx. Для этого в конец файла `/etc/audit/rules.d/audit.rules` добавляются строки:
```yml
    - name: NGINX | Audit config changes
      blockinfile:
        path: /etc/rsyslog.conf
        block: |
          -w /etc/nginx/nginx.conf -p wa -k nginx_conf
          -w /etc/nginx/default.d/ -p wa -k nginx_conf
      notify:
        - restart auditd
      tags:
        - auditd-conf
```
Данные правила позволяют контролировать запись (w) и измения атрибутов (a) в:
- /etc/nginx/nginx.conf
- Всех файлов каталога /etc/nginx/default.d/

Чтобы проверить, что логи аудита начали записываться локально, нужно внести изменения в файл /etc/nginx/nginx.conf или поменять его атрибут, затем посмотреть информацию об изменениях:
```sh
[root@web vagrant]# grep nginx_conf /var/log/audit/audit.log
node=web type=CONFIG_CHANGE msg=audit(1662222570.700:2855): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1662222570.700:2856): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
node=web type=CONFIG_CHANGE msg=audit(1662222586.168:2858): auid=1000 ses=19 op=updated_rules path="/etc/nginx/nginx.conf" key="nginx_conf" list=4 res=1
node=web type=SYSCALL msg=audit(1662222586.168:2859): arch=c000003e syscall=82 success=yes exit=0 a0=142d750 a1=1467d00 a2=fffffffffffffe80 a3=7ffe86fd79c0 items=4 ppid=26782 pid=26855 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=19 comm="vi" exe="/usr/bin/vi" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CONFIG_CHANGE msg=audit(1662222586.169:2860): auid=1000 ses=19 op=updated_rules path="/etc/nginx/nginx.conf" key="nginx_conf" list=4 res=1
node=web type=SYSCALL msg=audit(1662222586.169:2861): arch=c000003e syscall=2 success=yes exit=3 a0=142d750 a1=241 a2=1a4 a3=0 items=2 ppid=26782 pid=26855 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=19 comm="vi" exe="/usr/bin/vi" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=SYSCALL msg=audit(1662222586.187:2862): arch=c000003e syscall=188 success=yes exit=0 a0=142d750 a1=7f8667eecf6a a2=1467d30 a3=24 items=1 ppid=26782 pid=26855 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=19 comm="vi" exe="/usr/bin/vi" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=SYSCALL msg=audit(1662222586.188:2863): arch=c000003e syscall=90 success=yes exit=0 a0=142d750 a1=81a4 a2=0 a3=24 items=1 ppid=26782 pid=26855 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=19 comm="vi" exe="/usr/bin/vi" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=SYSCALL msg=audit(1662222586.188:2864): arch=c000003e syscall=188 success=yes exit=0 a0=142d750 a1=7f8667aa2e2f a2=1467e40 a3=1c items=1 ppid=26782 pid=26855 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=19 comm="vi" exe="/usr/bin/vi" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
```

Настраивается пересылка логов сервера web на удаленный сервер log. Требуется установить пакет audispd-plugins и внести изменения в конфиги `/etc/audit/auditd.conf`, `/etc/audisp/plugins.d/au-remote.conf`, `/etc/audisp/audisp-remote.conf`:

```yml
    - name: AUDITD | Install audispd-plugins package
      yum:
        name: audispd-plugins
        state: latest
      tags:
        - packages

    - name: AUDITD | Edit config
      lineinfile:
        path: "{{ item.file }}"
        regexp: "{{ item.search }}"
        line: "{{ item.replace }}"
      with_items: 
        - { search: '^log_format = .*', replace: 'log_format = RAW', file: '/etc/audit/auditd.conf' }
        - { search: '^name_format = .*', replace: 'name_format = HOSTNAME', file: '/etc/audit/auditd.conf' }
        - { search: '^active = .*', replace: 'active = yes', file: '/etc/audisp/plugins.d/au-remote.conf' }
        - { search: '^remote_server =.*', replace: 'remote_server = {{ log_vm_ip }}', file: '/etc/audisp/audisp-remote.conf' }
      notify:
        - restart auditd
      tags:
        - auditd-conf
```

Настраивается auditd Log-сервера, отрывается порт tcp\60:
```yml
    - name: AUDITD | Edit config
      replace:
        path: /etc/audit/auditd.conf
        regexp: '^\s*#*\s*(tcp_listen_port.*)'
        replace: '\1'
      notify:
        - restart auditd
      tags:
        - auditd-conf
```
Например при смене атрибута файла `/etc/nginx/nginx.conf` на сервере web, в журнале audit сервера log появляются записи:
```
[root@log vagrant]# tail /var/log/audit/audit.log
node=web type=SYSCALL msg=audit(1662284765.683:1623): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=1c370f0 a2=1ed a3=7ffec2a76030 items=1 ppid=5068 pid=5083 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=6 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CWD msg=audit(1662284765.683:1623):  cwd="/home/vagrant"
node=web type=PATH msg=audit(1662284765.683:1623): item=0 name="/etc/nginx/nginx.conf" inode=33565397 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
node=web type=PROCTITLE msg=audit(1662284765.683:1623): proctitle=63686D6F64002B78002F6574632F6E67696E782F6E67696E782E636F6E66
```

## Ansible
Структура проекта с использованием Ansible:
```sh
root@nas:/study/day24# tree
.
├── ansible
│   ├── hosts
│   └── provision.yml
├── ansible.cfg
└── Vagrantfile

1 directory, 4 files
```
Конфигурационный файл Ansible содержит параметры подключения:
```sh
root@nas:/study/day24# cat ansible.cfg
[defaults]
inventory = ansible/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```

Inventory:
```sh
root@nas:/study/day24# cat ansible/hosts
[stage]
web ansible_host=192.168.56.10 ansible_port=22 ansible_private_key_file=.vagrant/machines/web/virtualbox/private_key
log ansible_host=192.168.56.15 ansible_port=22 ansible_private_key_file=.vagrant/machines/log/virtualbox/private_key
```

# **Результаты**

Выполнено развёртывание стенда с настройкой пересылки и сбора событий на log-сервере.
Полученный в ходе работы `Vagrantfile` с настроенным Ansible provisioner помещен в публичный репозиторий.
При поднятии ВМ выполняется установка и настройка Nginx и Rsyslog с использованием Ansible.

- **GitHub** - https://github.com/jimidini77/otus-linux-day24
