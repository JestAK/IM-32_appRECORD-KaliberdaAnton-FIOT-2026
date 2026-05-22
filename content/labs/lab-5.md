## 1. Тема, Мета, Місце розташування

**1.1 Тема**
Безпека та продуктивність серверних додатків. Безпека Node.js-додатків, оптимізація запитів і кешування, тестування API.

**1.2 Мета**
Навчитись забезпечувати базову безпеку Node.js-додатків через Helmet, rate-limit та валідацію даних, використовувати кешування для зменшення навантаження на сервер, оптимізувати маршрути REST API, виконувати автоматизоване тестування API за допомогою Jest та Supertest, а також аналізувати продуктивність backend-застосунку.

**1.3 Місце розташування**

- GitHub репозиторій веб-застосунку: [посилання](https://github.com/JestAK/fandub-space)
- Live demo веб-застосунку (Frontend): [посилання](https://fandub-space-1.onrender.com)
- Live demo веб-застосунку (Backend): [посилання](https://fandub-space.onrender.com)
- GitHub репозиторій звіту: [посилання](https://github.com/JestAK/IM-32_appRECORD-KaliberdaAnton-FIOT-2026)
- Live demo звіту: [посилання]()

---

## 2. Виконання роботи

**2.1 Встановлення бібліотек**

До проєкту FanDub Space додано пакети `helmet`, `express-validator`, `node-cache`, `jest` та `supertest`.

```json
"dependencies": {
    "bcrypt": "^6.0.0",
    "cors": "^2.8.6",
    "dotenv": "^17.4.2",
    "express": "^5.2.1",
    "express-rate-limit": "^8.5.2",
    "express-validator": "^7.3.2",
    "helmet": "^8.2.0",
    "jsonwebtoken": "^9.0.3",
    "morgan": "^1.10.1",
    "multer": "^2.1.1",
    "mysql2": "^3.22.3",
    "node-cache": "^5.1.2",
    "passport": "^0.7.0",
    "passport-google-oauth20": "^2.0.0",
    "sequelize": "^6.37.8",
    "winston": "^3.19.0"
},
"devDependencies": {
    "jest": "^30.4.2",
    "supertest": "^7.2.2"
}
```

**2.2 Обмеження кількості запитів (rate limiting)**

Глобальний `limiter` обмежує кількість запитів з однієї IP-адреси: 20 запитів на хвилину.

```js
const rateLimit = require("express-rate-limit");

const limiter = rateLimit({
    windowMs: 1 * 60 * 1000,
    max: 20,
    message: { error: "Too many attempts, please try again later" },
});

app.use(limiter);
```

**2.3 Валідація даних через express-validator**

Усі маршрути, що приймають дані від клієнта, перевіряються валідацією з `express-validator`. Перевіряється формат email, довжина пароля, дозволені значення ролі, обов'язковість полів.

```js
const { body, validationResult } = require("express-validator");

app.post(
    "/register",
    body("email")
        .isEmail()
        .withMessage(
            "Invalid email format. Please enter a valid address (e.g., user@example.com)",
        ),
    body("password")
        .isLength({ min: 6 })
        .withMessage("Password must be at least 6 characters long"),
    body("name").notEmpty().withMessage("Name is required"),
    body("role")
        .isIn(["actor", "translator", "sound"])
        .withMessage(
            "Invalid role. Allowed roles are: actor, translator, sound",
        ),
    (req, res) => {
        const errors = validationResult(req);
        if (!errors.isEmpty()) {
            return res.status(400).json({ errors: errors.array() });
        }
        userController.registerUser(req, res);
    },
);
```

**2.4 Кешування відповідей через node-cache**

У файлі `middlewares.js` створено інстанс `NodeCache` з TTL 60 секунд та middleware `cacheMiddleware`. Якщо для поточного шляху у кеші вже є дані, вони повертаються одразу без виклику контролера.

```js
const NodeCache = require("node-cache");

const localCache = new NodeCache({ stdTTL: 60 });
const cacheMiddleware = (req, res, next) => {
    const key = req.originalUrl;
    const cachedData = localCache.get(key);

    if (cachedData) {
        return res.json(JSON.parse(cachedData));
    }

    res.sendResponse = res.json;
    res.json = (body) => {
        localCache.set(key, JSON.stringify(body));
        res.sendResponse(body);
    };

    next();
};
```

**2.5 Автоматизоване тестування API через Jest та Supertest**

У файлі `test/server.test.js` створено два тестові блоки: один по 10 разів звертається до кешованого `GET /posts`, другий до некешованого `GET /posts-non-opt`. Це дозволяє і перевірити коректність відповідей і одночасно зробити заміри продуктивності.

```js
const request = require("supertest");

describe("API with cache", () => {
    test("GET /post-non-opt", async () => {
        for (let i = 0; i < 10; i++) {
            const res = await request("http://localhost:3000").get(
                "/posts-non-opt",
            );
            expect(res.statusCode).toBe(200);
        }
    });

    test("GET /post", async () => {
        for (let i = 0; i < 10; i++) {
            const res = await request("http://localhost:3000").get("/posts");
            expect(res.statusCode).toBe(200);
        }
    });
});
```

---

## 3. Скріншоти

![Скрін 1](/assets/labs/lab-5/screen-1.png)
Відповідь сервера з захисними HTTP-заголовками від Helmet

![Скрін 2](/assets/labs/lab-5/screen-2.png)
POST /register з некоректними даними і відповідь з масивом помилок

![Скрін 3](/assets/labs/lab-5/screen-3.png)
![Скрін 4](/assets/labs/lab-5/screen-4.png)
GET /posts: перший запит з БД (повільніше) та наступні з кешу (швидше)

![Скрін 5](/assets/labs/lab-5/screen-5.png)
Вивід Jest з результатами тестів обох маршрутів

---

## Висновки

У ході виконання лабораторної роботи №5 бекенд проєкту FanDub Space розширено безпекою, кешуванням та тестуваннями. Для захисту додано Helmet, глобальний rate-limiter та валідацію даних через express-validator. Для оптимізації продуктивності реалізовано middleware `cacheMiddleware` та застосовано його до шляху `GET /posts`, також є шлях без кешування `GET /posts-non-opt`. Для тестування написано тести з 10 запитами до кешованого та некешованого шляху, що дозволяє оцінити різницю в продуктивності. І врешті кешований шлях відповідає значно швидше.

---

## Перелік використаних джерел

https://helmetjs.github.io/
https://express-validator.github.io/docs/
https://github.com/node-cache/node-cache
https://jestjs.io/docs/getting-started
https://github.com/ladjs/supertest
https://expressjs.com/en/advanced/best-practice-security.html
