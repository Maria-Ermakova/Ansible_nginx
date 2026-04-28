# Ansible_nginx
## the first steps with ansible

### Задание:
Используя Ansible необходимо развернуть nginx со следующими условиями:
- необходимо использовать модуль yum/apt
- конфигурационный файлы должны быть взяты из шаблона jinja2 с переменными
- после установки nginx должен быть в режиме enabled в systemd
- должен быть использован notify для старта nginx после установки
- сайт должен слушать на нестандартном порту - 8080, для этого использовать переменные в Ansible
* Сделать все это с использованием Ansible роли

### Решение:

Работаем на ВМ1 - Ansible

su -

убедиться что на сервере Ansible установлена версия Python не ниже 2.7, на современных системах он предустановлен версии 3 и выше

python3 --version

в случае успеха

apt install ansible

обеспечить подключение по ключу с ВМ1 Ansible на ВМ2 ClientAnsible

ssh-keygen -t ed25519 -C "ClientAnsible 912.168.56.18"     #укажу папку /home/leo/homework/ansible/private_key

ssh-copy-id -i /home/leo/homework/ansible/private_key.pub leo@192.168.56.18

создадим минимальную роль для nginx:

nginx_role/

├── tasks/main.yml        # Установка, конфигурация, запуск

├── handlers/main.yml     # Старт/рестарт NGINX

├── templates/nginx.conf.j2  # Шаблон с {{ nginx_listen_port }}

└── vars/main.yml         # nginx_listen_port: 8080

cd /home/leo/homework/ansible

mkdir -p nginx_role/{tasks,handlers,templates,vars}

cat >> nginx_role/tasks/main.yml << EOF
- name: update
  apt:
    update_cache: yes
  tags: 
    - update-apt  

- name: Install NGINX (Debian/Ubuntu)
  apt:
    name: nginx
    state: latest
  notify: start nginx
  tags: 
    - nginx-package

- name: Create NGINX config file from template
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: restart nginx
  tags: 
    - nginx-config
EOF

cat >> nginx_role/handlers/main.yml << EOF
- name: restart nginx
  systemd:
    name: nginx
    state: restarted
    daemon_reload: yes

- name: start nginx
  systemd:
    name: nginx
    state: started
    enabled: yes
EOF

cat >> nginx_role/templates/nginx.conf.j2 << EOF
#{{ ansible_managed }}
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
EOF

cat >> nginx_role/vars/main.yml << EOF
nginx_listen_port: 8080
EOF

теперь создадим плейбук с использованием роли
cat >> deploy_nginx.yml << EOF
- name: Deploy NGINX on a non-standard port using the role
  hosts: ClientAnsible
  become: yes
  gather_facts: yes  

  roles:
    - nginx_role
EOF

создадим инвентори файл

cat >> inventory.ini << EOF
[ClientAnsible]
192.168.56.18 ansible_user=leo ansible_ssh_private_key_file=/home/leo/homework/ansible/private_key ansible_become_password=2677
EOF

Проверка подключения

ansible -i inventory.ini ClientAnsible -m ping

Запуск плейбука в режиме проверки (dry-run)

ansible-playbook -i inventory.ini deploy_nginx.yml --check

Реальный запуск

ansible-playbook -i inventory.ini deploy_nginx.yml

Проверим доступность веб-сервера

curl http://192.168.56.18:8080
<img width="1913" height="567" alt="ansible_ok" src="https://github.com/user-attachments/assets/89265201-1044-4511-a763-68e11b17ed08" />
<img width="1173" height="560" alt="ansible_nginx_ok" src="https://github.com/user-attachments/assets/428b1060-6307-44bd-a898-d752611d5b6a" />

