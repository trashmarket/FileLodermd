# Пример: Отправка файла в формате Base64 через JSON

Вот пример, как это можно сделать front:
```js
const fileField = document.querySelector('input[type="file"]').files[0];

// Чтение файла и его преобразование в Base64
const toBase64 = file => new Promise((resolve, reject) => {
  const reader = new FileReader();
  reader.readAsDataURL(file);
  reader.onload = () => resolve(reader.result);
  reader.onerror = error => reject(error);
});

async function sendJsonWithFile() {
  try {
    // Кодируем файл в Base64
    const base64File = await toBase64(fileField);
    
    // Создаём JSON объект для отправки
    const data = {
      username: "abc123",
      avatar: base64File  // Добавляем файл в формате Base64
    };

    // Отправляем JSON данные
    const response = await fetch("https://example.com/profile/avatar", {
      method: "PUT",
      headers: {
        "Content-Type": "application/json",
        "Authorization": "Bearer YOUR_ACCESS_TOKEN", // если требуется токен
      },
      body: JSON.stringify(data),
    });

    const result = await response.json();
    console.log("Успех:", JSON.stringify(result));
  } catch (error) {
    console.error("Ошибка:", error);
  }
}

sendJsonWithFile();
```

# Пример обработки Base64 на сервере Node.js

```js
const express = require('express');
const fs = require('fs');
const path = require('path');

const app = express();

// Используем встроенные парсеры JSON и urlencoded
app.use(express.json({ limit: '10mb' })); // Лимит на размер JSON
app.use(express.urlencoded({ extended: true }));

app.put('/profile/avatar', (req, res) => {
  try {
    const { username, avatar } = req.body;

    // Проверка наличия Base64 строки
    if (!avatar || !avatar.startsWith("data:image")) {
      return res.status(400).json({ message: "Некорректный формат изображения" });
    }

    // Отделение Base64 данных от типа
    const base64Data = avatar.split(',')[1];

    // Декодирование Base64 строки в бинарный формат
    const binaryData = Buffer.from(base64Data, 'base64');

    // Сохранение файла
    const filePath = path.join(__dirname, 'uploads', `${username}_avatar.png`);
    fs.writeFileSync(filePath, binaryData);

    res.json({ message: "Аватар успешно сохранен", path: filePath });
  } catch (error) {
    console.error("Ошибка:", error);
    res.status(500).json({ message: "Ошибка сервера" });
  }
});

// Запуск сервера
app.listen(3000, () => {
  console.log('Сервер запущен на http://localhost:3000');
});
```

### Когда нужен express.urlencoded()
Если вы отправляете данные через HTML-форму, и у формы не указано enctype="multipart/form-data", то данные будут отправлены в application/x-www-form-urlencoded.
В этом случае express.urlencoded() позволяет серверу корректно распарсить эти данные.
#### Пример HTML-формы
Вот пример формы, которая отправляет данные в application/x-www-form-urlencoded формате:
```html
<form action="/submit" method="POST">
  <label for="username">Username:</label>
  <input type="text" id="username" name="username">
  
  <label for="email">Email:</label>
  <input type="email" id="email" name="email">
  
  <button type="submit">Отправить</button>
</form>
```

# Сервер на Node.js с использованием multer

## Установка multer
```bash
npm install multer
```

## Пример HTML-формы с multipart/form-data
```html
<form action="/upload" method="POST" enctype="multipart/form-data">
  <label for="username">Username:</label>
  <input type="text" id="username" name="username">

  <label for="avatar">Avatar:</label>
  <input type="file" id="avatar" name="avatar">
  
  <button type="submit">Загрузить</button>
</form>
```
### Теперь настроим сервер для обработки этой формы:
```js
const express = require('express');
const multer = require('multer');
const path = require('path');

// Настройка хранилища файлов с помощью multer
const storage = multer.diskStorage({
  destination: (req, file, cb) => {
    cb(null, 'uploads/'); // Папка для сохранения файлов
  },
  filename: (req, file, cb) => {
    const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
    cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
  }
});

const upload = multer({ storage: storage });

const app = express();

// Маршрут для загрузки файлов
app.post('/upload', upload.single('avatar'), (req, res) => {
  const username = req.body.username; // Получаем текстовое поле
  const file = req.file; // Получаем информацию о загруженном файле

  // Проверяем успешность загрузки
  if (!file) {
    return res.status(400).json({ message: 'Файл не загружен' });
  }

  res.json({
    message: 'Файл успешно загружен',
    file: {
      originalname: file.originalname,
      filename: file.filename,
      path: file.path,
      size: file.size,
    },
    username: username,
  });
});

// Запуск сервера
app.listen(3000, () => {
  console.log('Сервер запущен на http://localhost:3000');
});
```
***
Пояснение коду
Настройка хранения с помощью multer.diskStorage:
Указываем destination как папку, куда будут сохраняться файлы.
filename задает уникальное имя файла, добавляя к нему временную метку.
Маршрут /upload:
Используем upload.single('avatar') для обработки одного файла, переданного под именем avatar.
req.body.username позволяет получить текстовое поле username, переданное вместе с файлом.
req.file содержит информацию о загруженном файле: его оригинальное имя, новое имя, путь, размер и т. д.
***
