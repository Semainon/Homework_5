### Д3 5.1. 
### 5.1.1 _Terraform_
С помощью _terraform_ создать:
1. Изолированную частную сеть
2. Три сервера:
- _front_
   - 2 интерфейса: 1 – внешняя сеть, 2 – изолированная сеть,
   - ширина внешнего канала – 110 мбит/с,
- _back_ ( только интерфейс изолированной сети ),
- _db_ (только интерфейс изолированной сети ).

Подключение к созданным машинам сделать возможным только по ssh-ключу
На сервер _db_ должен быть дополнительный диск

Сформировать ansible-inventory с 3 группами: _front_, _back_ и _db_, где будут присутствовать созданные выше хосты


#### Предварительная структура проекта
```
Infrastructure-Automation/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tfvars
│   └── keys/                     # Папка для ключей SSH
│       ├── id_rsa_terraform      # Приватный ключ
│       └── id_rsa_terraform.pub  # Публичный ключ
│
├── ansible/
│   ├── inventory/
│   │   └── inventory.yml
│   ├── playbook.yml
│   ├── group_vars/
│   │   ├── all.yml
│   │   ├── front.yml
│   │   ├── back.yml
│   │   └── db.yml
│   └── roles/  
```
#### Terraform 
```bash 
# Устанавливаем Terraform 
[root@Zero ~]# wget https://releases.hashicorp.com/terraform/1.10.5/terraform_1.10.5_linux_amd64.zip 
[root@Zero ~]# unzip terraform_1.10.5_linux_amd64.zip
[root@Zero ~]# mv terraform /usr/local/bin/
[root@Zero ~]# chmod +x /usr/local/bin/terraform
[root@Zero ~]# terraform --version
Terraform v1.10.5
[root@Zero ~]# rm /root/terraform_1.10.5_linux_amd64.zip
# Устанавливаем Ansible
[root@Zero ~]# sudo yum install -y ansible
# Определяем предварительную структуру проекта, добавляем необходимые папки и файлы  
[root@Zero ~]# cd Infrastructure-Automation 
[root@Zero Infrastructure-Automation]# mkdir -p terraform/keys
[root@Zero Infrastructure-Automation]# touch terraform/main.tf
# Генерируем SSH-ключи 
[root@Zero ~]# ssh-keygen -t rsa -b 2048 -f ~/Infrastructure-Automation/terraform/keys/id_rsa_terraform
[root@Zero ~]# ssh-keygen -t rsa -b 2048 -f ~/Infrastructure-Automation/terraform/keys/id_rsa_terraform
# Устанавливаем Go (понадобится для сборки провайдера Docker из исходного кода)
[root@Zero ~]# sudo yum update -y
[root@Zero ~]# sudo yum install -y git gcc
[root@Zero ~]# wget https://golang.org/dl/go1.24.0.linux-amd64.tar.gz
[root@Zero ~]# sudo tar -C /usr/local -xzf go1.24.0.linux-amd64.tar.gz
[root@Zero ~]# echo "export PATH=$PATH:/usr/local/go/bin" >> ~/.bashrc
[root@Zero ~]# source ~/.bashrc
[root@Zero ~]# go version
[root@Zero ~]# go version
go version go1.24.0 linux/amd64
# Собираем провайдер Docker
[root@Zero ~]# cd Infrastructure-Automation/terraform
[root@Zero terraform]# ls
keys  main.tf  terraform-provider-docker  terraform.tfvars
[root@Zero terraform]# git clone https://github.com/kreuzwerker/terraform-provider-docker.git
[root@Zero terraform]# cd terraform-provider-docker
[root@Zero terraform-provider-docker]# go build
# Создаем директорию для провайдеров и перемещаем собранный файл:
[root@Zero terraform]# mkdir -p ~/.terraform.d/plugins/registry.terraform.io/kreuzwerker/docker/3.0.2/linux_amd64
[root@Zero terraform]# mv terraform-provider-docker ~/.terraform.d/plugins/registry.terraform.io/kreuzwerker/docker/3.0.2/linux_amd64/
[root@Zero terraform]# cd /root/Infrastructure-Automation/terraform
# Инициализируем Terraform:
[root@Zero terraform]# terraform init
# Проверка конфигурацииTerraform:
[root@Zero terraform]# terraform validate
Success! The configuration is valid.
[root@Zero terraform]# terraform plan
[root@Zero terraform]# terraform apply # вывод на скрине apply.png 
... 
... 
... 
Apply complete! Resources: 3 added, 0 changed, 3 destroyed.
# Проверяем работу контейнеров:
[root@Zero terraform]# docker ps
# Проверяем на соотвествие ТЗ 
# Получаем  IP-адреса 
[root@Zero terraform]# terraform show 
[root@Zero terraform]# docker network inspect bridge
[root@Zero terraform]# docker network inspect isolated_network
# Получили IP-адреса контейнеров
# Контейнер front:
# Внешняя сеть (bridge): 172.17.0.2.
# Изолированная сеть (isolated_network): 172.19.0.4.
# Контейнер back:
# Изолированная сеть (isolated_network): 172.19.0.3.
# Контейнер db:
# Изолированная сеть (isolated_network): 172.19.0.2.
# Подключение к контейнеру front, внешний интерфейс:
[root@Zero terraform]# ssh -i keys/id_rsa_terraform root@172.17.0.2   # успешно
# Подключение к контейнеру front, изолированный интерфейс 
[root@Zero terraform]# ssh -i keys/id_rsa_terraform root@172.19.0.4   # успешно
# Подключение к контейнеру back:
[root@Zero terraform]# ssh -i keys/id_rsa_terraform root@172.19.0.3   # успешно
# Подключение к контейнеру db:
[root@Zero terraform]# ssh -i keys/id_rsa_terraform root@172.19.0.2   # успешно
#  Проверка доступа по паролю:
[root@Zero terraform]# ssh root@172.17.0.2   
root@172.17.0.2: Permission denied (publickey). # SSH настроен правильно 
``` 
#### Ansible inventory
```bash
# Формируем ansible-inventory
[root@Zero Infrastructure-Automation]# mkdir ansible
[root@Zero Infrastructure-Automation]# cd ansible
[root@Zero ansible]# mkdir inventory
[root@Zero ansible]# nano inventory/inventory.yml
# Проверяем доступность хостов с помощью Ansible
[root@Zero Infrastructure-Automation]# ansible all -i ansible/inventory/inventory.yml -m ping
172.19.0.2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
172.19.0.3 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
172.19.0.4 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
172.17.0.2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
``` 

#### main.tf
```bash 
variable "docker_host" {
  description = "Docker host address"
  type        = string
}

terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "3.0.2"
    }
  }
}

provider "docker" {
  host = var.docker_host
}

# Создаем изолированную сеть
resource "docker_network" "isolated_network" {
  name = "isolated_network"
}

# Создаем том для базы данных
resource "docker_volume" "db_volume" {
  name = "db_data"
}

# Создаем контейнер front с двумя интерфейсами
resource "docker_container" "front" {
  image = "ubuntu:latest"
  name  = "front"
  privileged = true  # Добавляем привилегии для использования tc

  # Настройка внешнего интерфейса
  networks_advanced {
    name    = "bridge"  # Внешняя сеть
  }

  # Настройка изолированного интерфейса
  networks_advanced {
    name    = docker_network.isolated_network.name
  }

  command = ["tail", "-f", "/dev/null"]  # Удерживаем контейнер активным

  # Устанавливаем SSH-сервер и настраиваем его через local-exec
  provisioner "local-exec" {
    command = <<EOT
      docker exec ${self.name} apt-get update
      docker exec ${self.name} apt-get install -y openssh-server iproute2
      docker exec ${self.name} mkdir -p /root/.ssh
      docker exec ${self.name} sh -c 'echo "${file("keys/id_rsa_terraform.pub")}" > /root/.ssh/authorized_keys'
      docker exec ${self.name} chmod 600 /root/.ssh/authorized_keys
      docker exec ${self.name} chown root:root /root/.ssh/authorized_keys

      # Запрещаем доступ по паролю и настраиваем SSH
      docker exec ${self.name} sh -c 'echo "PasswordAuthentication no" >> /etc/ssh/sshd_config'
      docker exec ${self.name} sh -c 'echo "PermitRootLogin prohibit-password" >> /etc/ssh/sshd_config'
      docker exec ${self.name} service ssh restart
      sleep 10  # Даем SSH-серверу время на перезапуск
    EOT
  }

  # Настройка подключения для provisioners
  connection {
    type     = "ssh"
    host     = self.network_data[0].ip_address  # Используем IP-адрес контейнера
    user     = "root"
    private_key = file("keys/id_rsa_terraform")
    timeout  = "2m"  # Увеличиваем таймаут для подключения
  }

  # Ограничение ширины канала (используем утилиту tc)
  provisioner "remote-exec" {
    inline = [
      "tc qdisc add dev eth0 root tbf rate 110mbit burst 32kbit latency 400ms"
    ]
  }
}

# Создаем контейнер back с одним интерфейсом
resource "docker_container" "back" {
  image = "ubuntu:latest"
  name  = "back"

  # Настройка изолированного интерфейса
  networks_advanced {
    name    = docker_network.isolated_network.name
    aliases = ["back_internal"]
  }

  command = ["tail", "-f", "/dev/null"]  # Удерживаем контейнер активным

  # Устанавливаем SSH-сервер и настраиваем его через local-exec
  provisioner "local-exec" {
    command = <<EOT
      docker exec ${self.name} apt-get update
      docker exec ${self.name} apt-get install -y openssh-server
      docker exec ${self.name} mkdir -p /root/.ssh
      docker exec ${self.name} sh -c 'echo "${file("keys/id_rsa_terraform.pub")}" > /root/.ssh/authorized_keys'
      docker exec ${self.name} chmod 600 /root/.ssh/authorized_keys
      docker exec ${self.name} chown root:root /root/.ssh/authorized_keys

      # Запрещаем доступ по паролю и настраиваем SSH
      docker exec ${self.name} sh -c 'echo "PasswordAuthentication no" >> /etc/ssh/sshd_config'
      docker exec ${self.name} sh -c 'echo "PermitRootLogin prohibit-password" >> /etc/ssh/sshd_config'
      docker exec ${self.name} service ssh restart
      sleep 10  # Даем SSH-серверу время на перезапуск
    EOT
  }

  # Настройка подключения для provisioners
  connection {
    type     = "ssh"
    host     = self.network_data[0].ip_address  # Используем IP-адрес контейнера
    user     = "root"
    private_key = file("keys/id_rsa_terraform")
    timeout  = "2m"  # Увеличиваем таймаут для подключения
  }
}

# Создаем контейнер db с одним интерфейсом и дополнительным диском
resource "docker_container" "db" {
  image = "ubuntu:latest"
  name  = "db"

  # Настройка изолированного интерфейса
  networks_advanced {
    name    = docker_network.isolated_network.name
    aliases = ["db_internal"]
  }

  command = ["tail", "-f", "/dev/null"]  # Удерживаем контейнер активным

  # Устанавливаем SSH-сервер и настраиваем его через local-exec
  provisioner "local-exec" {
    command = <<EOT
      docker exec ${self.name} apt-get update
      docker exec ${self.name} apt-get install -y openssh-server
      docker exec ${self.name} mkdir -p /root/.ssh
      docker exec ${self.name} sh -c 'echo "${file("keys/id_rsa_terraform.pub")}" > /root/.ssh/authorized_keys'
      docker exec ${self.name} chmod 600 /root/.ssh/authorized_keys
      docker exec ${self.name} chown root:root /root/.ssh/authorized_keys

      # Запрещаем доступ по паролю и настраиваем SSH
      docker exec ${self.name} sh -c 'echo "PasswordAuthentication no" >> /etc/ssh/sshd_config'
      docker exec ${self.name} sh -c 'echo "PermitRootLogin prohibit-password" >> /etc/ssh/sshd_config'
      docker exec ${self.name} service ssh restart
      sleep 10  # Даем SSH-серверу время на перезапуск
    EOT
  }

  # Настройка подключения для provisioners
  connection {
    type     = "ssh"
    host     = self.network_data[0].ip_address  # Используем IP-адрес контейнера
    user     = "root"
    private_key = file("keys/id_rsa_terraform")
    timeout  = "2m"  # Увеличиваем таймаут для подключения
  }

  # Используем том для хранения данных
  mounts {
    source = docker_volume.db_volume.name
    target = "/var/lib/mysql"  # Путь, где будет храниться база данных
    type   = "volume"
  }
}
``` 

#### inventory.yml 
```bash  
[front]
172.17.0.2 ansible_ssh_private_key_file=/root/Infrastructure-Automation/terraform/keys/id_rsa_terraform
172.19.0.4 ansible_ssh_private_key_file=/root/Infrastructure-Automation/terraform/keys/id_rsa_terraform

[back]
172.19.0.3 ansible_ssh_private_key_file=/root/Infrastructure-Automation/terraform/keys/id_rsa_terraform

[db]
172.19.0.2 ansible_ssh_private_key_file=/root/Infrastructure-Automation/terraform/keys/id_rsa_terraform
```          
