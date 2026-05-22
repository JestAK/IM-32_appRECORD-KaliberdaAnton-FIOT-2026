## 1. Тема, Мета, Місце розташування

**1.1 Тема**
Розширені можливості Node.js-додатків: логування, завантаження файлів, моніторинг продуктивності.

**1.2 Мета**
Навчитись логувати запити і події через Morgan та Winston, налаштувати завантаження файлів на сервер за допомогою Multer з валідацією, додати моніторинг продуктивності застосунку та запустити сервер через менеджер процесів PM2.

**1.3 Місце розташування**

- GitHub репозиторій веб-застосунку: [посилання](https://github.com/JestAK/fandub-space)
- Live demo веб-застосунку (Frontend): [посилання](https://fandub-space-1.onrender.com)
- Live demo веб-застосунку (Backend): [посилання](https://fandub-space.onrender.com)
- GitHub репозиторій звіту: [посилання](https://github.com/JestAK/IM-32_appRECORD-KaliberdaAnton-FIOT-2026)
- Live demo звіту: [посилання]()

---

## 2. Виконання роботи

**2.1 Встановлення бібліотек**

До проєкту FanDub Space додано пакети `morgan`, `winston`, `multer` та `PM2`.

```json
"dependencies": {
    "bcrypt": "^6.0.0",
    "cors": "^2.8.6",
    "dotenv": "^17.4.2",
    "express": "^5.2.1",
    "express-rate-limit": "^8.5.2",
    "jsonwebtoken": "^9.0.3",
    "morgan": "^1.10.1",
    "multer": "^2.1.1",
    "mysql2": "^3.22.3",
    "passport": "^0.7.0",
    "passport-google-oauth20": "^2.0.0",
    "sequelize": "^6.37.8",
    "winston": "^3.19.0"
}
```

**2.2 Логування HTTP-запитів через Morgan**

Morgan підключений як глобальний middleware і виводить у консоль метод, шлях, HTTP-код та час обробки кожного запиту. Використано пресет `dev`.

```js
const morgan = require("morgan");

app.use(morgan("dev"));
```

**2.3 Логування подій через Winston**

У файлі `config/logger.js` створено інстанс Winston з двома транспортами файловим та консольним. Формат включає мітку часу і JSON-структуру.

```js
const winston = require("winston");
const path = require("path");

const logger = winston.createLogger({
    level: "info",
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
    ),
    transports: [
        new winston.transports.File({
            filename: path.join(__dirname, "../app.log"),
        }),
        new winston.transports.Console({
            format: winston.format.combine(
                winston.format.colorize(),
                winston.format.simple(),
            ),
        }),
    ],
});

module.exports = logger;
```

Приклад запису у `app.log`:

```json
{"level":"info","message":"GET /status - 200 - 5.463ms","timestamp":"2026-05-22T01:30:53.733Z"}
{"level":"info","message":"GET /status - 200 - 0.966ms","timestamp":"2026-05-22T01:31:07.857Z"}
```

**2.4 Вимірювання часу відповіді**

Middleware `responseTimeMiddleware` фіксує час старту запиту через `process.hrtime()`, а за подією `res.on('finish')` обчислює тривалість обробки та записує її через Winston. У `app.log` потрапляє інформація про метод, URL, HTTP-статус і час обробки.

```js
const responseTimeMiddleware = (req, res, next) => {
    const start = process.hrtime();
    res.on("finish", () => {
        const diff = process.hrtime(start);
        const timeInMs = (diff[0] * 1e3 + diff[1] * 1e-6).toFixed(3);
        logger.info(
            `${req.method} ${req.originalUrl} - ${res.statusCode} - ${timeInMs}ms`,
        );
    });
    next();
};
```

**2.5 Обробка помилок**

Глобальний error-handler підключений останнім у ланцюгу middleware. Він логує помилку через Winston, окремо обробляє `MulterError` та повертає клієнту JSON-відповідь з відповідним HTTP-кодом.

```js
const errorHandlerMiddleware = (err, req, res, next) => {
    logger.error(`${req.method} ${req.originalUrl} - Error: ${err.message}`);

    if (err instanceof require("multer").MulterError) {
        return res.status(400).json({ error: `Upload error: ${err.message}` });
    }

    res.status(500).json({ error: err.message || "Internal Server Error" });
};
```

**2.6 Налаштування Multer**

У файлі `config/multer.js` сконфігуровано `diskStorage` при старті створюється папка `uploads/`, якщо її ще немає. Кожен файл отримує унікальне ім'я з міткою часу та випадковим суфіксом. Через `fileFilter` дозволено лише `jpg`, `jpeg`, `png` та `pdf`, а параметр `limits.fileSize` обмежує розмір файлу 5 МБ.

```js
const multer = require("multer");
const path = require("path");
const fs = require("fs");

const uploadDir = path.join(__dirname, "../uploads");
if (!fs.existsSync(uploadDir)) {
    fs.mkdirSync(uploadDir, { recursive: true });
}

const storage = multer.diskStorage({
    destination: function (req, file, cb) {
        cb(null, uploadDir);
    },
    filename: function (req, file, cb) {
        const uniqueSuffix = Date.now() + "-" + Math.round(Math.random() * 1e9);
        cb(
            null,
            file.fieldname +
                "-" +
                uniqueSuffix +
                path.extname(file.originalname),
        );
    },
});

const fileFilter = (req, file, cb) => {
    const allowedTypes = /jpeg|jpg|png|pdf/;
    const extname = allowedTypes.test(
        path.extname(file.originalname).toLowerCase(),
    );
    const mimetype = allowedTypes.test(file.mimetype);

    if (extname && mimetype) {
        return cb(null, true);
    } else {
        cb(new Error("Invalid file type. Only jpg, png, and pdf are allowed."));
    }
};

const upload = multer({
    storage: storage,
    limits: { fileSize: 5 * 1024 * 1024 },
    fileFilter: fileFilter,
});

module.exports = upload;
```

**2.7 Завантаження одного та кількох файлів**

Контролер `systemController` має два обробники: `uploadSingleFile` для одного файлу та `uploadMultipleFiles` для масиву до 5 файлів. Обидва перевіряють наявність файлу, логують успіх через Winston і повертають клієнту метадані з Multer.

```js
const uploadSingleFile = (req, res) => {
    if (!req.file) {
        return res.status(400).json({ error: "No file uploaded" });
    }
    logger.info(`File uploaded successfully: ${req.file.filename}`);
    res.json({
        message: "File uploaded successfully",
        file: req.file,
    });
};

const uploadMultipleFiles = (req, res) => {
    if (!req.files || req.files.length === 0) {
        return res.status(400).json({ error: "No files uploaded" });
    }
    logger.info(`${req.files.length} files uploaded successfully`);
    res.json({
        message: "Files uploaded successfully",
        files: req.files,
    });
};
```

Маршрути:

```js
app.post("/upload", upload.single("file"), systemController.uploadSingleFile);
app.post(
    "/upload-multiple",
    upload.array("files", 5),
    systemController.uploadMultipleFiles,
);
```

**2.8 Моніторинг стану сервера**

Endpoint `/status` повертає `uptime` та структуру `process.memoryUsage()` з полями `rss`, `heapTotal`, `heapUsed`, `external`.

```js
const getStatus = (req, res) => {
    const status = {
        uptime: process.uptime(),
        memoryUsage: process.memoryUsage(),
        timestamp: new Date(),
    };
    res.json(status);
};
```

**2.9 Запуск через PM2**

Замість `node server.js` сервер запускається через PM2. Менеджер процесів контролює сервер у фоні, автоматично перезапускає його при падінні та дає інструменти моніторингу: `pm2 list` показує статус і метрики, `pm2 logs` транслює логи, `pm2 monit` відкриває інтерактивну панель з CPU/RAM.

```bash
pm2 start server.js
pm2 list
pm2 logs server
pm2 monit
```

---

## 3. Скріншоти

![Скрін 1](/assets/labs/lab-4/screen-1.png)
Morgan виводить запити у консоль

![Скрін 2](/assets/labs/lab-4/screen-2.png)
GET /status повернення `uptime` та `memoryUsage` процесу

![Скрін 3](/assets/labs/lab-4/screen-3.png)
POST /upload успішне завантаження одного файлу та відповідь з метаданими

![Скрін 4](/assets/labs/lab-4/screen-4.png)
POST /upload-multiple завантаження кількох файлів одночасно

![Скрін 5](/assets/labs/lab-4/screen-5.png)
POST /upload з помилкою через неправильний тип

![Скрін 6](/assets/labs/lab-4/screen-6.png)
Логи у файлі `app.log`

![Скрін 7](/assets/labs/lab-4/screen-7.png)
`pm2 monit` моніторинг процесу

---

## Висновки

У ході виконання лабораторної роботи №4 бекенд проєкт FanDub Space розширено новими можливостями. Підключено Morgan для логування HTTP-запитів у консоль та Winston для структурованого JSON-логування у файл `app.log` з мітками часу і рівнями `info` та `error`. Реалізовано глобальний middleware обробки помилок з окремою логікою для `MulterError`. Для завантаження файлів сконфігуровано Multer з валідацією типів: `jpg`, `jpeg`, `png`, `pdf`, та обмеженням розміру 5 МБ. Додано `/upload` та `/upload-multiple`. Для моніторингу продуктивності реалізовано `/status` та middleware `responseTimeMiddleware`, який вимірює час обробки кожного запиту і записує його у Winston. Застосунок запускається через PM2 з автоматичним рестартом та моніторингом у реальному часі.

---

## Перелік використаних джерел

https://expressjs.com/en/resources/middleware/morgan.html
https://github.com/winstonjs/winston
https://github.com/expressjs/multer
https://nodejs.org/api/process.html
https://pm2.keymetrics.io/docs/usage/quick-start/
https://expressjs.com/en/guide/error-handling.html
