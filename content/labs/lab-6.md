## 1. Тема, мета, посилання

### 1.1 Тема
«Документування API за допомогою Swagger. Деплой Node.js-додатку. Підсумковий проєкт: REST API з MySQL».

### 1.2 Мета
Інтегрувати Swagger/OpenAPI-документацію в існуючий backend-застосунок `lab1-rest-api`, описати основні REST endpoint-и для роботи зі студентами та групами, перевірити API через Swagger UI та підготувати проєкт до демонстрації як підсумковий REST API з MySQL.

### 1.3 Посилання
- Репозиторій backend REST API: [посилання](https://github.com/IvanBogus/lab1-rest-api.git)
- Репозиторій звітного HTML-документа: [посилання](https://github.com/IvanBogus/IO-35_appRECORD-Bohuslavets-Ivan-FIOT-2026.git)
- Звітний HTML-документ: [посилання](https://ivanbogus.github.io/IO-35_appRECORD-Bohuslavets-Ivan-FIOT-2026/)

---

## 2. Короткі теоретичні відомості

### 2.1 REST API
REST API - це спосіб організації взаємодії між клієнтом і сервером через HTTP-методи. У проєкті `lab1-rest-api` API використовується для роботи з навчальними групами, студентами, авторизацією, завантаженням файлів і службовими маршрутами.

Основні HTTP-методи:
- `GET` - отримання даних;
- `POST` - створення нового запису;
- `PUT` - повне оновлення запису;
- `DELETE` - видалення запису.

### 2.2 Swagger та OpenAPI
OpenAPI Specification - це стандарт формального опису REST API. Він дозволяє описати маршрути, параметри, body запитів, JSON-схеми моделей, коди відповідей і приклади даних.

Swagger UI використовує OpenAPI-документ і створює web-інтерфейс для перегляду та тестування API без Postman. У лабораторній роботі використано пакети:
- `swagger-jsdoc` - генерація OpenAPI-специфікації з JSDoc-коментарів;
- `swagger-ui-express` - підключення Swagger UI до Express-застосунку.

### 2.3 Деплой Node.js-застосунку
Деплой - це розгортання backend-застосунку на сервері або хмарній платформі. Для Node.js API важливо мати команду запуску, підтримку змінної середовища `PORT`, файл `.env.example` і коректне підключення до бази даних.

У проєкті вже використовується команда:

```bash
npm start
```

Сервер читає порт із `process.env.PORT` або використовує локальний порт `3000`.

---

## 3. Реалізований функціонал Lab 6

У межах лабораторної роботи реалізовано:
- інтеграцію Swagger UI в Express-застосунок;
- OpenAPI 3.0.0 специфікацію для StudentLab API;
- маршрут `GET /api-docs` для перегляду Swagger UI;
- маршрут `GET /api-docs.json` для перегляду OpenAPI JSON;
- опис моделей `Student`, `Group`, `StudentInput`, `PaginatedStudents`, `ErrorResponse`;
- документацію основних endpoint-ів Sequelize API;
- endpoint `GET /api/sequelize/students/:id` для отримання одного студента за id;
- тести для перевірки OpenAPI JSON і нового endpoint-а отримання студента.

---

## 4. Реалізація Swagger/OpenAPI

### 4.1 Встановлення залежностей
Для інтеграції Swagger додано залежності:

```bash
npm install swagger-ui-express swagger-jsdoc
```

У `package.json` з'явилися пакети:

```json
"swagger-jsdoc": "^6.2.8",
"swagger-ui-express": "^5.0.1"
```

### 4.2 OpenAPI-конфігурація
OpenAPI-конфігурацію винесено в окремий файл `docs/swagger.js`. У ньому описано загальну інформацію про API, локальний сервер і схеми даних.

```js
const swaggerSpec = swaggerJsdoc({
  definition: {
    openapi: "3.0.0",
    info: {
      title: "StudentLab API",
      version: "1.0.0",
      description: "REST API for managing student groups and students with MySQL and Sequelize."
    }
  },
  apis: ["./server.js", "./routes/sequelize/*.js"]
});
```

### 4.3 Підключення Swagger UI
У файлі `server.js` підключено `swagger-ui-express` і OpenAPI-специфікацію.

```js
const swaggerUi = require("swagger-ui-express");
const swaggerSpec = require("./docs/swagger");

app.use("/api-docs", swaggerUi.serve, swaggerUi.setup(swaggerSpec));
app.get("/api-docs.json", (req, res) => {
  res.json(swaggerSpec);
});
```

Після запуску backend документація доступна за адресами:
- `http://localhost:3000/api-docs`;
- `http://localhost:3000/api-docs.json`.

### 4.4 Документовані endpoint-и
У Swagger описано основні маршрути для роботи з групами та студентами:

| Метод | Endpoint | Опис |
|---|---|---|
| `GET` | `/api/sequelize/groups` | Отримання списку груп |
| `GET` | `/api/sequelize/students` | Отримання студентів з пагінацією |
| `GET` | `/api/sequelize/students/{id}` | Отримання одного студента за id |
| `POST` | `/api/sequelize/students` | Створення студента |
| `PUT` | `/api/sequelize/students/{id}` | Оновлення студента |
| `DELETE` | `/api/sequelize/students/{id}` | Видалення студента |

Для `POST` і `PUT` описано `requestBody` зі схемою `StudentInput`. Для всіх endpoint-ів описано основні коди відповідей: `200`, `201`, `400`, `404`, `500`.

### 4.5 Отримання студента за id
Для повного CRUD додано маршрут `GET /api/sequelize/students/:id`.

```js
router.get("/:id", async (req, res) => {
  const studentId = Number(req.params.id);

  if (!studentId) {
    return res.status(400).json({
      message: "Valid student id is required"
    });
  }

  const student = await Student.findByPk(studentId, {
    include: [
      {
        model: Group,
        attributes: ["id", "name", "code", "curator_name", "study_year"]
      }
    ]
  });

  if (!student) {
    return res.status(404).json({
      message: "Student not found"
    });
  }

  res.json(student);
});
```

---

## 5. Перевірка API через Swagger UI

Для перевірки потрібно запустити backend:

```bash
npm start
```

Після запуску відкрити:

```http
http://localhost:3000/api-docs
```

У Swagger UI потрібно перевірити:
- наявність групи endpoint-ів `Students`;
- наявність endpoint-а `GET /api/sequelize/groups`;
- виконання `GET /api/sequelize/students?page=1&limit=5`;
- виконання `GET /api/sequelize/students/{id}`;
- наявність body-схеми для `POST /api/sequelize/students`;
- наявність body-схеми для `PUT /api/sequelize/students/{id}`;
- виконання `DELETE /api/sequelize/students/{id}` для тестового запису.

Також можна перевірити OpenAPI JSON:

```http
http://localhost:3000/api-docs.json
```

---

## 6. Команди запуску та перевірки

### 6.1 Встановлення залежностей

```bash
npm install
```

### 6.2 Запуск API

```bash
npm start
```

Очікуваний результат:

```text
Sequelize connection is ready
Server running on http://localhost:3000
```

### 6.3 Запуск тестів

```bash
npm test
```

Очікуваний результат:

```text
Test Suites: 1 passed, 1 total
Tests:       6 passed, 6 total
```

---

## 7. Результати виконання

Скріншоти потрібно буде розмістити в директорії `static/assets/labs/lab-6/`.

Очікувані файли скріншотів:
- `static/assets/labs/lab-6/api_server_running.png`;
- `static/assets/labs/lab-6/swagger_ui_opened.png`;
- `static/assets/labs/lab-6/swagger_openapi_json.png`;
- `static/assets/labs/lab-6/swagger_students_list.png`;
- `static/assets/labs/lab-6/swagger_student_by_id.png`;
- `static/assets/labs/lab-6/swagger_student_post_schema.png`;
- `static/assets/labs/lab-6/swagger_student_put_schema.png`;
- `static/assets/labs/lab-6/swagger_student_delete.png`;
- `static/assets/labs/lab-6/jest_tests_success.png`.

![Успішний запуск backend-застосунку StudentLab API](/assets/labs/lab-6/api_server_running.png)
**Рис. 1 - Успішний запуск backend-застосунку `lab1-rest-api`. Файл: `static/assets/labs/lab-6/api_server_running.png`.**

![Swagger UI StudentLab API](/assets/labs/lab-6/swagger_ui_opened.png)
**Рис. 2 - Swagger UI за адресою `http://localhost:3000/api-docs` зі списком endpoint-ів. Файл: `static/assets/labs/lab-6/swagger_ui_opened.png`.**

![OpenAPI JSON документ](/assets/labs/lab-6/swagger_openapi_json.png)
**Рис. 3 - JSON-специфікація OpenAPI за маршрутом `GET /api-docs.json`. Файл: `static/assets/labs/lab-6/swagger_openapi_json.png`.**

![Перевірка списку студентів у Swagger UI](/assets/labs/lab-6/swagger_students_list.png)
**Рис. 4 - Виконання `GET /api/sequelize/students?page=1&limit=5` через Swagger UI. Файл: `static/assets/labs/lab-6/swagger_students_list.png`.**

![Перевірка отримання студента за id у Swagger UI](/assets/labs/lab-6/swagger_student_by_id.png)
**Рис. 5 - Виконання `GET /api/sequelize/students/{id}` через Swagger UI. Файл: `static/assets/labs/lab-6/swagger_student_by_id.png`.**

![Схема створення студента у Swagger UI](/assets/labs/lab-6/swagger_student_post_schema.png)
**Рис. 6 - Опис `requestBody` для `POST /api/sequelize/students` у Swagger UI. Файл: `static/assets/labs/lab-6/swagger_student_post_schema.png`.**

![Схема оновлення студента у Swagger UI](/assets/labs/lab-6/swagger_student_put_schema.png)
**Рис. 7 - Опис `requestBody` для `PUT /api/sequelize/students/{id}` у Swagger UI. Файл: `static/assets/labs/lab-6/swagger_student_put_schema.png`.**

![Видалення студента у Swagger UI](/assets/labs/lab-6/swagger_student_delete.png)
**Рис. 8 - Виконання `DELETE /api/sequelize/students/{id}` через Swagger UI. Файл: `static/assets/labs/lab-6/swagger_student_delete.png`.**

![Успішний запуск автоматизованих тестів Jest](/assets/labs/lab-6/jest_tests_success.png)
**Рис. 9 - Успішний запуск автоматизованих тестів командою `npm test`. Файл: `static/assets/labs/lab-6/jest_tests_success.png`.**

---

## 8. Підготовка до деплою

Проєкт підготовлено до production-запуску:
- у `package.json` є команда `"start": "node server.js"`;
- сервер використовує `process.env.PORT || 3000`;
- приклад змінних середовища зберігається у `.env.example`;
- підключення до MySQL виконується через Sequelize;
- API можна запустити локально командою `npm start`.

Для деплою на Render або іншу PaaS-платформу можна використати:
- Build Command: `npm install`;
- Start Command: `npm start`;
- Environment: `Node`;
- змінні середовища з `.env.example`;
- підключення до MySQL через параметри підключення в `.env`.

Фактичний деплой у межах цього етапу не виконувався, оскільки його потрібно робити окремо після підготовки репозиторію та хмарної бази даних.

---

## 9. Висновки

У межах лабораторної роботи до backend-застосунку `lab1-rest-api` інтегровано Swagger/OpenAPI-документацію. Для API створено OpenAPI 3.0.0 специфікацію з описом моделей даних, endpoint-ів, параметрів, request body та кодів відповідей.

Swagger UI доступний за маршрутом `/api-docs`, а JSON-специфікація - за маршрутом `/api-docs.json`. Через Swagger UI можна переглядати та тестувати основні CRUD-операції для ресурсу студентів і переглядати список груп.

Також додано endpoint для отримання одного студента за id, що завершує CRUD-набір для основного ресурсу. Автоматизовані тести перевіряють наявність OpenAPI JSON і роботу нового маршруту. Проєкт підготовлено до демонстрації як REST API з MySQL і Swagger-документацією.

---

## 10. Перелік використаних джерел

1. Документація Node.js.
2. Документація Express.js.
3. Документація OpenAPI Specification.
4. Документація Swagger UI.
5. Документація swagger-jsdoc.
6. Документація swagger-ui-express.
7. Документація Sequelize.
8. Документація Jest.
9. Документація Supertest.
