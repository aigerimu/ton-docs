# Словари в TON

:::warning
Эта страница переведена сообществом на русский язык, но нуждается в улучшениях. Если вы хотите принять участие в переводе свяжитесь с [@alexgton](https://t.me/alexgton).
:::

Смарт-контракты могут использовать словари — упорядоченные сопоставления ключ-значение. Внутренне они представлены деревьями ячеек.

:::warning
Working with potentially large trees of cells creates a couple of considerations:

1. Каждая операция обновления создает заметное количество ячеек (и каждая построенная ячейка стоит 500 газа, как можно найти на странице [Инструкции TVM](/v3/documentation/tvm/instructions#gas-prices), что означает, что эти операции могут исчерпать газ, если их использовать без должного внимания.
    - В частности, Wallet bot однажды столкнулся с такой проблемой при использовании кошелька highload-v2. Неограниченный цикл в сочетании с дорогостоящими обновлениями словаря на каждой итерации привел к исчерпанию газа и в конечном итоге к повторным транзакциям, таким как [fd78228f352f582a544ab7ad7eb716610668b23b88dae48e4f4dbd4404b5d7f6](https://tonviewer.com/transaction/fd78228f352f582a544ab7ad7eb716610668b23b88dae48e4f4dbd4404b5d7f6), опустошающим его баланс.
2. Двоичное дерево для N пар ключ-значение содержит N-1 разветвлений и, таким образом, в общей сложности не менее 2N-1 ячеек. Хранилище смарт-контрактов ограничено 65536 уникальными ячейками, поэтому максимальное количество записей в словаре составляет 32768 или немного больше, если есть повторяющиеся ячейки.
    :::

## Типы словарей

### "Хэш"карта

Очевидно, что наиболее известным и используемым типом словарей в TON является хэш-карта. Он имеет целый раздел, стоящий кодов операций TVM ([Инструкции TVM](/v3/documentation/tvm/instructions#quick-search) - Манипулирование словарем) и обычно используется в смарт-контрактах.

Эти словари представляют собой сопоставления ключей одинаковой длины (указанная длина указывается в качестве аргумента для всех функций) с фрагментами значений. В отличие от "хэширования" в названии, записи в них упорядочены и обеспечивают дешевое извлечение элемента по ключу, предыдущей или следующей паре ключ-значение. Значения помещаются в ту же ячейку, что и внутренние теги узлов и, возможно, ключевые части, поэтому они не могут использовать все 1023 бита; в такой ситуации обычно используется `~udict_set_ref`.

Пустая хэш-карта представляется TVM как `null`; таким образом, это не ячейка. Чтобы сохранить словарь в ячейке, сначала сохраняется один бит (0 для пустого, 1 в противном случае), а затем добавляется ссылка, если хэш-карта не пуста. Таким образом, `store_maybe_ref` и `store_dict` взаимозаменяемы, и некоторые авторы смарт-контрактов используют `load_dict` для загрузки `Maybe ^Cell` из входящего сообщения или хранилища.

Возможные операции для хэш-карт:

- загрузка из среза, сохранение в конструкторе
- получение/установка/удаление значения по ключу
- замена значения (установка нового значения, если ключ уже был) / добавление одного (если ключ отсутствовал)
- переход к следующей/предыдущей паре ключ-значение в порядке ключей (это можно использовать для [перебора словарей](/v3/documentation/smart-contracts/func/cookbook#how-to-iterate-dictionaries), если ограничение газа не имеет значения)
- извлечение минимального/максимального ключа с его значением
- получение функции (продолжение) по ключу и немедленное ее выполнение

Чтобы контракт не нарушался из-за превышения ограничения газа, при обработке одной транзакции должно происходить только ограниченное количество обновлений словаря. Если баланс контракта используется для поддержания карты в соответствии с условиями разработчика, контракт может отправить себе сообщение для продолжения очистки.

:::info
Существуют инструкции по извлечению подсловаря: подмножества записей в заданном диапазоне ключей. Они не были протестированы, поэтому вы можете проверить их только в форме сборки TVM: `SUBDICTGET` и аналогичной.
:::

#### Примеры хэш-карт

Давайте посмотрим, как выглядят хэш-карты, уделив особое внимание сопоставлению 257-битных целочисленных ключей с пустыми срезами значений (такая карта будет указывать только на наличие или отсутствие элемента).

Способ быстрой проверки — запустить следующий скрипт на Python (возможно, заменив `pytoniq` на другой SDK по мере необходимости):

```python
import pytoniq
k = pytoniq.HashMap(257)
em = pytoniq.begin_cell().to_slice()
k.set(5, em)
k.set(7, em)
k.set(5 - 2**256, em)
k.set(6 - 2**256, em)
print(str(pytoniq.begin_cell().store_maybe_ref(k.serialize()).end_cell()))
```

Структура представляет собой двоичное дерево, даже сбалансированное, если мы пропустим корневую ячейку.

```
1[80] -> {
	2[00] -> {
		265[9FC00000000000000000000000000000000000000000000000000000000000000080] -> {
			4[50],
			4[50]
		},
		266[9FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF40] -> {
			2[00],
			2[00]
		}
	}
}
```

В документации есть [еще примеры по разбору хэш-карт](/v3/documentation/data-formats/tlb/tl-b-types#hashmap-parsing-example).

### Расширенные карты (с дополнительными данными в каждом узле)

Эти карты используются внутри валидаторов TON для расчета общего баланса всех контрактов в шарде (использование карт с общим балансом поддерева с каждым узлом позволяет им очень быстро проверять обновления). Для работы с ними нет примитивов TVM.

### Словарь префиксов

:::info
Тестирование показывает, что для создания словарей префиксов недостаточно документации. Вам не следует использовать их в продакшен контрактах, если у вас нет полных знаний о том, как работают соответствующие opcode, `PFXDICTSET` и подобные.
:::
