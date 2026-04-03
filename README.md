# DevOps Troubeshooting
Всем хай! Я Junior DevOps-инженер 
Здесь я документирую сложные техпроблемы, с которыми я сталкивался на своём пути, чтобы в будущем было легче с ними работать

Здесь я опишу свои проблемы с которыми сталкивался в работе и дома. Вроде: 
- K8s
- Docker
- CI/CD
- Linux
- Сетевухи

## Кейсы 

<details>
<summary>🐳 Docker </summary>

| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Ошибка 'no such host' (IPv6) | Отключение IPv6 и настройка DNS | [Решение](./Docker-troubleshooting/combating-IPv6-priotization.md) |

</details>

<details>
<summary>⎈ Kubernetes</summary>
  
| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Динамически не настроились PVC и таймауты при деплое GitLab | Редактирование StorageClass, ручное создание бакетов и фикс MTU | [Решение](./K8s-troubleshooting/gitlab-k8s-pvc-network-fix.md) |
| Установка K8s через бинарники | Кратко не опишешь | [Решение](./K8s-troubleshooting/K8s-binary-installing.md)
</details>

<details>
  <summary> CI/CD </summary>

  
  
</details>

<details>
  <summary> Linux </summary>

    
</details>

<details>
  <summary> Сети </summary>

    
</details>

<details>
  <summary>📊 Мониторинг и парсинг логов  </summary>

| Проблема | Решение | Ссылка |
| :--- | :--- | :--- |
| Падение Es при старте из-за Race Condition | Внедрение Healthcheck и Sequential Startup | [Решение](./ELK-troubleshooting/fix-race-condition-elasticsearch.md) |
    
 
</details>
