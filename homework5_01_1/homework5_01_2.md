### Д3 5.1. 
### 5.1.2 _Ansible_
Все имена пользователей, пароли и иная чувствительная информация должна храниться в ansible-vault

Написать плейбук с ролями на свой выбор

На всех хостах должны быть установлены и настроены (при необходимости) через роль/роли пакеты:
- chrony
- yum-utils     
- mc
- curl
- htop
- atop
и удалены такие пакеты как AppArmor (в случае Debuntu), Cockpit

#### Предварительная структура проекта
```
Infrastructure-Automation/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tfvars
│   └── keys/
│       ├── id_rsa_terraform
│       └── id_rsa_terraform.pub
│
├── ansible/
│   ├── inventory/
│   │   └── inventory.yml
│   ├── playbook.yml
│   ├── vault/                          
│   │   └── secrets.yml               
│   └── roles/
│       ├── common/
│       │   ├── tasks/
│       │   │   └── main.yml
│       │   └── handlers/
│       │       └── main.yml
│       ├── back/
│       │   ├── tasks/
│       │   │   └── main.yml
│       │   └── handlers/
│       │       └── main.yml
│       ├── db/
│       │   ├── tasks/
│       │   │   └── main.yml
│       │   └── handlers/
│       │       └── main.yml
│       └── front/
│           ├── tasks/
│           │   └── main.yml
│           └── handlers/
│               └── main.yml
```
#### Ansible
```bash 
 [root@Zero ansible]# pwd
/root/Infrastructure-Automation/ansible
# Создаем папку для хранения зашифрованных данных
[root@Zero ansible]# mkdir -p vault/
[root@Zero ansible]# ansible-vault create vault/secrets.yml
# Создаем пароль для доступа к файлу с нашими чувствительными данными
New Vault password:    # 123...            
# После подтверждения пароля, в vim открывается файл, куда вносим нужные данные
Confirm New Vault password:      
[root@Zero ansible]# ansible-vault edit vault/secrets.yml  # команда для редактирования
# Вводим созданный ранее пароль и обновляем данные 
Vault password:  # 123... 
# Опционально: меняем редактор на nano   
[root@Zero ansible]# DITOR=nano ansible-vault edit vault/secrets.yml # файл откроется в nano
# Команда для просмотра secrets.yml без редактирования 
[root@Zero ansible]#  ansible-vault view  vault/secrets.yml  
# Создаем нужные папки и файлы
[root@Zero ansible]# mkdir -p roles/{common,back,db,front}/{tasks,handlers} 
# Проверяем стуктуру 
[root@Zero ansible]# tree
.
├── inventory
│   └── inventory.yml
├── roles
│   ├── back
│   │   ├── handlers
│   │   └── tasks
│   ├── common
│   │   ├── handlers
│   │   └── tasks
│   ├── db
│   │   ├── handlers
│   │   └── tasks
│   └── front
│       ├── handlers
│       └── tasks
└── vault
    └── secrets.yml

15 directories, 2 files

```
#### Роль common
Выполняет базовые настройки, которые по условию применяются ко всем хостам в данной инфраструктуре. 

_Устанавливает общие пакеты:_
- chrony — для синхронизации времени;
- yum-utils — утилиты для работы с пакетами (на CentOS/RHEL);
- mc (Midnight Commander) — файловый менеджер;
- curl — утилита для работы с HTTP-запросами;
- htop — мониторинг ресурсов системы;
- atop — расширенный мониторинг системы.

_Настраивает и запускает службу chrony:_
- убеждается, что служба chrony включена и работает.

_Удаляет ненужные пакеты:_
- apparmor — система безопасности (если используется);
- cockpit — веб-интерфейс для управления сервером.

_Рекомендации https://habr.com/ru/articles/508762/):_ 
- Play определяет, где и что выполнять.
- Роль используется для структурирования кода.
- Роль не принимает решений о том, где выполняться.
- Не нужно смешивать задачи и роли.
- Не следует злоупотреблять хэндлерами.
- Роль не должна пытаться быть "библиотечной функцией".


```bash 
# Код файлов 
root@Zero ansible]# cat roles/common/tasks/main.yml 
---
- name: Install common packages # Устанавливаем общие пакеты 
  apt:
    name: "{{ item }}"  # Устанавливаемый пакет
    state: present      # Убедиться, что пакет установлен
    update_cache: yes   # Обновить кеш пакетов перед установкой
  loop:
    - apt-utils        # Утилиты для работы с пакетами для Ubuntu
    - chrony           # Пакет для синхронизации времени
#   - yum-utils        # Утилиты для работы с пакетами (на CentOS/RHEL)
    - apt-utils        # Утилиты для работы с пакетами для Ubuntu 
    - mc               # Файловый менеджер Midnight Commander
    - curl             # Утилита для работы с HTTP-запросами
    - htop             # Мониторинг ресурсов системы
    - atop             # Расширенный мониторинг системы

- name: Remove unwanted packages # Удаляем ненужные пакеты 
  apt:
    name: "{{ item }}"  # Пакет для удаления
    state: absent       # Убедиться, что пакет удален
  loop:
    - apparmor         # Система безопасности (если используется на Debian/Ubuntu)
    - cockpit          # Веб-интерфейс для управления сервером

- name: Ensure chrony is running and enabled # Проверяем, что chrony запущен и включен
  service:
    name: chrony       # Название службы
    state: started     # Убедиться, что служба запущена
    enabled: yes       # Убедиться, что служба включена для автозапуска


[root@Zero ansible]# cat  playbook.yml
---
- name: Apply common configuration to all hosts
  hosts: all
  become: yes
  roles:
    - common

```
```bash
#  Выполняем плейбук Ansible, используя указанный файл inventory.yml
[root@Zero ansible]# ansible-playbook -i inventory/inventory.yml playbook.yml

PLAY [Apply common configuration to all hosts] ***********************************************************************

TASK [Gathering Facts] ***********************************************************************************************
ok: [172.19.0.2]
ok: [172.19.0.3]
ok: [172.17.0.2]
ok: [172.19.0.4]

TASK [common : Install common packages] *******************************************************************************
ok: [172.19.0.3] => (item=apt-utils)
ok: [172.19.0.2] => (item=apt-utils)
ok: [172.19.0.3] => (item=chrony)
ok: [172.19.0.2] => (item=chrony)
ok: [172.19.0.4] => (item=apt-utils)
ok: [172.17.0.2] => (item=apt-utils)
ok: [172.19.0.3] => (item=apt-utils)
ok: [172.19.0.2] => (item=apt-utils)
ok: [172.19.0.2] => (item=mc)
ok: [172.19.0.3] => (item=mc)
ok: [172.19.0.4] => (item=chrony)
ok: [172.19.0.2] => (item=curl)
ok: [172.19.0.3] => (item=curl)
ok: [172.17.0.2] => (item=chrony)
ok: [172.19.0.2] => (item=htop)
ok: [172.19.0.3] => (item=htop)
ok: [172.19.0.4] => (item=apt-utils)
ok: [172.19.0.2] => (item=atop)
ok: [172.19.0.3] => (item=atop)
ok: [172.17.0.2] => (item=apt-utils)
ok: [172.19.0.4] => (item=mc)
ok: [172.17.0.2] => (item=mc)
ok: [172.19.0.4] => (item=curl)
ok: [172.17.0.2] => (item=curl)
ok: [172.19.0.4] => (item=htop)
ok: [172.17.0.2] => (item=htop)
ok: [172.19.0.4] => (item=atop)
ok: [172.17.0.2] => (item=atop)

TASK [common : Remove unwanted packages] *****************************************************************************
ok: [172.19.0.3] => (item=apparmor)
ok: [172.19.0.2] => (item=apparmor)
ok: [172.19.0.2] => (item=cockpit)
ok: [172.19.0.3] => (item=cockpit)
ok: [172.17.0.2] => (item=apparmor)
ok: [172.19.0.4] => (item=apparmor)
ok: [172.19.0.4] => (item=cockpit)
ok: [172.17.0.2] => (item=cockpit)

TASK [common : Ensure chrony is running and enabled] *****************************************************************
ok: [172.19.0.3]
ok: [172.19.0.2]
ok: [172.17.0.2]
ok: [172.19.0.4]

PLAY RECAP ***********************************************************************************************************
172.17.0.2                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
172.19.0.2                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
172.19.0.3                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0   
172.19.0.4                 : ok=4    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0       

```
