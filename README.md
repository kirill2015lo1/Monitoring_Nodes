# Настройка мониторинга серверов с proxmox: логи, метрики

##### Стэк: `prometheus, victoriametrics, grafana, loki, cadvisor, alertmanager, node-exporter, pve-exporter, promtail.`

Метрики будут собирать по pull модели, а логи по push модели.  
Так как prometheus не поддерживает service discrovery, то нужно вручную указывать ip адреса, откуда ему забирать метрики. 
Promtail тоже не поддерживает SD, поэтому вручную нужно будет указывать адрес loki, куда он будет отправлять свои логи 
Плюсом, для удобного агрегирование логами и метриками, им нужно будет дать понятный лейбл

Простым языком все что нужно сделать, это на сервера с proxmox установить node-exporter, и установить promtail (в его конфигурации указать ip адрес Loki, куда он будет слать логи)
И уже далее в prometheus указать ip наших серверов с proxmox, куда он будет ходить забирать метрики. Все остальное уже настроено

Сразу дадим условность, что ip адреса наших трех нод с proxmox это 192.168.50.200, 192.168.50.201, 192.168.50.202  
А мониторить мы их хотим с виртуальной машины с ip 192.168.50.205

Настройка будет в 5 этапа:

### 1. Преднастройка
Установк докера 
Клонирование репозитория  
Настройка prometheus.yaml

### 2. Сбор метрик от node exporter (модель pull). Инструменты:

`node_exporter > prometheus > victoricametrics > grafana`

### 3. Сбор уже готовых метрик proxmox через pve-exporter (модель pull). Инструменты:

`proxmox node > pve-exporter с proxmox api token > prometheus > victoriametrics > grafana `

### 4. Сбор и отправка логов через Promtail в loki (модель push). Иструменты:

`promtail > loki > grafana`
### 5. Постнастройка

Настройка alertmanager, запуск docker-compose + полезные ссылки

# Первый этап

Установка docker: 

`https://docs.docker.com/engine/install/`

Копируем готовый репозиторий.
```
apt install -y git
git clone https://github.com/kirill2015lo1/Monitoring_Nodes.git
cd Monitoring_Nodes/
```
Далее открываем /prometheus/prometheus.yaml и указываем ip адреса наших нод, и указываем label server к каждому ip

Пример настройки 
```
#node_exporter
scrape_configs:
  - job_name: 'node_exporter' 
    static_configs:
      - targets:
          - '192.168.50.200:9100' # указать Ip откуда берется метрика
        labels:
          server: 'srv-hv1' # указать в ковычках название сервера, для удобного отображения в grafana

      - targets:
          - '192.168.50.201:9100'
        labels:
          server: 'srv-hv2'

      - targets:
          - '192.168.50.202:9100'
        labels:
          server: 'srv-hv3'

#promtail
  - job_name: 'promtail' 
    static_configs:
      - targets:
          - '192.168.50.200:9080'
        labels:
          server: 'srv-hv1'

      - targets:
          - '192.168.50.201:9080'
        labels:
          server: 'srv-hv2'

      - targets:
          - '192.168.50.202:9080'
        labels:
          server: 'srv-hv3'

#proxmox
  - job_name: 'proxmox'  
    static_configs:
      - targets:
          - '192.168.50.200'
        labels:
          server: 'srv-hv1'

      - targets:
          - '192.168.50.201'
        labels:
          server: 'srv-hv2'

      - targets:
          - '192.168.50.202'
        labels:
          server: 'srv-hv3'

```
Порты не стиратать, не менять. Лейблы указать одинаковые
Все остальное не трогать

# Этап второй

Установка node_exporter на ноды с Proxmox

Через ansible, или вручную подключиться к каждой ноде и выполнить скрипт (ctrl+c ctrl+v):
```
https://github.com/prometheus/node_exporter/releases

wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

tar -xvzf node_exporter-1.8.2.linux-amd64.tar.gz

rm -R node_exporter-1.8.2.linux-amd64.tar.gz

mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

useradd --no-create-home --shell /bin/false node_exporter

cat > /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=node_exporter
Group=node_exporter
ExecStart=/usr/local/bin/node_exporter
Restart=always
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start node_exporter
systemctl enable node_exporter
systemctl status node_exporter
```
Версию node_exporter можно поменять на более новую, в скрипте используется последняя на данный момент 1.8.2

После выполнения скрипты будут доступны метрики по порту 9100 
Для проверки можно выполнить:  

`curl localhost:9100/metrics`

# Этап третий 

Создание ключа для pve-exporter 

Proxmox уже отдает метрики, но чтобы их получить, нужно создать учетную запись,и api token для pve-exporter. Как визаульно это делается можно посмотреть в этом видео:  
`https://www.youtube.com/watch?v=PtsdThgnZqs&t=307s`

В proxmox переходим в `Datacenter`

Далее переходим в `Permissons => users`

Нажимаем `Add`, username указываем `pve-exporter`, realm указываем `proxmox ve auth`

Далее переходим в `Permissons` и нажимаем `Add => User permisson`

Выбираем `path = /`, `user = pve-exporter`, `role = PVEAuditor`

Далее переходим `Permissons => API Tokens`
 
Нажимаем `Add`, выбираем `User = pve-exporter@pve`, и `Token iD = pve-exporter`, убираем 'Privilege Separation' галочку 

Потом нам покажут Token ID и Secret, который нам надо указать в pve-exporter/pve.yml  
Например если нам выдало:  
`pve-exporter@pve!pve-exporter`
`7252731b-78c6-44e2-9de2-16853e1b578c`  

То pve.yml будет
```
default:
  user: "pve-exporter@pve" #левая часть до знака восклицания
  token_name: "pve-exporter" #правая часть после знака восклицания
  token_value: "7252731b-78c6-44e2-9de2-16853e1b578c"
  verify_ssl: false
```


# Этап четвертый

Для сборки логов, нужно зайти на каждый сервер с proxmox и установить одним скриптом promtail, который собирает логи dpkg и journal 

Далее идет скрипт, перед началом его выполнения, нужно скорректировать часть, с конфигурацией promtail. Так сам loki не знает откуда ему присылают логи, 
нам нужно указать promtail куда слать Логи, и указать лайблы, чтобы можно было понимать с какие серверов высланы логи.

Вот полный целый скрипт:
```
apt install sudo -y
apt install unzip -y
wget https://github.com/grafana/loki/releases/download/v3.4.2/promtail-linux-amd64.zip

unzip promtail-linux-amd64.zip

rm  promtail-linux-amd64.zip

sudo useradd --no-create-home --shell /bin/false promtail

chown promtail:promtail promtail-linux-amd64

mv promtail-linux-amd64 /usr/local/bin/promtail

sudo chmod +x /usr/local/bin/promtail

mkdir /etc/promtail

chown promtail:promtail /etc/promtail
sudo chmod 755 /etc/promtail

cat >  /etc/promtail/config.yaml <<EOF
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://192.168.50.145:3100/loki/api/v1/push

scrape_configs:
  - job_name: dpkg_logs
    static_configs:  
      - targets:
          - localhost
        labels:
          service_name: dpkg
          server: srv-hv
          __path__: /var/log/dpkg.log

  - job_name: journal_logs
    journal:
      max_age: 24h
      labels:
        service_name: systemd-journal
        server: srv-hv
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'

EOF

cat > /etc/systemd/system/promtail.service<<EOF             
[Unit]
Description=Promtail service
After=network.target
[Service]
Type=simple
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yaml
Restart=always
[Install]
WantedBy=multi-user.target
EOF
chown promtail:promtail /etc/promtail/config.yaml
sudo chmod 755 /etc/systemd/system/promtail.service
sudo systemctl daemon-reload
sudo systemctl start promtail
sudo systemctl enable promtail
sudo systemctl status promtail

```
В скрипте используется promtail v3.4.2, можно заменить на более новую

Нас интересует часть:  
```
cat >  /etc/promtail/config.yaml <<EOF
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://192.168.50.205:3100/loki/api/v1/push

scrape_configs:
  - job_name: dpkg_logs
    static_configs:  
      - targets:
          - localhost
        labels:
          service_name: dpkg
          server: srv-hv
          __path__: /var/log/dpkg.log

  - job_name: journal_logs
    journal:
      max_age: 24h
      labels:
        service_name: systemd-journal
        server: srv-hv
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'unit'

EOF
```
Нужно заменить оба лейбла server: srv-hv, на лейбл, который мы указывали для этого сервера в prometheus.yaml   
Например в prometheus.yaml мы указали для 192.168.50.200 значения лейбла server srv-hv1, то при выполнении скрипта нужно заменить оба лейбла server: srv-hv на server: srv-hv1  
Если в prometheus.yaml мы указали для 192.168.50.201 значения лейбла server srv-hv2, то при выполнении скрипта нужно заменить оба лейбла server: srv-hv на server: srv-hv2  

Далее нужно изменить куда мы хотим отправлять логи:

Нужно указать ip нашей виртуальной машины где у нас будет мониторинг, будет так:

clients:
  - url: http://192.168.50.205:3100/loki/api/v1/push

Теперь все, все другие настройки будут храниться в /etc/promtail/config.yaml

# Пятый этап

Создание telegram бота, получения bot_token и последующую настройку alertmanage/alertmanager.yaml можно сделать по видео:
`https://www.youtube.com/watch?v=nz5xMoY1d6c&t=1s`

Теперь делаем запуск:

`docker compose up -d`

