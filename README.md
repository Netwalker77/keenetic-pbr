# keenetic-pbr

![workflow status](https://img.shields.io/github/actions/workflow/status/maksimkurb/keenetic-pbr/.github%2Fworkflows%2Fbuild-ci.yml?branch=main)
![release](https://img.shields.io/github/v/release/maksimkurb/keenetic-pbr?sort=date)

> **keenetic-pbr** не является официальным продуктом компании **Keenetic** и никак с ней не связан. Этот пакет создан сторонним разработчиком и предоставляется "как есть" без какой-либо гарантии. Все вопросы и предложения касательно пакета можно направлять в GitHub Issue и в Telegram-чат: https://t.me/keenetic_pbr.

#### [> README in English <](./README.en.md)

keenetic-pbr — это пакет для маршрутизации на основе правил для роутеров Keenetic.

Telegram-чат проекта: https://t.me/keenetic_pbr

С помощью этого пакета можно настроить выборочную маршрутизацию для указанных IP-адресов, подсетей и доменов. Это необходимо, если вам понадобилось организовать защищенный доступ к определенным ресурсам, либо выборочно разделить трафик на несколько провайдеров (напр. трафик до сайта А идёт через одного оператора, а остальной трафик - через другого)

Пакет использует `ipset` для того, чтобы хранить большое количество адресов в памяти роутера без существенного увеличения нагрузки, а также `dnsmasq` для того, чтобы пополнять данный `ipset` IP-адресами, которые резолвят клиенты локальной сети.

Для настройки маршрутизации пакет создает скрипты в директории `/opt/etc/ndm/netfilter.d` и `/opt/etc/ndm/ifstatechanged.d`.

## Особенности

- Маршрутизация на основе доменов через `dnsmasq`
- Маршрутизация на основе IP-адресов через `ipset`
- Настраиваемые таблицы маршрутизации и приоритеты
- Автоматическая настройка для списков `dnsmasq`

## Принцип работы

Данный пакет содержит следующие скрипты и утилиты:
```
/opt
├── /usr
│   └── /bin
│       └── keenetic-pbr                    # Утилита для скачивания и обработки списков, их импорта в ipset, а также генерации файлов конфигурации для dnsmasq
└── /etc
    ├── /keenetic-pbr
    │   ├── /keenetic-pbr.conf              # Файл конфигурации keenetic-pbr
    │   └── /lists.d                        # В эту папку keenetic-pbr будет помещать скачанные и локальные списки. Не кладите сюда ничего сами, т.к. файлы из этой папки удаляются после каждого запуска команды "keenetic-pbr download".
    ├── /ndm
    │   ├── /netfilter.d
    │   │   └── 50-keenetic-pbr-fwmarks.sh  # Скрипт добавляет iptables правило для маркировки пакетов в ipset с определённым fwmark
    │   └── /ifstatechanged.d
    │       └── 50-keenetic-pbr-routing.sh  # Скрипт добавляет ip rule для направления пакетов с fwmark в нужную таблицу маршрутизации и создаёт её с нужным default gateway
    ├── /cron.daily
    │   └── 50-keenetic-pbr-lists-update.sh # Скрипт для автоматического ежедневного обновления списков
    └── /dnsmasq.d
        └── (config files)                  # Папка с сгенерированными конфигурациями для dnsmasq, заставляющими его класть IP-адреса доменов из списков в нужный ipset
```

### Маршрутизация пакетов на основе IP-адресов и подсетей
**keenetic-pbr** автоматически загружает ip-адреса и подсети из списков в нужные `ipset`. Далее пакеты на IP-адреса, которые попадают в этот `ipset`, маркируются определённым `fwmark` и на основе правил маршрутизации переадресовываются на конкретный интерфейс.

**Схема процесса:**
![IP routing scheme](./.github/docs/ip-routing.svg)

### Маршрутизация пакетов на основе доменов
Для маршрутизации на основе доменов используется `dnsmasq`. Каждый раз, когда клиенты локальной сети делают DNS-запрос, `dnsmasq` проверяет, есть ли домен в списках, и если есть, то добавляет его ip-адреса в `ipset`.

> [!NOTE]  
> Чтобы маршрутизация доменов работала, клиентские устройства не должны использовать собственные DNS-сервера. Их DNS-сервером должен быть IP роутера, иначе `dnsmasq` не увидит эти пакеты и не добавит ip-адреса в нужный `ipset`.

> [!IMPORTANT]  
> Некоторые приложения и игрушки используют собственные способы получения ip-адресов для своих серверов. Для таких приложений маршрутизация по доменам не будет работать, т.к. эти приложения не делают DNS-запросов. Вам придётся узнавать IP-адреса/подсети серверов этих приложений и добавлять их в списки самостоятельно. 

**Схема процесса:**
![Domain routing scheme](./.github/docs/domain-routing.svg)


## Предварительная подготовка роутера
1. Работоспособность пакета проверялась на **Keenetic OS** версии **4.2.1**. Работоспособность на версии **3.x.x** возможна, но не гарантируется.
2. Необходимо установить дополнительные компоненты на ваш роутер в разделе Управление -> Параметры системы:
   - **Сетевые функции / Протокол IPv6**
     - Этот компонент необходим для возможности установки компонента "Модули ядра подсистемы Netfilter". 
   - **Пакеты OPKG / Поддержка открытых пакетов**
   - **Пакеты OPKG / Модули ядра подсистемы Netfilter**
   - **Пакеты OPKG / Пакет расширения Xtables-addons для Netfilter**
     - На данный момент этот пакет не обязателен, поскольку его возможности не используются keenetic-pbr, но его возможности могут пригодиться в будущем. Инструкция к модулю [доступна по ссылке](https://manpages.ubuntu.com/manpages/trusty/en/man8/xtables-addons.8.html).
3. Вам необходимо установить среду Entware на Keenetic ([инструкция](https://help.keenetic.com/hc/ru/articles/360021214160-%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-%D1%81%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D1%8B-%D0%BF%D0%B0%D0%BA%D0%B5%D1%82%D0%BE%D0%B2-%D1%80%D0%B5%D0%BF%D0%BE%D0%B7%D0%B8%D1%82%D0%BE%D1%80%D0%B8%D1%8F-Entware-%D0%BD%D0%B0-USB-%D0%BD%D0%B0%D0%BA%D0%BE%D0%BF%D0%B8%D1%82%D0%B5%D0%BB%D1%8C)), для этого понадобится USB-накопитель, который будет постоянно вставлен в роутер
4. Также необходимо настроить второе (третье, четвёртое, ...) соединение, через которое вы хотите направить трафик попадающий под списки. Это может быть VPN-соединение или второй провайдер (multi-WAN).

## Установка

### Автоматическая установка

Подключитесь к вашему OPKG по SSH и выполните следующую команду:

```bash
opkg install curl jq && curl -sOfL https://raw.githubusercontent.com/maksimkurb/keenetic-pbr/refs/heads/main/install.sh && sh install.sh
```

### Ручная установка

1. Перейдите на страницу [релизов](https://github.com/maksimkurb/keenetic-pbr/releases) и скопируйте URL для последнего `.ipk` файла
   для вашей архитектуры.

2. Скачайте `.ipk` файл на ваш маршрутизатор:
   ```bash
   curl -LO <URL-to-latest-ipk-file-for-your-architecture>
   ```

3. Установите его с помощью OPKG:

   ```bash
   opkg install keenetic-pbr-*-entware.ipk
   ```

Во время установки пакет `keenetic-pbr` заменяет оригинальный файл конфигурации **dnsmasq**.
Резервная копия будет сохранена в `/opt/etc/dnsmasq.conf.orig`.

## Настройка

Отредактируйте конфигурацию в следующих файлах в соответствии с вашими потребностями:

- **(обязательно) Конфигурация keenetic-pbr:** `/opt/etc/keenetic-pbr/keenetic-pbr.conf`
  - В данном файле вы должны настроить необходимые ipset, списки и выходные интерфейсы
- **(опционально) Конфигурация dnsmasq:** `/opt/etc/dnsmasq.conf`
   - Данный файл можно перенастроить под свои нужды, например заменить upstream DNS сервер на свой
   - Рекомендуется поставить и настроить пакет `dnscrypt-proxy2`, а затем указать в `dnsmasq.conf` настройку `server=127.0.0.1#9153`, чтобы DNS-запросы были защищены DNS-over-HTTPS (DoH)

### 1. Редактирование `keenetic-pbr.conf`

Откройте `/opt/etc/keenetic-pbr/keenetic-pbr.conf` и отредактируйте его по мере необходимости:

1. Необходимо поправить поле `interface`, указав туда интерфейс, через который будет идти исходящий трафик, попавший под критерии списков.
2. Также необходимо добавить списки (локальный или удалённый по URL)

**Пример конфигурации:**
```toml
#---------------------#
#   Общие настройки   #
#---------------------#
[general]
ipset_path = "ipset"                                 # Путь к бинарному файлу `ipset`
lists_output_dir = "/opt/etc/keenetic-pbr/lists.d"   # В эту папку будут скачиваться списки
dnsmasq_lists_dir = "/opt/etc/dnsmasq.d"             # Загруженные списки будут сохранены в этом каталоге для dnsmasq
summarize = true                                     # Если true, keenetic-pbr будет суммировать IP-адреса и CIDR перед применением к ipset

#-------------#
#   IPSET 1   #
#-------------#
[[ipset]]
ipset_name = "vpn"              # Название ipset
flush_before_applying = true    # Очищать ipset каждый раз перед его заполнением

   [ipset.routing]
   interface = "nwg1"   # Куда будет направляться трафик для IP, попавших в этот ipset
   fwmark = 1001        # Этот fwmark будет применяться к пакетам, попавшим под критерии списков
   table = 1001         # Номер таблицы маршрутизации (ip route table), туда будет добавляться default gateway на интерфейс, указанный выше
   priority = 1001      # Приоритет правила маршрутизации (ip rule priority), чем число меньше, тем выше приоритет
   
   # Список 1 (ручное перечисление адресов)
   [[ipset.list]]
   name = "local"
   hosts = [
       "ifconfig.co",
       "myip2.ru",
       "1.2.3.4",
       "141.201.11.0/24",
   ]

   # Список 2 (скачивание через URL)
   [[ipset.list]]
   name = "remote-list-1"
   url = "https://some-url/list1.lst"  # Файл должен содержать домены, IP адреса и CIDR, по одному на каждой строчке

    # Список 3 (скачивание через URL)
   [[ipset.list]]
   name = "remote-list-2"
   url = "https://some-url/list2.lst"

# Вы можете добавлять столько ipset, сколько хотите:
# [[ipset]]
# ipset_name = "direct"
# ...
```

### 2. Скачивание и обработка списков

После редактирования конфигурационного файла введите данную команду, чтобы скачать файлы списков:

```bash
keenetic-pbr download
```

> [!IMPORTANT]  
> Эту команду необходимо запустить даже в том случае, если все списки являются локальными. Данная команда экспортирует все списки из настроек в папку `lists.d`.

### 3. Настройка DNS over HTTPS (DoH)
> [!TIP]  
> Обычный протокол DNS является не безопасным, поскольку все запросы передаются в открытом виде.
> Это значит, что провайдер или злоумышленники могут перехватить и подменить ваши DNS-запросы ([DNS spoofing](https://ru.wikipedia.org/wiki/DNS_spoofing)), направив вас на ненастоящий веб-сайт.
> 
> Чтобы обезопасить себя от этого, рекомендуется настроить пакет `dnscrypt-proxy2`, который будет использовать протокол **DNS-over-HTTPS** (**DoH**) для шифрования DNS-запросов.
> Подробнее о **DoH** [можно прочитать здесь](https://adguard-dns.io/ru/blog/adguard-dns-announcement.html).

Для настройки **DoH** на роутере необходимо выполнить следующие действия:
1. Скачиваем `dnscrypt-proxy2`
    ```bash
    keenetic-pbr download
    ```
2. Редактируем файл `/opt/etc/dnscrypt-proxy.toml`
    ```ini
   # ...
   
   # Указываем upstream-серверы (необходимо убрать решётку перед server_names)
   server_names = ['adguard-dns-doh', 'cloudflare-security', 'google']
   
   # Указываем порт 9153 для прослушивания DNS-запросов
   listen_addresses = ['[::]:9153']
   
   # ... 
    ```
3. Редактируем файл `/opt/etc/dnsmasq.conf`
    ```ini
   # ...
   
   # Меняем сервер по умолчанию 8.8.8.8 на наш dnscrypt-proxy2
   server=127.0.0.1#9153
   
   # ...
    ```

4. Перезапускаем `dnscrypt-proxy2` и `dnsmasq`
    ```bash
   /opt/etc/init.d/S09dnscrypt-proxy2 restart
   /opt/etc/init.d/S56dnsmasq restart
   ```

### 4. Включение DNS Override

Для того, чтобы Keenetic использовал `dnsmasq` в качестве DNS-сервера, необходимо включить DNS-Override.

> [!NOTE]  
> Данный этап не нужен, если ваши списки содержат только IP-адреса и CIDR и не указывают доменных имён.


1. Откройте следующий URL в браузере:
   ```
   http://<router-ip-address>/a
   ```
2. Введите следующие команды по очереди:
   - `opkg dns-override`
   - `system configuration save`

> [!TIP]
> Если вы решите отключить DNS-Override в будущем, выполните команды `no opkg dns-override` и `system configuration save`.

### 5. Перезапуск OPKG и проверка работы маршрутизации

Перезапустите OPKG и убедитесь, что маршрутизация на основе политики работает должным образом.

Для этого откройте адрес, которого нет в ваших списках (напр. https://2ip.ru) и адрес, который есть в ваших списках (напр. https://ifconfig.co) и сравните IP-адреса, они должны быть разными.

## Обновление списков

Списки обновляются автоматически ежедневно с помощью `cron`.

В случае, если вы редактировали настройки `keenetic-pbr.conf` и хотите обновить списки вручную, выполните следующие команды по SSH:

```bash
keenetic-pbr download
/opt/etc/init.d/S80keenetic-pbr restart
```

## Устранение неполадок

Если возникают проблемы, проверьте ваши конфигурационные файлы и логи.
Убедитесь, что списки были загружены правильно, и что `dnsmasq` работает с обновленной конфигурацией.

С вопросами можно обращаться в Telegram-чат проекта: https://t.me/keenetic_pbr

---

Приятного использования маршрутизации на основе политики с Keenetic-PBR!
