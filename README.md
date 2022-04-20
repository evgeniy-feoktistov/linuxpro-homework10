# linuxpro-homework10

Домашнее задание
Первые шаги с Ansible

Описание/Пошаговая инструкция выполнения домашнего задания:
Подготовить стенд на Vagrant как минимум с одним сервером. На этом сервере используя Ansible необходимо развернуть nginx со следующими условиями:

необходимо использовать модуль yum/apt;
конфигурационные файлы должны быть взяты из шаблона jinja2 с перемененными;
после установки nginx должен быть в режиме enabled в systemd;
должен быть использован notify для старта nginx после установки;
сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible.

***

### Установка Ansible

Проверяем версию Python
```
$ python3 -V
Python 3.8.10
```
Устанавливаем согласно документации для Ubuntu
```
$ sudo apt-get install software-properties-common
$ sudo apt-add-repository --yes --update ppa:ansible/ansible
$ sudo apt-get install ansible

$ ansible --version
ansible [core 2.12.4]
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/ujack/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  ansible collection location = /home/ujack/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/bin/ansible
  python version = 3.8.10 (default, Mar 15 2022, 12:22:08) [GCC 9.4.0]
  jinja version = 2.10.1
  libyaml = True
```
Создадим каталог **Ansible** и положем в него **Vagrant** файл
```
$ mkdir Ansible
$ cat Ansible/Vagrantfile
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :nginx => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.150'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "200"]
          end

          box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh; cp ~vagrant/.ssh/auth* ~root/.ssh
            sed -i '65s/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config
            systemctl restart sshd
          SHELL

      end
  end
end
```
Проверим, что VM запускается и мы можем к ней подключится
```
$ vagrant ssh
[vagrant@nginx ~]$ logout
Connection to 127.0.0.1 closed.
```
Посмотрим параметры подключения по ssh
```
$ vagrant ssh-config
Host nginx
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /home/ujack/otus-linuxadminpro/linuxpro-homework10/Ansible/.vagrant/machines/nginx/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```
Создадим файл **inventory** со следующим содержанием
```
$ $ cat inventory
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_user=vagrant ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```
Проверим, что Ansible может управлять нашим хостом
```
$ ansible nginx -i inventory -m ping
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ECDSA key fingerprint is SHA256:sqcO7BLI5euhJixktD2Dh1YX3Hs3cdqdrM5sPMoF3gs.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
При первом запуске нужно подтверждение при подключении по ssh (like as always)

Чтобы каждый раз при запуске ansible не указывать инвентору файл создаим в текущей директории файл ansible.cfg со следующим содержанием, а из файл inventory уберем указание пользователя
```
$ cat ansible.cfg
[defaults]
inventory = inventory
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
ujack@ubuntu-otus-admlinuxpro:~/otus-linuxadminpro/linuxpro-homework10/Ansible$ cat inventory
[web]
nginx ansible_host=127.0.0.1 ansible_port=2222 ansible_private_key_file=.vagrant/machines/nginx/virtualbox/private_key
```
Тогда команда
```
$ ansible nginx -m ping
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```
отрабатывает.
Теперь мы можем управлять нашим хостом.
Посмотрим, какое ядро установлено в VM
```
$ ansible nginx -m command -a "uname -r"
nginx | CHANGED | rc=0 >>
3.10.0-1127.el7.x86_64
```
Проверим статус firewalld
```
$ ansible nginx -m systemd -a name=firewalld
nginx | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "name": "firewalld",
    "status": {
        ...
        "ActiveState": "inactive",
        ...
    }
}

```
"ActiveState": "inactive"
Установим пакет epel-release в нашу VM
```
$ ansible nginx -m yum -a "name=epel-release state=present" -b
nginx | SUCCESS => {
"changed": true
...
```
Создадим Playbook по установке epel
Для этого создадим файл **epel.yml** со следующим содержанием
```
$ cat epel.yml
---
- name: Install EPEL Repo
  hosts: nginx
  become: true
  tasks:
   - name: Install EPEL Repo package from standard repo
     yum:
      name: epel-release
      state: present
```
Запустим нам **playbook** командой **ansible-playbook epel.yml**
```
$ ansible-playbook epel.yml

PLAY [Install EPEL Repo] ***********************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standard repo] ********************************************************************************
ok: [nginx]

PLAY RECAP *************************************************************************************************************************
nginx                      : ok=2    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
Выполним команду **ansible nginx -m yum -a "name=epel-release state=absent" -b**
```
$ ansible nginx -m yum -a "name=epel-release
> state=absent" -b
nginx | CHANGED => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": true,
    "changes": {
        "removed": [
            "epel-release"
        ]
    },
    "msg": "",
    "rc": 0,
    "results": [
        "Loaded plugins: fastestmirror\nResolving Dependencies\n--> Running transaction check\n---> Package epel-release.noarch 0:7-11 will be erased\n--> Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package                Arch             Version        Repository         Size\n================================================================================\nRemoving:\n epel-release           noarch           7-11           @extras            24 k\n\nTransaction Summary\n================================================================================\nRemove  1 Package\n\nInstalled size: 24 k\nDownloading packages:\nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Erasing    : epel-release-7-11.noarch                                     1/1 \n  Verifying  : epel-release-7-11.noarch                                     1/1 \n\nRemoved:\n  epel-release.noarch 0:7-11                                                    \n\nComplete!\n"
    ]
}
```
Данная команда удаляет **epel** из системы.
И запустим наш **playbook** еще раз
```
$ ansible-playbook epel.yml

PLAY [Install EPEL Repo] ***********************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************
ok: [nginx]

TASK [Install EPEL Repo package from standard repo] ********************************************************************************
changed: [nginx]

PLAY RECAP *************************************************************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```
В прошлый раз изменений никаких у нас не было, так как **epel** был уже установлен в системе. Теперь мы видим, что **changed=1**

Приступим к написанию **playbook** для установки **nginx**
Скопируем наш **epel.yml** в **nginx.yml** и добавим в него установку пакета **nginx**. Добавим **tags**
```
$ cp epel.yml nginx.yml
$ cat nginx.yml
---
- name: Install EPEL Repo
  hosts: nginx
  become: true
  tasks:
   - name: Install EPEL Repo package from standard repo
     yum:
      name: epel-release
      state: present
     tags:
       - epel-pakage
       - packages

   - name: Install nginx packege from epel repo
     yum:
      name: nginx
      state: latest
     tags:
       - nginx-package
       - packages
```
Теперь мы можем получить список тегов, и выполнять действия только для определенных из них.
```
$ ansible-playbook nginx.yml --list-tags

playbook: nginx.yml

  play #1 (nginx): Install EPEL Repo    TAGS: []
      TASK TAGS: [epel-pakage, nginx-package, packages]

```
Например, запустим только установку **nginx**
```
$ ansible-playbook nginx.yml -t nginx-package

PLAY [Install EPEL Repo] ***********************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************
ok: [nginx]

TASK [Install nginx packege from epel repo] ****************************************************************************************
changed: [nginx]

PLAY RECAP *************************************************************************************************************************
nginx                      : ok=2    changed=1    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
Далее добавим шаблон для конфига **nginx** и модуль, который будет копировать этот шаблон в ВМ:
```
$ cat nginx.yml
---
- name: Install EPEL Repo
  hosts: Nginx | Install and configure nginx
  become: true
  vars:
    nginx_listen_port: 8080
  tasks:
   - name: Install EPEL Repo package from standard repo
     yum:
       name: epel-release
       state: present
     tags:
       - epel-pakage
       - packages

   - name: Install nginx packege from epel repo
     yum:
       name: nginx
       state: latest
     tags:
       - nginx-package
       - packages

   - name: Nginx | Create nginx config file from template
     template:
       src: templates/nginx.conf.j2
       dest: /tmp/nginx.conf
     tags:
       - nginx-configuration
```
Создадим сам шаблон
```
$ cat templates/nginx.conf.j2
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
Теперь создадим handler и добавим notify к копированию шаблона. Теперь каждый раз когда конфиг будет изменяться - сервис перезагрузится.
Конфиг примет вид:
```
$ cat nginx.yml
---
- name: Nginx | Install and configure nginx
  hosts: nginx
  become: true
  vars:
    nginx_listen_port: 8080
  tasks:
   - name: Install EPEL Repo package from standard repo
     yum:
       name: epel-release
       state: present
     tags:
       - epel-pakage
       - packages

   - name: Install nginx packege from epel repo
     yum:
       name: nginx
       state: latest
     notify:
       - restart nginx
     tags:
       - nginx-package
       - packages

   - name: Nginx | Create nginx config file from template
     template:
       src: templates/nginx.conf.j2
       dest: /tmp/nginx.conf
     notify:
       - reload nginx
     tags:
       - nginx-configuration

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enable: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```
Запустим его:
```
$ ansible-playbook nginx.yml

PLAY [Nginx | Install and configure nginx] *****************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************
ok: [nginx]

PLAY [Install EPEL Repo] ***********************************************************************************************************

TASK [Install EPEL Repo package from standard repo] ********************************************************************************
ok: [nginx]

TASK [Install nginx packege from epel repo] ****************************************************************************************
ok: [nginx]

TASK [Nginx | Create nginx config file from template] ******************************************************************************
changed: [nginx]

RUNNING HANDLER [reload nginx] *****************************************************************************************************
changed: [nginx]

PLAY RECAP *************************************************************************************************************************
nginx                      : ok=5    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```
И проверим доступность **nginx**
```
$ curl http://192.168.56.150:8080
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css">
...
```

Теперь создадим роль.
```
Ansible$ ansible-galaxy init nginx-role
Ansible$ ll nginx-role/
total 48
drwxrwxr-x 10 ujack ujack 4096 апр 20 13:24 ./
drwxrwxr-x  5 ujack ujack 4096 апр 20 13:24 ../
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 defaults/
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 files/
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 handlers/
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 meta/
-rw-rw-r--  1 ujack ujack 1328 апр 20 13:24 README.md
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 tasks/
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 templates/
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 tests/
-rw-rw-r--  1 ujack ujack  539 апр 20 13:24 .travis.yml
drwxrwxr-x  2 ujack ujack 4096 апр 20 13:24 vars/
```
Создалось необходимое дерево каталогов для роли
Для порядка создадим каталог **roles**, перенесем в него каталог с нашей ролью **nginx-roles**
```
Ansible$ mkdir roles
Ansible$ mv nginx-role/ roles/
Ansible$ ll roles/
total 12
drwxrwxr-x  3 ujack ujack 4096 апр 20 13:33 ./
drwxrwxr-x  5 ujack ujack 4096 апр 20 13:33 ../
drwxrwxr-x 10 ujack ujack 4096 апр 20 13:24 nginx-role/
```
Сделаем отдельный playbook для использования роли
```
Ansible$ cat nginx-role.yml
---
- name: Nginx | Install and configure nginx
  become: yes
  hosts: nginx
  vars:
    nginx_listen_port: 8080

  roles:
    - nginx-role
```
В **Ansible.cfg** укажем где искать роли чекрез переменную **roles_path**
```
$ cat ansible.cfg
[defaults]
inventory = inventory
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
roles_path = roles
```
Перенесем наши таски и хэндлеры в соответствующие файлы роли
```
Ansible$ cat roles/nginx-role/tasks/main.yml
---
# tasks file for nginx-role
   - name: Install EPEL Repo package from standard repo
     yum:
       name: epel-release
       state: present
     tags:
       - epel-pakage
       - packages

   - name: Install nginx packege from epel repo
     yum:
       name: nginx
       state: latest
     notify:
       - restart nginx
     tags:
       - nginx-package
       - packages

   - name: Nginx | Create nginx config file from template
     template:
       src: nginx.conf.j2
       dest: /etc/nginx/nginx.conf
     notify:
       - reload nginx
     tags:
       - nginx-configuration
```

```
Ansible$ cat roles/nginx-role/handlers/main.yml
---
# handlers file for nginx-role
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enable: yes

    - name: reload nginx
      systemd:
        name: nginx
        state: reloaded
```
Запустим наш плейбук с ролью
```
Ansible$ ansible-playbook nginx-role.yml

PLAY [Nginx | Install and configure nginx] **********************************************************************************************************************************************************************************************************************************

TASK [Gathering Facts] ******************************************************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [nginx-role : Install EPEL Repo package from standard repo] ************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [nginx-role : Install nginx packege from epel repo] ********************************************************************************************************************************************************************************************************************
ok: [nginx]

TASK [nginx-role : Nginx | Create nginx config file from template] **********************************************************************************************************************************************************************************************************
ok: [nginx]

PLAY RECAP ******************************************************************************************************************************************************************************************************************************************************************
nginx                      : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

Done. Имеем и отдельный плэйбук со всеми тасками все в одном и плэйбук с ролью.

З.Ы.
Для наглядности nginx.conf.j2 содержит разные данные.

