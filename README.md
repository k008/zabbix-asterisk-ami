# Zabbix мониторинг Asterisk через WEB AMI
Автор: Станислав Ярошевский <1@3sla.ru>

Данный шаблон позволяет без установки дополнительных скриптов и ПО отслеживать состояние SIP регистраций и состояние пиров.

При отслеживании состояний SIP регистраций существует 3 значения, которые сохраняются в Zabbix:
- 1 - SIP успешно зарегистрирован
- 0 - SIP умеет статус "Unregistered"
- -1 - Прочие статусы SIP регистрации, например "Auth. Sent"

Для отслеживания состояний SIP регистраций включен триггер.

При отслеживании состояний Peers заббикс собирает 2 метрики:
- Первая метрика это состояние самого пира
- Вторая метрика, логирует время отклика SIP устройства в 
миллисекундах. Если статус устройства отличный от "OK", то принимается 
значение "-1". Значение говорит о том, что устройство не используется 
или потеряло связь с АТС

## Настройка Asterisk
В файл /etc/asterisk/manager.conf, в секцию [general] нужно добавить:
```
[general]
...
webenabled = yes
httptimeout = 600 ;Время жизни сессии
...
```
Данные параметры не перезаписываются FreePBX

Если у Вас FreePBX, нужно включить mini-HTTP server, опции Enable TLS for the mini-HTTP Server и Enable TLS for the mini-HTTP Server в Settings->Advanced Settings 

Если у Вас голый Asterisk, то в файл /etc/asterisk/http.conf, в секцию [general] нужно добавить: 
```
[general]
enabled=no
enablestatic=no
bindaddr=::
bindport=8088
prefix=
sessionlimit=100
session_inactivity=30000
session_keep_alive=15000
tlsenable=yes
tlsbindaddr=[::]:8089
tlscertfile=/etc/asterisk/keys/integration/certificate.pem
tlsprivatekey=/etc/asterisk/keys/integration/webserver.key
```

Теперь нужно создать пользователя для AMI. 

Если у Вас FreePBX, идем Settings->Asterisk Manager Users и добавляем пользователя с правами reporting на чтение и запись
Если у Вас голый Asterisk, то в файл /etc/asterisk/manager.conf нужно добавить следующие строки.
```
...
[zabbix] ;соответствует логину
secret = zabbix ;пароль
deny=0.0.0.0/0.0.0.0
permit=127.0.0.1/255.255.255.0
permit= 192.168.99.44/255.255.255.255 ;откуда разрешен доступ
read = reporting ;права на чтение
write = reporting ;права на запись
...
```
Не забываем указать, из каких сетей разрешен доступ.
После этого, применяем настройки и перезагружаем АТС.Настройки самой АТС завершены.

## Настройка хоста в Zabbix
После того, как хост АТС был добавлен в Zabbix, назначаем на него шаблон "Template Asterisk AMI"
В шаблоне есть следующие макросы, который при необходимости можно менять на уровне хоста:
```
{$ZABBIX_API_URL} — путь до API интерфейса. В зависимости от архитектуры обращение к вебу может идти по другому URL. В той ситуации когда Zabbix-server и zabbix-frontend находятся на одном сервере и включен SSL путь такой: https://localhost/zabbix/api_jsonrpc.php
{$ASTERISK_AMI_PORT} — порт. Для HTTP по умолчанию используется 8088, для HTTPS - 8089
{$ASTERISK_AMI_PROTOCOL} — протокол, в зависимости от вашего выбора. Либо http, либо https
{$ASTERISK_AMI_SECRET} -  пароль, который будет использоваться для авторизации в AMI
{$ASTERISK_AMI_USERNAME} — логин, который будет использоваться для авторизации в AMI
{$ZABBIX_API_LOGIN} - логин учетной запись в Заббиксе, через которую можно обновить макрос.
{$ZABBIX_API_PASSWORD} - пароль запись в Заббиксе, через которую можно обновить макрос.
```
ВАЖНО: Проверяйте пару ЛОГИН/ПАРОЛЬ для API Zabbix можно авторизоваться в вебе и там отображается нужны нам хост.

Теперь необходимо в ХОСТЕ АТС создать 2 макроса:
- {$HOST_ID} — и внести в него ID данного хоста. ID можно увидеть в строке URL браузера.
- {$ASTERISK_AMI_COOKIE} - оставить пустым.

После всех этих операция назначаем наш шаблон на хост. Если все было  сделано правильно, то в макросе {$ASTERISK_AMI_COOKIE} должен появиться идентификатор сессии. Если где-то проблема, об этом сообщат триггеры.


