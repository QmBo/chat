# Микросервис chat

## Задачи

1. Принимает подключение с фронта по WebSocket.
2. Принимает сообщения от подключенного пользователя по WebSocket.
3. Отправляет сообщения подключенному пользователю по WebSocket.
4. Авторизует пользователя.

_Сервис chat имеют свою базу данных_

## Укрупненно

1. Фронт запрашивает подключение по WebSocket на сервере.
	- в запросе содержится токен авторизации.
	- если у пользователя есть сообщения, то сервер высылает их пользователю.
2. Подключенный пользователь оправляет на сервер сообщения для оператора. 
3. Высылает подключенному клиенту сообщения от оператора. 
4. Выдает клиенту токен авторизации.

## База данных

Таблица Messages

| имя столбца     |                                           |
|-----------------|-------------------------------------------|
| id              | INT UNSIGNED, PRIMARY KEY, AUTO_INCREMENT |
| user_id         | INT FOREIGN KEY REFERENCES USERS(id)      |
| invader_name    | VARCHAR(255) NOT NULL                     |
| message         | TEXT                                      |
| created         | DATETIME NOT NULL DEFAULT NOW()           |
| is_from_support | BOOL NOT NULL                             |

Таблица Users

| имя столбца   |                                           |
|---------------|-------------------------------------------|
| id            | INT UNSIGNED, PRIMARY KEY, AUTO_INCREMENT |
| user_name     | VARCHAR(255) NOT NULL                     |
| user_password | VARCHAR(255) NOT NULL                     |
| token         | VARCHAR(255)                              |
| is_admin      | BOOL NOT NULL, DEFAULT FALSE              |

## Логика работы

**1. С фронта приходит запрос на соединение.**

1.1. Фронт запрашивает авторизацию с помощью сообщения содержащего токен:

```json
{
	"method": "message.auth",
	"params": [
		"QWERTYUIOP"
	],
	"id": 1
}
```

1.1.1. Если токена нет или он не соответствует токену из таблицы:
- разорвать соединение подключение
- завершить обработку запроса

1.1.2. Если переданный токен соответствует токену в таблице, перейти к выполнению следующего пункта.

1.2. Выбрать из таблицы сообщений пользователя.

1.2.1 Если записи есть:

Сформировать json объект на основе этих записей.

```json
{
	"error": null,
	"result": {
		"messages": [
			{
				"message_id": 1,
				"message_text": "Message from Client",
				"invader_name": "",
				"is_from_support": false,
				"created": 1678794555
			},
			{
				"message_id": 2,
				"message_text": "Message from Operator",
				"invader_name": "Operator1",
				"is_from_support": true,
				"created": 1678794555
			}
		]
	},
	"id": 1
}
```

1.2.2 Если записей нет:

Сформировать json объект:

```json
{
	"error": null,
	"result": {
		"messages": null
	},
	"id": 1
}
```

1.2.3. Отправить json объект клиенту.


**2. Подключенный пользователь оправляет на сервер сообщения для оператора:**

```json
{
	"method": "message.send",
	"params": [
		"message"
	],
	"id": 1
}
```
2.1.1 Если аргументы params не соответствуют ожидаемому типу, отправить сообщение об ошибке:

```json
{
	"error": {
		"code": 1,
		"message": "Incorrect argument."
	},
	"result": null,
	"id": 1
}
```
2.1.2. Прекратить обработку запроса.

2.2.1. Сохранить сообщение в таблицу.

2.2.2. Отправить сообщение о положительном выполнении:

```json
{
	"error": null,
	"result": { 
		"message":
			{
				"message_id": 3,
				"message_text": "Message from Client",
				"invader_name": "",
				"is_from_support": false,
				"created": 1678794555
			}
	},
	"id": 3
}
```

2.3. Перейти к пункту 4.

**3. Высылает подключенному клиенту новые сообщения.**

3.1. Когда в переписке появляется новое сообщение оно высылается пользователю если тот всё еще подключен:

```json
{
	"error": null,
	"result": { 
		"message":
			{
				"message_id": 3,
				"message_text": "Message from Client",
				"invader_name": "",
				"is_from_support": true,
				"created": 1678794555
			}
	},
	"id": 3
}
```

**4. Подключенный оператор оправляет на сервер сообщения для пользователя:**

```json
{
	"method": "message.operatorSend",
	"params": [
		"message"
	],
	"id": 1
}
```
4.1.1 Если аргументы params не соответствуют ожидаемому типу, отправить сообщение об ошибке:

```json
{
	"error": {
		"code": 1,
		"message": "Incorrect argument."
	},
	"result": null,
	"id": 1
}
```
4.1.2. Прекратить обработку запроса

4.2.1. Сохранить сообщение в таблицу.

4.2.2. Отправить сообщение о положительном выполнении:

```json
{
	"error": null,
	"result": { 
		"message":
			{
				"message_id": 3,
				"message_text": "Message from Client",
				"invader_name": "",
				"is_from_support": false,
				"created": 1678794555
			}
	},
	"id": 3
}
```
4.3. Перейти к пункту 4.



**5. Обработка запроса на авторизацию.**

5.1. С фронта приходит http POST запрос на /v1/auth/.

5.1.1 В теле запроса ожидается json объект формата:

```json
{
	"user_name": "user1",
	"password": "password1"
}
```

5.1.1.1. Если в теле есть объект в указанном формате, но передаваемые данные не соответствуют записи о пользователе:

Вернуть пустой ответ со статус кодом 403.

5.1.1.2. Если в теле есть объект в указанном формате и переданный данные соответствуют записи о пользователе:

Вернуть json объект с токином для данного клиента со статус кодом 200:

```json
{
	"error": null,
	"result": {
		"token": "QWERTYUIOP"
	},
	"id": 6
}
```

5.1.1.3. Во всех остальных случаях вернуть пустой ответ со статус кодом 404.
