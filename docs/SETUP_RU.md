# Пошаговая установка VPN-сервера

Сценарий: есть свежий VPS, `root` и пароль от провайдера, нужно получить сервер с 3X-UI, веб-панелями и SSH только по ключу.

## 0. Что понадобится

- VPS на Ubuntu/Debian.
- Домен, A-запись которого смотрит на IP сервера.
- SSH-ключ на локальной машине.
- Ansible на WSL/Linux.
- Желательно 4 GB RAM. На 2 GB лучше добавить swap.

Проверить публичный ключ:

```bash
cat ~/.ssh/id_ed25519.pub
```

Если ключ лежит в Windows, в WSL он может быть здесь:

```bash
ls -la /mnt/c/Users/YOUR_WINDOWS_USER/.ssh
```

## 1. Подготовить файлы

```bash
cp inventory/hosts.ini.example inventory/hosts.ini
cp group_vars/vpn.yml.example group_vars/vpn.yml
ansible-galaxy collection install -r requirements.yml
```

Если проект лежит в `/mnt/d/...`, Ansible может игнорировать `ansible.cfg`. Тогда запускайте команды с:

```bash
ANSIBLE_ROLES_PATH="$PWD/roles"
```

## 2. Первый inventory: root и пароль

Откройте:

```bash
vim inventory/hosts.ini
```

Для первого запуска:

```ini
[vpn]
vpn1 ansible_host=YOUR_SERVER_IP ansible_user=root ansible_port=22

[vpn:vars]
ansible_python_interpreter=/usr/bin/python3
```

## 3. Заполнить переменные

Откройте:

```bash
vim group_vars/vpn.yml
```

Минимум:

```yaml
bootstrap_user: deployer
bootstrap_sudo_nopasswd: true
bootstrap_ssh_public_key: "ssh-ed25519 AAAA... your_key_comment"

torotin_webdomain: "vpn.example.com"
torotin_user_ssh: "{{ bootstrap_user }}"
torotin_pass_ssh: "CHANGE_ME_TEMP_PASSWORD"
torotin_ssh_port: 22022
torotin_ssh_public_key: "{{ bootstrap_ssh_public_key }}"

torotin_user_web: "admin"
torotin_pass_web: "CHANGE_ME_STRONG_PASSWORD"
```

Важно:

- `torotin_webdomain` должен быть реальным доменом.
- DNS A-запись домена должна смотреть на IP сервера.
- Не оставляйте `CHANGE_ME`.
- `torotin_pass_web` это пароль от веб-панелей.

## 4. Добавить fingerprint сервера

Ansible с `--ask-pass` не умеет сам принять host key. Один раз зайдите руками:

```bash
ssh root@YOUR_SERVER_IP
```

Ответьте `yes`, введите пароль, затем выйдите:

```bash
exit
```

## 5. Bootstrap: создать deployer и ключ

Если нет `sshpass`:

```bash
sudo apt update
sudo apt install -y sshpass
```

Запуск:

```bash
ANSIBLE_ROLES_PATH="$PWD/roles" \
ansible-playbook -i inventory/hosts.ini playbooks/00-bootstrap-user.yml --ask-pass
```

Результат: на сервере создан пользователь `deployer`, добавлен SSH-ключ и sudo без пароля.

## 6. Проверить вход по ключу

```bash
ssh deployer@YOUR_SERVER_IP
```

Если ключ не стандартный:

```bash
ssh -i /path/to/id_ed25519 deployer@YOUR_SERVER_IP
```

## 7. Второй inventory: deployer и ключ

Теперь `inventory/hosts.ini`:

```ini
[vpn]
vpn1 ansible_host=YOUR_SERVER_IP ansible_user=deployer ansible_port=22 ansible_ssh_private_key_file=~/.ssh/id_ed25519

[vpn:vars]
ansible_python_interpreter=/usr/bin/python3
```

Если запускаете Ansible из-под `root` в WSL, `~/.ssh/id_ed25519` означает `/root/.ssh/id_ed25519`. Тогда укажите реальный путь:

```ini
ansible_ssh_private_key_file=/mnt/c/Users/YOUR_WINDOWS_USER/.ssh/id_ed25519
```

## 8. Установить Torotin/3X-UI stack

```bash
ANSIBLE_ROLES_PATH="$PWD/roles" \
ansible-playbook -i inventory/hosts.ini playbooks/10-install-3xui.yml
```

На 2 GB RAM установка может идти долго. Если сервер упирается в память, добавьте swap:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

## 9. После установки: SSH порт 22022

Installer меняет SSH-порт на `torotin_ssh_port`, по умолчанию `22022`.

Проверьте:

```bash
ssh -p 22022 deployer@YOUR_SERVER_IP
```

После этого обновите `inventory/hosts.ini`:

```ini
[vpn]
vpn1 ansible_host=YOUR_SERVER_IP ansible_user=deployer ansible_port=22022 ansible_ssh_private_key_file=~/.ssh/id_ed25519

[vpn:vars]
ansible_python_interpreter=/usr/bin/python3
```

Проверить Ansible:

```bash
ANSIBLE_ROLES_PATH="$PWD/roles" \
ansible -i inventory/hosts.ini vpn -m ping
```

## 10. Где взять веб-ссылки

На сервере:

```bash
ssh -p 22022 deployer@YOUR_SERVER_IP
sudo cat /srv/3x-ui_pro_Docker/script/install-state/install.summary
```

Там будут:

- `Homepage` - стартовая страница со ссылками и статусами.
- `3X-UI Panel` - основная панель для VPN-клиентов.
- `AdGuard Home` - DNS-фильтрация.
- `AdGuard DoH` - endpoint DNS-over-HTTPS, это не обычная веб-страница.
- `Dozzle Logs` - веб-логи контейнеров.
- `Traefik Dashboard` - диагностика HTTPS-маршрутов.
- `Telemt Panel` - прокси/маршрутизация/маскировочная прослойка.

## 11. Как подключить VPN-клиент

Стандартный Windows VPN не подходит. Нужен клиент для Xray/VLESS:

- Hiddify Next
- v2rayN
- NekoRay/Nekobox

В 3X-UI:

1. Откройте `3X-UI Panel`.
2. Перейдите в `Inbounds`.
3. Создайте или откройте inbound.
4. Создайте клиента.
5. Скопируйте ссылку вида `vless://...` или subscription URL.
6. Импортируйте ссылку в Hiddify/v2rayN.

Для Hiddify Next:

```text
Add profile -> Import from clipboard
```

Для v2rayN:

```text
Servers -> Import bulk URL from clipboard
```

## 12. Полезные команды

```bash
sudo docker ps
sudo docker ps -a
sudo /srv/docker-proxy/compose.d/run-compose.sh ps
sudo /srv/docker-proxy/compose.d/run-compose.sh validate
sudo tail -n 200 /var/log/torotin-3xui-install.log
sudo tail -n 200 /srv/3x-ui_pro_Docker/script/install-state/install.log
sudo /srv/3x-ui_pro_Docker/script/install.sh doctor
sudo cat /srv/3x-ui_pro_Docker/script/install-state/install.summary
```

## 13. Частые ошибки

`sshpass program not installed`

```bash
sudo apt install -y sshpass
```

`Host Key checking is enabled`

```bash
ssh root@YOUR_SERVER_IP
```

`no such identity: /root/.ssh/id_ed25519`

Укажите правильный путь к ключу в `inventory/hosts.ini`.

`No inventory was parsed` или `role not found`

```bash
ANSIBLE_ROLES_PATH="$PWD/roles" \
ansible-playbook -i inventory/hosts.ini playbooks/10-install-3xui.yml
```

`docker ps` пустой

```bash
sudo /srv/docker-proxy/compose.d/run-compose.sh ps
sudo tail -n 200 /var/log/torotin-3xui-install.log
```
