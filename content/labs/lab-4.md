## 1. Тема, мета, посилання

### 1.1 Тема
«Розширені можливості Node.js-додатків: логування, завантаження файлів, моніторинг продуктивності».

### 1.2 Мета
Розширити серверну частину проєкту `lab1-rest-api` засобами логування HTTP-запитів і подій, обробки файлових завантажень та базового моніторингу стану Node.js-процесу. У межах роботи потрібно реалізувати завантаження одного й кількох файлів із валідацією, запис подій і помилок у лог, endpoint стану сервера та запуск застосунку через PM2.

### 1.3 Посилання
- Репозиторій backend REST API: [посилання](https://github.com/IvanBogus/lab1-rest-api.git)
- Репозиторій звітного HTML-документа: [посилання](https://github.com/IvanBogus/IO-35_appRECORD-Bohuslavets-Ivan-FIOT-2026.git)
- Звітний HTML-документ: [посилання](https://ivanbogus.github.io/IO-35_appRECORD-Bohuslavets-Ivan-FIOT-2026/)

---

## 2. Короткі теоретичні відомості

### 2.1 Логування у Node.js-застосунках
Логування використовується для фіксації HTTP-запитів, службових подій, помилок і часу відповіді сервера. У проєкті використано два інструменти: `Morgan` для виведення HTTP-запитів у консоль під час розробки та `Winston` для запису структурованих логів у файл `app.log`.

### 2.2 Завантаження файлів
Файли з клієнта передаються на сервер у форматі `multipart/form-data`. Для Express-застосунків такий формат зручно обробляти через `Multer`. Він приймає файл із форми, перевіряє обмеження та зберігає файл у визначену директорію.

У лабораторній роботі дозволено завантаження файлів форматів `jpg`, `png` і `pdf`. Також встановлено обмеження розміру одного файла.

### 2.3 Моніторинг продуктивності
Базовий моніторинг Node.js-застосунку можна реалізувати через вбудований об'єкт `process`. Для цієї роботи використано:
- `process.uptime()` — час роботи процесу;
- `process.memoryUsage()` — використання оперативної пам'яті.

Окремо для кожного HTTP-запиту вимірюється час обробки, після чого результат записується у лог.

---

## 3. Реалізований функціонал Lab 4

### 3.1 Основні сценарії
У межах лабораторної роботи реалізовано такі можливості:
- логування HTTP-запитів у консоль через Morgan;
- файлове логування подій через Winston;
- запис `info` та `error` логів у `logs/app.log`;
- централізована обробка помилок;
- вимірювання часу відповіді кожного HTTP-запиту;
- endpoint `GET /status` для перегляду `uptime` та `memoryUsage`;
- endpoint `POST /upload` для завантаження одного файла;
- endpoint `POST /upload-multiple` для завантаження кількох файлів;
- валідація типу файла;
- обмеження розміру файла;
- PM2-конфігурація для запуску API як окремого процесу.

### 3.2 Адаптація під існуючий проєкт
Проєкт `lab1-rest-api` уже був побудований на `Express.js`, тому реалізацію виконано без створення окремого демонстраційного сервера. Нові можливості додано до наявного backend-застосунку:
- Morgan підключено у `server.js`;
- Winston logger винесено в `utils/logger.js`;
- маршрути завантаження файлів винесено в `routes/upload.js`;
- існуючий `errorMiddleware` розширено логуванням помилок;
- PM2-конфігурацію додано у файл `ecosystem.config.js`.

---

## 4. Реалізація backend-частини

### 4.1 Підключення Morgan і middleware часу відповіді
У файлі `server.js` підключено Morgan для логування HTTP-запитів у консоль.

```js
const morgan = require("morgan");

app.use(morgan("dev"));
```

Також додано middleware, яке вимірює час обробки кожного запиту та записує результат через Winston.

```js
app.use((req, res, next) => {
  const startedAt = process.hrtime.bigint();

  res.on("finish", () => {
    const durationMs = Number(process.hrtime.bigint() - startedAt) / 1_000_000;

    logger.info("HTTP request completed", {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      durationMs: Number(durationMs.toFixed(2))
    });
  });

  next();
});
```

### 4.2 Файловий логер Winston
Для логування створено файл `utils/logger.js`. Він створює директорію `logs` і записує події у файл `app.log`.

```js
const logger = winston.createLogger({
  level: "info",
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({
      filename: path.join(logsDir, "app.log")
    })
  ]
});
```

Помилки додатково логуються в `middleware/errorMiddleware.js`.

```js
logger.error("Request failed", {
  method: req.method,
  url: req.originalUrl,
  statusCode,
  ...errorDetails
});
```

### 4.3 Моніторинг стану сервера
Маршрут `GET /status` реалізовано у файлі `server.js`. Він повертає поточний стан Node.js-процесу.

```js
app.get("/status", (req, res) => {
  res.json({
    status: "ok",
    uptime: process.uptime(),
    memoryUsage: process.memoryUsage()
  });
});
```

### 4.4 Завантаження одного файла
Для завантаження одного файла реалізовано маршрут `POST /upload`. Він очікує поле `file` у `multipart/form-data`.

```js
router.post("/upload", upload.single("file"), (req, res) => {
  if (!req.file) {
    return res.status(400).json({ message: "File is required" });
  }

  const file = formatFile(req.file);
  logger.info("Single file uploaded", file);

  res.status(201).json({
    message: "File uploaded successfully",
    file
  });
});
```

Після успішного завантаження API повертає назву збереженого файла, початкову назву, MIME-тип, розмір і шлях.

### 4.5 Завантаження кількох файлів
Маршрут `POST /upload-multiple` приймає кілька файлів у полі `files`.

```js
router.post("/upload-multiple", upload.array("files", 5), (req, res) => {
  if (!req.files?.length) {
    return res.status(400).json({ message: "At least one file is required" });
  }

  const files = req.files.map(formatFile);
  logger.info("Multiple files uploaded", { count: files.length, files });

  res.status(201).json({
    message: "Files uploaded successfully",
    files
  });
});
```

Максимальна кількість файлів в одному запиті — 5.

### 4.6 Валідація файлів
Перед збереженням перевіряється MIME-тип файла. Дозволені типи:
- `image/jpeg`;
- `image/png`;
- `application/pdf`.

```js
const allowedMimeTypes = new Set(["image/jpeg", "image/png", "application/pdf"]);
```

Також встановлено обмеження розміру файла:

```js
const maxFileSizeBytes = 5 * 1024 * 1024;
```

Якщо файл має непідтримуваний тип, API повертає статус `400` і повідомлення про помилку.

### 4.7 PM2-конфігурація
Для запуску застосунку через PM2 створено файл `ecosystem.config.js`.

```js
module.exports = {
  apps: [
    {
      name: "studentlab-api",
      script: "server.js",
      cwd: __dirname,
      env: {
        NODE_ENV: "production",
        PORT: process.env.PORT || 3000
      }
    }
  ]
};
```

Після цього застосунок можна запустити командою `npx pm2 start ecosystem.config.js`.

---

## 5. Перевірка API

Перевірка виконувалася через браузер, Postman і консоль. Для файлових запитів у Postman використовувався тип тіла `form-data`.

Перевірені запити:
- `GET /`;
- `GET /status`;
- `POST /upload` — завантаження одного файла;
- `POST /upload-multiple` — завантаження кількох файлів;
- `POST /upload` з файлом непідтримуваного типу;
- перегляд файла `logs/app.log`;
- запуск застосунку через PM2.

Для `POST /upload` використовується поле `file`, для `POST /upload-multiple` — поле `files`.

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

### 6.3 Перегляд логів
```powershell
Get-Content .\logs\app.log -Tail 15
```

### 6.4 Запуск через PM2
```powershell
npx pm2 start ecosystem.config.js
npx pm2 list
npx pm2 logs studentlab-api
npx pm2 restart studentlab-api
```

У поточному локальному середовищі для PM2 використовувалась окрема директорія:

```powershell
$env:PM2_HOME = "$PWD\.pm2"
npx pm2 start ecosystem.config.js
```

---

## 7. Результати виконання

Скріншоти потрібно розмістити в директорії `static/assets/labs/lab-4/`.

![Успішний запуск backend-застосунку StudentLab API](/assets/labs/lab-4/server-start.png)
**Рис. 1 - Успішний запуск backend-застосунку `lab1-rest-api`.**

![Перевірка маршруту GET /](/assets/labs/lab-4/get-root.png)
**Рис. 2 - Перевірка маршруту `GET /`.**

![Перевірка маршруту GET /status у Postman](/assets/labs/lab-4/get-status.png)
**Рис. 3 - Перевірка маршруту `GET /status` з `uptime` та `memoryUsage`.**

![Успішне завантаження одного файла через Postman](/assets/labs/lab-4/upload-single-success.png)
**Рис. 4 - Успішне завантаження одного файла через `POST /upload`.**

![Помилка валідації типу файла у Postman](/assets/labs/lab-4/upload-invalid-type.png)
**Рис. 5 - Помилка валідації під час завантаження файла непідтримуваного типу.**

![Успішне завантаження кількох файлів через Postman](/assets/labs/lab-4/upload-multiple-success.png)
**Рис. 6 - Успішне завантаження кількох файлів через `POST /upload-multiple`.**

![Вміст директорії uploads](/assets/labs/lab-4/uploads-directory.png)
**Рис. 7 - Файли, збережені сервером у директорії `uploads`.**

![Вміст файла app log з HTTP-запитами та помилками](/assets/labs/lab-4/app-log.png)
**Рис. 8 - Вміст файла `app.log` з подіями, помилками та часом відповіді.**

![Логування HTTP-запитів Morgan у консолі](/assets/labs/lab-4/morgan-console-log.png)
**Рис. 9 - Виведення HTTP-запитів у консоль через Morgan.**

![Процес studentlab-api у PM2](/assets/labs/lab-4/pm2-list.png)
**Рис. 10 - Запуск API через PM2 та перегляд процесу `studentlab-api`.**

---

## 8. Висновки

У межах лабораторної роботи серверну частину StudentLab API розширено можливостями логування, завантаження файлів і моніторингу стану процесу. Для HTTP-логування використано Morgan, а для файлових логів — Winston, який записує інформаційні події та помилки у файл `app.log`. Додатково для кожного HTTP-запиту вимірюється час відповіді.

Для роботи з файлами реалізовано два маршрути: `POST /upload` для одного файла та `POST /upload-multiple` для кількох файлів. Додано перевірку MIME-типу й обмеження розміру файла, тому сервер приймає лише дозволені формати та повертає зрозумілу помилку у випадку некоректного запиту.

Маршрут `GET /status` повертає uptime і використання пам'яті, що демонструє базовий моніторинг Node.js-застосунку. Також підготовлено конфігурацію PM2 для запуску backend як окремого керованого процесу.

---

## 9. Перелік використаних джерел
1. Документація Node.js (`process`, `fs`, `path`).
2. Документація Express.js.
3. Документація Morgan.
4. Документація Winston.
5. Документація Multer.
6. Документація PM2.
