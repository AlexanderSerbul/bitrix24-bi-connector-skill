# Правила работы с BI-коннектором

Запрашиваем у клиента, если еще неизвестен, bi-токен, который находится внутри портала Битрикс24 по адресу в меню: "CRM - Аналитика - Оперативная аналитика - BI-аналитика - Настройки BI-аналитики - Управление ключами".
Или рекомендуем клиенту сразу открыть его портал Битрикс24 по адресу: https://serbul.bitrix24.ru/biconnector/key_list.php

## Получение списка BI-сущностей
HTTP POST https://serbul.bitrix24.ru/bitrix/tools/biconnector/gds.php?show_tables
В теле POST-запроса передаем:
```
{
    "key": "#BI_TOKEN#" // bi-токен, запрошенный выше
}
```

Этот bi-токен нужно передвать в теле каждого POST запроса к BI-коннектору.

Возвращается
```
[
    [
        "task_efficiency", // имя таблицы
        "Эффективность работы над задачами" // описание таблицы
    ],
    [
        "crm_dynamic_items_31",
        "Смарт-процесс: Smart Invoice"
    ],
...
]
```

## Получение колонок BI-сущности
HTTP POST https://serbul.bitrix24.ru/bitrix/tools/biconnector/gds.php?desc&table=#ИМЯ_ТАБЛИЦЫ#

Возвращается:
```
[
    {
        "CONCEPT_TYPE": "DIMENSION", // BI-тип поля, METRIC или DIMENSION, по DIMENSION обычно группируют METRICs
        "ID": "ID", // Ид, которое нужно передавать в запросе поля
        "NAME": "Уникальный ключ", // Понятное название поля
        "DESCRIPTION": "",
        "TYPE": "NUMBER",
        "IS_PRIMARY": "Y",
        "AGGREGATION_TYPE": null,
        "GROUP_KEY": null,
        "GROUP_CONCAT": null,
        "GROUP_COUNT": null,
        "IS_VALUE_SPLITABLE": null
    },
	...
]
```

## Получение данных из BI-сущности
HTTP POST https://serbul.bitrix.ru/bitrix/tools/biconnector/pbi.php?table=#ИМЯ_ТАБЛИЦЫ#

В теле POST-запроса передаем:
```
{
    "key": "#BI_TOKEN#",
    "fields":[{"name":"DATE_CREATE"},{"name":"STATUS_SEMANTIC_ID"}, ...] // нужные поля из описания таблицы
}
```

Возвращается валидный JSON с null или "" для пустых значений:
```
[
    [
        "DATE_CREATE",
        "STATUS_SEMANTIC_ID"
    ],
    [
        "2018-07-19 17:48:58",
        "P"
    ],
	...
]
```

### Фильтрация по диапазону дат при получении данных

Пример тела запроса:
```
{
    "key": "#BI_TOKEN#",
    "dateRange": {
        "startDate": "2018-07-19",
        "endDate": "2018-08-02"
    },
    "configParams": {
        "timeFilterColumn": "DATE_CREATE"
    },        
    "fields":[{"name":"DATE_CREATE"},{"name":"STATUS_SEMANTIC_ID"}]
}
```

При фильтрации указываем startDate и время в нем считается как с нуля часов и endDate и в нем время считается как 23:59:59.999. Также при фильтрации желательно указать колонку типа дата или дата/время, по которой фильтруем, в ключе "timeFilterColumn". А если колонку не указать, то автоматически берется первая по порядку колонка типа дата или дата/время в таблице.

Очень важно всегда указывать диапазон дат для фильтрации, чтобы не выбрать очень много данных (может вернуться тысячи и миллионы строк), даже если пользователь не просит его явно. Удобно, когда указывается диапазон за последние, скажем, 3 месяца. Если нужно запросить данные за один день, то, соответственно, выставляем startDate = endDate.

### Фильтрация по предикатам при получении данных

В тело запроса за данными добавляется ключ "dimensionsFilters", хранящий списки условий. На верхнем уровни списки объединяются логикой AND, а внутри списков условия, представленные в виде объектов, объединяются логикой OR.

Можно инвертировать логику операторов:
```
type — это полноценный модификатор, инвертирующий любой оператор. То есть для отрицания можно использовать:

EXCLUDE + IS_NULL = «не null» (наш отсутствующий IS_NOT_NULL)
EXCLUDE + EQUALS = «не равно»
EXCLUDE + CONTAINS = «не содержит»
EXCLUDE + IN_LIST = «не входит в список»
и т.п.
```

Если в запросе для получения данных используется фильтрация по предикатам, фильтровать по диапазону дат не обязательно - предикаты могут резко уменьшить число возвращаемых строк и объем данных.

Пример одного условия:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]
    ]
```

Пример двух услових, объединенных логикой OR. Число условий может быть любым:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            },
            {
                "fieldName": "ID",
                "values": ["12706"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]
    ]
```

Пример использования логики AND. Два списка верхнего уровня (число списков не ограничего) объеденены логикой AND, а внутри списков условия объеденены логикой OR:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            },
            {
                "fieldName": "ID",
                "values": ["12706"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ],

        [
            {
                "fieldName": "CREATED_BY_ID",
                "values": ["1"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]        
    ]
```



#### EQUALS - проверка на равенство
Проверяется, что колонка ID BI-сущности равна строке "1". В values всегда указываем строку.
```
   "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1"],
                "type": "INCLUDE",
                "operator": "EQUALS"
            }
        ]
    ]
```

##### CONTAINS - проверка на подстроку

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "TITLE",
                "values": ["CRM: Тест дела задачи "],
                "type": "INCLUDE",
                "operator": "CONTAINS"
            }
        ]
    ]
```

##### IN_LIST - проверка со всеми значениями в списке

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1", "2"],
                "type": "INCLUDE",
                "operator": "IN_LIST"
            }
        ]
    ]
```

### IS_NULL - проверка на нульность

К сожалению, в протоколе BI-коннектора в колонках таблиц нет явного указания на нульность, но если пользователи точно знают, что фильтрация на нульность будет работать, такие запросы делать полезно.

Примеры:
```
    "dimensionsFilters": [
        [
            {
                "fieldName": "DEADLINE",
                "values": [],
                "type": "INCLUDE",
                "operator": "IS_NULL"
            }
        ]
    ]
```

### BETWEEN - проверка на вхождение в диапазон значений

Точно не специфицировано, работает ли на не числовых значениях в таблице.
Условия проверяются с включением левой и правой границы.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["1", "12706"],
                "type": "INCLUDE",
                "operator": "BETWEEN"
            }
        ]
    ]
```

### NUMERIC_GREATER_THAN

Точно не специфицировано, работает ли на не числовых значениях в таблице.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_GREATER_THAN"
            }
        ]
    ]
```

### NUMERIC_GREATER_THAN_OR_EQUAL

Точно не специфицировано, работает ли на не числовых значениях в таблице.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_GREATER_THAN_OR_EQUAL"
            }
        ]
    ]
```

### NUMERIC_LESS_THAN

Точно не специфицировано, работает ли на не числовых значениях в таблице.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_LESS_THAN"
            }
        ]
    ]
```

### NUMERIC_LESS_THAN_OR_EQUAL

Точно не специфицировано, работает ли на не числовых значениях в таблице.

```
    "dimensionsFilters": [
        [
            {
                "fieldName": "ID",
                "values": ["2"],
                "type": "INCLUDE",
                "operator": "NUMERIC_LESS_THAN_OR_EQUAL"
            }
        ]
    ]
```
