# ArgoCD не может подключиться к GitLab (Repo failed)

### Стенд
- 2 Виртуальные машины в одной сети
- VM1 - Gitlab CE in Docker
- VM2 k8s (kubeadm, single-node, Calico CNI)
- ArgoCD v3.3.4

### Описание проблемы (Issue)
- Репозиторий показывается Failed при попытке законнектить ArgoCD к проекту. В логах `dial tcp ... connect: connection timed out`

---

## Полный путь фикса 

### 1. Что делал ДО появления ошибки.
<details> <summary>  Начальная установка ArgoCD </summary>

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
- Чекаем, чтоб поды поднялись:
```bash
kubectl get pods -n argocd
```
  - Если будет ErrImagePull, то проверяем пинг:
  ```bash
  curl -i https://quay.io
  ```
</details>

<details> <summary> Создание Ingress для ArgoCD </summary>

- ArgoCD использует HTTPS + gRPC, поэтому используем ssl-passthrough:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-ingress
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  ingressClassName: nginx
  rules:
  - host: <ARGOCDNAME>
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: argocd-server
          port:
            number: 443
```
- Применяем конфу
```bash
 kubectl apply -f argocd-ingress.yaml
```

- Получаем порт Ingress Controller'a
kubectl get svc -n ingress-nginx

</details>
<details>
<summary> Послеустановочные действия </summary>

- Добавляем в hosts на хостовой машине 
`*VIRTUAL IP* *ARGOCDNAME*` (например: 192.168.0.110 argocd.localname) 

- Получаем пароль:
  ```bash
  kubectl get secret argocd-initial-admin-secret -b argocd -o jsonpath="{.data.password}" | base64 -d
  ```

 - Открываем https://\*ARGOCDNAME\*.30966
</details>

<details><summary> Базовая работа с GitLab </summary> 
  
- Генерим ключ `ssh-keygen -t ed25519` (либо любой другой метод криптографии)
- Добавляем публичный ключ в настройках 
- Получаем known hosts: ssh-keyscan -p 2222 YOUIP
</details>
<details> <summary> Последующая работа с ArgoCD </summary> 
- В ArgoCD мы подключаемся по SSH:
  Settings > Repositories > Connect Repo (По SSH)    
Важно: Для нестандартного SSH порта URL должен быть в формате ssh://git@host:port.git, а не git@host:path.git  

  И дальше идёт проблема!  
</details>

### 2. Анализ проблемы
- Проблема: статус репозитория Failed.   
- В логах описывается `dial tcp <IP>:<PORT> connect: connection timed out  `
- При этом с самой виртуалки GitLab доступен
`nc -zv <IP_GITLAB> <PORT_GITLAB> - Connection succeeded `
- Получается проблема не в машине, а в поде либо самом кубере

### 3. Поиск проблемы
- Каждый под живёт в изолированном сетевом пространстве со своей таблицей маршрутов.
```bash
sudo crictl inspect $(sudo crictl ps | grep repo-server | awk '{print $1}')
```
- Выдаётся результат, после чего заходим в неймспейс пода
```bash
sudo nsenter -t <POD_NAMESPACES> -n ip route show
```
Результат(на своём примере): 
`default via 169.254.1.1 dev eth0   
169.254.1.1 dev eth0 scope link `
Всего два маршрута. Под без понятия о существующей сети хоста. Весь трафик идёт на calico

- Ищем причину конфликта
```bash
kubectl get ippool -o yaml | grep cidr # cidr: 192.168.0.0/16
```
- Вот и проблема. Calico использовал диапазон для подов в котором существовал GitLab. В итоге когда под пытался подключиться к Гиту, calico думал, что это его собственный под, пытался найти его внутри кластера, не находил его и в итоге дропал пакет

### 4. Решение проблемы
- Меняем CIDR через tigera-operator
```bash
kubectl edit installation default
```
 Находим строку с `calicoNetwork.ipPools`
 меняем `cidr`: Вместо `192.168.0.0/16` на `cidr: 10.244.0.0/16`

- Пересоздаём IPPool
```bash
kubectl delete ippool default-ipv4-ippool
```
```bash
cat <<EOF > new-ippool.yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  ipipMode: Never
  natOutgoing: true
  nodeSelector: all()
  vxlanMode: CrossSubnet
  blockSize: 26
  allowedUses:
  - Workload
  - Tunnel
EOF
```
```bash
kubectl apply -f newest-ippool.yaml
```
- Перезапускаем поды чтобы они получили новый IP из другого диапазона
```bash
kubectl rollout restart deployment -n argocd
kubectl rollout restart statefulset -n argocd
```
- Ждём когда появятся поды:
```bash
kubectl get pods -n argocd -o wide
```

### 5. Просто деплоим приложение 
- Конфигурируем Application манифест
<details> <summary> argocd-conf.yaml </summary>

```yaml
apiVersion: argocdproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default

source:
  repoURL: ssh://git@IP:PORT/path/to/file
  targetRevision: HEAD
  path: guestbook

destination:
  server: https://kubernetes.default.svc
  namespace: guestbook

syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
    -  PrunePropagationPolicy=foreground
  retry:
    limit: 3
    backoff:
      duration: 5s
      maxDuration: 3m
      factor: 2
```

```bash
kubectl apply -f argocd-conf.yaml
kubectl get application -n argocd # чтобы проверить, что всё работает
```
</summary></details>

### 6. Делаем Ingress
<details> <summary> ingress.nginx.yaml </summary>

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <INGRESSNAME>
  namespace: <NAME>
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
  - host: <HOSTNAME>
    http:
      paths: /
      pathType: Prefix
      backend:
        service:
          name: <INGRESSNAME>
          port:
            number: 80
```

- Добавляем всё также в hosts. Вводим туда имя из host строки конфига
</details>

- Открываем!
http://<INGRESSHOSTNAME>:31951

### Итоги 
- Узнал, что CNI может спокойно захватить всю физическую подсеть. Теперь проверяю чтоб cni не пересекался с с сетью хоста
- kubectl exec не всегда доступен. В итоге приходится пользоваться crictl и nsenter
