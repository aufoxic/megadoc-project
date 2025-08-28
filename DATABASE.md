# DATABASE.md
# Структура базы данных MegaDoc (MySQL)

## 📌 Общие соглашения

### Идентификаторы (ID)
- Все ID (веток, сообщений) должны соответствовать шаблону: `[a-zA-Z0-9_-]+`
- Запрещено использовать пробелы, кириллицу, спецсимволы кроме дефиса и подчеркивания
- Примеры валидных ID: `MEGADOC_CORE`, `new-feature-2025`, `task_123`

### Многоязычная поддержка
- Система поддерживает хранение и отображение контента на разных языках
- При отсутствии перевода используется fallback на язык оригинала
- Переводы могут добавляться автоматически (через API) или вручную

### Неизменность истории
- Данные никогда не удаляются физически из базы
- Используется механизм soft delete (пометка об удалении)
- Сохраняется оригинальный контент и история изменений

## 🗂 Таблицы

### Таблица `branches`
Содержит информацию о всех ветках системы.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(50)` | Уникальный идентификатор ветки | `PRIMARY KEY` |
| `name` | `VARCHAR(255)` | Название ветки (оригинал) | `NOT NULL` |
| `description` | `TEXT` | Описание ветки (оригинал) | |
| `parent_id` | `VARCHAR(50)` | ID родительской ветки | `FOREIGN KEY REFERENCES branches(id)` |
| `is_visible` | `BOOLEAN` | Флаг видимости ветки | `DEFAULT TRUE` |
| `sort_order` | `INT` | Порядок сортировки | `DEFAULT 0` |
| `created_at` | `DATETIME` | Дата и время создания | `DEFAULT CURRENT_TIMESTAMP` |
| `updated_at` | `DATETIME` | Дата и время обновления | `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` |
| `created_by` | `BIGINT` | user_id создателя ветки | `NOT NULL` |

### Таблица `branch_translations`
Переводимые поля для веток.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `branch_id` | `VARCHAR(50)` | ID ветки | `FOREIGN KEY, PRIMARY KEY` |
| `language_code` | `VARCHAR(5)` | Код языка | `PRIMARY KEY, FOREIGN KEY REFERENCES languages(code)` |
| `name` | `VARCHAR(255)` | Перевод названия | |
| `description` | `TEXT` | Перевод описания | |

### Таблица `messages`
Хранит все сообщения, привязанные к веткам.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(255)` | Уникальный ID сообщения (UUID) | `PRIMARY KEY` |
| `branch_id` | `VARCHAR(50)` | ID ветки | `FOREIGN KEY NOT NULL` |
| `user_id` | `BIGINT` | user_id отправителя | `NOT NULL` |
| `content` | `TEXT` | Текущий текст сообщения | `NOT NULL` |
| `type` | `ENUM('message','decision','problem','idea')` | Тип сообщения | `DEFAULT 'message'` |
| `reply_to_id` | `VARCHAR(255)` | ID сообщения-оригинала | `FOREIGN KEY` |
| `timestamp` | `DATETIME` | Время создания | `DEFAULT CURRENT_TIMESTAMP` |
| `is_edited` | `BOOLEAN` | Флаг редактирования | `DEFAULT FALSE` |
| `is_deleted` | `BOOLEAN` | Флаг удаления (soft delete) | `DEFAULT FALSE` |
| `original_content` | `TEXT` | Оригинальный текст до редактирования | |
| `edited_at` | `DATETIME` | Время последнего редактирования | |

### Таблица `message_translations`
Переводимые поля для сообщений.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `message_id` | `VARCHAR(255)` | ID сообщения | `FOREIGN KEY, PRIMARY KEY` |
| `language_code` | `VARCHAR(5)` | Код языка | `PRIMARY KEY, FOREIGN KEY REFERENCES languages(code)` |
| `content` | `TEXT` | Перевод содержимого | |

### Таблица `users`
Информация о пользователях бота.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | user_id из Telegram | `PRIMARY KEY` |
| `username` | `VARCHAR(255)` | @username пользователя | |
| `first_name` | `VARCHAR(255)` | Имя пользователя | |
| `role` | `ENUM('admin','moderator','user')` | Роль в системе | `DEFAULT 'user'` |
| `language_code` | `VARCHAR(5)` | Предпочтительный язык | `FOREIGN KEY REFERENCES languages(code)` |

### Таблица `languages`
Справочник поддерживаемых языков.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `code` | `VARCHAR(5)` | Код языка (ru, en, zh, etc.) | `PRIMARY KEY` |
| `name` | `VARCHAR(50)` | Название языка | `NOT NULL` |
| `is_active` | `BOOLEAN` | Флаг активности языка | `DEFAULT TRUE` |

### Таблица `branch_tags`
Связь "многие-ко-многим" между ветками и тегами.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `branch_id` | `VARCHAR(50)` | ID ветки | `FOREIGN KEY, PRIMARY KEY` |
| `tag` | `VARCHAR(255)` | Текст тега (например, "#KAP") | `PRIMARY KEY` |

## ⚙️ Триггеры и процедуры

-   **`prevent_cyclic_branch`**: Триггер для проверки на циклические связи перед обновлением `parent_id`
-   **`update_branch_children`**: Процедура для каскадного обновления дочерних веток
-   **`save_message_history`**: Триггер для сохранения оригинального контента при первом редактировании

## 🔗 Связи и целостность данных

-   При удалении ветки все связанные переводы и теги должны удаляться (`ON DELETE CASCADE`)
-   Поле `parent_id` должно ссылаться на существующую ветку или быть `NULL`
-   Все внешние ключи должны быть properly indexed для производительности

## 🌐 Механизм работы с переводами

1. **Определение языка пользователя:**
   - При первом контакте с ботом определяется язык пользователя
   - Сохраняется в поле `users.language_code`
   - Пользователь может сменить язык через команду `/language`

2. **Получение переведенного контента:**
   - При запросе данных система ищет перевод для языка пользователя
   - Если перевод отсутствует - возвращается оригинальный текст
   - В интерфейсе пометка "перевод недоступен" для непереведенного контента

3. **Добавление переводов:**
   - Автоматически: через интеграцию с API переводчиков (Yandex, Google, OpenAI)
   - Вручную: через модераторский интерфейс с предложением правок

## 🚀 Примеры использования

-- Создание новой ветки с английским названием
INSERT INTO branches (id, name, description, created_by)
VALUES ('new_feature', 'New Feature', 'Description of new feature', 123456789);

-- Добавление русского перевода для ветки
INSERT INTO branch_translations (branch_id, language_code, name, description)
VALUES ('new_feature', 'ru', 'Новая функция', 'Описание новой функции');

-- Получение переведенного содержимого для пользователя
SELECT 
  COALESCE(bt.name, b.name) as name,
  COALESCE(bt.description, b.description) as description
FROM branches b
LEFT JOIN branch_translations bt ON b.id = bt.branch_id AND bt.language_code = 'ru'
WHERE b.id = 'new_feature';
