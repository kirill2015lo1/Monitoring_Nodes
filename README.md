# Настройка мониторинга серверов с proxmox: логи, метрики

##### Стэк: `prometheus, victoriametrics, grafana, loki, cadvisor, alertmanager, node-exporter, pve-exporter, promtail.`

Метрики будут собирать по pull модели, а логи будут оптравляться по push модели.  
Так как prometheus не поддерживает service discrovery, то нужно вручную указывать ip адреса, откуда ему забирать метрики. 
Promtail тоже не поддерживает SD, поэтому вручную нужно будет указывать адрес loki, куда он будет отправлять свои логи 
Плюсом, для удобного агрегирование логами и метриками, им нужно будет дать понятный лейбл

Простым языком все что нужно сделать, это на сервера с proxmox установить node-exporter, и установить promtail (в его конфигурации указать ip адрес Loki, куда он будет слать логи)
И уже далее в prometheus указать ip наших серверов с proxmox, куда он будет ходить забирать метрики.


Настройка будет в 3 этапа:

### 1. Преднастройка

Клонирование репозитория

### 2. Сбор метрик от node exporter (модель pull). Инструменты:

`node_exporter > prometheus > victoricametrics > grafana`

### 3. Сбор уже готовых метрик proxmox через pve-exporter (модель pull). Инструменты:

`proxmox node > pve-exporter с proxmox api token > prometheus > victoriametrics > grafana `

### 4. Сбор и отправка логов через Promtail в loki (модель push). Иструменты:

`promtail > loki > grafana`



# Первый этап

Копируем готовый репозиторий, с уже наст
