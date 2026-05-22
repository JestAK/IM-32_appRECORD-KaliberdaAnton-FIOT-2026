## 1. Тема, Мета, Місце розташування

**1.1 Тема**
Документування API за допомогою Swagger. Деплой Node.js-додатку.

**1.2 Мета**
Навчитись документувати REST API через Swagger/OpenAPI, інтегрувати Swagger UI у Node.js + Express застосунок, описувати endpoint, параметри та схеми даних, виконувати деплой Node.js-додатку на хмарний сервіс та тестувати API через Swagger UI.

**1.3 Місце розташування**

- GitHub репозиторій веб-застосунку: [посилання](https://github.com/JestAK/fandub-space)
- Live demo веб-застосунку (Frontend): [посилання](https://fandub-space-1.onrender.com)
- Live demo веб-застосунку (Backend): [посилання](https://fandub-space.onrender.com)
- Swagger документація: [посилання](https://fandub-space.onrender.com/docs)
- GitHub репозиторій звіту: [посилання](https://github.com/JestAK/IM-32_appRECORD-KaliberdaAnton-FIOT-2026)
- Live demo звіту: [посилання]()

---

## 2. Виконання роботи

**2.1 Встановлення бібліотек**

До проєкту додано пакети `swagger-jsdoc` та `swagger-ui-express`.

```json
"dependencies": {
    "swagger-jsdoc": "^6.2.8",
    "swagger-ui-express": "^5.0.1"
}
```

**2.2 Підключення Swagger UI**

У `server.js` Swagger UI підключається до маршруту `/docs`. Після цього документація доступна за шляхом `GET /docs`.

```js
const { swaggerUi, specs } = require("./config/swagger");

app.use("/docs", swaggerUi.serve, swaggerUi.setup(specs));
```

У документації вказана інформація по всіх наявних шляхах додатка.

**2.3 Деплой**

Деплой виконано за допомогою сервісів `Render` та `Aiven`. При цьому фронтенд, бекенд і бази даних знаходяться на окремих серверах.

---

## 3. Скріншоти

![Скрін 1](/assets/labs/lab-6/screen-1.png)
Swagger UI

![Скрін 2](/assets/labs/lab-6/screen-2.png)
![Скрін 3](/assets/labs/lab-6/screen-3.png)
Render Dashboard з серверами фронтенду та бекенду

![Скрін 4](/assets/labs/lab-6/screen-4.png)
Aiven Dashboard з сервером БД

---

## Висновки

У ході виконання лабораторної роботи №6 бекенд проєкту FanDub Space створено інтерактивну документацію по всіх наявних адресах веб-застосунку за допомогою SwaggerUI/OpenAPI. Також виконано деплой застосунку на хмарних сервісах `Render` та `Aiven`, де фронтенд та бекенд знаходяться на сервісі `Render`, але в окремих інстансах, а на сервісі `Aiven` знахоиться БД.

---

## Перелік використаних джерел

https://swagger.io/specification/
https://github.com/Surnet/swagger-jsdoc
https://github.com/scottie1984/swagger-ui-express
