# Решение проблемы с первоначальной развёртывании ELK-стэка

### Описание проблемы (Issue)
При развёртывании ELK-стэка (9.3.1) через docker copmose, контейнер Elasticsearch падал на этапе инициализации, когда другие сервисы спокойно запускались

<details>
  <summary>Конфиг до фикса docker-compose.yml   </summary>
    
```yaml
services:
  elasticsearch:
    image: elasticsearch:9.3.1
    volumes:
      - ./elasticsearch/config.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - ./docker_volumes/elasticsearch/data:/usr/share/elasticsearch/data
    environment:
      ES_JAVA_OPTS: "*-Xmx512m -Xms512m"
      ELASTIC_USERNAME: "***"
      ELASTIC_PASSWORD: "***"
      discovery.type: single-node
    networks:
      - elk
    ports:
      - "9200:9200"
      - "9300:9300"

  logstash:
    image: logstash:9.3.1
    volumes:
      - ./logstash/config.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/pipelines:/usr/share/logstash/config/pipelines:ro
      - ./logstash/pipelines.yml:/usr/share/logstash/config/pipelines.yml:ro
    environment:
      LS_JAVA_OPTS: "-Xmx512m -Xms512m"
    ports:
      - "5044:5044"
      - "5000:5000"
      - "9600:9600"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:9.3.1
    depends_on:
      - elasticsearch
    volumes:
      - ./kibana/config.yml:/usr/share/kibana/config/kibana.yml:ro
    networks:
      - elk
    ports:
      - "5601:5601"
  filebeat:
    image: elastic/filebeat:9.3.1
    volumes:
      - ./filebeat/config.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./host_metrics_app/:/host_metrics_app/:ro
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge
```
</details>

Проблемы: 
- После падения Elasticsearch не мог подняться автоматически. Использование `restart: always` или `unless-stopped` не помогало
- Прочитав логи данного приложения, никаких ошибок не было выявлено. Строки кода были без критических ошибок, а код выхода был 0
- При этом автоматические запуски и дописывание `depends_on` ситуацию не меняло
- Остальные компоненты ELK спокойно запускались без ключевого приложения

--- 

### Полное решение

#### 1. Диагностика
- При анализе логов, никаких ошибок не было выявлено. При старте я проверил нагрузку через `htop` (был высокий LA), но при этом нагрузка на CPU и RAM была не максимальна из-за чего я проверил диски через `iotop` и `iostat`, что позволило мне выяснить, что проблема была в недостатке пропускной способности дисков
- Понял, что Elasticsearch просто не успевает инициализироваться и падает, так как он самый требовательный из-за JVM, он превышал внутренние таймауты приложения. В итоге контейнер завершался с обычным кодом выхода 0, не успев зафиксировать ошибку в логах. `depends_on` тут не помог бы, так как он чекает только запуск контейнера, а не готовность

#### 2. Внедрение Healthcheck как гарантию запуска и дополнение к depends_on
- Чтобы приложения успевали запускаться из-за медленных I/O, я добавил блок с healthcheck. Он проверяет реальное состояние кластера через API, прежде чем дать команду стартовать другим сервисам. 

<details>
  <summary> Сам healthcheck блок </summary>

```yaml
healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --user elastic:${ELASTIC_PASSWORD} -X GET http://localhost:9200/_cluster/health?pretty | grep status | grep -q '\\(green\\|yellow\\)'"   ## Проверяет здоровье эластика и ищет в выводе зелёный либо жёлтый коды статуса
        ]
      interval: 10s
      timeout: 5s
      retries: 24
      start_period: 40s
```


</details>

- Для остальных приложений изменил логику зависимости. Они теперь жду успешную проверку, а не просто старт запуска
```yaml
depends_on:
  elasticsearch:
    condition: service_healthy
```

### 3. Результат
Благодаря внедрению хелзчека и выходящему из этого последовательного запуска, стэк стал полностью автономным. Система спокойно поднимается после перезагрузки виртуалки на HDD, исключая необходить постоянного ввода команды `docker compose up`

<details>
  <summary> Сам docker-compose.yaml </summary>

```yaml
  services:
  elasticsearch:
    image: elasticsearch:9.3.1
    volumes:
      - ./elasticsearch/config.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - ./docker_volumes/elasticsearch/data:/usr/share/elasticsearch/data
    environment:
      ES_JAVA_OPTS: "-Xmx512m -Xms512m"
      ELASTIC_USERNAME: "elastic"
      ELASTIC_PASSWORD: "Passw0rd"
      discovery.type: single-node
    networks:
      - elk
    ports:
      - "9200:9200"
      - "9300:9300"
    restart: always
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "curl -s --user elastic:${ELASTIC_PASSWORD} -X GET http://localhost:9200/_cluster/health?pretty | grep status | grep -q '\\(green\\|yellow\\)'"
        ]
      interval: 10s
      timeout: 5s
      retries: 24
      start_period: 40s

  logstash:
    image: logstash:9.3.1
    volumes:
      - ./logstash/config.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logstash/pipelines:/usr/share/logstash/config/pipelines:ro
      - ./logstash/pipelines.yml:/usr/share/logstash/config/pipelines.yml:ro
    environment:
      LS_JAVA_OPTS: "-Xmx512m -Xms512m"
    restart: unless-stopped
    ports:
      - "5044:5044"
      - "5000:5000"
      - "9600:9600"
    networks:
      - elk
    depends_on:
      elasticsearch:
        condition: service_healthy

  kibana:
    image: kibana:9.3.1
    depends_on:
      - elasticsearch
    volumes:
      - ./kibana/config.yml:/usr/share/kibana/config/kibana.yml:ro
    networks:
      - elk
    restart: unless-stopped
    ports:
      - "5601:5601"

  filebeat:
    image: elastic/filebeat:9.3.1
    volumes:
      - ./filebeat/config.yml:/usr/share/filebeat/filebeat.yml:ro
      - ./host_metrics_app/:/host_metrics_app/:ro
    networks:
      - elk
    depends_on:
      - elasticsearch
    #restart: unless-stopped

networks:
  elk:
    driver: bridge
```

</details>
