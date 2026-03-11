Ниже представлен SQL запрос кода на создание таблицы

```SQL
CREATE TABLE users (
	id BIGINT NOT NULL PRIMARY KEY,
	first_name VARCHAR(64) NOT NULL,
	last_name VARCHAR(64) NOT NULL,
	email VARCHART(128) NOT NULL
);
```

SQL запрос к базе данных на добавление информации в таблицу

```SQL
INSERT INTO users (id, first_name, last_name, email)
VALUES (1, 'Vlad', 'Zagvo', 'example@gmail.com');
```

SQL запрос к базе данных на обновление информации в таблицы

```SQL
UPDATE users SET
email = 'example@outlook.com', first_name = 'Elon'
WHERE id = 1
```

SQL запрос к базе данных на запрос выбора информации в таблице

```SQL
SELECT * FROM users
WHERE id = 1
```

SQL запрос к базе данных на удаление информации в таблице

```SQL
DELETE FROM users
WHERE id = 2 OR id = 3
```

SQL запрос на добавление таблицы и связывание между таблицами 1:1

```SQL
CREATE TABLE price(
	id BIGINT NOT NULL PRIMARY KEY,
	price INT NOT NULL,
	created_at TIMESTAMP DEFAULT now(),
	user_id BIGINT NOT NULL,
	-- Добавление отношения между таблицами
	CONSTRAINT user_id_fk FOREIGN KEY (user_id) REFERENCES users (id)
)
```
