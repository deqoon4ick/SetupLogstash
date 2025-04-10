

```
#!/bin/bash

OPENSEARCH_HOST="http://188.72.88.96:9200"
OPENSEARCH_USER="admin"
OPENSEARCH_PASS="KellyOne667"
INDEX_NAME="centos-nginx-logs-%{+YYYY.MM.dd}"
SERVER_NAME="centos-nginx"
NGINX_LOG="/var/log/nginx/access.log"



if ls /tmp/java-11-openjdk*.rpm 1> /dev/null 2>&1; then
    sudo yum localinstall -y /tmp/java-11-openjdk*.rpm
fi
if ls /tmp/wget*.rpm 1> /dev/null 2>&1; then
    sudo yum localinstall -y /tmp/wget*.rpm
fi
if ls /tmp/curl*.rpm 1> /dev/null 2>&1; then
    sudo yum localinstall -y /tmp/curl*.rpm
fi

# === 2. Устанавливаем Logstash из локального RPM из /tmp ===
echo "Устанавливаем Logstash из RPM..."
if ls /tmp/logstash-*.rpm 1> /dev/null 2>&1; then
    sudo yum localinstall -y /tmp/logstash-*.rpm
else
    echo "Ошибка: Не найден RPM Logstash в /tmp"
    exit 1
fi

# Увеличиваем память для установки плагинов (например, до 2 ГБ) ===
export LS_JAVA_OPTS="-Xmx2g"


echo "Устанавливаем плагин logstash-output-opensearch..."
if ls /tmp/logstash-output-opensearch*.gem 1> /dev/null 2>&1; then
    sudo /usr/share/logstash/bin/logstash-plugin install --local /tmp/logstash-output-opensearch*.gem
else
    echo "Предупреждение: Gem-файл плагина logstash-output-opensearch не найден в /tmp. Пропускаем установку плагина."
fi


if [ -f "$NGINX_LOG" ]; then
    OWNER_GROUP=$(stat -c "%G" "$NGINX_LOG")
    echo "Файл $NGINX_LOG принадлежит группе $OWNER_GROUP. Добавляем пользователя logstash в эту группу..."
    sudo usermod -aG "$OWNER_GROUP" logstash
    # Обеспечиваем, чтобы файл был доступен для чтения
    sudo chmod 644 "$NGINX_LOG"
else
    echo "Ошибка: Файл $NGINX_LOG не найден. Убедитесь, что NGINX установлен и лог файл ведется."
    exit 1
fi

echo "Создаем конфигурационный файл /etc/logstash/conf.d/nginx_logs.conf..."
sudo mkdir -p /etc/logstash/conf.d
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


echo "Настраиваем systemd и запускаем Logstash..."
sudo systemctl daemon-reload
sudo systemctl enable logstash
sudo systemctl restart logstash

echo "✅ Logstash установлен и запущен на CentOS! Проверь статус:"
sudo systemctl status logstash
echo "Проверь логи через: sudo journalctl -u logstash -f"
echo "NGINX логи будут передаваться в OpenSearch в индекс: $INDEX_NAME"


```
