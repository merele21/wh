# <p align=center>Сканирование сети</p>

## Ping-сканирование

**_Основная задача_** — обнаружить «живые» узлы в сети. Под ping-сканированием понимают широковещательную рассылку пакетов **_ICMP_**. Сканер рассылает **пакеты типа Echo REQUEST** по указанным IP-адресам и ожидает в ответ пакеты типа **Echo REPLY**. Если ответ получен, считается, что узел присутствует в сети по указанному **IP-адресу**.

**_Протокол ICMP_** широко используется администраторами сетей для диагностики, поэтому, чтобы избежать разглашения информации об узлах, важна корректная настройка средств защиты периметра. Для корпоративных сетей такой вид сканирования не релевантен при внешнем сканировании, потому что большинство средств защиты по умолчанию блокируют протокол ICMP либо ответы по этому протоколу. При отсутствии нестандартных задач в корпоративной сети на выход, как правило, разрешены следующие виды **_`ICMP-сообщений: Destination Unreachable, Echo REQUEST, Bad IP header`_**, а на вход разрешены **_`Echo REPLY, Destination Unreachable, Source Quench, Time Exceeded, Bad IP header.`_** В локальных сетях не такая строгая политика безопасности, и злоумышленники могут применять этот способ, когда уже проникли в сеть, однако это легко детектируется.

## Сканирование портов

Объединим **_TCP-сканирование_** и **_UDP-сканирование_** под общим названием — сканирование портов. Сканирование этими методами определяет доступные порты на узлах, а затем на основе полученных данных делается предположение о типе используемой операционной системы или конкретного приложения, запущенного на конечном узле. Под сканированием портов понимают пробные попытки подключения к внешним узлам.<br>

### Рассмотрим основные методы, реализованные в автоматизированных сетевых сканерах:

**1. TCP SYN**

- **_Метод TCP SYN_** — наиболее популярен, используется в 95% случаев. Его называют сканированием с установкой полуоткрытого соединения, так как соединение не устанавливается до конца. На исследуемый порт посылается `сообщение SYN`, затем идет ожидание ответа, на основании которого определяется статус порта. `Ответы SYN/ACK` говорят о том, что порт прослушивается (открыт), а `ответ RST` говорит о том, что не прослушивается.<br>
  Если после нескольких запросов не приходит никакого ответа, то сетевой трафик до порта узла назначения фильтруется средствами межсетевого экранирования (далее будем использовать термин «порт фильтруется»). Также порт помечается как фильтруемый, если в ответ приходит `сообщение ICMP с ошибкой достижимости (Destination Unreachable)` и определенными кодами и флагами.

**2. TCP CONNECT**

- **_Метод TCP CONNECT_** менее популярен, чем TCP SYN, но все-таки часто встречается на практике. При реализации метода TCP CONNECT производится попытка установить соединение по протоколу TCP к нужному порту с процедурой `handshake`. Процедура заключается в обмене сообщениями для согласования параметров соединения, то есть служебными сообщениями `SYN, SYN/ACK, ACK, между узлами.` Соединение устанавливается на уровне операционной системы, поэтому существует шанс, что оно будет заблокировано средством защиты и попадет в журнал событий.

**3. UDP scan**

- **_UDP-сканирование_** медленнее и сложнее, чем TCP-сканирование. Из-за специфики сканирования UDP-портов о них часто забывают, ведь **_полное время сканирование 65 535 UDP-портов со стандартными параметрами на один узел занимает у большинства автоматизированных сканеров до 18 часов. Это время можно уменьшить за счет распараллеливания процесса сканирования и рядом других способов._** Следует уделять внимание поиску UDP-служб, потому что UDP-службы реализуют обмен данными с большим числом инфраструктурных сервисов, которые, как правило, вызывают интерес злоумышленников.

На сетевых периметрах часто встречаются UDP-сервисы **`DNS` (53), `NTP` (123), `SNMP` (161), `VPN` (500, 1194, 4500), `RDG` (3391)**. Реже встречаются сервисные службы типа **`echo` (7), `discard` (9), `chargen` (19)**, а также **`DAYTIME` (13), `TFTP` (69), `SIP` (5060)**, сервисы **`NFS` (2049), `RPC` (111, 137-139, 761 и др.), `СУБД` (1434).**

Для определения статуса порта посылается пустой `UDP-заголовок`, и если в ответ приходит ошибка достижимости `ICMP Destination Unreachable` с кодом **`Destination port unreachable`**, это значит, что порт закрыт; другие ошибки достижимости `ICMP (Destination host unreachable, Destination protocol unreachable, Network administratively prohibited, Host administratively prohibited, Communication administratively prohibited)` означают, что порт фильтруется. Если порт отвечает `UDP-пакетом`, значит, он открыт. Из-за специфики UDP и потери пакетов запросы повторяются несколько раз, обычно три и более. Как правило, если ответ не получен, статус порта определяется в состоянии «открыт» или «фильтруется», поскольку непонятно, что стало причиной — блокировка трафика средством защиты или потеря пакетов.

Для точности определения статуса порта и самой службы, запущенной на `UDP-порте`, используется специальная полезная нагрузка, наличие которой должно вызвать определенную реакцию у исследуемого приложения.

## Редкие методы сканирования

### Методы, которые практически не используются:

**1. TCP ACK**

- **_Прямое назначение метода ACK-сканирования_** — выявить правила средств защиты, а также определить фильтруемые порты. В пакете запроса при таком типе сканирования установлен только `ACK-флаг.` Открытые и закрытые порты вернут `RST-пакет`, так как порты достижимы для `ACK-пакетов`, но состояние неизвестно. Порты, которые не отвечают или посылают в ответ `ICMP-сообщение Destination Unreachable` с определенными кодами считаются фильтруемыми.

**2. TCP NULL, FIN, Xmas**

- **_Методы TCP NULL, FIN, Xmas_** заключаются в отправке пакетов с отключенными флагами в заголовке `TCP`. При `NULL-сканировании` не устанавливаются никакие биты, при `FIN-сканировании` устанавливается бит `TCP FIN`, а в `Xmas-сканировании` устанавливаются флаги **FIN, PSH и URG**. Методы основаны на особенности спецификации `RFC 793`, согласно которой при закрытом порте входящий сегмент, не содержащий `RST`, повлечет за собой отправку `RST` в ответ. `Когда порт открыт, ответа не будет`. **`Ошибка достижимости ICMP означает, что порт фильтруется.`** Эти методы считаются более скрытными, чем `SYN-сканирование`, однако и менее точны, потому что не все системы придерживаются `RFC 793.`

**3. «Ленивое сканирование»**

- **_«Ленивое сканирование»_** является самым скрытным из методов, поскольку для сканирования используется `другой узел сети, который называется зомби-узлом`. Метод применяется злоумышленниками для разведки. Преимущество такого сканирования в том, что статус портов определяется для зомби-узла, поэтому, используя разные узлы, можно установить доверительные связи между узлами сети. Полное описание метода доступно по **[ссылке][1].**

## Процесс выявления уязвимостей

Под уязвимостью будем понимать слабое место узла в целом или его отдельных программных компонентов, которое может быть использовано для реализации атаки. В стандартной ситуации наличие уязвимостей объясняется ошибками в программном коде или используемой библиотеке, а также ошибками конфигурации.

Уязвимость регистрируется в `MITRE CVE`, а подробности публикуются в `NVD`. Уязвимости присваивается идентификатор `CVE`, а также общий балл системы оценки уязвимости `CVSS`, отражающий уровень риска, который уязвимость представляет для конечной системы. Подробно об оценке уязвимостей написано **[here][2].**<br>
**_Централизованный список MITRE CVE_** — ориентир для сканеров уязвимостей, ведь задача сканирования — обнаружить уязвимое программное обеспечение.

**_Ошибка конфигурации_** — тоже уязвимость, но подобные уязвимости нечасто попадают в базу `MITRE`; впрочем, они все равно попадают в базы знаний сканеров с внутренними идентификаторами. В базы знаний сканеров попадают и другие типы уязвимостей, которых нет в `MITRE CVE`, поэтому при выборе инструмента для сканирования важно обращать внимание на экспертизу его разработчика. Сканер уязвимостей будет опрашивать узлы и сравнивать собранную информацию с базой данных уязвимостей или списком известных уязвимостей. **Чем больше информации у сканера, тем точнее результат.**

## Параметры сканирования

За месяц периметр организации может неоднократно поменяться. Проводя сканирование периметра в лоб можно затратить время, за которое результаты станут нерелевантными. При сильном увеличении скорости сканирования сервисы могут «упасть». Надо найти баланс и правильно выбрать параметры сканирования. От выбора зависят потраченное время, точность и релевантность результатов. Всего можно сканировать **`65 535 TCP-портов`** и столько же `UDP-портов`. По нашему опыту, среднестатистический периметр компании, который попадает в пул сканирования, `составляет две полных сети класса «С» с маской 24.`

### Основные параметры:

**1. Количество портов**

По количеству портов сканирование можно разделить на три вида — `сканирование по всему списку TCP- и UDP-портов, сканирование по всему списку TCP-портов и популярных UDP-портов, сканирование популярных TCP- и UDP-портов.`<br>
Как определить популярности порта? В утилите `nmap` на основе статистики, которую собирает разработчик утилиты, тысяча наиболее популярных портов определена в конфигурационном файле. Коммерческие сканеры также имеют преднастроенные профили, включающие `до 3500 портов.`

Если в сети используются сервисы на нестандартных портах, их также стоит добавить в список сканируемых.<br>
**_Для регулярного сканирования_** мы рекомендуем использовать средний вариант, при котором `сканируются все TCP-порты и популярные UDP-порты.` Такой вариант `наиболее сбалансирован по времени и точности результатов.` При проведении тестирования на проникновение или полного аудита сетевого периметра `рекомендуется сканировать все TCP- и UDP-порты.`

> Важная ремарка: не получится увидеть реальную картину периметра, сканируя из локальной сети, потому что на сканер будут действовать правила межсетевых экранов для трафика из внутренней сети. `Сканирование периметра` необходимо проводить с одной или нескольких внешних площадок; в использовании разных площадок есть смысл, только если они расположены в разных странах.

**2. Глубина сканирования**

Под глубиной сканирования подразумевается количество данных, которые собираются о цели сканирования. Сюда входит операционная система, версии программного обеспечения, информация об используемой криптографии по различным протоколам, информация о веб-приложениях. При этом имеется прямая зависимость: чем больше хотим узнать, тем дольше сканер будет работать и собирать информацию об узлах.

**3. Скорость сканирования**

При выборе скорости необходимо руководствоваться пропускной способностью канала, с которого происходит сканирование, пропускной способностью канала, который сканируется, и возможностями сканера. Существуют пороговые значения, превышение которых не позволяет гарантировать точность результатов, сохранение работоспособности сканируемых узлов и отдельных служб. Не забывайте учитывать время, за которое необходимо успеть провести сканирование.

**4. Параметры определения уязвимостей**

**_Параметры определения уязвимостей_** — наиболее обширный раздел параметров сканирования, от которого зависит скорость сканирования и объем уязвимостей, которые могут быть обнаружены. Например, баннерные проверки не займут много времени. Имитации атак будут проведены только для отдельных сервисов и тоже не займут много времени. `**Самый долгий вид** — веб-сканирование.`

> **_Полное сканирование_** сотни веб-приложений может длиться неделями, так как зависит от используемых словарей и количества входных точек приложения, которые необходимо проверить. **Важно понимать**, что из-за особенностей реализации веб-модулей и веб-сканеров инструментальная проверка веб-уязвимостей `не даст стопроцентной точности, но может очень сильно замедлить весь процесс.` `Веб-сканирование лучше проводить отдельно от регулярного`, тщательно выбирая приложения для проверки.

> **_Для глубокого анализа_** использовать инструменты статического и динамического анализа приложений или услуги тестирования на проникновение. Мы не рекомендуем использовать опасные проверки при проведении регулярного сканирования, поскольку существует риск нарушения работоспособности сервисов.

## Инструментарий

Каждый из инструментов сканирования служит своей цели, поэтому при выборе инструмента должно быть понимание, зачем он используется. Иногда правильно применять несколько сканеров для получения полных и точных результатов.

**_Сетевые сканеры:_** `Masscan , Zmap , nmap`. На самом деле утилит для сканирования сети намного больше, однако для сканирования периметра вряд ли вам понадобятся другие. Эти утилиты позволяют решить большинство задач, связанных со сканированием портов и служб.

**_Поисковики по интернету вещей, или онлайн-сканеры_** — важные инструменты для сбора информации об интернете в целом. Они предоставляют сводку о принадлежности узлов к организации, сведения о сертификатах, активных службах и иную информацию. С разработчиками этого типа сканеров можно договориться об исключении ваших ресурсов из списка сканирования или о сохранении информации о ресурсах только для корпоративного пользования. **_Наиболее известные поисковики:_** `Shodan , Censys , Fofa`.

Для решения задачи не обязательно применять сложный коммерческий инструмент с большим числом проверок: это излишне для сканирования пары «легких» приложений и сервисов. **_Наиболее известные:_** `Skipfish , Nikto , ZAP , Acunetix , SQLmap`.

**_При тщательном ручном анализе будут полезны инструменты_** `Burp Suite, Metasploit и OpenVAS`. Недавно вышел `сканер Tsunami компании Google`.

> Отдельной строкой стоит упомянуть об `онлайн-поисковике уязвимостей Vulners`. `Это большая база данных контента информационной безопасности`, где собирается информация об уязвимостях с большого количества источников, куда, кроме типовых баз, входят вендорские бюллетени безопасности, программы `bug bounty` и другие тематические ресурсы. Ресурс предоставляет `API`, через который можно `забирать результаты`, поэтому можно `реализовать баннерные проверки своих систем без фактического сканирования здесь и сейчас`. Либо использовать `Vulners vulnerability scanner`, `который будет собирать информацию об операционной системе, установленных пакетах и проверять уязвимости через API Vulners.`

**_Коммерческие сканеры уязвимостей_**

Все коммерческие системы защиты поддерживают основные режимы сканирования, интеграцию с различными внешними системами, такими как `SIEM-системы, patch management systems, CMBD, системы тикетов.` Коммерческие сканеры могут присылать оповещения по разным критериям, а поддерживают различные форматы и типы отчетов. Все разработчики систем сканирования используют общие базы уязвимостей, а также собственные базы знаний, которые постоянно обновляются на основе исследований.

> Основные различия между коммерческими сканерами — `поддерживаемые стандарты, лицензии государственных структур, количество и качество реализованных проверок, а также направленность на тот или иной рынок сбыта, например поддержка сканирования отечественного ПО`. У каждого сканера есть свои преимущества и недостатки. Для анализа защищенности подходят все перечисленные средства, **можно использовать их комбинации:** `Qualys , Max Patrol 8 , Tenable SecurityCenter.`

## Как работают сканеры уязвимостей

### Режимы сканирования реализованы по трем схожим принципам:

**- Аудит, или режим белого ящика**

**_Аудит_** — `режим белого ящика`, который позволяет провести `полную инвентаризацию сети, обнаружить все ПО, определить его версии и параметры и на основе этого сделать выводы об уязвимости систем на детальном уровне, а также проверить системы на использование слабых паролей.` Процесс сканирования требует `определенной степени интеграции с корпоративной сетью`, в частности `необходимы учетные записи для авторизации на узлах.`
**Авторизованному пользователю**, в роли которого выступает `сканер`, значительно проще получать детальную информацию об узле, его программном обеспечении и конфигурационных параметрах. При сканировании используются различные механизмы и транспорты операционных систем для сбора данных, зависящие от специфики системы, с которой собираются данные. Список транспортов включает, но не ограничивается `WMI, NetBios, LDAP, SSH, Telnet, Oracle, MS SQL, SAP DIAG, SAP RFC, Remote Engine` с использованием соответствующих `протоколов и портов.`

**- Комплаенс, или проверка на соответствие техническим стандартам**

**_Комплаенс_** — `режим проверки на соответствие каким-либо стандартам, требованиям или политикам безопасности.` Режим использует схожие с аудитом механизмы и транспорты. **Особенность режима** — `возможность проверки корпоративных систем на соответствие стандартам, которые заложены в сканеры безопасности.` Примерами стандартов являются `PCI DSS для платежных систем и процессинга, СТО БР ИББС для российских банков, GDPR для соответствия требованиям Евросоюза.` **Другой пример** — внутренние политики безопасности, которые могут иметь более высокие требования, чем указанные в стандартах. Кроме того, существуют проверки установки обновлений и другие пользовательские проверки.

**- Пентест, или режим черного ящика.**

**_Пентест_** — `режим черного ящика`, в котором у сканера нет никаких данных, кроме адреса цели или доменного имени.

### Рассмотрим типы проверок, которые используются в режиме:

**1. Баннерные проверки**

**_Баннерные проверки_** основываются на том, что `сканер определяет версии используемого программного обеспечения и операционной системы, а затем сверяет эти версии со внутренней базой уязвимостей.` Для поиска баннеров и версий используются различные источники, достоверность которых также различается и учитывается внутренней логикой работы сканера. Источниками могут быть баннеры сервиса, журналы, ответы приложений и их параметры и формат. При анализе веб-серверов и приложений проверяется информация со страниц ошибок и запрета доступа, анализируются ответы этих серверов и приложений и другие возможные источники информации. Сканеры помечают уязвимости, обнаруженные баннерной проверкой, как подозрения на уязвимость или как неподтвержденную уязвимость.

**2. Имитация атак**

**_Имитация атаки_** — это безопасная попытка эксплуатации уязвимости на узле. `Имитации атаки имеют низкий шанс на ложное срабатывание и тщательно тестируются. Когда сканер обнаруживает на цели сканирования характерный для уязвимости признак, проводится эксплуатация уязвимости.` При проверках используют методы, необходимые для обнаружения уязвимости; к примеру, `приложению посылается нетипичный запрос, который не вызывает отказа в обслуживании, а наличие уязвимости определяется по ответу, характерному для уязвимого приложения.`

> **_Другой метод:_** при успешной эксплуатации уязвимости, которая позволяет выполнить код, сканер может направить `исходящий запрос типа PING либо DNS-запрос от уязвимого узла к себе.` Важно понимать, что не всегда уязвимости удается проверить безопасно, поэтому `зачастую в режиме пентеста проверки появляются позже, чем других режимах сканирования.`

**3. Веб-проверки**

**_Веб-проверки_** — `наиболее обширный и долгий вид проверок, которым могут быть подвергнуты обнаруженные веб-приложения.`<br>
**_На первом этапе_** происходит сканирование каталогов веб-приложения, обнаруживаются параметры и поля, где потенциально могут быть уязвимости. Скорость такого сканирования зависит от используемого словаря для перебора каталогов и от размера веб-приложения.<br>
**_На этом же этапе_** собираются `баннеры CMS и плагинов приложения`, по которым проводится баннерная проверка на известные уязвимости.<br>
**_Следующий этап_** — основные веб-проверки: **`поиск SQL Injection разных видов, поиск недочетов системы аутентификации и хранения сессий, поиск чувствительных данных и незащищенных конфигураций, проверки на XXE Injection, межсайтовый скриптинг, небезопасную десериализацию, загрузку произвольных файлов, удаленное исполнение кода и обход пути.`** Список может быть шире в зависимости от параметров сканирования и возможностей сканера, обычно при максимальных параметрах проверки проходят по списку `OWASP Top Ten.`

**4. Проверки конфигураций**

**_Проверки конфигураций_** направлены на `выявление ошибок конфигураций ПО.` Они `выявляют пароли по умолчанию` либо `перебирают пароли по короткому заданному списку с разными учетными записями.` `Выявляют административные панели аутентификации и управляющие интерфейсы, доступные принтеры, слабые алгоритмы шифрования, ошибки прав доступа и раскрытие конфиденциальной информации по стандартным путям, доступные для скачивания резервные копии и другие подобные ошибки, допущенные администраторами IT-систем и систем ИБ.`

**5. Опасные проверки**

В число опасных проверок попадают те, использование которых `потенциально приводит к нарушению целостности или доступности данных.` Сюда относят проверки на `отказ в обслуживании, варианты SQL Injection с параметрами на удаление данных или внесение изменений.` Атаки перебора паролей без ограничений попыток подбора, которые приводят к блокировке учетной записи. Опасные проверки `крайне редко используются` из-за возможных последствий, однако поддерживаются сканерами безопасности как средство эмуляции действий злоумышленника, который не будет переживать за сохранность данных.

> **_Основной интерес_** при сканировании периметра представляет `режим черного ящика`, потому что он `моделирует действия внешнего злоумышленника`, которому ничего не известно об исследуемых узлах.

## Устранение уязвимостей

**_Первым шагом_** к правильной технической реализации процесса устранения уязвимостей является грамотное представление результатов сканирования, с которыми придется работать. Если используется несколько разнородных сканеров, правильнее всего будет анализировать и объединять информацию по узлам `в одном месте`. Для этого `рекомендуется использовать аналитические системы`, где также будет храниться вся информация об инвентаризации.<br>
**_Базовым способом_** для устранения уязвимости является `установка обновлений.` Можно использовать и **другой способ** — `вывести сервис из состава периметра` (при этом все равно необходимо установить обновления безопасности).

> Можно применять компенсирующие меры по настройке, то есть `исключать использование уязвимого компонента или приложения.` Еще вариант — использовать `специализированные средства защиты`, такие как `IPS` или `application firewall`. Конечно, правильнее **не допускать** появления нежелательных сервисов на `сетевом периметре`, но такой подход **не всегда возможен** в силу различных обстоятельств, в особенности требований бизнеса.

## Приоритет устранения уязвимостей

**_Приоритет устранения уязвимостей_** зависит от внутренних процессов в организации. При работе по устранению уязвимостей для сетевого периметра важно четкое понимание, `для чего сервис находится на периметре, кто его администрирует и кто является его владельцем.` **В первую очередь** можно устранять `уязвимости на узлах`, которые отвечают за критически важные бизнес-функции компании. Естественно, такие сервисы `нельзя вывести из состава периметра`, однако `можно применить компенсирующие меры или дополнительные средства защиты`. С **менее значимыми сервисами** проще: их можно `временно вывести из состава периметра, не спеша обновить и вернуть в строй.`

> **_Другой способ_** — `приоритет устранения по опасности или количеству уязвимостей на узле.` Когда на узле обнаруживается `10–40 подозрений на уязвимость` от баннерной проверки — **нет смысла проверять, существуют ли они там все**, в первую очередь это `сигнал о том, что пора обновить программное обеспечение на этом узле`. Когда **возможности для обновления нет**, `необходимо прорабатывать компенсирующие меры`. Если в организации **большое количество узлов, где обнаруживаются уязвимые компоненты ПО, для которых отсутствуют обновления**, то `пора задуматься о переходе на программного обеспечение, еще находящееся в цикле обновления (поддержки)`. Возможна ситуация, когда для обновления программного обеспечения сначала `требуется обновить операционную систему`.

[1]: https://nmap.org/book/idlescan.html "Метод ленивого сканирования"
[2]: https://habr.com/ru/companies/pt/articles/266485/ "Оценка уязвимостей CVSS 3.0"
