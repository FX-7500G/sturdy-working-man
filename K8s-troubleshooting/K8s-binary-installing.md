# Установка Кубера через бинарные файлы

Делал этот мануал из-за невозможности установить Кубер через `apt`. Установим бинарники напрямую и настроим systemd юниты вручную

### 1. Подготовка системы
Обновляем пакеты, отключаем Swap

```bash
sudo apt update -y && sudo apt full-upgrade -y
sudo swapoff -a 
sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
```
Настраиваем сетевые модули, открываем порты

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

```bash
sudo ufw allow 6443/tcp       
sudo ufw allow 2379:2380/tcp   
sudo ufw allow 10250/tcp      
sudo ufw allow 10257/tcp       
sudo ufw allow 10259/tcp      
sudo ufw allow 30000:32767/tcp 
sudo ufw allow ssh             
sudo ufw reload
```

### 2. Установка бинарных файлов

```bash
#(ПРОВЕРЯЙТЕ АКТУАЛЬНОСТЬ ВЕРСИИ!)
cd /tmp
curl -LO https://dl.k8s.io/release/v1.34.3/bin/linux/amd64/kubelet
curl -LO https://dl.k8s.io/release/v1.34.3/bin/linux/amd64/kubeadm
curl -LO https://dl.k8s.io/release/v1.34.3/bin/linux/amd64/kubectl
chmod +x kubelet kubeadm kubectl
sudo mv kubelet kubeadm kubectl /usr/local/bin/
```

### 3. Настройка Kubelet 
Так как ставим кубер вручную, нужно создать `systemd` unit, чтобы `kubelet` запускался автоматически

```bash
sudo tee /etc/systemd/system/kubelet.service > /dev/null <<EOF
[Unit]
Description=kubelet: The Kubernetes Node Agent
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet
Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now kubelet
```

### 4. Разные возможные ошибки при инициализации

#### 4.1. Ошибка с hostname. 
K8s не любит заглавные буквы в хостнеймах. Только строчные

#### 4.2. Проблема с pause-контейнера
Если `kubeadm init` виснет, проверь версию `pause`. У меня он искал 3.8, а в новых версиях качается 3.10.1+
```bash
sudo sed -i 's|registry.k8s.io/pause:3.8|registry.k8s.io/pause:3.10.1|g' /etc/containerd/config.toml
sudo systemctl restart containerd
```
#### 4.3 Невозможность установить cri-tools через APT
```bash
VERSION="v1.34.0"
curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/${VERSION}/crictl-${VERSION}-linux-amd64.tar.gz
tar -xvf crictl-${VERSION}-linux-amd64.tar.gz
sudo mv crictl /usr/local/bin/

sudo tee /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock 
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
EOF
```

### 5. Финальная настройка после инициализации
После успешной kubeadm init, не забудь сделать базу. Настрой доступ для kubectl
```bash
rm -rf $HOME/.kube 
mkdir -p $HOME/.kube 
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
И добавь StorageClass, чтобы поды могли сохранять данные
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml 
kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### 6. Настройка сети кластера (CNI) 
Узлы то не будут работать, пока не установлен CNI. Я использовал Calico

**Важно**! При установке Calico, убедись, что параметр pod-network-cidr в манфисете совпадает с тем, что ты указывал при Kubeadm init (в моём случае 10.244.0.0/16)

```bash
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl apply -f calico.yaml
```

## Итог
Это точно не продакшн вариант, так как уверен упустил некоторые моменты, но для собственного развития это мне очень сильно помогло. У меня не были доступны официальные репозитории K8s по неизвестной мне причине. Решение проблем с резолвом ДНС, Настройкой IPv6/IPv4, проверки системных характеристик и стабильности интернета мне не помогло для обычной установки через APT. 
В итоге реализовал альтернативный метод через установку бинарников напрямую. Потратил где-то 1.5 недели на ломание головы, но смог самостоятельно всё собрать и на удивление корректно запустить! На этом кубере у меня даже ГитЛаб свой работает :)
