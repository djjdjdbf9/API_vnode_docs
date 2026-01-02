
# 🧬 Serotonin Auth API

![API Version](https://img.shields.io/badge/API-v1.0-blue?style=flat-square)
![Format](https://img.shields.io/badge/format-JSON-orange?style=flat-square)
![Protocol](https://img.shields.io/badge/protocol-HTTPS-green?style=flat-square)

Документация для интеграции клиентского программного обеспечения (Loader/Client) с сервером авторизации **Serotonin**.
API предоставляет безопасные методы для аутентификации пользователей и получения статусов активных подписок без передачи HWID.

## 📋 Основные сведения

Взаимодействие с API происходит посредством HTTP-запросов (POST). Данные передаются и принимаются в формате **JSON**.

- **Base URL:** `https://api.serotonin.su`
- **Endpoint:** `/api.php`
- **Content-Type:** `application/json`
- **Кодировка:** `UTF-8`

> ⚠️ **Важно:** Все запросы должны отправляться по протоколу **HTTPS** для обеспечения безопасности передаваемых данных (паролей).

---

## 🔐 Авторизация (Login)

Единственный метод для валидации пользователя. При успешном входе возвращает список доступных продуктов, их статусы и срок действия.

### Запрос (Request)

- **URL:** `https://api.serotonin.su/api.php`
- **Method:** `POST`
- **Header:** `Content-Type: application/json`

#### Параметры тела запроса

| Параметр | Тип | Обязательно | Описание |
| :--- | :--- | :--- | :--- |
| `username` | `string` | Да | Логин пользователя в системе. |
| `password` | `string` | Да | Пароль пользователя (Plain Text). |

**Пример запроса (JSON):**

```json
{
  "username": "Customer1337",
  "password": "MySuperSecretPassword"
}

```

---

### Ответ (Response)

Сервер всегда возвращает объект JSON. Логика приложения должна основываться на поле `status`.

#### ✅ Сценарий: Успешный вход (`success`)

Возвращается, если учетные данные верны, и у пользователя нет активного бана.

```json
{
  "status": "success",
  "username": "Customer1337",
  "subscriptions": [
    {
      "name": "Serotonin: CS2 Legion",
      "status": "Undetected",
      "expires_at": "2026-05-20 14:00:00"
    },
    {
      "name": "Serotonin: Rust Private",
      "status": "Updating",
      "expires_at": "2026-06-01 00:00:00"
    }
  ]
}

```

**Описание полей ответа:**

| Поле | Тип | Описание |
| --- | --- | --- |
| `status` | `string` | Всегда `success`. |
| `username` | `string` | Имя пользователя (как записано в базе данных). |
| `subscriptions` | `array` | Список активных подписок. Если массив пустой (`[]`) — у пользователя нет купленных продуктов. |

**Объект подписки (Subscription Object):**

| Поле | Тип | Описание |
| --- | --- | --- |
| `name` | `string` | Полное название продукта. |
| `status` | `string` | Текущий статус безопасности (см. раздел "Статусы"). |
| `expires_at` | `string` | Дата истечения подписки (формат `YYYY-MM-DD HH:MM:SS`). |

#### ❌ Сценарий: Ошибка (`error`)

Возвращается при неверном пароле, отсутствии пользователя в базе или блокировке аккаунта.

```json
{
  "status": "error",
  "message": "Incorrect username or password"
}

```

| Поле | Тип | Описание |
| --- | --- | --- |
| `status` | `string` | Всегда `error`. |
| `message` | `string` | Текст ошибки для отображения пользователю (например: `User is Banned`, `Invalid Credentials`). |

---

## 🚦 Статусы продуктов

Клиентское приложение (Loader) должно обрабатывать поле `status` для каждого продукта, чтобы разрешать или запрещать запуск (Inject).

| Статус | Цвет индикатора | Значение | Действие клиента |
| --- | --- | --- | --- |
| `Undetected` | 🟢 Green | Софт безопасен и работает. | **Разрешить** запуск. |
| `Testing` | 🔵 Cyan | Софт на тестах (возможны баги). | **Предупредить** пользователя (Use at own risk). |
| `Updating` | 🟠 Orange | Софт обновляется разработчиком. | **Блокировать** запуск. |
| `Detected` | 🔴 Red | Софт обнаружен античитом. | **Блокировать** запуск (Critical). |

---

## 💻 Пример интеграции (C++)

Ниже приведен пример реализации авторизации на C++ с использованием библиотек `libcurl` и `nlohmann/json`.

### Зависимости

* [libcurl](https://curl.se/libcurl/) — для сетевых запросов.
* [nlohmann/json](https://github.com/nlohmann/json) — для парсинга JSON.

### Код клиента

```cpp
#include <iostream>
#include <string>
#include <curl/curl.h>
#include <nlohmann/json.hpp>

using json = nlohmann::json;

// Функция для записи ответа сервера в строку
size_t WriteCallback(void* contents, size_t size, size_t nmemb, void* userp) {
    ((std::string*)userp)->append((char*)contents, size * nmemb);
    return size * nmemb;
}

void AuthUser(std::string username, std::string password) {
    CURL* curl;
    CURLcode res;
    std::string readBuffer;

    curl = curl_easy_init();
    if (curl) {
        // 1. Настройка URL
        curl_easy_setopt(curl, CURLOPT_URL, "[https://api.serotonin.su/api.php](https://api.serotonin.su/api.php)");
        
        // 2. SSL Verification (Рекомендуется включить в Release!)
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, 0L); 
        curl_easy_setopt(curl, CURLOPT_SSL_VERIFYHOST, 0L);

        // 3. Формирование JSON
        json reqJson;
        reqJson["username"] = username;
        reqJson["password"] = password;
        std::string jsonStr = reqJson.dump();

        // 4. Заголовки
        struct curl_slist* headers = NULL;
        headers = curl_slist_append(headers, "Content-Type: application/json");
        curl_easy_setopt(curl, CURLOPT_HTTPHEADER, headers);

        // 5. Параметры POST
        curl_easy_setopt(curl, CURLOPT_POSTFIELDS, jsonStr.c_str());
        curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallback);
        curl_easy_setopt(curl, CURLOPT_WRITEDATA, &readBuffer);

        // 6. Выполнение
        res = curl_easy_perform(curl);

        if (res == CURLE_OK) {
            try {
                // Парсинг ответа
                auto response = json::parse(readBuffer);
                
                if (response["status"] == "success") {
                    std::cout << "[+] Auth Success! User: " << response["username"] << std::endl;
                    
                    // Итерация по подпискам
                    for (const auto& sub : response["subscriptions"]) {
                        std::cout << " -> Product: " << sub["name"] 
                                  << " [" << sub["status"] << "]"
                                  << " Exp: " << sub["expires_at"] << std::endl;
                    }
                } else {
                    std::cout << "[-] Error: " << response["message"] << std::endl;
                }
            } catch (json::parse_error& e) {
                std::cerr << "JSON Parse failed: " << e.what() << std::endl;
            }
        } else {
            std::cerr << "Request failed: " << curl_easy_strerror(res) << std::endl;
        }
        
        // Очистка памяти
        curl_easy_cleanup(curl);
        curl_slist_free_all(headers);
    }
}

```

---

## 🛠 Troubleshooting

### 1. Ответ не приходит (Connection Failed)

* Проверьте подключение к интернету.
* Убедитесь, что фаервол не блокирует запросы к `api.serotonin.su`.
* Попробуйте открыть URL в браузере (должна быть пустая страница или ошибка метода, но не ошибка подключения).

### 2. JSON Parse Error

* Убедитесь, что сервер возвращает корректный JSON (`application/json`), а не HTML страницу ошибки (например, 502 Bad Gateway или 404 Not Found).
* Выведите сырой ответ сервера (`readBuffer`) в консоль перед парсингом для отладки.

---

© 2026 Serotonin Project. All rights reserved.

```

```
