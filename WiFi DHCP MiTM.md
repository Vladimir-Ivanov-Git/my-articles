# Еще один MiTM в WiFi сети

В данной статье мы расскажем про еще один способ осуществления **MiTM в WiFi сети**, не самый простой, но все же. Прежде чем читать эту статью настоятельно рекомендую ознакомиться с другой моей [статьей](https://habrahabr.ru/company/dsec/blog/333978/) в которой я попытался объяснить принцип работы протокола DHCP и как с его помощью атаковать клиентов и сервер.

В данной статье мы найдем еще один способ организовать MiTM в WiFi сети. Как и всегда есть пару ограничений:
1. Мы должны быть подключены к атакуемой точке доступа.
2. У нас должна быть возможность прослушивать широковещательный трафик в сети.

И так как же это работает? Атака разделяется на несколько этапов:
1. Производим атаку DHCP Starvation;
2. Отправляем WiFi DeAuth пакеты;
3. Перехватываем ARP запросы от клиентов, отвечаем на них, чтобы создать конфликт IP-адресов и принудить клиента отправить DHCPDECLINE;
4. Перехватываем DHCPDISCOVER и DHCPREQUEST запросы, отвечаем на них;
5. Profit!

Разберемся в этой схеме поподробнее.

## DHCP Starvation
Подключаемся к атакуемой WiFi сети и производим атаку DHCP Starvation с целью переполнить пул свободных IP-адресов на точке доступа.
Как это работает:

1. Формируем и отправляем широковещательный DHCPDISCOVER-запрос, при этом представляемся как DHCP relay agent. В поле **giaddr** (Relay agent IP) указываем свой IP-адрес **192.168.1.198**, в поле **chaddr** (Client MAC address) - рандомный MAC **00:23:32:B1:A9:75**, при этом на канальном уровне в SRC MAC выставляем свой MAC-адрес.
![DHCPDISCOVER](https://dl.dropboxusercontent.com/s/eb3q2s6y8okxwej/DHCP%20starvation%20send%20discover.png)

2. Сервер отвечает сообщением DHCPOFFER агенту ретрансляции (нам), и предлагает клиенту с MAC-адресом **00:23:32:B1:A9:75** IP-адрес **192.168.1.121**
![DHCPOFFER](https://dl.dropboxusercontent.com/s/m9rszt1gie4jkjg/DHCP%20starvation%20recieve%20offer.png)

3. После получения DHCPOFFER, отправляем широковещательный DHCPREQUEST-запрос, при этом в DHCP-опции с кодом 50 ([Requested IP address](https://tools.ietf.org/html/rfc2132#section-9.1)) выставляем предложений клиенту IP-адрес **192.168.1.121**, в опции с кодом 12 ([Host Name Option](https://tools.ietf.org/html/rfc2132#section-3.14)) - рандомную строку **lyzEKAAr**. **Важно:** значения полей **xid** (Transaction ID) и **chaddr** (Client MAC address) в DHCPREQUEST и DHCPDISCOVER должны быть одинаковыми, иначе сервер отбросит запрос, ведь это будет выглядеть, как другая транзакция от того же клиента, либо другой клиент с той же транзакцией.
![DHCPREQUEST](https://dl.dropboxusercontent.com/s/s345vssxo7lnj8o/DHCP%20starvation%20send%20request.png)

4. Сервер отправляет агенту ретрансляции сообщение DHCPACK. С этого момента IP-адрес **192.168.1.121** считается зарезервированным за клиентом с MAC-адресом **00:23:32:B1:A9:75** на 12 часов (время аренды по умолчанию).
![DHCPACK](https://dl.dropboxusercontent.com/s/skflvfoyf6ut4hf/DHCP%20starvation%20recieve%20ack.png)
![DHCP clients](https://dl.dropboxusercontent.com/s/to9acs67jtp7bkb/DHCP%20clients.png)

## WiFi DeAuth
Работает эта схема примерно так:
![WiFi DeAuth](https://upload.wikimedia.org/wikipedia/commons/9/95/Deauth_attack_sequence_diagram.svg)

Отправляем deauth пакеты с целью отсоединить клиентов WiFi сети от точки доступа.

## DHCPDECLINE
После того как клиент отсоединился от точки доступа он заново подключаются к WiFi сети и получить IP-адрес по DHCP. Так как ранее он уже подключались этой сети то будет отравлять только широковещательный DHCPREQUEST.
![Send DHCPREQUEST after DeAuth](https://dl.dropboxusercontent.com/s/cm0a8s7sgl97oef/dhcprequest%20after%20deauth.png)

Мы перехватываем запрос клиента, но ответить быстрее точки доступа мы естественно не успеем. Поэтому клиент получает IP-адрес от точки доступа полученный ранее **192.168.1.102**. Далее клиент с помощью протокола ARP пытается обнаружить конфликт IP-адресов в сети.
![Try detect IP address conflict](https://dl.dropboxusercontent.com/s/kl9q8cbb0le3x11/ip%20address%20conflict%20detection.png)

Естественно такой запрос широковещательный и можем его перехватить и ответить на него.
![IP address conflict detected](https://dl.dropboxusercontent.com/s/foc4apwpu5xankl/ip%20address%20conflict%20detected.png)

После чего клиент фиксирует конфликт IP-адресов и отправляет широковещательное сообщение отказа DHCP - DHCPDECLINE.
![Send DHCPDECLINE](https://dl.dropboxusercontent.com/s/4l71v3j3ynap1km/send%20dhcp%20decline.png)

## DHCPDISCOVER
И так последний этап атаки. После отправки DHCPDECLINE клиент с самого начала проходит процедуру получения IP-адреса, а именно отправляет широковещательный DHCPDISCOVER. Легитимный DHCP-сервер не может ответить на этот запрос, так как пул свободных IP-адресов переполнен после проведения атаки DHCP starvation и поэтому заметно тормозит, зато на DHCPDISCOVER можем ответить мы - **192.168.1.172**.
![DHCPDISCOVER](https://dl.dropboxusercontent.com/s/ycdn0xrqu6mexk2/send%20dhcp%20discover.png)
![Attacker DHCPOFFER](https://dl.dropboxusercontent.com/s/xi1njdgkxfxnxg7/attacker%20dhcp%20offer.png)

В DHCPOFFER мы предоставили клиенту IP-адрес **192.168.1.2** клиент получив данное только от нас отправляет DHCPREQUEST, выставляя при этом в requested ip значение **192.168.1.2** и мы отвечаем DHCPACK.
![DHCPREQUEST](https://dl.dropboxusercontent.com/s/4bz9ames6o76gqb/send%20dhcp%20request.png)
![Attacker DHCPACK](https://dl.dropboxusercontent.com/s/mm2h6t3782de3io/attacker%20dhcp%20ack.png)

Клиент принимает наш DHCPACK и в качестве шлюза по умолчанию и DNS-сервера выставляет наш IP-адрес **192.168.1.172**, а DHCPNAK от точки доступа присланный на 2 секунды позже просто проигнорирует.
![Client network settings]()

Почему точка доступа прислала DHCPOFFER и DHCPNAK на 2 секунды позже, да еще и предложила тот же IP-адрес **192.168.1.102**, ведь клиент отказался от него?
![DHCPOFFER from AP](https://dl.dropboxusercontent.com/s/rfx4m994i66xzrw/dhcpoffer%20from%20AP.png)

Чтобы ответить на данный вопрос немного изменим фильтр в WireShark и посмотрим ARP запросы от точки доступа:
![Add ARP requests from AP](https://dl.dropboxusercontent.com/s/yiso0fktggvw868/ip%20address%20conflict%20detection%20from%20AP.png)

После проведения атаки DHCP Starvation у DHCP-сервера не оказалось свободных IP-адресов кроме того от которого отказался один из клиентов **192.168.1.102**. Поэтому получив DHCPDISCOVER-запрос DHCP-сервер в течении 2-ух секунд отправляет три ARP-запроса чтобы узнать кто использует IP-адрес **192.168.1.102** и после того как сервер убедился что данный IP-адрес свободен, так как никто не ответил, выдает его клиенту, но уже слишком поздно.

Таким образом мы можем выставлять любые сетевые параметры, а именно: шлюз по умолчанию, DNS сервер и т.д. любому подключенному или новому клиенту в атакуемой WiFi сети.

Видео проведения атаки на MacOS Siera и Windows 10:
[![WiFi DHCP MiTM](https://j.gifs.com/2R6OEz.gif)](https://youtu.be/OBXol-o2PEU)

Ну и конечно же [PoC](https://github.com/Vladimir-Ivanov-Git/raw-packet).

# Apple DHCP client

Кстати по поводу баг и фич. Как оказалось MacOS и iOS переплюнули всех, когда эти операционные системы отправляют DHCPREQUEST точка доступа отвечает им DHCPACK:
![MacOS legal DHCPACK](https://dl.dropboxusercontent.com/s/k9ji5zi7uf74m95/MacOS%20legal%20DHCPACK.png)

Но проблема в том что DHCPREQUEST широковещательный и мы без проблем можем его перехватить и тоже можем ответить DHCPACK, но конечно позднее точки доступа. Как вы думаете какие настройки выставляют данные операционные системы? Ответ: те настройки которые будут содержаться в нашем DHCPACK сообщении (во втором DHCPACK) !
![MacOS attacker DHCPACK](https://dl.dropboxusercontent.com/s/ffln8lh31m6eqzx/MacOS%20attacker%20DHCPACK.png?dl=0)

Ну и конечно же видео:
[![MacOS DHCP client vulnerability](https://j.gifs.com/k5zJk6.gif)](https://youtu.be/XSVT4BFUqsU)

Как Вы думаете баг это или фича? Я подумал баг и завел заявку на apple bug reporter скоро этой заявке исполнится месяц, но ни одного комментария от специалистов Apple я так и не получил.
![Apple bugreporter](https://dl.dropboxusercontent.com/s/yh5hg3pdgcb4mjd/Apple%20bugreporter.PNG)
