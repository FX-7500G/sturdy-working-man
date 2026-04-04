# DevOps Troubleshooting
Hi, Everyone! A collection of real infrastructure problems I've debugged and solved.

---  
## Cases 

<details>
<summary>🐳 Docker </summary>

| Problem | Solution | Link |
| :--- | :--- | :--- |
| Ошибка 'no such host' (IPv6) | Отключение IPv6 и настройка DNS | [Решение](./Docker-troubleshooting/combating-IPv6-priotization.md) |
| Миграция конфигов Nginx и SSL | Извлечение конфигов из работающего конта через bash -c, настройка HTTPS редиректа | ./  |

</details>

<details>
<summary>⎈ Kubernetes</summary>
  
| Problem | Solution | Link |
| :--- | :--- | :--- |
| Динамически не настроились PVC и таймауты при деплое GitLab | Редактирование StorageClass, ручное создание бакетов и фикс MTU | [Решение](./K8s-troubleshooting/gitlab-k8s-pvc-network-fix.md) |  
| Недоступность `apt`-репозиториев K8s | Установка бинарников напрямую, настройка containerd, ручное создание systemd-unit | [Решение](./K8s-troubleshooting/K8s-binary-installing.md) |
</details>

<details open>
  <summary> CI/CD </summary>
    
| Problem | Solution | Link |
| :--- | :--- | :--- |
| Legacy Pipeline Refactoring | Устранение ошибок SSL , миграция переменных, Внедрение цикла ожидания готовности Docker-демона | [Решение](./CI-CD-troubleshooting/legacy-k8s-deploy.md) |
  
</details>

<details>
  <summary> Linux </summary>

| Problem | Solution | Link |
| :--- | :--- | :--- |


</details>
<details>
  <summary> IaC </summary>

| Problem | Solution | Link |
| :--- | :--- | :--- |
| Автоматизация развёртывания Docker (Ansible) | Написание плейбука для DNF/YUM, управление правами пользователя | [Решение](./IaC-troubleshooting/Ansible-docker-automation.md) |
  
</details>

<details>
  <summary> Network </summary>
  
| Problem | Solution | Link |
| :--- | :--- | :--- |
| Отсутствие удаленного доступа к виртуальной машине через SSH | Смена сетевого интерфейса с NAT на Bridged для получения общего IP, конфигурация статического IP через netplan, настройка UFW | [Решение](./Network-troubleshooting/vm-ssh-access.md) |

</details>

<details>
  <summary>📊 Monitoring and parsing logs  </summary>

| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Падение Es при старте из-за Race Condition | Внедрение Healthcheck и Sequential Startup | [Решение](./ELK-troubleshooting/fix-race-condition-elasticsearch.md) |
    
</details>
