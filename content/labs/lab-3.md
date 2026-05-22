## 1. Тема, Мета, Місце розташування

**1.1 Тема**
Розробка функціонального REST API. Реєстрація та авторизація користувачів. Валідація даних і обробка помилок.

**1.2 Мета**
Вивчити принципи побудови REST API, набути практичних навичок розробки серверного застосунку на Node.js і Express, реалізувати механізми реєстрації та авторизації користувачів, забезпечити валідацію вхідних даних та обробку помилок, організувати захищений доступ до ресурсів із використанням JWT-токенів та системи ролей.

**1.3 Місце розташування**

- GitHub репозиторій веб-застосунку: [посилання](https://github.com/JestAK/fandub-space)
- Live demo веб-застосунку (Frontend): [посилання](https://fandub-space-1.onrender.com)
- Live demo веб-застосунку (Backend): [посилання](https://fandub-space.onrender.com)
- GitHub репозиторій звіту: [посилання](https://github.com/JestAK/IM-32_appRECORD-KaliberdaAnton-FIOT-2026)
- Live demo звіту: [посилання]()

---

## 2. Виконання роботи

**2.1 Встановлення необхідних бібліотек**

Для реалізації REST API, аутентифікації та авторизації у проєкт встановлено набір бібліотек: `express`, `bcrypt`, `jsonwebtoken`, `express-rate-limit`, `passport`, `passport-google-oauth20`, `cors`, `dotenv`.

```json
"dependencies": {
    "bcrypt": "^6.0.0",
    "cors": "^2.8.6",
    "dotenv": "^17.4.2",
    "express": "^5.2.1",
    "express-rate-limit": "^8.5.2",
    "jsonwebtoken": "^9.0.3",
    "mysql2": "^3.22.3",
    "passport": "^0.7.0",
    "passport-google-oauth20": "^2.0.0",
    "sequelize": "^6.37.8"
}
```

**2.2 Реєстрація користувача з валідацією та підтвердженням пароля**

Реєстрація реалізована у контролері `userController`. Виконується валідація обов'язкових полів, перевірка формату email через регулярний вираз, перевірка довжини пароля та збіг пароля з полем підтвердження. Допустимі ролі обмежені переліком `['actor', 'translator', 'sound']`. Пароль хешується через `bcrypt` перед збереженням у БД.

```js
const registerUser = async (req, res) => {
    try {
        const { email, password, confirmedPassword, name, role } = req.body;

        if (!email || !password || !confirmedPassword || !name || !role) {
            return res.status(400).json({ error: "Fill necessary data" });
        }

        const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
        if (!emailRegex.test(email)) {
            return res.status(400).json({
                error: "Invalid email format. Please enter a valid address (e.g., user@example.com)",
            });
        }

        if (password.length < 6) {
            return res
                .status(400)
                .json({ error: "Password must be at least 6 characters long" });
        }

        if (password !== confirmedPassword) {
            return res.status(400).json({ error: "Passwords do not match" });
        }

        const allowedRoles = ["actor", "translator", "sound"];
        if (!allowedRoles.includes(role)) {
            return res.status(400).json({
                error: `Invalid role. Allowed roles are: ${allowedRoles.join(", ")}`,
            });
        }

        const hashedPassword = await bcrypt.hash(password, 10);
        const user = await createUser({
            email,
            password: hashedPassword,
            name,
            role,
            isAdmin: false,
        });
        const userResponse = user.toJSON();
        delete userResponse.password;
        res.status(201).json(userResponse);
    } catch (error) {
        if (error.message === "User with this email already exists") {
            return res.status(400).json({ error: error.message });
        }
        console.error("Registration Error:", error);
        res.status(500).json({ error: error.message });
    }
};
```

**2.3 Авторизація користувача, JWT-токени та обмеження спроб входу**

При успішному вході сервер генерує два токени: короткоживучий Access Token для авторизації запитів та Refresh Token для оновлення сесії без повторного введення пароля. Refresh token зберігається в окремій таблиці БД. Для захисту від брутфорсу маршрут `/login` обмежений через `express-rate-limit` — максимум 5 спроб на 15 хвилин з однієї IP-адреси.

```js
const loginLimiter = rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 5,
    message: { error: "Too many login attempts, please try again later" },
});

app.post("/login", loginLimiter, userController.loginUser);
```

```js
const loginUser = async (req, res) => {
    try {
        const { email, password } = req.body;

        if (!email || !password) {
            return res
                .status(400)
                .json({ error: "Email and password are required" });
        }

        const user = await User.findOne({ where: { email } });
        if (!user) {
            return res.status(401).json({ error: "Invalid credentials" });
        }

        const isPasswordValid = await bcrypt.compare(password, user.password);
        if (!isPasswordValid) {
            return res.status(401).json({ error: "Invalid credentials" });
        }

        const token = jwt.sign(
            { id: user.id, email: user.email, isAdmin: user.isAdmin },
            ACCESS_SECRET,
            { expiresIn: "1h" },
        );

        const refreshToken = jwt.sign({ id: user.id }, REFRESH_SECRET, {
            expiresIn: "7d",
        });

        const expiryDate = new Date();
        expiryDate.setDate(expiryDate.getDate() + 7);

        await RefreshToken.create({
            token: refreshToken,
            expiresAt: expiryDate,
            UserId: user.id,
        });

        res.json({ token, refreshToken });
    } catch (error) {
        console.error("Login Error:", error);
        res.status(500).json({ error: error.message });
    }
};
```

**2.4 Middleware для перевірки токена та ролі користувача**

`authMiddleware` витягує JWT з заголовка `Authorization`, верифікує його через `jwt.verify` і додає декодовані дані користувача (`id`, `email`, `isAdmin`) до `req.user`. `adminMiddleware` перевіряє наявність прав адміністратора для маршрутів модерації.

```js
const authMiddleware = (req, res, next) => {
    const token = req.headers["authorization"].split(" ")[1];

    if (!token) {
        return res
            .status(401)
            .json({ error: "Access denied. No token provided." });
    }

    try {
        const decoded = jwt.verify(token, ACCESS_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        console.error("Auth Middleware Error:", error.message);
        res.status(401).json({ error: "Invalid or expired token." });
    }
};

const adminMiddleware = (req, res, next) => {
    if (!req.user || !req.user.isAdmin) {
        return res
            .status(403)
            .json({ error: "Forbidden. Admin privileges required." });
    }
    next();
};
```

**2.5 Захищений маршрут отримання профілю**

Маршрут `/profile` доступний лише за наявності валідного Access Token. Контролер повертає дані поточного користувача разом із його постами.

```js
const getUserProfile = async (req, res) => {
    try {
        const userId = req.user.id;
        const user = await User.findByPk(userId, { include: Post });

        if (!user) {
            return res.status(404).json({ error: "User not found" });
        }

        const userResponse = user.toJSON();
        delete userResponse.password;
        res.json(userResponse);
    } catch (error) {
        console.error("Profile Retrieval Error:", error);
        res.status(500).json({ error: error.message });
    }
};
```

**2.6 Refresh token та logout**

Якщо Access Token закінчився, клієнт надсилає Refresh Token на `/refresh-token` і отримує новий Access Token без повторного логіну. При logout запис Refresh Token видаляється з БД.

```js
const refreshToken = async (req, res) => {
    try {
        const { refreshToken } = req.body;

        if (!refreshToken) {
            return res.status(400).json({ error: "Refresh token is required" });
        }

        const storedToken = await RefreshToken.findOne({
            where: { token: refreshToken },
        });
        if (!storedToken) {
            return res.status(401).json({ error: "Invalid refresh token" });
        }

        if (new Date() > storedToken.expiresAt) {
            await storedToken.destroy();
            return res.status(401).json({ error: "Refresh token expired" });
        }

        const decoded = jwt.verify(refreshToken, REFRESH_SECRET);
        const user = await User.findByPk(decoded.id);

        if (!user) {
            return res.status(404).json({ error: "User not found" });
        }

        const newAccessToken = jwt.sign(
            { id: user.id, email: user.email, isAdmin: user.isAdmin },
            ACCESS_SECRET,
            { expiresIn: "1h" },
        );

        res.json({ token: newAccessToken });
    } catch (error) {
        console.error("Refresh Token Error:", error);
        res.status(500).json({ error: error.message });
    }
};

const logoutUser = async (req, res) => {
    try {
        const { refreshToken } = req.body;

        if (!refreshToken) {
            return res.status(400).json({ error: "Refresh token is required" });
        }

        await RefreshToken.destroy({ where: { token: refreshToken } });
        res.json({ message: "Logged out successfully" });
    } catch (error) {
        console.error("Logout Error:", error);
        res.status(500).json({ error: error.message });
    }
};
```

**2.7 Оновлення профілю, зміна пароля та видалення користувача**

Користувач може оновити власні `name` та `role`, змінити пароль або видалити обліковий запис. Усі ці маршрути захищені `authMiddleware`.

```js
const updateUserProfile = async (req, res) => {
    try {
        const userId = req.user.id;
        const { name, role } = req.body;
        const fieldsToUpdate = {};

        if (name) fieldsToUpdate.name = name;

        if (role) {
            const allowedRoles = ["actor", "translator", "sound"];
            if (!allowedRoles.includes(role)) {
                return res.status(400).json({
                    error: `Invalid role. Allowed roles are: ${allowedRoles.join(", ")}`,
                });
            }
            fieldsToUpdate.role = role;
        }

        if (Object.keys(fieldsToUpdate).length === 0) {
            return res
                .status(400)
                .json({ error: "Please provide name or role to update" });
        }

        const updatedUser = await updateUser(userId, fieldsToUpdate);
        res.json({
            message: "Profile updated successfully",
            user: updatedUser,
        });
    } catch (error) {
        console.error("Update User Error:", error);
        if (error.message === "User not found") {
            return res.status(404).json({ error: error.message });
        }
        res.status(500).json({ error: error.message });
    }
};

const changeUserPassword = async (req, res) => {
    try {
        const userId = req.user.id;
        const { oldPassword, newPassword, confirmedNewPassword } = req.body;

        if (!oldPassword || !newPassword || !confirmedNewPassword) {
            return res
                .status(400)
                .json({ error: "Please fill in all password fields" });
        }

        if (newPassword.length < 6) {
            return res.status(400).json({
                error: "New password must be at least 6 characters long",
            });
        }

        if (newPassword !== confirmedNewPassword) {
            return res
                .status(400)
                .json({ error: "New passwords do not match" });
        }

        if (oldPassword === newPassword) {
            return res.status(400).json({
                error: "New password cannot be the same as the old password",
            });
        }

        await changePassword(userId, oldPassword, newPassword);
        res.json({ message: "Password changed successfully" });
    } catch (error) {
        console.error("Change Password Error:", error);
        if (error.message === "User not found")
            return res.status(404).json({ error: error.message });
        if (error.message === "Incorrect old password")
            return res.status(400).json({ error: error.message });
        res.status(500).json({ error: error.message });
    }
};

const deleteUserProfile = async (req, res) => {
    try {
        const userId = req.user.id;
        await deleteUser(userId);
        res.json({
            message:
                "User account and all associated data deleted successfully",
        });
    } catch (error) {
        console.error("Delete User Error:", error);
        if (error.message === "User not found") {
            return res.status(404).json({ error: error.message });
        }
        res.status(500).json({ error: error.message });
    }
};
```

**2.8 Система ролей користувачів та модерація постів**

У моделі `User` присутні ролі `actor` / `translator` / `sound` та привілеї `isAdmin`. Адмін-маршрути перевіряють `req.user.isAdmin` і дозволяють переглядати пости зі статусом `pending` та модерувати їх `approved` / `rejected`.

```js
const moderateUserPost = async (req, res) => {
    try {
        if (!req.user.isAdmin) {
            return res
                .status(403)
                .json({ error: "Access denied. Admins only." });
        }

        const { id } = req.params;
        const { status } = req.body;

        if (!["approved", "rejected"].includes(status)) {
            return res
                .status(400)
                .json({ error: "Invalid status. Choose approved or rejected" });
        }

        const post = await updatePostStatus(id, status);
        res.json({
            message: `Post status successfully updated to ${status}`,
            post,
        });
    } catch (error) {
        if (error.message === "Post not found") {
            return res.status(404).json({ error: error.message });
        }
        res.status(500).json({ error: error.message });
    }
};
```

**2.9 OAuth-аутентифікація через Google**

Окрім класичної реєстрації за email/password, реалізовано вхід через Google за допомогою `passport-google-oauth20`. Стратегія перевіряє, чи існує користувач із таким email, і за необхідності створює нового. Після успішної аутентифікації сервер генерує власні JWT-токени та редіректить користувача на фронтенд із токенами в query-параметрах.

```js
passport.use(
    new GoogleStrategy(
        {
            clientID: process.env.GOOGLE_CLIENT_ID,
            clientSecret: process.env.GOOGLE_CLIENT_SECRET,
            callbackURL: `${process.env.BASE_URL}/auth/google/callback`,
        },
        async (accessToken, refreshToken, profile, done) => {
            try {
                const email = profile.emails[0].value;
                const name = profile.displayName;

                let user = await User.findOne({ where: { email } });
                const generatedPassword = Math.random().toString(36).slice(-8);

                if (!user) {
                    user = await User.create({
                        email: email,
                        name: name,
                        role: "actor",
                        password: generatedPassword,
                        isAdmin: false,
                    });
                }

                return done(null, user);
            } catch (error) {
                return done(error, null);
            }
        },
    ),
);
```

**2.10 Обробка помилок та логування**

У кожному контролері критична логіка обгорнута в `try/catch`. Очікувані помилки повертають конкретні HTTP-коди, а несподівані падіння логуються через `console.error` та повертають `500`.

```js
} catch (error) {
    console.error("Registration Error:", error);
    res.status(500).json({ error: error.message });
}
```

**2.11 Маршрути сервера**

У `server.js` усі маршрути авторизації та профілю підключаються до контролерів, а захищені обгорнуті в `authMiddleware`.

```js
app.post("/register", userController.registerUser);
app.post("/login", loginLimiter, userController.loginUser);
app.get("/profile", authMiddleware, userController.getUserProfile);
app.post("/refresh-token", userController.refreshToken);
app.post("/logout", userController.logoutUser);
app.post("/profile/update", authMiddleware, userController.updateUserProfile);
app.post("/profile/delete", authMiddleware, userController.deleteUserProfile);
app.post(
    "/profile/change-password",
    authMiddleware,
    userController.changeUserPassword,
);

app.get(
    "/auth/google",
    passport.authenticate("google", { scope: ["profile", "email"] }),
);
app.get(
    "/auth/google/callback",
    passport.authenticate("google", {
        session: false,
        failureRedirect: `${process.env.FRONTEND_URL}/login?error=google_failed`,
    }),
    userController.googleAuthCallback,
);

app.post("/posts", authMiddleware, postController.createUserPost);
app.get("/posts", postController.getApprovedUserPosts);
app.put("/posts/:id", authMiddleware, postController.updateUserPost);
app.delete("/posts/:id", authMiddleware, postController.deleteUserPost);

app.get(
    "/admin/posts/pending",
    authMiddleware,
    postController.getPendingUserPosts,
);
app.put(
    "/admin/posts/:id/moderate",
    authMiddleware,
    postController.moderateUserPost,
);
```

---

## 3. Скріншоти

![Скрін 1](/assets/labs/lab-3/screen-1.png)
POST /register успішна реєстрація користувача з валідацією полів та підтвердженням пароля

![Скрін 2](/assets/labs/lab-3/screen-2.png)
POST /login успішний вхід: отримання Access Token та Refresh Token

![Скрін 3](/assets/labs/lab-3/screen-3.png)
POST /login спрацювання rate-limiter після 5 невдалих спроб входу

![Скрін 4](/assets/labs/lab-3/screen-4.png)
GET /profile звернення до захищеного маршруту з валідним токеном (`Authorization: Bearer ...`)

![Скрін 5](/assets/labs/lab-3/screen-5.png)
POST /refresh-token отримання нового Access Token за допомогою Refresh Token

![Скрін 6](/assets/labs/lab-3/screen-6.png)
POST /profile/change-password зміна пароля з перевіркою старого

![Скрін 7](/assets/labs/lab-3/screen-7.png)
POST /logout вихід

---

## Висновки

У ході виконання лабораторної роботи на Express побудовано REST API для проєкту FanDub Space, що включає: реєстрацію з валідацією і підтвердженням пароля, авторизацію з JWT, оновлення профілю, зміну пароля, видалення облікового запису та вихід з акаунту. Реалізовано систему ролей із розмежуванням прав звичайних користувачів та адміністраторів, додано middleware для перевірки токена, обмеження кількості спроб входу через `express-rate-limit`, а також альтернативний спосіб входу через Google OAuth.

---

## Перелік використаних джерел

https://expressjs.com/en/guide/routing.html
https://jwt.io/introduction
https://github.com/auth0/node-jsonwebtoken
https://www.npmjs.com/package/bcrypt
https://www.npmjs.com/package/express-rate-limit
https://www.passportjs.org/packages/passport-google-oauth20/
https://developer.mozilla.org/en-US/docs/Web/HTTP/Status
