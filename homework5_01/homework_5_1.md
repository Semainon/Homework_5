### Д3 4.1. Написать Dockerfile, который собирает:
- **nginx как http-интерфейс для Wordpress**
- **PostgreSQL**
- **Wordpress**

### Предварительная структура файлов проекта
```bash
# Несколько Dockerfile дают возможность изолировать сервисы (для лучшей управляемости и масштабирумости)
wwordpress_nginx_postgres_setup/
├── .env
├── docker-compose.yml
├── nginx/
│   ├── Dockerfile
│   ├── html/
│   └── nginx.conf
├── php-fpm/
│   ├── Dockerfile
│   ├── php-fpm.conf
│   └── www.conf
├── python_server/
│   ├── app.py
│   └── Dockerfile
├── wordpress/
│   ├── Dockerfile
│   ├── install-wordpress.sh
│   └── wp-config.php
└── grafana/
```
### Установка Docker и Docker Compose	
```bash
sudo yum update -y
sudo yum install -y yum-utils

# Установка Docker    
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo 
sudo yum install -y docker-ce docker-ce-cli containerd.io 
sudo systemctl start docker  # запускаем Docker 
sudo systemctl enable docker # добавляем Docker в автозагругку
# Опционально: добавляем пользователя в группу Docker, чтобы запускать команды Docker без sudo
# sudo systemctl stop sssd
# sudo rm -rf /var/lib/sss/db/*
# sudo systemctl start sssd
sudo usermod -aG docker $USER  
# Проверяем установку
sudo docker --version
sudo docker run hello-world
sudo systemctl status docker

# Установка Docker Compose   
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose yml
sudo chmod +x /usr/local/bin/docker-compose 
docker-compose --version 
```
```bash
[root@Zero ~]# cd /root/wordpress_nginx_postgres_setup && mkdir nginx wordpress python_server grafana
[root@Zero wordpress_nginx_postgres_setup]# cd nginx
[root@Zero nginx]# nano nginx.conf
[root@Zero nginx]# nano Dockerfile
[root@Zero nginx]# cd ..
[root@Zero wordpress_nginx_postgres_setup]# cd wordpress && nano Dockerfile
```
###  Dockerfile для nginx
```bash
# Используем официальный образ NGINX
FROM nginx:latest

# Копируем файл конфигурации nginx.conf в контейнер
COPY ./nginx/nginx.conf /etc/nginx/nginx.conf

# Копируем файлы  веб-приложения из папки wordpress
COPY ./wordpress /usr/share/nginx/html

# Открываем порт 80 для HTTP
EXPOSE 80

# Запускаем NGINX с указанием конфигурационного файла
CMD ["nginx", "-g", "daemon off;"]

[root@Zero nginx]# cd ..
[root@Zero wordpress_nginx_postgres_setup]# cd wordpress && nano Dockerfile
```
### Dockerfile для wordpress
```bash 
# Используем официальный образ WordPress
FROM wordpress:latest

# Установка необходимых расширений PHP для PostgreSQL и MySQL 
RUN docker-php-ext-install pdo pdo_pgsql mysqli

# Установка прав доступа
RUN chown -R www-data:www-data /var/www/html

# Открытие порта 80 для HTTP
EXPOSE 80
```