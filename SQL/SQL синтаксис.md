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

