# DevOps Troubleshooting
Всем хай! Я Junior DevOps-инженер. Здесь собраны разборы реальных технических кейсов, с которыми я столкнулся при развертывании и эксплуатации как чужой, так и своей инфраструктуры

Технологический стек:    
Оркестрация/Контейниризация: Kubernetes, Docker, Docker Compose    
Мониторинг/Логи: ELK Stack (Elasticsearch, Logstash, Kibana), Grafana + Prometheus           
Система: Linux (Ubuntu/Debian), Windows    
CI/CD: GitLab CI, ArgoCD  
IaC: Ansible, Vagrant   
Виртуализация: Proxmox VE   

---
###  Контактная информация:
[![Telegram](https://img.shields.io/badge/Telegram-2CA5E0?style=for-the-badge&logo=telegram&logoColor=white)](https://t.me/FX_7500G)  
Telegram: @FX_7500G    
Email: alekcwdsa@gmail.com    

## Кейсы 

<details>
<summary>🐳 Docker </summary>

| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Ошибка 'no such host' (IPv6) | Отключение IPv6 и настройка DNS | [Решение](./Docker-troubleshooting/combating-IPv6-priotization.md) |
| Миграция конфигов Nginx и SSL | Извлечение конфигов из работающего конта через bash -c, настройка HTTPS редиректа | ./  |

</details>

<details>
<summary>⎈ Kubernetes</summary>
  
| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Динамически не настроились PVC и таймауты при деплое GitLab | Редактирование StorageClass, ручное создание бакетов и фикс MTU | [Решение](./K8s-troubleshooting/gitlab-k8s-pvc-network-fix.md) |  
| Недоступность `apt`-репозиториев K8s | Установка бинарников напрямую, настройка containerd, ручное создание systemd-unit | [Решение](./K8s-troubleshooting/K8s-binary-installing.md) |
</details>

<details open>
  <summary> CI/CD </summary>
    
 | Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Legacy Pipeline Refactoring | Устранение ошибок SSL , миграция переменных, Внедрение цикла ожидания готовности Docker-демона | [Решение](./CI-CD-troubleshooting/legacy-k8s-deploy.md) |
  
</details>

<details>
  <summary> Linux </summary>

| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |


</details>
<details>
  <summary> IaC </summary>

| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Автоматизация развёртывания Docker (Ansible) | Написание плейбука для DNF/YUM, управление правами пользователя | [Решение](./IaC-troubleshooting/Ansible-docker-automation.md) |
  
</details>

<details>
  <summary> Сети </summary>
  
| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |

</details>

<details>
  <summary>📊 Мониторинг и парсинг логов  </summary>

| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Падение Es при старте из-за Race Condition | Внедрение Healthcheck и Sequential Startup | [Решение](./ELK-troubleshooting/fix-race-condition-elasticsearch.md) |
    
</details>
