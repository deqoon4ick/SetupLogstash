# SetupLogstash
Setting up Logstash on RedHat7
# 🚀 Logstash NGINX Auto Installer for CentOS/RedHat 7

Скрипт для быстрой установки и настройки Logstash для сбора логов NGINX и отправки их в OpenSearch.

## 📦 Что делает скрипт

- Устанавливает Java и Logstash
- Подключает репозиторий Elastic
- Устанавливает плагин `logstash-output-opensearch`
- Настраивает права доступа к логам NGINX
- Создаёт конфиг-файл Logstash
- Запускает Logstash как systemd-сервис

## ⚙️ Параметры

Изменяемые параметры находятся в начале скрипта:

```#!/bin/bash
# Скрипт установки Logstash для сбора NGINX логов на RedHat 7/CentOS 7

# --- Параметры конфигурации ---
OPENSEARCH_HOST="http://188.72.88.96:9200"
OPENSEARCH_USER="admin"
OPENSEARCH_PASS="KellyOne667"
INDEX_NAME="almalinux-nginx-logs-%{+YYYY.MM.dd}"
SERVER_NAME="almalinux-nginx"
NGINX_LOG="/var/log/nginx/access.log"

# --- 1. Обновляем систему и устанавливаем зависимости ---
echo "Обновляем систему и устанавливаем зависимости..."
sudo yum update -y
sudo yum install -y java-11-openjdk wget curl

# --- 2. Добавляем GPG-ключ и репозиторий Elastic для Logstash ---
echo "Добавляем репозиторий Logstash..."
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
cat <<EOF | sudo tee /etc/yum.repos.d/logstash.repo
[logstash-7.x]
name=Elastic repository for Logstash 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
enabled=1
autorefresh=1
type=rpm-md
EOF

# --- 3. Устанавливаем Logstash ---
echo "Устанавливаем Logstash..."
sudo yum install -y logstash

# --- 4. Увеличиваем память для установки плагина (чтобы избежать OutOfMemoryError) ---
export LS_JAVA_OPTS="-Xmx2g"

# --- 5. Устанавливаем плагин logstash-output-opensearch ---
echo "Устанавливаем плагин logstash-output-opensearch..."
sudo /usr/share/logstash/bin/logstash-plugin install logstash-output-opensearch

# --- 6. Настройка прав доступа к файлу NGINX логов ---
if [ -f "$NGINX_LOG" ]; then
    OWNER_GROUP=$(stat -c "%G" "$NGINX_LOG")
    echo "Файл $NGINX_LOG принадлежит группе $OWNER_GROUP. Добавляем пользователя logstash в эту группу..."
    sudo usermod -aG "$OWNER_GROUP" logstash
    # Для тестовой установки можно выставить права 644 (чтобы Logstash точно мог читать)
    sudo chmod 644 "$NGINX_LOG"
else
    echo "Ошибка: файл $NGINX_LOG не существует. Проверьте, что NGINX установлен и ведет логи."
    exit 1
fi

# --- 7. Создаем конфигурационный файл Logstash для NGINX ---
echo "Создаем конфигурацию Logstash в /etc/logstash/conf.d/nginx_logs.conf..."
sudo tee /etc/logstash/conf.d/nginx_logs.conf > /dev/null <<EOF
input {
  file {
    path => "$NGINX_LOG"
    start_position => "beginning"
    sincedb_path => "/dev/null"
    type => "nginx_access"
  }
}

filter {
  grok {
    match => {
      "message" => '%{IPORHOST:remote_addr} - %{DATA:remote_user} \[%{HTTPDATE:time_local}\] "%{WORD:method} %{URIPATH:request_path}(?:\?%{URIPARAM:request_params})? HTTP/%{NUMBER:http_version}" %{NUMBER:status} %{NUMBER:body_bytes_sent} "(?:%{URI:http_referer}|-)" "%{DATA:http_user_agent}"'
    }
    remove_field => ["message"]
  }

  date {
    match => [ "time_local", "dd/MMM/yyyy:HH:mm:ss Z" ]
    target => "@timestamp"
  }

  mutate {
    convert => {
      "status" => "integer"
      "body_bytes_sent" => "integer"
    }
    add_field => {
      "server_name" => "$SERVER_NAME"
    }
  }

  useragent {
    source => "http_user_agent"
    target => "user_agent"
  }

  if [status] >= 500 {
    mutate { add_tag => [ "server_error" ] }
  } else if [status] >= 400 {
    mutate { add_tag => [ "client_error" ] }
  }
}

output {
  opensearch {
    hosts => ["$OPENSEARCH_HOST"]
    user => "$OPENSEARCH_USER"
    password => "$OPENSEARCH_PASS"
    index => "$INDEX_NAME"
    ssl => false
    manage_template => false
    ecs_compatibility => "disabled"
  }

  stdout { codec => rubydebug }
}
EOF

# --- 8. Настраиваем автозапуск и запускаем Logstash как systemd-сервис ---
echo "Перезагружаем systemd и запускаем Logstash..."
sudo systemctl daemon-reload
sudo systemctl enable logstash
sudo systemctl restart logstash

echo "✅ Logstash установлен и запущен!"
echo "Проверь статус командой: sudo systemctl status logstash"
echo "Проверь логи через: sudo journalctl -u logstash -f"
echo "Логи NGINX будут отправляться в OpenSearch в индекс: $INDEX_NAME"
```


## 📦 Как запустить скрипт

1. Сохраните скрипт в файл, например:

```bash
nano install_logstash_nginx.sh
```

2. Вставьте содержимое, затем сохраните и выйдите:  

3. Сделайте скрипт исполняемым:

```bash
chmod +x install_logstash_nginx.sh
```

4. Запустите скрипт с правами суперпользователя:

```bash
sudo ./install_logstash_nginx.sh
```


Запустите скрипт с правами суперпользователя:

```bash
sudo ./install_logstash_nginx.sh
```
```

