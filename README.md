Описание:

На виртуальной машине установите любую open source CMS, которая включает в себя следующие компоненты: nginx, php-fpm, database (MySQL or Postgresql)
На этой же виртуальной машине установите Prometheus exporters для сбора метрик со всех компонентов системы (начиная с VM и заканчивая DB, не забудьте про blackbox exporter, который будет проверять доступность вашей CMS)
На этой же или дополнительной виртуальной машине установите Prometheus, задачей которого будет раз в 5 секунд собирать метрики с экспортеров.
  ==================================================================

Созданы 2 виртуальные машины с адресами 192.168.192.141 и 192.168.192.248
На одной из установлена October CMS (https://octobercms.com/)
Проверяем и обновляем пакеты системы
  sudo apt update -y
  sudo apt upgrade -y

1. Устанавливаем Node Exporter

Скачиваем дистрибутив (https://github.com/prometheus/node_exporter/releases)
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz
        
Распаковываем
tar -xzvf node_exporter-1.8.2.linux-amd64.tar.gz

node_exporter-1.8.2.linux-amd64
├── LICENSE
├── node_exporter
└── NOTICE

Создаем пользователя prometheus
sudo useradd -rs /bin/false prometheus

Копируем бинарный файл экспортера в /usr/local/bin
cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

Наделяем правами пользователя prometheus
sudo chown prometheus:prometheus /usr/local/bin/node_exporter

Создаем сервис
sudo nano /etc/systemd/system/node_exporter.service

[Unit]
Description=Node Exporter
After=network.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target

Применяем
sudo systemctl daemon-reload

Запускаем сервис и устанавливаем автозапуск, проверяем статус
sudo systemctl start node_exporter.service
sudo systemctl enable node_exporter.service
sudo systemctl status node_exporter.service

● node_exporter.service - Node Exporter
     Loaded: loaded (/etc/systemd/system/node_exporter.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-01-02 11:34:39 UTC; 1min 39s ago
   Main PID: 2562 (node_exporter)
      Tasks: 3 (limit: 2274)
     Memory: 7.1M (peak: 7.4M)
        CPU: 205ms
     CGroup: /system.slice/node_exporter.service
             └─2562 /usr/local/bin/node_exporter

2. Устанавливаем Prometheus на 192.168.192.141
Скачиваем дистрибутив (https://github.com/prometheus/prometheus/releases)
wget https://github.com/prometheus/prometheus/releases/download/v3.0.1/prometheus-3.0.1.linux-amd64.tar.gz
        
Распаковываем
tar -xzvf prometheus-3.0.1.linux-amd64.tar.gz

prometheus-3.0.1.linux-amd64
├── LICENSE
├── NOTICE
├── prometheus
├── prometheus.yml
└── promtool

Копируем бинарные файлы в /usr/local/bin
cp prometheus-3.0.1.linux-amd64/prometheus /usr/local/bin/
cp prometheus-3.0.1.linux-amd64/prometool /usr/local/bin/

Наделяем правами пользователя prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometool

Создаем папки
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
Выдаем права
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
Копируем файл конфигурации
sudo cp prometheus-3.0.1.linux-amd64/prometheus.yml /etc/prometheus/

Прописываем в файле prometheus.yml
prometheus и node_exporter

# my global config
global:
  scrape_interval: 10s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  # evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus_master"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    scrape_interval: 5s
    static_configs:
     - targets: ["localhost:9090"]

  - job_name: 'node_exporter_master'
    scrape_interval: 5s
    static_configs:
     - targets: ['localhost:9100']

  - job_name: 'node_exporter_cms'
    scrape_interval: 5s
    static_configs:
     - targets: ["192.168.192.248:9100"]


Создаем сервис
sudo nano /etc/systemd/system/prometheus.service

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/
[Install]
WantedBy=multi-user.target

Применяем
sudo systemctl daemon-reload

Запускаем сервис и устанавливаем автозапуск, проверяем статус
sudo systemctl start prometheus.service
sudo systemctl enable prometheus.service
sudo systemctl status prometheus.service

● prometheus.service - Prometheus
     Loaded: loaded (/etc/systemd/system/prometheus.service; enabled; preset: enabled)
     Active: active (running) since Thu 2025-01-02 09:48:36 UTC; 2h 41min ago
   Main PID: 1171 (prometheus)
      Tasks: 7 (limit: 2274)
     Memory: 97.7M (peak: 102.6M)
        CPU: 36.132s
     CGroup: /system.slice/prometheus.service
             └─1171 /usr/local/bin/prometheus --config.file /etc/prometheus/prometheus.yml --storage.tsdb.path /var/lib/prometheus/

Проверяем http://192.168.192.141:9090/targets
![image](https://github.com/user-attachments/assets/2ef3231d-ac11-4b60-83f8-9d997fc9c465)

