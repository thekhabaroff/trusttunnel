# trusttunnel.sh

Скрипт-установщик и менеджер для [TrustTunnel](https://github.com/TrustTunnel/TrustTunnel)
VPN-эндпоинта. Привязан к апстрим-версии **v1.0.33** и ребрендирован под
форк `thekhabaroff`.

Всё делает один bash-файл: ставит бинарь `trusttunnel_endpoint` через
апстрим-инсталлер, генерирует четыре TOML-конфига
(`vpn.toml`, `hosts.toml`, `credentials.toml`, `rules.toml`), поднимает
systemd-юнит `trusttunnel.service`, а при повторных запусках открывает
интерактивное меню управления.

## Быстрая установка

```shell
curl -fsSL https://raw.githubusercontent.com/thekhabaroff/trusttunnel/master/trusttunnel.sh \
  -o /root/trusttunnel.sh && chmod +x /root/trusttunnel.sh && sudo /root/trusttunnel.sh
```

После установки в `PATH` появляется шорткат `tt` → `/root/trusttunnel.sh`,
так что повторный заход в меню — одной командой:

```shell
tt
```

## Размещение

| Элемент | Путь |
| --- | --- |
| Каталог установки / конфигов | `/opt/trusttunnel` |
| systemd-юнит | `/etc/systemd/system/trusttunnel.service` |
| Маркер конфигурации | `/opt/trusttunnel/.trusttunnel_configured` |
| Скрипт-менеджер | `/root/trusttunnel.sh` |
| Шорткат в PATH | `/usr/local/bin/tt` → `/root/trusttunnel.sh` |
| Апстрим-инсталлер | <https://raw.githubusercontent.com/TrustTunnel/TrustTunnel/refs/heads/master/scripts/install.sh> |
| URL для self-update | <https://raw.githubusercontent.com/thekhabaroff/trusttunnel/master/trusttunnel.sh> |

## Что изменилось по сравнению с апстрим-скриптом `tt-installer`

Скрипт начинался от `deathline94/tt-installer/main/installer.sh` и
приведён в соответствие с TrustTunnel **v1.0.33**:

- **Пин версии эндпоинта** — `install_trusttunnel` вызывает апстрим
  инсталлер с `-V 1.0.33 -o /opt/trusttunnel`, чтобы версия бинаря
  совпадала со схемой конфигов, которые генерирует этот скрипт.
- **Let's Encrypt из коробки** — при выборе варианта 1 скрипт сам ставит
  `certbot` (apt/dnf/yum), выпускает сертификат через HTTP-01 на порту 80,
  прописывает в `hosts.toml` пути к live-симлинкам
  `/etc/letsencrypt/live/<domain>/{fullchain,privkey}.pem` и ставит
  deploy-хук `/etc/letsencrypt/renewal-hooks/deploy/trusttunnel.sh`,
  который автоматически перезапускает службу после каждого продления.
  Если выпуск не удался (DNS не смотрит сюда, `:80` занят или недоступен
  извне) — печатается точная команда для ручного восстановления, служба
  не стартует с пустыми ключами.
- **Диагностические хендлеры** (добавлены в TrustTunnel 1.0.17):
  в `vpn.toml` теперь пишутся `ping_enable`, `ping_path`,
  `speedtest_enable`, `speedtest_path`. Визард спрашивает перед включением,
  по умолчанию оба `false`.
- **HTTP-код при неудачной аутентификации** (1.0.17):
  `auth_failure_status_code = 407` пишется явно, поведение стабильно
  между апгрейдами.
- **Per-client лимиты соединений** (1.0.7): визард спрашивает глобальные
  значения `default_max_http2_conns_per_client` /
  `default_max_http3_conns_per_client`, а пункт меню «Add User» принимает
  индивидуальные `max_http2_conns` / `max_http3_conns`.
- **Правила `client_random_prefix`** (1.0.1 / 1.0.28):
    - `rules.toml` генерируется с комментариями, документирующими
      матчеры `cidr` / `client_random_prefix` (точное значение и
      битовый формат `prefix/mask`).
    - `show_client_config` вызывает
      `trusttunnel_endpoint -c USER -a ADDR --client-random-prefix`,
      чтобы эндпоинт сам сгенерировал prefix, добавил соответствующее
      правило `allow` в `rules.toml` и вшил значение в экспортируемый
      клиентский конфиг / deep-link. Есть fallback на старый вызов,
      если флаг не поддерживается.
- **Deep-link `tt://?...`** (1.0.13): баннер после установки
  показывает новый формат, клиенты также принимают легаси-форму `tt://`.
- **Переиспользование systemd-юнита**: если апстрим-инсталлер положил
  `trusttunnel.service.template`, `setup_systemd_service` копирует его
  как есть в `/etc/systemd/system/trusttunnel.service`, вместо того чтобы
  везти свою копию — любые изменения `ExecStart` между релизами
  подхватятся автоматически. Если шаблона нет — используется
  встроенный fallback-юнит.
- **Отображение статуса и конфигурации** теперь показывает тумблеры
  диагностических хендлеров, per-client лимиты, HTTP-код ошибки авторизации
  и реальную установленную версию бинаря (`trusttunnel_endpoint --version`).
- **Маркер-файл** хранит новые поля (`PING_ENABLED`,
  `SPEEDTEST_ENABLED`, `DEFAULT_MAX_HTTP2_CONNS`,
  `DEFAULT_MAX_HTTP3_CONNS`, `TRUSTTUNNEL_VERSION`). Значения пишутся
  через `printf %q`, поэтому имена пользователей и пароли с пробелами
  и метасимволами корректно восстанавливаются через `source`.
- **Ребрендинг и пути** под этот форк:
  `INSTALLER_URL`, `MANAGER_SCRIPT=/root/trusttunnel.sh`, шорткат `tt`,
  обновлённый баннер.

## Требования

- Linux, архитектура `x86_64` или `aarch64`.
- `root` (`sudo`).
- `bash`, `curl`, `openssl`, `systemctl`.
- Для варианта Let's Encrypt: открытый снаружи и свободный `:80/tcp`,
  DNS-запись, указывающая на этот сервер, корректный email.

## Меню управления

После первой установки повторный запуск (`tt` или
`bash /root/trusttunnel.sh`) открывает меню, учитывающее текущее
состояние сервиса:

| # | Действие |
| --- | --- |
| 1 | Старт сервиса |
| 2 | Стоп сервиса |
| 3 | Рестарт сервиса |
| 4 | Просмотр live `journalctl` логов |
| 5 | Показать статус |
| 6 | Редактировать один из четырёх TOML-файлов |
| 7 | Добавить пользователя (с опциональными per-user H2/H3 лимитами) |
| 8 | Показать клиентский конфиг / deep-link |
| 9 | Переустановить эндпоинт (конфиги сохраняются) |
| 10 | Полное удаление (сносит всё в `/opt/trusttunnel` и шорткат `tt`) |

## Лицензия

MIT, как и весь репозиторий.
