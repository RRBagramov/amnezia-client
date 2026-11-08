# AmneziaVPN с подключением к серверу по IPv6

Форк [amnezia-vpn/amnezia-client](https://github.com/amnezia-vpn/amnezia-client),
ветка `feature/ipv6-endpoint` от `dev`. Цель — подключаться к своему серверу
по его публичному IPv6-адресу, когда IPv4-адрес сервера заблокирован у
провайдера клиента, и управлять таким сервером из приложения.

Подробный аудит кода и план работ: [ipv6-implementation-plan.md](ipv6-implementation-plan.md).

## Зачем

Провайдеры и мобильные операторы блокируют IPv4-адреса зарубежных VPS. У
многих таких серверов есть и IPv6-адрес, который не заблокирован. В исходном
клиенте подключиться по нему нельзя:

- адрес сервера `host:port` делился по каждому `:`, поэтому IPv6 ломал ввод
  учётных данных SSH и импорт конфигов;
- адрес WireGuard/AmneziaWG всегда уходил демону как IPv4, и UAPI получал
  `endpoint=2001:db8::10:51820` без скобок;
- kill switch на macOS и Linux блокировал весь IPv6, включая сам сервер;
- выгруженные конфиги писали `Endpoint = 2001:db8::10:51820` без скобок.

## Что изменено

| Коммит | Что делает |
|---|---|
| `feat(core): parse host and port with IPv6 support` | Общий разбор `host`, `host:port`, `[IPv6]`, `[IPv6]:port` через `QHostAddress`/`QUrl` (`NetworkUtilities::parseHostPort`, `endpointHost`, `endpoint`). IPv6 без скобок никогда не делится на адрес и порт. Используется в окне SSH-подключения, `PayloadSender` и импорте WireGuard. |
| `fix(wireguard): connect to IPv6 endpoints` | Адрес сервера уходит демону в поле своего семейства; скобки в wg-quick (Windows), `WGConfig.swift` (iOS) и шаблонах выгрузки конфигов (`$SERVER_ENDPOINT_HOST`); поиск адаптера на Windows через `GetBestRoute2`. |
| `fix(killswitch): allow IPv6 VPN server` | macOS: правило `allownets` с `quick`. Linux: `110.allowNets` для обоих семейств и до `250.blockIPv6`, IPv6-адреса в `ip6tables`. Сервис: проверка IPv6/CIDR в списках адресов. |
| `fix(wireguard): pass IPv6 tunnel ranges and addresses with their family` | IPv6-диапазоны AllowedIPs помечаются как IPv6 (по умолчанию `/128`), `Address = v4, v6` делится на два адреса. |
| `docs: add IPv6 support audit and plan` | Аудит и план. |
| `fix(macos): build installer without a signing identity` | `.pkg` собирается без сертификата разработчика (раньше `pkgbuild` падал на пустом `--sign`). |
| `fix(killswitch): flush stale pf sub-anchors on reinstall` | Сервис при переустановке pf очищает все подъякоря `amn/*`, а не только корень. Без этого после обновления приложения в pf оставались старые правила, и новые файлы правил не загружались. |
| `fix(macos): ad-hoc sign binaries without a signing identity` | Без сертификата бинарники подписываются ad-hoc без hardened runtime. Без этого собранное приложение падало при запуске: `macdeployqt` портит подписи Qt. |
| `docs: describe the IPv6 fork` | Этот документ. |

Тесты: `client/tests/testHostPortParsing.cpp` (разбор адресов) и
`client/tests/testWireGuardEndpointImport.cpp` (импорт `.conf` с IPv6).

## Что проверено

- macOS (Apple Silicon), собранный `.pkg`: подключение AmneziaWG к серверу по
  `Endpoint = [IPv6]:port`, IPv4-трафик внутри туннеля, kill switch включён.
- macOS: добавление self-hosted сервера по `[IPv6]:port`, управление им по SSH
  через IPv6 при подключённом VPN, добавление пользователей.
- `test_host_port_parsing`: 43 из 43, собран отдельно против Qt.
- Docker на сервере публикует порт контейнера AmneziaWG и на IPv6-адресе хоста,
  без правки серверных скриптов (подтверждено подключением выше).

Не проверялось: сборки Windows и Linux (код для них не компилировался),
Android- и iOS-сборки этого форка, `test_wireguard_endpoint_import` (нужна
полная тестовая сборка), протоколы кроме AmneziaWG.

## Сборка на macOS

Нужны Xcode Command Line Tools, Homebrew и примерно 30 ГБ места: Conan при
первой настройке собирает нативные зависимости (десятки минут).

```bash
brew install cmake ninja python@3.14
python3.14 -m venv ~/amnezia-venv
~/amnezia-venv/bin/pip install conan aqtinstall
~/amnezia-venv/bin/aqt install-qt mac desktop 6.10.1 clang_64 \
  -m qtremoteobjects qt5compat qtshadertools qthttpserver qtwebsockets -O ~/Qt
```

```bash
git clone -b feature/ipv6-endpoint https://github.com/RRBagramov/amnezia-client.git
cd amnezia-client
PATH=~/amnezia-venv/bin:$PATH deploy/build.sh --generator Ninja --installer all
```

`deploy/build.sh` сам находит Qt в `~/Qt`. Результат —
`deploy/build/AmneziaVPN_<версия>_macos_x64.pkg` (универсальный arm64 + x86_64,
подпись ad-hoc).

Повторная сборка после правок:

```bash
cd deploy/build
cmake --build . --parallel
PATH=~/amnezia-venv/bin:$PATH cpack -G productbuild
```

Установка (системный сервис ставится вместе с приложением):

```bash
sudo installer -pkg deploy/build/AmneziaVPN_*_macos_x64.pkg -target /
```

После установки закройте приложение, которое запустил установщик, и откройте
его заново из «Программ» (см. «Грабли», п. 3).

Тесты:

```bash
cmake -S . -B build -DCMAKE_PREFIX_PATH=$HOME/Qt/6.10.1/macos -DAMNEZIA_BUILD_TESTS=ON
cmake --build build --parallel
ctest --test-dir build/client/tests -R "HostPortParsing|WireGuardEndpointImport" --output-on-failure
```

## Как подключиться по IPv6

1. Узнать IPv6 сервера: `ip -6 addr show scope global` на сервере.
2. Убедиться, что порт AmneziaWG слушается на IPv6: `ss -ulpn | grep <порт>`
   должен показать `[::]:<порт>`.
3. Добавить сервер в приложение по адресу `[2001:db8::10]:22` (порт SSH) и
   выдать новое подключение — в нём уже будет IPv6. Или в готовом `.conf`
   заменить `Endpoint` на `[2001:db8::10]:<порт>` и импортировать.

## Грабли

1. **Ошибка 302 в окне подключения к серверу.** IPv6 с портом вводится только
   в квадратных скобках: `[2001:db8::10]:2222`. Без скобок последнюю группу
   адреса невозможно отличить от порта.
2. **Ошибка 302 при добавлении пользователей с включённым kill switch
   (macOS).** pf не пускал SSH к IPv6-адресу сервера: в подъякорях `amn/*`
   оставались правила от прежней версии приложения. Исправлено в этом форке.
   На старой сборке: отключить VPN, `sudo pfctl -a amn/110.allowNets -F all`,
   подключиться снова. Проверка: `sudo pfctl -a amn/110.allowNets -s rules`
   должен показать `pass out quick`.
3. **Ошибка 1200 сразу после установки `.pkg`.** Скрипт установщика запускает
   приложение со своим окружением, в том числе `TMPDIR` во временной папке
   установщика, которую потом удаляет; временный файл для загрузки на сервер
   не создаётся. Закрыть приложение и открыть заново. Ошибка есть и в исходном
   проекте (`deploy/data/macos/post_install.sh`).
4. **Приложение падает при запуске (`Code Signature Invalid`).** Сборка без
   сертификата до исправления подписи. Пересобрать с текущей веткой.
5. **У клиента должен быть IPv6.** Подключение по IPv6-адресу сервера
   работает только из сети, где у устройства есть рабочий IPv6. Проверка —
   [test-ipv6.com](https://test-ipv6.com) без VPN. Ошибка
   `sendmsg: network is unreachable` в логе AmneziaWG означает, что у сети
   устройства нет IPv6-маршрута. У мобильных операторов IPv6 часто нет или он
   включается в настройках APN (тип протокола IPv4/IPv6).
6. **fail2ban на сервере.** Если в `ignoreip` указан только домашний IPv4, то
   при управлении по IPv6 неудачные входы банят IPv6-адрес. Добавьте в
   `ignoreip` и свой IPv6-префикс; у домашних провайдеров адреса часто
   динамические.
7. **Отдельное приложение AmneziaWG** (`org.amnezia.awg`) — другой проект,
   изменения этого форка его не касаются.

## Ограничения

- **Сеть клиента без IPv6** (многие мобильные операторы) не достучится до
  сервера по IPv6. Если IPv4-адрес сервера заблокирован везде, нужен
  промежуточный сервер в доступной сети, пересылающий порт на основной сервер,
  или новый IPv4-адрес.
- **XRay на десктопе** через IPv6-сервер не работает: `StopRoutingIpv6`
  отключает IPv6 в системе (Linux) или заворачивает его в loopback (Windows).
  Для IPv6 используйте AmneziaWG.
- **IPv6 внутри туннеля при включённом kill switch** (macOS, Linux)
  блокируется целиком, кроме адреса сервера: утечек нет, но и IPv6 через VPN
  нет.
- **Свой сервер Amnezia не выдаёт клиентам IPv6 в туннеле**, IPv6-трафик через
  VPN возможен только с конфигами, где это настроено на сервере.
- **Сервер только с IPv6** (без IPv4) не поддерживается: серверные скрипты и
  сети Docker работают с IPv4.
- **DNS-серверы IPv6** в импортируемых конфигах (`DNS =`) не распознаются.
- **Подсказка в окне подключения** по-прежнему показывает пример с IPv4.
- **Windows, Linux, Android и iOS** из этого форка не собирались и не
  проверялись.
