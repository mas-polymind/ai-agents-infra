# Финальное руководство: Создание переносимого Git-репозитория для AI-Агента

Мы создаем чистую, изолированную архитектуру без GPU, где все зависимости и связи управляются внутри Docker-сети, а пути настраиваются через относительные указатели.

## Структура вашего будущего GitHub-репозитория

Создайте локально на Windows (или сразу на сервере) папку проекта и инициализируйте Git. Структура файлов должна выглядеть так:
```text
ai-agent-infra/
├── .gitignore
├── .env.example
├── config.toml
└── docker-compose.yml
```

---

## Шаг 1: Исходный код файлов репозитория

Создайте эти файлы со следующим содержимым:

### 1. File: `.gitignore`
Этот файл критически важен. Он гарантирует, что тяжелые модели Ollama и ваш созданный ИИ код (проекты) никогда не улетят в публичный доступ на GitHub.
```text
# Игнорируем папки с данными контейнеров
/ollama_data/
/openhands_workspace/

# Игнорируем локальный файл секретных переменных
.env
```

### 2. File: `.env.example`
Шаблон переменных окружения, который будет храниться в Git.
```text
# Версии образов контейнеров
OPENHANDS_VERSION=0.18
OLLAMA_VERSION=latest

# Внутренний порт для доступа к веб-интерфейсу OpenHands
OPENHANDS_PORT=3000
```

### 3. File: `config.toml`
Статический файл конфигурации OpenHands. Здесь жестко прописано имя сервиса `ollama` вместо IP-адресов.
```toml
[core]
workspace_base = "/opt/openhands/workspace"
run_as_user = true

[llm]
model = "ollama/deepseek-r1:7b"
base_url = "http://ollama:11434"
api_key = "none"
embedding_model = "local"
```

### 4. File: `docker-compose.yml`
Манифест, который создает единую сеть, связывает контейнеры и монтирует папки относительно корня репозитория (`./`).
```yaml
version: '3.8'

networks:
  ai-agents-network:
    driver: bridge

services:
  ollama:
    image: ollama/ollama:\${OLLAMA_VERSION}
    container_name: ollama
    networks:
      - ai-agents-network
    volumes:
      - ./ollama_data:/root/.ollama
    ports:
      - "11434:11434"
    restart: unless-stopped

  openhands:
    image: docker.all-hands.dev/all-hands-ai/openhands:\${OPENHANDS_VERSION}
    container_name: openhands-agent
    networks:
      - ai-agents-network
    ports:
      - "\${OPENHANDS_PORT}:3000"
    user: "\({UID}:\){GID}"
    environment:
      - SANDBOX_USER_ID=\${UID}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config.toml:/opt/openhands/config/config.toml
      - ./openhands_workspace:/opt/openhands/workspace
    depends_on:
      - ollama
    restart: unless-stopped
```

---

## Шаг 2: Быстрый запуск на любом сервере за 4 шага

Когда вы перенесете этот репозиторий на любой сервер (через `git clone`), процесс запуска будет выглядеть так:

**1. Подготовка окружения (создание локального `.env`):**
```bash
cp .env.example .env
```

Сначала установите Docker:


# 1. Установите Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# 2. Перезагрузите сессию или выполните
newgrp docker

# 3. Проверьте установку
docker --version

# 4. Теперь запускайте compose правильно
cd ~/ai-agents-infra
export UID=$(id -u)
export GID=$(id -g)
docker compose up -d

# Или одной строкой
UID=$(id -u) GID=$(id -g) docker compose up -d



**3. Скачивание модели DeepSeek-R1 внутрь контейнера:**
```bash
docker exec -it ollama ollama run deepseek-r1:7b
```
*(После завершения скачивания на 100% нажмите `Ctrl + D` или введите `/exit`)*

**4. Настройка Multicoder в локальном VS Code (Windows):**
Вам больше не нужно управлять демонами через SSH-команды. Просто укажите Multicoder, где запущен агент и где лежит папка с его проектами:

```json
{
  "multicoder.agent.provider": "openhands",
  "multicoder.agent.connectionMode": "ssh",
  "multicoder.ssh.host": "<IP_ВАШЕГО_СЕРВЕРА>",
  "multicoder.ssh.username": "ubuntu",
  "multicoder.ssh.authMethod": "publickey",
  "multicoder.ssh.privateKeyPath": "~/.ssh/id_ed25519-llm",
  
  "multicoder.workspace.remotePath": "/home/ubuntu/ai-agent-infra/openhands_workspace",
  "multicoder.workspace.localSync": true,
  
  "multicoder.ui.portForwarding": true,
  "multicoder.ui.localPort": 3000
}
```



# Настройка пути к SSH-ключу в Windows для VS Code

Параметр `"${userHome}/.ssh/id_rsa"` — это шаблонный путь, который указывает VS Code искать ваш **приватный** SSH-ключ в домашней папке пользователя Windows. 

Чтобы этот параметр работал без ошибок, на вашем компьютере с Windows должен физически существовать этот файл ключа. Настроить это можно двумя путями.

---

### Шаг 1: Проверка наличия ключа в Windows

1. Откройте на Windows обычную командную строку (`cmd`) или PowerShell.
2. Выполните команду проверки:
   ```cmd
   dir %USERPROFILE%\.ssh
   ```
3. **Что вы должны увидеть:**
   * Если в списке есть файл `id_rsa` (без расширения), значит, всё готово. В `settings.json` VS Code вы оставляете строку как есть: `"${userHome}/.ssh/id_rsa"`.
   * Если команда выдала ошибку "Файл не найден", значит, у вас на Windows еще нет SSH-ключа, и его нужно создать.

---

### Шаг 2: Создание ключа (если его нет)

Если файла `id_rsa` не оказалось, выполните прямо в терминале Windows команду:
```cmd
ssh-keygen -t rsa -b 4096
```
* **Важно:** Нажимайте `Enter` на все вопросы (не вводите пароль/passphrase, чтобы Multicoder мог подключаться в фоновом режиме автоматически).
* Эта команда создаст в папке `C:\Users\<Ваш_Юзер>\.ssh\` два файла: `id_rsa` (приватный ключ) и `id_rsa.pub` (публичный ключ).

---

### Шаг 3: Перенос ключа на сервер VPS

Чтобы сервер пустил вас по этому ключу, вам нужно скопировать содержимое публичного ключа (`id_rsa.pub`) на ваш Ubuntu-сервер.

1. Откройте файл `id_rsa.pub` на Windows через Блокнот и **скопируйте весь текст** (он начинается с `ssh-rsa ...`).
2. Подключитесь к вашему VPS-серверу (как вы делали это обычно) и откройте файл авторизованных ключей пользователя `ubuntu`:
   ```bash
   nano ~/.ssh/authorized_keys
   ```
3. Вставьте скопированную строку с новой строки в самый конец файла, сохраните (`Ctrl + O`, затем `Enter`) и выйдите (`Ctrl + X`).

---

### Важные нюансы для Windows (Если имя ключа другое)

* **Если вы используете ED25519 вместо RSA:** Иногда современные ключи создаются с именем `id_ed25519`. Если у вас именно такой файл, просто измените строку в `settings.json` на:
  ```json
  "multicoder.ssh.privateKeyPath": "\${userHome}/.ssh/id_ed25519",
  ```
* **Прямой путь (альтернатива):** Если VS Code по какой-то причине не распознает переменную `${userHome}`, вы можете прописать абсолютный путь к вашему пользователю Windows вручную, заменив обратные слэши на прямые:
  ```json
  "multicoder.ssh.privateKeyPath": "C:/Users/Имя_Вашего_Пользователя_Windows/.ssh/id_rsa",
  ```


---

## Важные ответы на ваши вопросы по этой архитектуре:
* **Как обновлять?** Измените версию в файле `.env` (например, `OPENHANDS_VERSION=0.19`) и введите команду `docker compose up -d --pull always`. Docker сам перекачает и перезапустит только то, что изменилось.
* **Как изолированы проекты?** Внутри папки `./openhands_workspace` будут появляться папки ваших проектов (например, `./openhands_workspace/my-python-app/`). Вы можете заходить в любую из них, инициализировать там отдельный `git init` и связывать конкретный проект со своим репозиторием на GitHub.

Схема полностью готова к переносу в ваш приватный GitHub-репозиторий. Хотите ли вы **разобрать процесс инициализации Git** и первой отправки кода, или этот шаг вам знаком?




# Создание пользователя ubuntu и настройка прав за 4 шага

Раз на сервере сейчас есть только `root`, вам нужно создать пользователя `ubuntu` вручную, выдать ему права на использование Docker и настроить его домашнюю папку. 

Выполните следующие команды в терминале от имени пользователя `root`:

---

### 1. Создание пользователя и его домашней папки
```bash
# Создаем пользователя ubuntu и автоматически создаем его домашнюю директорию
useradd -m -s /bin/bash ubuntu

# Задайте пароль для пользователя ubuntu (введите его дважды, символы при вводе не видны)
passwd ubuntu
```


### 2. Выдача прав администратора (Sudo)
Чтобы пользователь мог при необходимости выполнять команды с повышенными правами, добавьте его в группу `sudo`:
```bash
usermod -aG sudo ubuntu
```

### 3. Добавление пользователя в группу Docker
Это самый важный шаг. Он позволит пользователю `ubuntu` управлять контейнерами OpenHands и Ollama без использования `sudo`, что критично для интеграции с VS Code:
```bash
usermod -aG docker ubuntu
```

### 4. Перенос SSH-ключей (чтобы заходить без пароля)
Поскольку ранее вы, скорее всего, настраивали доступ по SSH-ключу для `root`, нужно скопировать этот ключ пользователю `ubuntu`, чтобы вы могли подключаться через VS Code:
```bash
# Создаем папку для ключей в профиле ubuntu
mkdir -p /home/ubuntu/.ssh

# Копируем авторизованные ключи из root к ubuntu
cp /root/.ssh/authorized_keys /home/ubuntu/.ssh/

# Выставляем правильного владельца на файлы
chown -R ubuntu:ubuntu /home/ubuntu/.ssh

# Выставляем безопасные права Linux (строго 700 на папку и 600 на файл)
chmod 700 /home/ubuntu/.ssh
chmod 600 /home/ubuntu/.ssh/authorized_keys
```

---

## Что делать дальше:

Теперь всё готово. Переключитесь на созданного пользователя прямо в терминале:
```bash
su - ubuntu
```
Убедитесь, что команда `whoami` теперь возвращает `ubuntu`, а команда `pwd` показывает `/home/ubuntu`. После этого вы можете безопасно клонировать ваш Git-репозиторий в эту папку.
