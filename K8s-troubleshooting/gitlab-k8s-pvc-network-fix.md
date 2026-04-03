# Решение трабла с хранилищем при развёртывании GitLab в K8s

## Описание проблемы (Issue)
При установке GitLab через Helm возникла проблема с PVC для внутренних компонентов. Поды зависают в статусе `Pending` либо `Init:2/3`

---

### Полное решение

#### 1. Проверка PVC и StorageClass
Если поды MinIo не стартуют, проверяем события: `kubectl describe pod "pods-names" -n gitlab`.
Если видим `persistentvolumeclaim not found`, значит динамический провижининг не отработал

Настраиваем дефолтный StorageClass: 
```bash
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### 2. Создание бакетов в MinIO вручную
MinIO бывает запускается пустым, и миграции GitLab падают с ошибкой NoSuchBucket. Используем toolbox для ручного создания структуры бакетов:
```bash
TOOLBOX_POD=$(kubectl get pods -n gitlab -l app=toolbox -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $TOOLBOX_POD -n gitlab -- s3cmd mb s3://registry
kubectl exec -it $TOOLBOX_POD -n gitlab -- s3cmd mb s3://git-lfs
kubectl exec -it $TOOLBOX_POD -n gitlab -- s3cmd mb s3://artifacts
# После подготовки хранилища, стоит принудительно запустить миграцию
kubectl exec -it $TOOLBOX_POD -n gitlab -- gitlab-rake db:migrate
```

#### 3. Настройка сети и MTU
Таймауты при обращении компонентов происходят из-за того, что пакеты не проходят через интерфейсы из-за их размера (1500 MTU чаще всего)
```bash
kubectl patch installation.operator.tigera.io default --type merge -p '{"spec":{"calicoNetwork":{"mtu":1400}}}'
kubectl rollout restart daemonset -n kube-system calico-node
```

#### 4. Финал
Обычно после ручной установки которая изложена в вышеуказанных трёх пунктах всё должно работать. Далее идёт уже то, что я делал для работы на свой железе

##### 4.1 Настройка MetalLB
- Устанавливаем всё по официальным мануалам (https://metallb.io/installation/)
- Проверяем наличие CRD через `kubectl get crd | grep metallb`
- Создаём файл metallb-config.yaml с нужным диапозоном свободных IP роутера
<details>
  <summary>
    Пример моего конфига:
  </summary>

  ```yaml
  apiVersion: metallb.io/v1beta1
    kind: IPAddressPool
    metadata:
      name: default
      namespace: metallb-system
    spec:
      addresses:
      - 192.168.0.200-192.168.0.210
    ---
    apiVersion: metallb.io/v1beta1
    kind: L2Advertisement
    metadata:
      name: l2-adv
      namespace: metallb-system
    spec:
      ipAddressPools:
      - default
  ```
</details>

##### 4.2 Настройка хостов
Смотрим какие домены нужны на хосте через `kubectl get ingress -n gitlab` и прописываем их в файле `hosts` на хостовой ОС (подсказка. На винде это путь `C:\Windows\System32\drivers\etc\hosts` )

##### 4.3 Проброс маршрута
Пробрасываем маршруты на хосте для получения доступа к сервисам виртуалки
```DOS
route add 192.168.0.<required MetalLB_IP> mask 255.255.255.255 <Virtual machine IP>
```
