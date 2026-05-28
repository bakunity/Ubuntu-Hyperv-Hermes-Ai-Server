# Гайд: Ubuntu Server + Hyper‑V + Docker + VPN TUN + Hermes Agent + VS Code Remote SSH

## Цель

Собрать отдельную Linux‑машину для AI‑агентов в Hyper‑V:

```text
Windows Host
  ↓
Hyper‑V VM
  ↓
Ubuntu Server 24.04 LTS
  ↓
Docker
  ↓
sing-box VPN в TUN mode
  ↓
Hermes Agent / AI tools / Docker containers
  ↓
GPT / Gemini / OpenRouter / Claude
```

В результате получится отдельный AI‑сервер, который:

- запускается в Hyper‑V;
- имеет стабильный IP;
- доступен из VS Code через Remote SSH;
- работает через системный VPN;
- гонит Docker‑контейнеры через VPN;
- готов для Hermes Agent и других AI‑агентов.

---

# 0. Переменные, которые нужно заменить под себя

В этом гайде все личные данные заменены на placeholders.

Перед выполнением команд подставь свои значения:

```text
<USERNAME>       — Linux username, например aiuser
<SERVER_NAME>    — hostname сервера, например ai-server
<CURRENT_IP>     — текущий DHCP IP Ubuntu до настройки static IP
<STATIC_IP>      — будущий постоянный IP Ubuntu
<GATEWAY_IP>     — gateway твоей подсети
<PREFIX>         — маска сети, например /20 или /24
<VPN_SERVER>     — сервер VPN из подписки
<VPN_PORT>       — порт VPN
<VPN_UUID>       — UUID / client id из VPN
<VPN_SNI>        — SNI / server_name из VPN
<VPN_LOCATION>   — выбранная страна/локация VPN
```

Никогда не публикуй в открытый доступ:

```text
VPN subscription URL
VPN UUID / private keys
API keys
.env файлы
SSH private keys
```

---

# 1. Что понадобится

## На Windows

- Windows 10/11 Pro, Enterprise или Education.
- Hyper‑V включён.
- Visual Studio Code.
- Расширение VS Code: **Remote - SSH**.
- PowerShell или Windows Terminal.

## Для Ubuntu

- ISO: **Ubuntu Server 24.04 LTS amd64**.
- Рекомендуемый диск VM: 80 GB+.
- RAM: 8–16 GB, если будут AI‑агенты и Docker.
- CPU: 4 vCPU или больше.

## Для VPN

Нужна одна стабильная конфигурация VPN, например:

```text
vless://...
```

или экспортированный JSON / TXT / subscription dump от VPN‑провайдера.

---

# 2. Создание VM в Hyper‑V

## Рекомендуемые настройки VM

```text
Generation: 2
RAM: 8 GB или больше
Dynamic Memory: OFF
CPU: 4 vCPU или больше
Disk: VHDX 80 GB или больше
Network: Default Switch или External Switch
Secure Boot: ON, Microsoft UEFI Certificate Authority
```

Для максимальной стабильности сети лучше использовать **External Switch**, но можно использовать и **Default Switch** + статический IP внутри подсети Hyper‑V.

---

# 3. Установка Ubuntu Server

Загружаемся с ISO Ubuntu Server.

## Boot menu

Выбрать:

```text
Try or Install Ubuntu Server
```

Не выбирать HWE kernel, если цель — стабильность.

## Обновление installer

Если установщик предлагает обновиться:

```text
Continue without updating
```

Систему потом обновим через `apt`.

## Keyboard

Можно выбрать:

```text
Russian
English (US)
```

Переключение удобно поставить:

```text
Alt + Shift
```

Для Linux/dev желательно иметь **English (US)**.

## Type of install

Выбрать:

```text
Ubuntu Server
```

Не выбирать minimized.

## Network

Если DHCP выдал IP — ничего не менять.

## Proxy

Оставить пустым.

## Mirror

Оставить default, если проверка прошла.

## Disk layout

Выбрать:

```text
Use an entire disk
```

Снять галочку:

```text
Set up this disk as an LVM group
```

Не включать LUKS encryption, если нет отдельной необходимости.

Итог:

```text
ext4
без LVM
без encryption
```

## Profile setup

Пример безопасного шаблона:

```text
Your name: Your Name
Server name: <SERVER_NAME>
Username: <USERNAME>
Password: свой пароль
```

Username лучше делать маленькими английскими буквами:

```text
aiuser
```

## SSH

Обязательно включить:

```text
Install OpenSSH server
Allow password authentication over SSH
```

SSH key пока можно не импортировать.

## Featured snaps

Ничего не выбирать.

После установки нажать:

```text
Reboot Now
```

Если появится:

```text
Please remove the installation medium, then press ENTER
```

В Hyper‑V отключить ISO:

```text
Media → DVD Drive → Eject
```

Потом нажать Enter.

---

# 4. Первый вход и обновление системы

После загрузки Ubuntu будет только терминал. Это нормально: установлена Server‑версия без GUI.

Войти под своим пользователем:

```text
<USERNAME>
```

Обновить систему:

```bash
sudo apt update && sudo apt upgrade -y
```

Перезагрузить:

```bash
sudo reboot
```

---

# 5. SSH подключение из Windows

Узнать IP в Ubuntu:

```bash
ip a
```

Текущий IP обозначим так:

```text
<CURRENT_IP>
```

На Windows в PowerShell:

```powershell
ssh <USERNAME>@<CURRENT_IP>
```

При первом подключении написать:

```text
yes
```

Если SSH не работает, проверить в Ubuntu:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh
```

Должно быть:

```text
active (running)
```

---

# 6. Установка базовых утилит

```bash
sudo apt install -y git curl wget unzip htop btop net-tools build-essential software-properties-common apt-transport-https ca-certificates gnupg lsb-release
```

---

# 7. Установка Docker

```bash
sudo apt install -y docker.io docker-compose-v2
```

Включить Docker:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

Добавить пользователя в группу `docker`:

```bash
sudo usermod -aG docker $USER
```

Что это значит: после этого можно запускать Docker без `sudo`:

```bash
docker ps
```

После добавления в группу нужен новый login или reboot:

```bash
sudo reboot
```

После перезагрузки проверить:

```bash
docker ps
```

Ожидаемый результат:

```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

Пустая таблица — это нормально.

---

# 8. VS Code Remote SSH

## Установить на Windows

В VS Code установить extension:

```text
Remote - SSH
```

## Подключение

В VS Code:

```text
F1 → Remote-SSH: Connect to Host
```

Ввести:

```text
ssh <USERNAME>@<CURRENT_IP>
```

Если спросит platform:

```text
Linux
```

Когда подключится, открыть папку:

```text
/home/<USERNAME>
```

Если VS Code спрашивает доверять папке:

```text
Trust Folder & Continue
```

---

# 9. SSH без пароля для VS Code

Ключ нужно создавать на Windows, потому что VS Code подключается с Windows.

## На Windows PowerShell

```powershell
ssh-keygen
```

Везде нажать Enter. Passphrase оставить пустым, если нужно подключение без запроса пароля.

Ключи появятся здесь:

```text
C:\Users\<WINDOWS_USER>\.ssh\id_ed25519
C:\Users\<WINDOWS_USER>\.ssh\id_ed25519.pub
```

## Отправить ключ на Ubuntu

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <USERNAME>@<CURRENT_IP> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

Если всё равно просит пароль, вручную проверить на Ubuntu:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Вставить содержимое `id_ed25519.pub`, сохранить:

```text
CTRL + O
ENTER
CTRL + X
```

Поставить права:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Проверка с Windows:

```powershell
ssh <USERNAME>@<CURRENT_IP>
```

Пароль спрашивать уже не должно.

---

# 10. Статический IP для Ubuntu

Hyper‑V Default Switch может менять IP после перезагрузки. Поэтому задаём статический IP.

Сначала узнать текущую сеть:

```bash
ip a
ip route
```

Нужно определить:

```text
<CURRENT_IP>/<PREFIX>
<GATEWAY_IP>
```

Выбрать свободный IP в этой же подсети:

```text
<STATIC_IP>/<PREFIX>
```

Открыть netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Шаблон:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: false

      addresses:
        - <STATIC_IP>/<PREFIX>

      routes:
        - to: default
          via: <GATEWAY_IP>

      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Пример формата, не копировать без замены:

```yaml
addresses:
  - <STATIC_IP>/<PREFIX>
routes:
  - to: default
    via: <GATEWAY_IP>
```

Сохранить:

```text
CTRL + O
ENTER
CTRL + X
```

Применить:

```bash
sudo netplan apply
```

SSH может оборваться — это нормально, потому что IP изменился.

Подключиться заново:

```powershell
ssh <USERNAME>@<STATIC_IP>
```

В VS Code нужно поменять SSH config:

```text
F1 → Remote-SSH: Open SSH Configuration File
```

Пример:

```sshconfig
Host ai-server
    HostName <STATIC_IP>
    User <USERNAME>
```

Важно: если Hyper‑V Default Switch полностью сменит подсеть, этот static IP может перестать работать. Для максимально постоянного IP лучше использовать External Switch или DHCP reservation на роутере.

---

# 11. Установка Hermes Agent

В Ubuntu / VS Code terminal:

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
```

Во время установки:

## Install ripgrep / ffmpeg

Выбрать:

```text
y
```

или Enter.

## Install build tools

Выбрать:

```text
y
```

или Enter.

## Provider setup

Если VPN ещё не настроен, выбрать:

```text
Leave unchanged
```

## Execution backend

Для старта выбрать:

```text
Local - run directly on this machine
```

## Gateway working directory

Оставить default.

## Enable sudo support

Лучше выбрать:

```text
N
```

Чтобы Hermes не хранил sudo password.

## Telegram / Discord gateway

На первом этапе:

```text
Skip — set up later
```

---

# 12. Установка sing-box VPN

Установка:

```bash
curl -fsSL https://sing-box.app/install.sh | sudo bash
```

Создать config directory:

```bash
sudo mkdir -p /etc/sing-box
```

---

# 13. VPN TUN config

Ниже пример для VLESS TLS. Реальные значения нужно взять из своей подписки / export:

```text
<VPN_SERVER>
<VPN_PORT>
<VPN_UUID>
<VPN_SNI>
```

Создать конфиг:

```bash
sudo tee /etc/sing-box/config.json > /dev/null << 'EOF'
{
  "log": {
    "level": "info"
  },

  "dns": {
    "servers": [
      {
        "address": "8.8.8.8",
        "tag": "google"
      }
    ],

    "rules": [
      {
        "outbound": "any",
        "server": "google"
      }
    ]
  },

  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "singtun0",
      "address": [
        "<TUN_INTERNAL_IP>/30"
      ],
      "mtu": 1500,
      "auto_route": true,
      "strict_route": true,
      "stack": "system"
    }
  ],

  "outbounds": [
    {
      "type": "vless",
      "tag": "proxy",

      "server": "<VPN_SERVER>",
      "server_port": <VPN_PORT>,

      "uuid": "<VPN_UUID>",

      "tls": {
        "enabled": true,
        "server_name": "<VPN_SNI>",

        "alpn": [
          "h2"
        ],

        "utls": {
          "enabled": true,
          "fingerprint": "chrome"
        }
      }
    },

    {
      "type": "direct",
      "tag": "direct"
    }
  ],

  "route": {
    "auto_detect_interface": true,
    "final": "proxy"
  }
}
EOF
```

Для `<TUN_INTERNAL_IP>` можно использовать любой свободный внутренний диапазон, который не конфликтует с твоей сетью, например из private range. Это не публичный IP.

Проверить config:

```bash
sudo sing-box check -c /etc/sing-box/config.json
```

Если команда ничего не вывела — config валидный.

Запустить:

```bash
sudo systemctl restart sing-box
sudo systemctl enable sing-box
```

Проверить статус:

```bash
sudo systemctl status sing-box
```

Ожидается:

```text
Active: active (running)
```

---

# 14. Проверка VPN

Проверить внешний IP:

```bash
curl ifconfig.me
```

Проверить гео:

```bash
curl ipinfo.io/json
```

Ожидаемый результат:

```json
{
  "city": "<VPN_LOCATION>",
  "country": "<VPN_COUNTRY_CODE>"
}
```

Если DNS не работает:

```text
curl: (6) Could not resolve host
```

Проверить:

```bash
sudo systemctl status sing-box
sudo journalctl -u sing-box -n 100 --no-pager
```

---

# 15. Проверка Docker через VPN

```bash
docker run --rm curlimages/curl https://ipinfo.io/json
```

Если Docker тоже показывает твою VPN‑локацию — значит контейнеры идут через VPN.

---

# 16. Проверка после reboot

Перезагрузить:

```bash
sudo reboot
```

С Windows подключиться:

```powershell
ssh <USERNAME>@<STATIC_IP>
```

Проверить VPN:

```bash
curl ipinfo.io/json
```

Проверить Docker через VPN:

```bash
docker run --rm curlimages/curl https://ipinfo.io/json
```

Если оба показывают VPN‑локацию — всё готово.

---

# 17. VS Code AI agent внутри Ubuntu

Можно поставить AI‑агента прямо в VS Code:

## Вариант 1: Cline

Хороший вариант для новичка:

- видит папку проекта;
- может менять файлы;
- может запускать команды;
- обычно спрашивает разрешение.

## Вариант 2: Continue

Хорош для:

- chat по кодовой базе;
- autocomplete;
- подключения разных моделей.

Важно: AI‑агент в VS Code будет работать внутри Remote SSH окружения, то есть в Ubuntu VM, а не напрямую на Windows host. Это хорошо для изоляции.

---

# 18. Что НЕ давать AI‑агенту

Не давать бездумно:

```text
sudo/root доступ
полный доступ к Windows host
секреты/API keys без .gitignore
авто-выполнение любых команд без подтверждения
```

Правильная схема:

```text
AI agent работает внутри Ubuntu VM
видит только рабочую папку проекта
опасные команды подтверждаются вручную
секреты лежат в .env и не попадают в git
```

---

# 19. Backup / Checkpoint / Export VM

## Checkpoint

В Hyper‑V Manager:

```text
ПКМ по VM → Checkpoint
```

Это быстрый rollback.

## Export VM

Перед export лучше выключить VM:

```bash
sudo shutdown now
```

Потом в Hyper‑V Manager:

```text
ПКМ по VM → Export...
```

Так можно перенести готовую AI‑машину на другой ПК.

Важно: перед передачей другому человеку удалить секреты:

- VPN subscription;
- API keys;
- `.env`;
- SSH keys;
- shell history.

---

# 20. Текущий итог готовой системы

Готово:

```text
Ubuntu Server 24.04 LTS
Hyper‑V VM
Static IP
SSH без пароля
VS Code Remote SSH
Docker
Docker без sudo
sing-box VPN
TUN mode
system-wide VPN
Docker через VPN
Hermes Agent установлен
```

Проверки:

```bash
ip a
curl ipinfo.io/json
docker ps
docker run --rm curlimages/curl https://ipinfo.io/json
sudo systemctl status sing-box
sudo systemctl status docker
```

---

# 21. Следующий план

## Этап 1: модели и providers

- OpenRouter;
- OpenAI;
- Gemini;
- Claude;
- Hermes provider config.

## Этап 2: AI tooling

- Playwright;
- browser-use;
- MCP servers;
- Cline / Continue в VS Code.

## Этап 3: self-hosted UI

- Open WebUI;
- Ollama, если нужны локальные модели;
- n8n / Flowise / Langflow по желанию.

## Этап 4: production hardening

- firewall;
- SSH hardening;
- subscription auto-update для VPN;
- backup/export VM;
- monitoring.

---

# 22. Быстрые команды проверки

## VPN

```bash
curl ipinfo.io/json
```

## Docker через VPN

```bash
docker run --rm curlimages/curl https://ipinfo.io/json
```

## sing-box

```bash
sudo systemctl status sing-box
sudo journalctl -u sing-box -n 100 --no-pager
```

## Docker

```bash
docker ps
docker --version
```

## IP Ubuntu

```bash
ip a
ip route
```

## SSH

```bash
sudo systemctl status ssh
```

---

# 23. Частые проблемы

## VS Code просит пароль

Проверить SSH key:

```powershell
ssh <USERNAME>@<STATIC_IP>
```

Если просит пароль — ключ не добавлен или права неправильные.

На Ubuntu:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## IP поменялся

Проверить netplan:

```bash
cat /etc/netplan/50-cloud-init.yaml
ip a
ip route
```

## VPN запущен, но сайты не открываются

```bash
sudo journalctl -u sing-box -n 100 --no-pager
sudo sing-box check -c /etc/sing-box/config.json
```

## Docker не работает без sudo

```bash
sudo usermod -aG docker $USER
sudo reboot
```

## DNS ошибка

```text
Could not resolve host
```

Проверить sing-box config DNS и статус:

```bash
sudo systemctl status sing-box
curl ipinfo.io/json
```

---

# Финальный результат

После всех шагов получается готовая база для AI‑агентов:

```text
Windows Host
  ↓
Hyper‑V Ubuntu Server
  ↓
Static IP + SSH keys
  ↓
VS Code Remote SSH
  ↓
Docker
  ↓
sing-box TUN VPN
  ↓
Hermes Agent
  ↓
GPT / Gemini / OpenRouter / Claude
```

Это можно использовать как личный AI server template или golden image.

