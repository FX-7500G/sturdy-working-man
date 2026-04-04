# Автоматизация установки Docker через Ansible

## Проблема
Проблемы самой по себе нет. Тут будет обычный гайд как написан конфиг для моих целей. Писал для группы хостов на CentOS/Fedora/RHEL и обеспечив работу юзера без `sudo`

## Решение
- Был написан playbook, который использует нативные модули Ansible 

<details>
  <summary> Сам плейбук </summary>

```yaml
- name: Install and Configure Docker Engine
  hosts: localhost
  connection: local
  become: true

  tasks:
    - name: Install prerequisite packages
      ansible.builtin.dnf:
        name:
          - dnf-plugins-core
          - ca-certificates
          - curl
        state: present

    - name: Add Docker CE repository
      ansible.builtin.yum_repository:
        name: docker-ce
        description: Docker CE Stable - $basearch
        baseurl: [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo)
        enabled: yes
        gpgcheck: yes
        gpgkey: [https://download.docker.com/linux/centos/gpg](https://download.docker.com/linux/centos/gpg)

    - name: Install Docker Engine components
      ansible.builtin.dnf:
        name:
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present

    - name: Ensure docker group exists
      ansible.builtin.group:
        name: docker
        state: present

    - name: Add current user to docker group
      ansible.builtin.user:
        name: "{{ ansible_user_id }}"
        groups: docker
        append: yes
      notify: Restart Docker

  handlers:
    - name: Restart Docker
      ansible.builtin.systemd:
        name: docker
        state: restarted
        enabled: yes
```
</details>

### Подробное ручное решение
#### 1. Выбор инструментов
- Использовал модули dnf, yum_repository, group, user, systemd
- Целевые ОС: CentOS/Fedora/RHEL
#### 2. Установка зависимостей
- Используем модуль ansible.builtin.dnf для установки базовых пакетов
- Добавляем официальный репозиторий Docker CE с проверкой GPG Key, чтобы установить последнюю стабильную версию Докера без сторонних источников
#### 3. Установка Docker Engine и компонентов
- Устанавливаем основной движок, CLI, Buildx, Docker compose через модуль dnf
#### 4. Настройка юзера
- Создаём группу docker, добавляем текущего пользователя, чтобы работать с докер без `sudo`
<details>
  <summary> Добавление юзера </summary>

```yaml
- name: Ensure docker group exists
  ansible.builtin.group:
    name: docker
    state: present

- name: Add current user to docker group
  ansible.builtin.user:
    name: "{{ ansible_user_id }}"
    groups: docker
    append: yes
  notify: Restart Docker
```
</details>
- Уведомление notify перезапустит докер после всех изменений, чтоб изменения вступили в силу

#### 5. Рестарт докера
- Используем хендлер для перезапуска демона и включения его в автозапуск
<details>
  <summary> handlers block </summary>

  ```yaml
handlers:
  - name: Restart Docker
    ansible.builtin.systemd:
      name: docker
      state: restarted
      enabled: yes
```
</details>

## Итог
- Docker установлен на всех хостах целевых ОС
- Целевой юзер может работать с Docker без `sudo`
