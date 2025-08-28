# DATABASE.md
# Структура базы данных MegaDoc (MySQL)

## 📌 Общая схема

База данных состоит из нескольких взаимосвязанных таблиц, обеспечивающих работу древовидной структуры веток, сообщений и пользователей.

## 🗂 Таблицы

### Таблица `branches`
Содержит информацию о всех ветках системы.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(50)` | Уникальный идентификатор ветки (человекочитаемый) | `PRIMARY KEY` |
| `name` | `VARCHAR(255)` | Название ветки | `NOT NULL` |
| `description` | `TEXT` | Описание ветки | |
| `parent_id` | `VARCHAR(50)` | ID родительской ветки | `FOREIGN KEY REFERENCES branches(id)` |
| `is_visible` | `BOOLEAN` | Флаг видимости ветки | `DEFAULT TRUE` |
| `sort_order` | `INT` | Порядок сортировки среди веток одного уровня | `DEFAULT 0` |
| `created_at` | `DATETIME` | Дата и время создания | `DEFAULT CURRENT_TIMESTAMP` |
| `updated_at` | `DATETIME` | Дата и время последнего обновления | `DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP` |
| `created_by` | `BIGINT` | user_id создателя ветки | `NOT NULL` |

### Таблица `branch_tags`
Связь "многие-ко-многим" между ветками и тегами. Реализует механизм тегирования.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `branch_id` | `VARCHAR(50)` | ID ветки | `FOREIGN KEY, PRIMARY KEY` |
| `tag` | `VARCHAR(255)` | Текст тега (например, "#KAP") | `PRIMARY KEY` |

### Таблица `messages`
Хранит все сообщения, привязанные к веткам.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `id` | `VARCHAR(255)` | Уникальный ID сообщения (UUID) | `PRIMARY KEY` |
| `branch_id` | `VARCHAR(50)` | ID ветки, к которой привязано сообщение | `FOREIGN KEY NOT NULL` |
| `user_id` | `BIGINT` | user_id отправителя (из Telegram) | `NOT NULL` |
| `content` | `TEXT` | Текст сообщения | `NOT NULL` |
| `type` | `ENUM(...)` | Тип: 'message', 'decision', 'problem', 'idea' | `DEFAULT 'message'` |
| `reply_to_id` | `VARCHAR(255)` | ID сообщения, на которое дан ответ | `FOREIGN KEY` |
| `timestamp` | `DATETIME` | Время отправки | `DEFAULT CURRENT_TIMESTAMP` |

### Таблица `users`
Информация о пользователях бота.
| Поле | Тип | Описание | Ограничения |
| :--- | :--- | :--- | :--- |
| `id` | `BIGINT` | user_id из Telegram | `PRIMARY KEY` |
| `username` | `VARCHAR(255)` | @username пользователя | |
| `first_name` | `VARCHAR(255)` | Имя пользователя | |
| `role` | `ENUM('admin','moderator','user')` | Роль в системе | `DEFAULT 'user'` |

## ⚙️ Триггеры и процедуры (для реализации позже)

-   **`prevent_cyclic_branch`**: Триггер для проверки на циклические связи перед обновлением `parent_id`.
-   **`update_branch_children`**: Процедура для каскадного обновления дочерних веток.

## 🔗 Связи и целостность данных

-   При удалении ветки все её сообщения должны удаляться (`ON DELETE CASCADE`).
-   Поле `parent_id` должно ссылаться на существующую ветку или быть `NULL` (для корневых веток).

---
*Документ в процессе разработки. Обновлён: 28.08.2025*
