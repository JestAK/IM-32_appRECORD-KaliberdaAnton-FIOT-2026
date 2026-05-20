## 1. Тема, Мета, Місце розташування

**1.1 Тема**
Створення бази даних у MySQL. Підключення Node.js до MySQL. Робота з ORM Sequelize.

**1.2 Мета**
Навчитися створювати базу даних у MySQL, виконувати SQL-запити (SELECT, INSERT, UPDATE, DELETE), підключити серверну програму на Node.js до бази даних через драйвер `mysql2`, використати ORM Sequelize для роботи з БД та реалізувати зв'язок One-to-Many між таблицями `Users` та `Posts`.

**1.3 Місце розташування**

- GitHub репозиторій веб-застосунку: [посилання](https://github.com/JestAK/fandub-space)
- Live demo веб-застосунку (Frontend): [посилання](https://fandub-space-1.onrender.com)
- Live demo веб-застосунку (Backend): [посилання](https://fandub-space.onrender.com)
- GitHub репозиторій звіту: [посилання](https://github.com/JestAK/IM-32_appRECORD-KaliberdaAnton-FIOT-2026)
- Live demo звіту: [посилання]()

---

## 2. Виконання роботи

**2.1 Створення бази даних та таблиць у MySQL**

Для проєкту створено базу даних `fandub-space`. Таблиці `Users` та `Posts` створюються автоматично через `sequelize.sync({ alter: true })` при старті сервера.

**2.2 Підключення Node.js до MySQL через драйвер mysql2**

Підключення винесене у файл `config/database.js`. Креденшіали зчитуються зі змінних оточення через `dotenv`, що дозволяє не зберігати їх у репозиторії.

```js
require("dotenv").config();
const { Sequelize } = require("sequelize");

const sequelize = new Sequelize(
    process.env.DB_NAME,
    process.env.DB_USER,
    process.env.DB_PASSWORD,
    {
        host: "localhost",
        dialect: "mysql",
    },
);

module.exports = sequelize;
```

**2.3 SQL-запити через mysql2 (SELECT, INSERT, UPDATE, DELETE)**

Для демонстрації виконання SQL-запитів безпосередньо через драйвер `mysql2` (без ORM) створено окремий файл `sqlTest.js`. Він послідовно виконує INSERT → SELECT → UPDATE → SELECT → DELETE для таблиці `Users` та виводить результати в консоль. Використовуються prepared statements (`?`-плейсхолдери) для безпечної передачі параметрів та захисту від SQL-ін'єкцій.

```js
const mysql = require("mysql2/promise");
require("dotenv").config();

async function runTests() {
    console.log("--- Starting DB connection test ---");

    const connection = await mysql.createConnection({
        host: "localhost",
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD,
        database: process.env.DB_NAME,
    });
    console.log("Connected successfully!");

    console.log("--- INSERT ---");
    const [insertRes] = await connection.execute(
        "INSERT INTO Users (name, role, createdAt, updatedAt) VALUES (?, ?, NOW(), NOW())",
        ["Test User", "translator"],
    );
    const userId = insertRes.insertId;
    console.log(`User inserted with ID: ${userId}`);

    console.log("--- SELECT ---");
    const [rows] = await connection.execute(
        "SELECT id, name, role FROM Users WHERE id = ?",
        [userId],
    );
    console.log("User data from DB:", rows);

    console.log("--- UPDATE ---");
    await connection.execute(
        "UPDATE Users SET name = ?, role = ?, updatedAt = NOW() WHERE id = ?",
        ["Updated Name", "actor", userId],
    );

    const [updatedRows] = await connection.execute(
        "SELECT id, name, role FROM Users WHERE id = ?",
        [userId],
    );
    console.log("Data after update:", updatedRows);

    console.log("--- DELETE ---");
    await connection.execute("DELETE FROM Users WHERE id = ?", [userId]);

    const [checkRows] = await connection.execute(
        "SELECT id FROM Users WHERE id = ?",
        [userId],
    );
    if (checkRows.length === 0) {
        console.log("User deleted successfully!");
    } else {
        console.log("Error: User still exists.");
    }

    await connection.end();
    console.log("--- Test finished, connection closed ---");
}

runTests();
```

**2.4 Модель User**

Модель `User` описує учасника фандаб-спільноти. Поле `role` обмежене переліком значень через `ENUM`.

```js
const { DataTypes } = require("sequelize");
const sequelize = require("../config/database");

const User = sequelize.define("User", {
    name: {
        type: DataTypes.STRING,
        allowNull: false,
    },
    role: {
        type: DataTypes.ENUM,
        values: ["actor", "translator", "sound"],
    },
});

module.exports = User;
```

**2.5 Модель Post та зв'язок One-to-Many**

Модель `Post` описує роботу учасника. Зв'язок між `User` та `Post` оголошується відразу у файлі моделі: `User.hasMany(Post)` та `Post.belongsTo(User)`. Sequelize автоматично додає поле `UserId` до таблиці `Posts`.

```js
const { DataTypes } = require("sequelize");
const sequelize = require("../config/database");
const User = require("./User");

const Post = sequelize.define("Post", {
    title: {
        type: DataTypes.STRING,
        allowNull: false,
    },
    content: {
        type: DataTypes.TEXT,
        allowNull: false,
    },
});

User.hasMany(Post);
Post.belongsTo(User);

module.exports = Post;
```

**2.6 Контролери**

CRUD-операції винесені в контролери, аби розвантажити `server.js` та дотриматись принципу розділення відповідальностей.

`controllers/userController.js`:

```js
const User = require("../models/User");
const Post = require("../models/Post");

const getAllUsers = async (req, res) => {
    try {
        const users = await User.findAll({ include: Post });
        res.json(users);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

const createUser = async (req, res) => {
    try {
        const { name, role } = req.body;

        if (!name || !role) {
            return res
                .status(400)
                .json({ error: "Name and role are required" });
        }

        const allowedRoles = ["actor", "translator", "sound"];
        if (!allowedRoles.includes(role)) {
            return res.status(400).json({
                error: `Invalid role. Allowed roles are: ${allowedRoles.join(", ")}`,
            });
        }

        const user = await User.create({ name, role });
        res.status(201).json(user);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

getUserById = async (req, res) => {
    try {
        const { id } = req.params;
        const user = await User.findByPk(id, { include: Post });

        if (!user) {
            return res.status(404).json({ error: "User not found" });
        }

        res.json(user);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

module.exports = {
    getAllUsers,
    createUser,
    getUserById,
};
```

`controllers/postController.js`:

```js
const Post = require("../models/Post");

const createPost = async (req, res) => {
    try {
        const { title, content, userId } = req.body;
        if (!title || !content || !userId) {
            return res
                .status(400)
                .json({ error: "Title, content and userId are required" });
        }

        const post = await Post.create({ title, content, UserId: userId });
        res.status(201).json(post);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

const updatePost = async (req, res) => {
    try {
        const { id } = req.params;
        const { title, content } = req.body;

        const [updatedRows] = await Post.update(
            { title, content },
            { where: { id } },
        );
        if (updatedRows === 0) {
            return res.status(404).json({ error: "Post not found" });
        }

        res.json({ message: "Post updated" });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

const deletePost = async (req, res) => {
    try {
        const { id } = req.params;
        const deletedRows = await Post.destroy({ where: { id } });
        if (deletedRows === 0) {
            return res.status(404).json({ error: "Post not found" });
        }

        res.json({ message: "Post deleted" });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
};

module.exports = {
    createPost,
    updatePost,
    deletePost,
};
```

**2.7 Маршрути та запуск сервера**

У `server.js` маршрути підключаються до контролерів, а сама синхронізація моделей з БД виконується через `sequelize.sync({ alter: true })`:

```js
const express = require("express");
const sequelize = require("./config/database");
const userController = require("./controllers/userController");
const postController = require("./controllers/postController");
const app = express();
const port = 3000;

app.use(express.json());

sequelize
    .sync({ alter: true })
    .then(() => {
        console.log("Tables created / DB Synced");
    })
    .catch((err) => {
        console.error("Connection error:", err);
    });

app.get("/users", userController.getAllUsers);
app.post("/users", userController.createUser);

app.post("/posts", postController.createPost);
app.put("/posts/:id", postController.updatePost);
app.delete("/posts/:id", postController.deletePost);

app.listen(port, () => {
    console.log(`App listening on port ${port}`);
});
```

---

## 3. Скріншоти

![Скрін 1](/assets/labs/lab-2/screen-1.png)
Запуск сервера: підключення до БД та синхронізація таблиць (`Tables created / DB Synced`)

![Скрін 2](/assets/labs/lab-2/screen-2.png)
Виконання SQL-запитів через драйвер `mysql2` (`sqlTest.js`): INSERT, SELECT, UPDATE, DELETE

![Скрін 3](/assets/labs/lab-2/screen-3.png)
POST /users — створення нового користувача через Sequelize

![Скрін 4](/assets/labs/lab-2/screen-4.png)
GET /users — отримання списку користувачів з їхніми постами (`include: Post`)

![Скрін 5](/assets/labs/lab-2/screen-5.png)
POST /posts — створення поста, прив'язаного до користувача (`UserId`)

![Скрін 6](/assets/labs/lab-2/screen-6.png)
PUT /posts/:id — оновлення поста

![Скрін 7](/assets/labs/lab-2/screen-7.png)
DELETE /posts/:id — видалення поста

---

## Висновки

У ході виконання лабораторної роботи створено базу даних `fandub-space` у MySQL та підключено до неї Node.js-сервер проєкту FanDub Space. Виконано SQL-запити (SELECT, INSERT, UPDATE, DELETE) безпосередньо через драйвер `mysql2` з використанням prepared statements. На основі ORM Sequelize описано моделі `User` та `Post`, реалізовано зв'язок One-to-Many та винесено CRUD-логіку у контролери. Секрети БД винесено у змінні оточення через `dotenv`.

---

## Перелік використаних джерел

https://dev.mysql.com/doc/
https://github.com/sidorares/node-mysql2
https://sequelize.org/docs/v6/
https://expressjs.com/en/guide/routing.html
https://www.npmjs.com/package/dotenv
