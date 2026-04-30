## 1. Тема, мета та завдання лабораторної роботи

**Тема:** Реєстрація, авторизація та захист маршрутів у REST API з використанням Node.js, Express.js, MySQL, Sequelize ORM та JWT.

**Мета:** реалізувати у backend-проєкті `lab1-rest-api` модуль автентифікації користувачів для системи StudentLab, забезпечити збереження користувачів у базі даних MySQL, хешування паролів, роботу з JWT-токенами, refresh token, захищеними маршрутами, ролями користувачів та базовою обробкою помилок.

**Завдання лабораторної роботи:**
1. Додати модель користувача до існуючого backend-проєкту.
2. Реалізувати реєстрацію та вхід користувача.
3. Забезпечити хешування паролів за допомогою `bcryptjs`.
4. Реалізувати access token та refresh token на основі `jsonwebtoken`.
5. Додати захищений маршрут для отримання профілю користувача.
6. Додати ролі `user` та `admin`.
7. Реалізувати middleware для перевірки токена та ролі користувача.
8. Додати валідацію вхідних даних, централізовану обробку помилок та логування.
9. Реалізувати оновлення профілю, зміну пароля, вихід із системи, відновлення пароля, підтвердження email та спрощений Google login endpoint.
10. Перевірити роботу API через Postman.

---

## 2. Аналіз вимог лабораторної роботи

У межах лабораторної роботи потрібно було розширити існуючий REST API, а не створювати новий backend-проєкт. Тому реалізація виконана всередині проєкту `lab1-rest-api`, який уже містив маршрути для роботи зі студентами та групами.

Основні вимоги були такими:
- backend має працювати на `Node.js` та `Express.js`;
- дані користувачів мають зберігатися в MySQL через `Sequelize`;
- пароль не повинен зберігатися у відкритому вигляді;
- після реєстрації або входу користувач має отримувати `accessToken` і `refreshToken`;
- захищені маршрути мають вимагати заголовок `Authorization: Bearer <token>`;
- для адміністративних дій має використовуватися роль `admin`;
- помилки валідації, авторизації та серверні помилки мають повертатися у зрозумілому JSON-форматі.

У реалізації також залишено попередню функціональність StudentLab. Старі маршрути не були видалені і доступні за такими адресами:

```text
/api/legacy/students
/api/mysql2/groups
/api/mysql2/students
/api/sequelize/groups
/api/sequelize/students
```

---

## 3. Структура backend-проєкту

Після виконання лабораторної роботи структура backend-проєкту була розширена файлами для автентифікації, middleware та допоміжних функцій.

```text
lab1-rest-api/
├── config/
│   ├── database.js
│   └── sequelize.js
├── middleware/
│   ├── authMiddleware.js
│   ├── errorMiddleware.js
│   ├── roleMiddleware.js
│   └── validate.js
├── models/
│   ├── group.js
│   ├── index.js
│   ├── student.js
│   └── user.js
├── routes/
│   ├── auth.js
│   ├── legacy/
│   ├── mysql2/
│   └── sequelize/
├── sql/
│   └── schema.sql
├── utils/
│   ├── tokens.js
│   └── userResponse.js
├── package.json
└── server.js
```

Файл `server.js` відповідає за запуск Express-сервера, підключення маршрутів та middleware обробки помилок. Маршрути автентифікації підключено з префіксом `/api/auth`.

```js
app.use("/api/auth", authRoutes);
app.use(errorMiddleware);
```

---

## 4. Опис моделі користувача

Для збереження користувачів створено Sequelize-модель `User` у файлі `models/user.js`. Вона відповідає таблиці `users` у базі даних MySQL.

Основні поля моделі:
- `id` — унікальний ідентифікатор користувача;
- `name` — ім'я користувача;
- `email` — електронна пошта, яка має бути унікальною;
- `password_hash` — хеш пароля;
- `role` — роль користувача, значення `user` або `admin`;
- `refresh_token_hash` — хеш поточного refresh token;
- `email_confirmed` — ознака підтвердження email;
- `email_confirmation_token` — токен для підтвердження email;
- `reset_password_token` — токен для відновлення пароля;
- `google_id` — ідентифікатор для спрощеного Google login;
- `login_attempts` і `locked_until` — поля для обмеження кількості невдалих спроб входу.

Фрагмент моделі:

```js
role: {
  type: DataTypes.ENUM("admin", "user"),
  allowNull: false,
  defaultValue: "user"
},
email: {
  type: DataTypes.STRING(100),
  allowNull: false,
  unique: true,
  validate: {
    isEmail: true
  }
}
```

Пароль зберігається не напряму, а у полі `password_hash`. Це зменшує ризики у випадку несанкціонованого доступу до бази даних.

---

## 5. Опис реалізованих API endpoints

Усі маршрути автентифікації мають префікс `/api/auth`.

| Метод | URL | Призначення |
|---|---|---|
| `POST` | `/api/auth/register` | Реєстрація нового користувача |
| `POST` | `/api/auth/login` | Вхід користувача |
| `POST` | `/api/auth/logout` | Вихід користувача та скидання refresh token |
| `POST` | `/api/auth/refresh` | Оновлення access token за refresh token |
| `GET` | `/api/auth/profile` | Отримання профілю поточного користувача |
| `PUT` | `/api/auth/profile` | Оновлення імені або email поточного користувача |
| `PUT` | `/api/auth/change-password` | Зміна пароля |
| `DELETE` | `/api/auth/users/:id` | Видалення користувача, доступне лише адміністратору |
| `POST` | `/api/auth/forgot-password` | Генерація токена відновлення пароля |
| `POST` | `/api/auth/confirm-email` | Підтвердження email за токеном |
| `POST` | `/api/auth/google` | Спрощений endpoint для Google login |

Приклад відповіді після успішної реєстрації:

```json
{
  "message": "User registered successfully",
  "user": {
    "id": 1,
    "name": "Ivan Student",
    "email": "ivan.student@studentlab.ua",
    "role": "user",
    "email_confirmed": false
  },
  "accessToken": "...",
  "refreshToken": "...",
  "emailConfirmationToken": "..."
}
```

---

## 6. Реєстрація та авторизація користувача

Реєстрація виконується через маршрут `POST /api/auth/register`. Для створення користувача потрібно передати `name`, `email`, `password` та `passwordConfirmation`. Також можна передати роль `user` або `admin`.

Приклад тіла запиту:

```json
{
  "name": "Ivan Student",
  "email": "ivan.student@studentlab.ua",
  "password": "password123",
  "passwordConfirmation": "password123",
  "role": "user"
}
```

Перед створенням користувача API перевіряє наявність обов'язкових полів, формат email, мінімальну довжину пароля та збіг підтвердження пароля. Якщо email уже існує в базі, повертається статус `409`.

Для хешування пароля використовується бібліотека `bcryptjs`:

```js
password_hash: await bcrypt.hash(password, 10)
```

Вхід користувача виконується через `POST /api/auth/login`. Після успішного входу сервер повертає дані користувача, `accessToken` та `refreshToken`. Якщо email або пароль неправильні, повертається статус `401`.

---

## 7. JWT, refresh token і захищені маршрути

У проєкті використовується бібліотека `jsonwebtoken`. Access token містить ідентифікатор користувача, email та роль. Він використовується для доступу до захищених маршрутів.

Створення access token:

```js
function createAccessToken(user) {
  return jwt.sign(
    {
      id: user.id,
      email: user.email,
      role: user.role
    },
    accessTokenSecret,
    { expiresIn: accessTokenExpiresIn }
  );
}
```

Refresh token створюється окремо і зберігається в базі не у відкритому вигляді, а як SHA-256 хеш у полі `refresh_token_hash`. При запиті `POST /api/auth/refresh` сервер перевіряє refresh token, видає новий access token і новий refresh token.

Захист маршрутів реалізовано у файлі `middleware/authMiddleware.js`. Middleware читає заголовок:

```text
Authorization: Bearer <accessToken>
```

Якщо токен відсутній, неправильний або прострочений, API повертає статус `401`. Маршрут `GET /api/auth/profile` працює лише після успішної перевірки токена.

Для адміністративних дій використовується `roleMiddleware.js`. Наприклад, маршрут `DELETE /api/auth/users/:id` доступний тільки користувачу з роллю `admin`.

---

## 8. Валідація даних та обробка помилок

Валідація реалізована у файлі `middleware/validate.js` та безпосередньо у маршрутах автентифікації. Перевіряються:
- наявність обов'язкових полів;
- формат email;
- мінімальна довжина пароля;
- збіг пароля і підтвердження;
- допустимість ролі `user` або `admin`.

Приклад відповіді при помилці валідації:

```json
{
  "message": "Validation error",
  "errors": ["password confirmation does not match"]
}
```

Централізована обробка помилок реалізована у файлі `middleware/errorMiddleware.js`. Вона обробляє помилки Sequelize, зокрема дублювання унікального значення, та виводить помилки в консоль:

```js
console.error(`[${new Date().toISOString()}] ${req.method} ${req.originalUrl}`, {
  message: error.message,
  stack: error.stack
});
```

Також реалізовано обмеження кількості невдалих спроб входу. Після перевищення ліміту користувач тимчасово блокується, а API повертає статус `429`.

---

## 9. Тестування API в Postman

Тестування виконувалося через Postman за адресою:

```text
http://localhost:3000
```

Для запуску backend потрібно встановити залежності, налаштувати `.env`, створити базу даних та запустити сервер:

```bash
npm install
mysql -u root -p < sql/schema.sql
npm start
```

Основний порядок тестування:
1. Зареєструвати користувача через `POST /api/auth/register`.
2. Перевірити помилки валідації при реєстрації.
3. Увійти через `POST /api/auth/login`.
4. Скопіювати `accessToken`.
5. Викликати `GET /api/auth/profile` без токена і з Bearer-токеном.
6. Оновити access token через `POST /api/auth/refresh`.
7. Оновити профіль через `PUT /api/auth/profile`.
8. Змінити пароль через `PUT /api/auth/change-password`.
9. Виконати вихід через `POST /api/auth/logout`.

Під час тестування було перевірено як успішні сценарії, так і помилки: відсутні поля, неправильне підтвердження пароля, неправильні облікові дані та відсутність токена для захищеного маршруту.

---

## 10. Скріншоти результатів

![Успішна реєстрація користувача](/assets/labs/lab-3/register%20%20-%20success.png)

**Рис. 1 - Успішна реєстрація користувача через `POST /api/auth/register`.**

![Помилка валідації при реєстрації](/assets/labs/lab-3/register%20-%20validation%20error.png)

**Рис. 2 - Помилка валідації при реєстрації через `POST /api/auth/register`.**

![Помилка підтвердження пароля](/assets/labs/lab-3/register%20-%20password%20mismatch.png)

**Рис. 3 - Помилка підтвердження пароля при реєстрації через `POST /api/auth/register`.**

![Успішний вхід користувача](/assets/labs/lab-3/login-success.png)

**Рис. 4 - Успішний вхід користувача через `POST /api/auth/login`.**

![Помилка входу](/assets/labs/lab-3/login%20-%20invalid%20credentials.png)

**Рис. 5 - Помилка входу з неправильними обліковими даними через `POST /api/auth/login`.**

![Профіль без токена](/assets/labs/lab-3/profile%20%20-%20without%20token.png)

**Рис. 6 - Відмова доступу до `GET /api/auth/profile` без токена.**

![Профіль з токеном](/assets/labs/lab-3/profile%20-%20with%20token.png)

**Рис. 7 - Успішний доступ до `GET /api/auth/profile` з Bearer-токеном.**

![Оновлення accessToken](/assets/labs/lab-3/refresh%20-%20success.png)

**Рис. 8 - Оновлення `accessToken` через `POST /api/auth/refresh`.**

> Скріншот для оновлення профілю через `PUT /api/auth/profile` у папці `static/assets/labs/lab-3` відсутній, тому він не був вставлений до звіту.

![Успішна зміна пароля](/assets/labs/lab-3/change-password%20-%20success.png)

**Рис. 9 - Успішна зміна пароля через `PUT /api/auth/change-password`.**

![Успішний вихід](/assets/labs/lab-3/logout%20-%20success.png)

**Рис. 10 - Успішний вихід користувача через `POST /api/auth/logout`.**

---

## 11. Висновок

У межах лабораторної роботи було розширено backend-проєкт `lab1-rest-api` для системи StudentLab. Було додано модель користувача, маршрути реєстрації та авторизації, хешування паролів за допомогою `bcryptjs`, роботу з JWT access token та refresh token, захищений маршрут профілю, ролі `user` і `admin`, middleware для перевірки токена та ролі.

Також реалізовано оновлення профілю, зміну пароля, вихід користувача, генерацію токена відновлення пароля, токен підтвердження email, спрощений Google login endpoint, обмеження кількості спроб входу, валідацію даних і централізовану обробку помилок. Попередні маршрути StudentLab для роботи зі студентами та групами залишилися доступними, тому функціональність попередніх лабораторних робіт не була порушена.

Результати тестування в Postman підтвердили, що основні сценарії Lab 3 працюють: користувач може зареєструватися, увійти в систему, отримати профіль за Bearer-токеном, оновити access token, змінити профіль, змінити пароль і виконати logout.
