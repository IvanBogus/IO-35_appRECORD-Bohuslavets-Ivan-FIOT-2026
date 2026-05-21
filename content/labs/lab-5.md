## 1. Тема, мета, посилання

### 1.1 Тема
«Безпека та продуктивність серверних додатків. Безпека Node.js-додатків, оптимізація запитів, кешування, тестування API».

### 1.2 Мета
Розширити серверну частину проєкту `lab1-rest-api` засобами базового захисту HTTP API, обмеження кількості запитів, валідації вхідних даних, кешування відповідей, оптимізації маршруту за допомогою пагінації та автоматизованого тестування.

У межах роботи потрібно показати, як сервер захищається від типових небезпечних сценаріїв, як зменшується кількість повторних звернень до бази даних завдяки кешу та як працездатність API перевіряється тестами.

### 1.3 Посилання
- Репозиторій backend REST API: [посилання](https://github.com/IvanBogus/lab1-rest-api.git)
- Репозиторій звітного HTML-документа: [посилання](https://github.com/IvanBogus/IO-35_appRECORD-Bohuslavets-Ivan-FIOT-2026.git)
- Звітний HTML-документ: [посилання](https://ivanbogus.github.io/IO-35_appRECORD-Bohuslavets-Ivan-FIOT-2026/)

---

## 2. Короткі теоретичні відомості

### 2.1 Безпека Node.js API
Для backend-застосунків важливо захищати HTTP-рівень, перевіряти вхідні дані та обмежувати надмірну кількість запитів. У роботі використано `Helmet`, який додає захисні HTTP-заголовки, та `express-rate-limit`, який обмежує частоту звернень до API.

У проєкті також уже реалізовано JWT-автентифікацію, хешування паролів і middleware для перевірки токена. У межах ЛР5 ці механізми доповнено глобальним захистом HTTP-відповідей та rate limiting.

### 2.2 Валідація даних
Валідація потрібна для того, щоб сервер не обробляв некоректні або неповні дані. У маршрутах студентів перевіряються обов'язкові поля, формат email, рік вступу та коректність `group_id`. Якщо запит некоректний, API повертає статус `400` і список помилок.

### 2.3 Кешування
Кешування зберігає результат запиту на короткий час і дозволяє повторно повернути його без звернення до бази даних. Для лабораторної роботи використано in-memory cache через `node-cache`. Такий варіант не потребує окремого Redis-сервера і достатній для демонстрації принципу кешування.

Для перевірки кешу API повертає заголовок:
- `x-cache: MISS` - дані отримано з бази даних і записано в кеш;
- `x-cache: HIT` - дані повернуто з кешу.

### 2.4 Оптимізація API
Одним із простих способів оптимізації спискових маршрутів є пагінація. Замість повернення всіх записів одразу сервер приймає параметри `page` і `limit`, обмежує кількість записів у відповіді та повертає метадані про сторінку.

### 2.5 Автоматизоване тестування
Автоматизовані тести дозволяють швидко перевірити, що основні сценарії API працюють після змін. У роботі використано `Jest` і `Supertest`. Тести перевіряють security headers, rate limit headers, валідацію, пагінацію та кешування.

---

## 3. Реалізований функціонал Lab 5

### 3.1 Основні сценарії
У межах лабораторної роботи реалізовано:
- захисні HTTP-заголовки через `helmet`;
- обмеження кількості запитів через `express-rate-limit`;
- покращену валідацію `POST` і `PUT` запитів для студентів;
- кешування відповіді маршруту `GET /api/sequelize/groups`;
- заголовки `x-cache: MISS` і `x-cache: HIT` для демонстрації кешу;
- пагінацію маршруту `GET /api/sequelize/students`;
- автоматизовані тести через `Jest` і `Supertest`;
- можливість імпорту Express-застосунку в тестах без запуску окремого HTTP-сервера.

### 3.2 Адаптація під існуючий проєкт
Проєкт `lab1-rest-api` уже був реалізований на `Express.js`, `Sequelize` і `MySQL`, тому лабораторну роботу виконано як розширення наявної backend-частини:
- `server.js` доповнено middleware безпеки;
- `routes/sequelize/groups.js` доповнено кешуванням;
- `routes/sequelize/students.js` доповнено пагінацією та валідацією;
- `utils/cache.js` створено як окремий модуль кешу;
- `tests/api.test.js` додано для автоматизованої перевірки API.

---

## 4. Реалізація backend-частини

### 4.1 Helmet і rate limiting
У файлі `server.js` підключено `helmet` і `express-rate-limit`.

```js
app.use(helmet());
app.use(
  rateLimit({
    windowMs: 60 * 1000,
    limit: 100,
    standardHeaders: true,
    legacyHeaders: false,
    message: {
      message: "Too many requests, please try again later"
    }
  })
);
```

Після цього відповіді API містять захисні HTTP-заголовки, наприклад `X-Content-Type-Options: nosniff`, а також заголовки rate limiting.

### 4.2 Кешування списку груп
Для кешування створено файл `utils/cache.js`, який використовує `node-cache`.

```js
const cache = new NodeCache({
  stdTTL: 60,
  checkperiod: 120,
  useClones: false
});
```

Маршрут `GET /api/sequelize/groups` спочатку перевіряє кеш. Якщо дані вже збережені, API повертає їх без повторного звернення до бази даних.

```js
const cachedGroups = getCached(groupsCacheKey);

if (cachedGroups) {
  res.set("x-cache", "HIT");
  return res.json(cachedGroups);
}
```

Якщо кеш порожній, дані отримуються з бази даних, записуються в кеш і повертаються з заголовком `x-cache: MISS`.

### 4.3 Пагінація списку студентів
Маршрут `GET /api/sequelize/students` оптимізовано за допомогою параметрів `page` і `limit`.

```js
const page = parsePositiveInteger(req.query.page, 1);
const limit = Math.min(parsePositiveInteger(req.query.limit, 10), 50);
const offset = (page - 1) * limit;
```

Для запиту до Sequelize використовується `findAndCountAll`, що дозволяє отримати записи поточної сторінки та загальну кількість студентів.

```js
const { count, rows } = await Student.findAndCountAll({
  include: [
    {
      model: Group,
      attributes: ["id", "name", "code", "curator_name", "study_year"]
    }
  ],
  order: [["id", "ASC"]],
  limit,
  offset
});
```

Відповідь містить масив `data` і службову інформацію `meta`.

```js
res.json({
  data: rows,
  meta: {
    total: count,
    page,
    limit,
    totalPages: Math.ceil(count / limit),
    source: "database"
  }
});
```

### 4.4 Валідація даних студента
Для створення та оновлення студента додано перевірку обов'язкових полів, email, року вступу та `group_id`.

```js
if (!email || !isEmail(email)) {
  errors.push("Valid email is required");
}

if (!Number.isInteger(Number(group_id)) || Number(group_id) <= 0) {
  errors.push("group_id must be a positive integer");
}
```

Якщо валідація не пройдена, сервер повертає відповідь:

```json
{
  "message": "Validation error",
  "errors": ["Valid email is required"]
}
```

### 4.5 Автоматизовані тести
Для перевірки ЛР5 додано файл `tests/api.test.js`. У тестах використано `Supertest`, а моделі Sequelize замокано, щоб перевірка не залежала від локальної MySQL-бази.

Тести перевіряють:
- наявність security headers;
- наявність rate limit headers;
- відхилення некоректного `POST /api/sequelize/students`;
- пагінацію `GET /api/sequelize/students`;
- кешування `GET /api/sequelize/groups`.

---

## 5. Перевірка API

Перевірка виконується через Postman і консоль.

Основні запити для демонстрації:
- `GET http://localhost:3000/status` - перевірка стану API, Helmet і rate limit headers;
- `GET http://localhost:3000/api/sequelize/students?page=1&limit=5` - перевірка пагінації;
- `GET http://localhost:3000/api/sequelize/groups` - перший запит із `x-cache: MISS`;
- `GET http://localhost:3000/api/sequelize/groups` - повторний запит із `x-cache: HIT`;
- `POST http://localhost:3000/api/sequelize/students` з некоректним body - перевірка валідації;
- `npm test` - запуск автоматизованих тестів.

Приклад некоректного body для перевірки валідації:

```json
{
  "first_name": "",
  "email": "invalid-email"
}
```

---

## 6. Команди для запуску

### 6.1 Встановлення залежностей
```bash
npm install
```

### 6.2 Запуск API
```bash
npm start
```

Після запуску API доступний за адресою `http://localhost:3000`.

### 6.3 Запуск тестів
```bash
npm test
```

Очікуваний результат:

```text
Test Suites: 1 passed, 1 total
Tests:       4 passed, 4 total
```

---

## 7. Результати виконання

Скріншоти потрібно буде розмістити в директорії `static/assets/labs/lab-5/`.

Очікувані файли скріншотів:
- `static/assets/labs/lab-5/api_server_running.png`;
- `static/assets/labs/lab-5/postman_security_headers.png`;
- `static/assets/labs/lab-5/postman_students_pagination.png`;
- `static/assets/labs/lab-5/postman_groups_cache_miss.png`;
- `static/assets/labs/lab-5/postman_groups_cache_hit.png`;
- `static/assets/labs/lab-5/postman_validation_error.png`;
- `static/assets/labs/lab-5/jest_tests_success.png`.

![Успішний запуск backend-застосунку StudentLab API](/assets/labs/lab-5/api_server_running.png)
**Рис. 1 - Успішний запуск backend-застосунку `lab1-rest-api`. Файл: `static/assets/labs/lab-5/api_server_running.png`.**

![Security headers і rate limit headers у Postman](/assets/labs/lab-5/postman_security_headers.png)
**Рис. 2 - Перевірка `GET /status`: захисні заголовки Helmet і заголовки rate limiting. Файл: `static/assets/labs/lab-5/postman_security_headers.png`.**

![Пагінація студентів у Postman](/assets/labs/lab-5/postman_students_pagination.png)
**Рис. 3 - Перевірка пагінації `GET /api/sequelize/students?page=1&limit=5`. Файл: `static/assets/labs/lab-5/postman_students_pagination.png`.**

![Перший запит списку груп із cache MISS](/assets/labs/lab-5/postman_groups_cache_miss.png)
**Рис. 4 - Перший запит `GET /api/sequelize/groups` із заголовком `x-cache: MISS`. Файл: `static/assets/labs/lab-5/postman_groups_cache_miss.png`.**

![Повторний запит списку груп із cache HIT](/assets/labs/lab-5/postman_groups_cache_hit.png)
**Рис. 5 - Повторний запит `GET /api/sequelize/groups` із заголовком `x-cache: HIT`. Файл: `static/assets/labs/lab-5/postman_groups_cache_hit.png`.**

![Помилка валідації студента у Postman](/assets/labs/lab-5/postman_validation_error.png)
**Рис. 6 - Помилка валідації під час `POST /api/sequelize/students` з некоректним body. Файл: `static/assets/labs/lab-5/postman_validation_error.png`.**

![Успішний запуск автоматизованих тестів Jest](/assets/labs/lab-5/jest_tests_success.png)
**Рис. 7 - Успішний запуск автоматизованих тестів командою `npm test`. Файл: `static/assets/labs/lab-5/jest_tests_success.png`.**

---

## 8. Висновки

У межах лабораторної роботи серверну частину StudentLab API розширено засобами безпеки, кешування, оптимізації та тестування. Через `helmet` додано захисні HTTP-заголовки, а через `express-rate-limit` реалізовано обмеження кількості запитів до API.

Для підвищення продуктивності реалізовано in-memory кешування списку груп через `node-cache`. Перший запит отримує дані з бази даних і повертає `x-cache: MISS`, а повторний запит повертає ті самі дані з кешу з `x-cache: HIT`.

Маршрут списку студентів оптимізовано за допомогою пагінації. API приймає параметри `page` і `limit`, повертає тільки потрібну частину записів і додає метадані про загальну кількість елементів та сторінок.

Також додано автоматизовані тести на `Jest` і `Supertest`, які перевіряють ключові сценарії ЛР5 без підключення до реальної бази даних. Це дає змогу швидко перевірити, що middleware безпеки, валідація, пагінація та кешування працюють коректно.

---

## 9. Перелік використаних джерел
1. Документація Node.js.
2. Документація Express.js.
3. Документація Helmet.
4. Документація express-rate-limit.
5. Документація node-cache.
6. Документація Sequelize.
7. Документація Jest.
8. Документація Supertest.
