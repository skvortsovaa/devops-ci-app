# Сервер приложения DevOps

## Проверенный образ из практической работы №10

- Репозиторий: https://github.com/skvortsovaa/devops-ci-app
- GHCR: `ghcr.io/skvortsovaa/devops-ci-app`
- SHA: `3397614b7b716f91fb6a82d039d8d3f150e7a4ab`
- Digest: `sha256:cbe9a284476030bcc94b1e8cb37c459364aca526dc8b1894a0cccf3712ea4846`
- Проверенный CI: https://github.com/skvortsovaa/devops-ci-app/actions/runs/45658745256
- Пакет публичный, анонимное чтение не требует токена.

## 10. Ansible

Проект находится в `ansible/`. Управляющий узел — Ubuntu в WSL на Windows-хосте,
рабочий каталог `/home/devops/devops-ci-app/ansible`, вне `/mnt/c`.
Управляемый узел — Ubuntu Server 24.04 amd64 на VirtualBox VM `devops-vm`,
восстанавливаемой к снимку `01-clean-install` перед первым запуском.
Текущее состояние предыдущих работ предварительно сохранено отдельным снимком.

Схема: WSL (Ansible) → SSH 22 → VM; клиент → Nginx 80 →
`127.0.0.1:5000` → контейнер `devops-ci-app:5000`.

`inventory/bootstrap.ini` использует `student@192.168.56.101:22` с запросами
пароля SSH и sudo. `inventory/prod.ini` использует `devops` и ключ
`~/.ssh/devops_vm`. Закрытый ключ хранится только в домашнем каталоге WSL,
с правами `0600`, и не входит в репозиторий. Ключ хоста сверяется с ранее
доверенным ключом VM; проверка ключа хоста включена.

Роли выполняются в следующем порядке:

- `base`: проверяет Ubuntu 24.04 amd64, устанавливает базовые пакеты,
  `python3-debian` и `python3-requests`, добавляет официальный репозиторий
  Docker, устанавливает и запускает Docker Engine.
- `ssh`: создаёт пользователя и каталог SSH, устанавливает открытый ключ,
  проверяемый через `visudo` файл sudoers, затем проверяемую конфигурацию
  SSH с запретом паролей и root-входа; разрешает UFW порты 22 и 80.
- `nginx`: устанавливает Nginx, разворачивает шаблон reverse proxy, отключает
  default site; перед reload обработчик выполняет `nginx -t`.
- `app`: загружает проверенный GHCR-образ до изменения контейнера, запускает
  контейнер и ожидает HTTP 200 с JSON `{"status":"ok"}` на `/health`.

Версия задаётся переменной `app_tag` в `group_vars/all.yml`; используется
полный SHA выше. Сборка на VM не выполняется. Порт контейнера 5000 публикуется
только как `127.0.0.1:5000:5000`, поэтому внешний запрос проходит через Nginx.

Подготовка управляющего узла:

```sh
sudo apt update
sudo apt install -y ansible sshpass
cd ~/devops-ci-app
ansible-galaxy collection install -r ansible/requirements.yml
cd ansible
ansible-inventory -i inventory/bootstrap.ini --graph
ansible-inventory --graph
ansible-playbook -i inventory/bootstrap.ini site.yml --syntax-check
ansible-playbook site.yml --syntax-check
```

Первичный запуск:

```sh
ansible-playbook -i inventory/bootstrap.ini site.yml --ask-pass --ask-become-pass
```

Повторный неинтерактивный запуск и аудит:

```sh
ansible-playbook site.yml
ansible-playbook verify.yml
curl --fail http://192.168.56.101/health
```

`force_handlers = True` обеспечивает выполнение уже уведомлённых handlers
при отказе последующей задачи. Это не скрывает ошибку самого play.
`verify.yml` только читает состояние SSH, контейнера и `/health`;
`changed_when: false` у `sshd -T` сохраняет корректный результат `changed=0`.

Проверка восстановления: удалить учебный контейнер и заменить upstream Nginx
на порт 5001; затем восстановить конфигурацию только `site.yml`.
Проверка отказа версии: временно установить SHA из сорока нулей в
`group_vars/all.yml`, выполнить play и проверить, что прежний контейнер
остаётся доступен; затем вернуть файл через Git и повторить play.

`NOPASSWD:ALL` и членство в группе `docker` дают пользователю полный
административный доступ и применяются в рамках этой учебной VM.
Для следующего занятия GitHub-hosted runner не достигнет Host-only адреса:
понадобится сетевой туннель, облачная VM либо защищённый self-hosted runner.
Ключ деплоя и другие секреты не следует сохранять в исходниках и журналах.

### Проверки 26 ноября 2026 года

Управляющий узел: WSL Ubuntu 26.04, Ansible core 2.20.1, Python 3.14.
Управляемый узел: Ubuntu Server 24.04.5, Python 3.12.
Коллекции закреплены в `requirements.yml` согласно заданию.

- Bootstrap от `student`: `ok=27 changed=18 failed=0`.
- Первый повтор через prod: `ok=24 changed=0 failed=0`, handlers не запускались.
- Вход `devops` по ключу и `sudo -n true` успешны; password-only SSH отклонён
  с `Permission denied (publickey)`.
- `/health` и `/` через Nginx вернули HTTP 200. Внешний порт 5000 недоступен;
  Docker публикует его только на `127.0.0.1`.
- После удаления контейнера и изменения upstream на 5001 получен HTTP 502.
  Только `site.yml` восстановил шаблон и контейнер, выполнил reload Nginx:
  `ok=26 changed=3 failed=0`; внешний `/health` снова вернул 200.
- Тег из сорока нулей дал 404 в `Pull reviewed application image`:
  `ok=21 changed=0 failed=1`. До container task play не дошёл. Полный ID
  контейнера и его `Config.Image` не изменились, `/health` остался доступен.
  После `git restore group_vars/all.yml` получено `changed=0 failed=0`.
- Два запуска `verify.yml`: `ok=7 changed=0 failed=0` каждый.
  После `docker stop` аудит обнаружил `State.Running=false` и завершился
  `ok=4 changed=0 failed=1`. Для восстановления применяется `site.yml`.

Ansible 2.20 выводит deprecation warnings для обращения к facts и импортов
в закреплённых версиях коллекций. Эти предупреждения не меняют результаты
проверок и сохранены в исходных журналах практической работы.
