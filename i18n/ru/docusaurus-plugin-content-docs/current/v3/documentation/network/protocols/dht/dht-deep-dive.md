# DHT

:::warning
Эта страница переведена сообществом на русский язык, но нуждается в улучшениях. Если вы хотите принять участие в переводе свяжитесь с [@alexgton](https://t.me/alexgton).
:::

Распределенная хеш-таблица (DHT - Distributed Hash Table) по сути является распределенной базой данных "ключ-значение",
где каждый участник сети может хранить что-то, например, информацию о себе.

Реализация DHT в TON по своей сути похожа на реализацию [Kademlia](https://codethechange.stanford.edu/guides/guide_kademlia.html), которая используется в IPFS.
Любой участник сети может запустить узел DHT, сгенерировать ключи и хранить данные.
Для этого ему нужно сгенерировать случайный идентификатор и сообщить другим узлам о себе.

Чтобы определить, на каком узле хранить данные, используется алгоритм, определяющий "расстояние" между узлом и ключом.
Алгоритм прост: берем идентификатор узла и идентификатор ключа, выполняем операцию XOR. Чем меньше значение, тем ближе узел.
Задача — хранить ключ на узлах, максимально приближенных к ключу, чтобы другие участники сети могли, используя тот же алгоритм, найти узел, который может предоставить данные по этому ключу.

## Поиск значения по ключу

Рассмотрим пример с поиском ключа, [подключаемся к любому узлу DHT и устанавливаем соединение по ADNL UDP](/v3/documentation/network/protocols/adnl/adnl-udp#packet-structure-and-communication).

Например, мы хотим найти адрес и открытый ключ для подключения к узлу, на котором размещен сайт foundation.ton.
Допустим, мы уже получили адрес ADNL этого сайта, выполнив Get метод контракта DNS.
Адрес ADNL в шестнадцатеричном представлении - `516618cf6cbe9004f6883e742c9a2e3ca53ed02e3e36f4cef62a98ee1e449174`.
Теперь наша цель - найти ip, порт и открытый ключ узла, имеющего этот адрес.

Для этого нам нужно получить идентификатор ключа DHT, для начала заполним схему ключа DHT:

```tlb
dht.key id:int256 name:bytes idx:int = dht.Key
```

`name` - это тип ключа, для адресов ADNL используется слово `address`, а, например, для поиска узлов шардчейна - `nodes`. Но типом ключа может быть любой массив байт, в зависимости от значения, которое вы ищете.

Заполнив эту схему, получаем:

```
8fde67f6                                                           -- TL ID dht.key
516618cf6cbe9004f6883e742c9a2e3ca53ed02e3e36f4cef62a98ee1e449174   -- our searched ADNL address
07 61646472657373                                                  -- key type, the word "address" as an TL array of bytes
00000000                                                           -- index 0 because there is only 1 key
```

Далее - получаем идентификатор ключа, хэш sha256 из байтов, сериализованных выше. Это будет `b30af0538916421b46df4ce580bf3a29316831e0c3323a7f156df0236c5b2f75`

Теперь мы можем начать поиск. Для этого нам нужно выполнить запрос, который имеет [схему](https://github.com/ton-blockchain/ton/blob/ad736c6bc3c06ad54dc6e40d62acbaf5dae41584/tl/generate/scheme/ton_api.tl#L197):

```tlb
dht.findValue key:int256 k:int = dht.ValueResult
```

`key` — это идентификатор нашего ключа DHT, а `k` — это "ширина" поиска, чем он меньше, тем точнее, но меньше потенциальных узлов для запроса. Максимальное k для узлов в TON — 10, обычно используется 6.

Давайте заполним эту структуру, сериализуем и отправим запрос с помощью схемы `adnl.message.query`. [Подробнее об этом можно прочитать в другой статье](/v3/documentation/network/protocols/adnl/adnl-udp#packet-structure-and-communication).

В ответ мы можем получить:

- `dht.valueNotFound` - если значение не найдено.
- `dht.valueFound` - если значение найдено на этом узле.

##### dht.valueNotFound

Если мы получим `dht.valueNotFound`, то ответ будет содержать список узлов, которые известны запрошенному нами узлу и максимально приближены к запрошенному нами ключу из списка известных ему узлов. В этом случае нам нужно подключиться и добавить полученные узлы в список известных нам.
После этого из списка всех известных нам узлов выбираем ближайший, доступный и еще не запрошенный и делаем к нему такой же запрос. И так до тех пор, пока не перепробуем все узлы в выбранном нами диапазоне или пока не перестанем получать новые узлы.

Давайте подробнее разберем поля ответа, используемые схемы:

```tlb
adnl.address.udp ip:int port:int = adnl.Address;
adnl.addressList addrs:(vector adnl.Address) version:int reinit_date:int priority:int expire_at:int = adnl.AddressList;

dht.node id:PublicKey addr_list:adnl.addressList version:int signature:bytes = dht.Node;
dht.nodes nodes:(vector dht.node) = dht.Nodes;

dht.valueNotFound nodes:dht.nodes = dht.ValueResult;
```

`dht.nodes -> nodes` - список узлов DHT (массив).

У каждого узла есть `id`, который является его открытым ключом, обычно [pub.ed25519](https://github.com/ton-blockchain/ton/blob/ad736c6bc3c06ad54dc6e40d62acbaf5dae41584/tl/generate/scheme/ton_api.tl#L47), используемым как ключ сервера для подключения к узлу через ADNL. Также у каждого узла есть список адресов `addr_list:adnl.addressList`, версия и подпись.

Нам нужно проверить подпись каждого узла, для этого мы считываем значение `signature` и устанавливаем поле в ноль (делаем его пустым массивом байтов). После - сериализуем структуру TL `dht.node` с пустой подписью и проверяем поле `signature`, которое было до опустошения.
Проверяем полученные сериализованные байты, используя открытый ключ из поля `id`. [Пример реализации](https://github.com/xssnick/tonutils-go/blob/46dbf5f820af066ab10c5639a508b4295e5aa0fb/adnl/dht/client.go#L91)

Из списка `addrs:(vector adnl.Address)` берем адрес и пытаемся установить соединение ADNL UDP, в качестве ключа сервера используем `id`, который является открытым ключом.

Чтобы узнать "расстояние" до этого узла - нам нужно взять [идентификатор ключа](/v3/documentation/network/protocols/adnl/adnl-tcp#getting-key-id) из ключа из поля `id` и проверить расстояние операцией XOR из идентификатора ключа узла и нужного ключа.
Если расстояние достаточно мало, мы можем сделать тот же запрос к этому узлу. И так далее, пока не найдем значение или не останется новых узлов.

##### dht.valueFound

Ответ будет содержать само значение, полную информацию о ключе и, возможно, подпись (зависит от типа значения).

Давайте подробнее проанализируем поля ответа, используемые схемы:

```tlb
adnl.address.udp ip:int port:int = adnl.Address;
adnl.addressList addrs:(vector adnl.Address) version:int reinit_date:int priority:int expire_at:int = adnl.AddressList;

dht.key id:int256 name:bytes idx:int = dht.Key;

dht.updateRule.signature = dht.UpdateRule;
dht.updateRule.anybody = dht.UpdateRule;
dht.updateRule.overlayNodes = dht.UpdateRule;

dht.keyDescription key:dht.key id:PublicKey update_rule:dht.UpdateRule signature:bytes = dht.KeyDescription;

dht.value key:dht.keyDescription value:bytes ttl:int signature:bytes = dht.Value; 

dht.valueFound value:dht.Value = dht.ValueResult;
```

Сначала проанализируем `key:dht.keyDescription`, это полное описание ключа, сам ключ и информация о том, кто и как может обновить значение.

- `key:dht.key` - ключ должен совпадать с тем, из которого мы взяли идентификатор ключа для поиска.
- `id:PublicKey` - открытый ключ владельца записи.
- `update_rule:dht.UpdateRule` - правило обновления записи.
- - `dht.updateRule.signature` - только владелец закрытого ключа может обновить запись, `signature` как ключа, так и значения должны быть действительными
- - `dht.updateRule.anybody` - все могут обновить запись, `signature` пустое и не проверяется
- - `dht.updateRule.overlayNodes` - узлы из одного и того же оверлея могут обновить ключ, используется для поиска узлов того же оверлея и добавления себя

###### dht.updateRule.signature

После прочтения описания ключа действуем в зависимости от `updateRule`, для случая поиска адреса ADNL тип всегда `dht.updateRule.signature`.
Проверяем подпись ключа так же, как и в прошлый раз, делаем подпись пустым массивом байтов, сериализуем и проверяем. После - повторяем то же самое для значения, т.е. для всего объекта `dht.value` (при этом возвращая ключевую подпись на место).

[Пример реализации](https://github.com/xssnick/tonutils-go/blob/46dbf5f820af066ab10c5639a508b4295e5aa0fb/adnl/dht/client.go#L331)

###### dht.updateRule.overlayNodes

Используется для ключей, содержащих информацию о других узлах-шардах воркчейна в сети, значение всегда имеет структуру TL `overlay.nodes`.
Поле value должно быть пустым.

```tlb
overlay.node id:PublicKey overlay:int256 version:int signature:bytes = overlay.Node;
overlay.nodes nodes:(vector overlay.node) = overlay.Nodes;
```

Для проверки на валидность мы должны проверить все `nodes` и для каждого проверить `signature` по его `id`, сериализовав структуру TL:

```tlb
overlay.node.toSign id:adnl.id.short overlay:int256 version:int = overlay.node.ToSign;
```

Как видим, id следует заменить на adnl.id.short, что является идентификатором ключа (хешем) поля `id` из исходной структуры. После сериализации - сверяем подпись с данными.

В результате получаем валидный список узлов, которые могут предоставить нам информацию о нужном нам шарде воркчейна.

###### dht.updateRule.anybody

Подписей нет, обновлять может кто угодно.

#### Использование значения

Когда все проверено и значение `ttl:int` не истекло, мы можем начать работать с самим значением, т. е. `value:bytes`. Для адреса ADNL внутри должна быть структура `adnl.addressList`.
В ней будут находиться ip-адреса и порты серверов, соответствующие запрашиваемому адресу ADNL. В нашем случае, скорее всего, будет 1 адрес RLDP-HTTP сервиса `foundation.ton`.
В качестве ключа сервера мы будем использовать открытый ключ `id:PublicKey` из информации о ключе DHT.

После установки соединения мы можем запрашивать страницы сайта по протоколу RLDP. Задача со стороны DHT на этом этапе выполнена.

### Поиск узлов, хранящих состояние блокчейна

DHT также используется для поиска информации об узлах, хранящих данные воркчейнов и их шардов. Процесс такой же, как и при поиске любого ключа, разница только в сериализации самого ключа и валидации ответа, эти моменты мы разберем в этом разделе.

Чтобы получить данные, например, мастерчейна и его шардов, нам нужно заполнить структуру TL:

```
tonNode.shardPublicOverlayId workchain:int shard:long zero_state_file_hash:int256 = tonNode.ShardPublicOverlayId;
```

Где `workchain` в случае мастерчейна будет равен -1, его шард будет равен -922337203685477580 (0xFFFFFFFFFFFFFFFF), а `zero_state_file_hash` - это хэш нулевого состояния цепочки (file_hash), как и другие данные, его можно взять из глобальной конфигурации сети, в поле `"validator"`

```json
"zero_state": {
  "workchain": -1,
  "shard": -9223372036854775808, 
  "seqno": 0,
  "root_hash": "F6OpKZKqvqeFp6CQmFomXNMfMj2EnaUSOXN+Mh+wVWk=",
  "file_hash": "XplPz01CXAps5qeSWUtxcyBfdAo5zVb1N979KLSKD24="
}
```

После того, как мы заполнили `tonNode.shardPublicOverlayId`, мы сериализуем его и получаем из него идентификатор ключа путем хэширования (как всегда).

Нам нужно использовать полученный идентификатор ключа как `name` для заполнения структуры `pub.overlay name:bytes = PublicKey`, обернув ее в массив байтов TL. Затем мы сериализуем его и получаем из него идентификатор ключа.

Полученный идентификатор будет ключом для использования в

```bash
dht.findValue
```

, а значение поля `name` будет словом `nodes`. Мы повторяем процесс из предыдущего раздела, все то же самое, что и в прошлый раз, но `updateRule` будет [dht.updateRule.overlayNodes](#dhtupdateruleoverlaynodes).

После проверки мы получим открытые ключи (`id`) узлов, которые содержат информацию о нашем воркчейне и шарде. Чтобы получить адреса ADNL узлов, нам нужно создать идентификаторы из ключей (используя метод хеширования) и повторить процедуру, описанную выше, для каждого из адресов ADNL, как и для адреса ADNL домена `foundation.ton`.

В результате мы получим адреса узлов, из которых, при желании, можно узнать адреса других узлов этой цепочки с помощью [overlay.getRandomPeers](https://github.com/ton-blockchain/ton/blob/ad736c6bc3c06ad54dc6e40d62acbaf5dae41584/tl/generate/scheme/ton_api.tl#L237).
Также мы сможем получать всю информацию о блоках с этих узлов.

## Ссылки

*Вот [ссылка на оригинальную статью](https://github.com/xssnick/ton-deep-doc/blob/master/DHT.md) [Олега Баранова](https://github.com/xssnick).*
