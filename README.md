# AntiPlagiarism-KPO-DZ3

> Учебный микросервисный проект по КПО: система проверки студенческих работ на плагиат.

---
## Содержание

- [Общее описание](#общее-описание)
- [Архитектура](#архитектура)
  - [File Storage](#file-storage)
  - [Analysis](#analysis)
  - [Gateway](#gateway)
- [Структура репозитория](#структура-репозитория)
- [Запуск](#запуск)
  - [Через Visual Studio](#через-visual-studio)
  - [Через dotnet run](#через-dotnet-run)
- [Настройки и порты](#настройки-и-порты)
- [API-спецификация](#api-спецификация)
  - [Gateway API](#gateway-api)
  - [Analysis API](#analysis-api)
  - [File Storage API](#file-storage-api)
  - [Проверка через test-gateway.html](#Проверка-через-test-gateway.html)
- [Алгоритм проверки на плагиат](#алгоритм-проверки-на-плагиат)
- [Типичный сценарий использования](#типичный-сценарий-использования)


---

## Общее описание

Система имитирует проверку студенческих работ на плагиат:

1. Студент отправляет файл через **Gateway**.
2. Gateway:
   - сохраняет файл в **File Storage**;
   - регистрирует сдачу в **Analysis**;
   - возвращает студенту результат проверки.
3. Преподаватель может просмотреть:
   - отчёт по конкретной сдаче;
   - все отчёты по одной работе (контрольной).

Для простоты всё хранение реализовано **в памяти** (in-memory репозитории).

---

## Архитектура

Проект состоит из трёх микросервисов:

- `AntiPlagiarism` – **File Storage**
- `AntiPlagiarism.Analysis.Api` – **Analysis**
- `AntiPlagiarism.Gateway.Api` – **Gateway**

Общая схема взаимодействия:

```text
   [Student Browser]
          |
          v
   +--------------+
   |   Gateway    |
   | (API / UI)   |
   +--------------+
     |         |
     | HTTP    | HTTP
     v         v
+-----------+  +----------------+
|FileStorage|  |    Analysis    |
+-----------+  +----------------+
     |               |
 In-memory       In-memory
 file meta       submissions/reports
```

### File Storage

Отвечает за:

- приём файлов (`multipart/form-data`);
- присвоение `fileId`;
- хранение метаданных (`StoredFileMetadata`);
- выдачу метаданных по `fileId`.



---

### Analysis

Отвечает за:

- регистрацию сдачи (`Submission`);
- хранение отчётов (`Report`);
- анализ плагиата среди работ одной и той же `workId`
- предоставление отчётов по `reportId` и по `workId`.

Работает с `fileId`, полученным из File Storage.

---

### Gateway

Единственная точка входа для внешнего клиента:

- принимает файлы от студентов;
- общается с File Storage и Analysis;
- возвращает агрегированный результат;
- предоставляет удобный API и Swagger для преподавателя.

---

## Структура репозитория

В корне:

```text
AntiPlagiarism-KPO-DZ3/
│
├─ AntiPlagiarism.sln                
│
├─ AntiPlagiarism/                   # File Storage сервис
│   ├─ Program.cs
│   ├─ appsettings.json
│   ├─ Contracts/
│   ├─ Models/
│   └─ Services/
│
├─ AntiPlagiarism.Analysis.Api/      # Analysis сервис
│   ├─ Program.cs
│   ├─ appsettings.json
│   ├─ Contracts/
│   ├─ Models/
│   ├─ Services/
│   └─ Repositories/
│
├─ AntiPlagiarism.Gateway.Api/       # Gateway сервис
│   ├─ Program.cs
│   ├─ appsettings.json
│   ├─ Contracts/
│   └─ Clients/
│
├─ README.md
└─ .gitignore
```

---

## Настройки и порты

Порты могут отличаться, смотрите `Now listening on:` в логах.  
Типовая конфигурация (для примера):

- File Storage: `http://localhost:5110`
- Analysis: `http://localhost:5173`
- Gateway: `http://localhost:5053`

Gateway берёт адреса внутренних сервисов из `appsettings.json`:

```json
"Downstream": {
  "FileStorage": {
    "BaseUrl": "http://localhost:5110"
  },
  "Analysis": {
    "BaseUrl": "http://localhost:5173"
  }
}
```

При необходимости можно изменить порты в настройках запуска проектов.

---

## API-спецификация

У каждого сервиса есть свой Swagger UI:

- File Storage: `http://localhost:5110/swagger`
- Analysis: `http://localhost:5173/swagger`
- Gateway: `http://localhost:5053/swagger`

Ниже кратко описаны ключевые эндпоинты.

---

## Gateway API

**Базовый URL:** `http://localhost:5053`

### 1. `POST /api/works/{workId}/submit`

Отправка студентом работы на проверку.

**Параметры маршрута**:

- `workId` – строка, идентификатор работы (например, `"control-1"`).

**Тело запроса** (`multipart/form-data`):

| Поле       | Тип    | Обязательно | Описание                           |
|-----------|--------|-------------|------------------------------------|
| `file`     | file   | да          | Файл работы                        |
| `studentId`| string | да          | Идентификатор студента             |
| `fileKind` | string | нет         | Тип файла (по умолчанию `Work`)   |

**Пример ответа 200 OK** (`SubmissionAnalysisResponseDto`):

```json
{
  "submissionId": "3f8a5f64-5717-4562-b3fc-2c963f66afa6",
  "workId": "control-1",
  "studentId": "s1",
  "fileId": "0c5c47cd-841c-4650-b704-272aafa0ba48",
  "status": "Completed",
  "reportId": "27a5d01a-d4f1-476b-aace-7272769e5c47",
  "isPlagiarism": false,
  "similarityPercent": 0,
  "baseSubmissionId": null,
  "createdAtUtc": "2025-12-12T01:11:28.1944388Z"
}
```

**Что делает Gateway под капотом:**

1. Отправляет файл в File Storage: `POST /files`.
2. Получает `fileId`.
3. Создаёт сабмишен в Analysis: `POST /submissions`.
4. Ждёт, пока Analysis выполнит проверку и сформирует отчёт.
5. Возвращает агрегированный результат клиенту.

---

### 2. `GET /api/reports/{reportId}`

Получение одного отчёта по идентификатору.

**Параметры маршрута**:

- `reportId` – GUID отчёта.

**Пример ответа 200 OK** (`ReportResponseDto`):

```json
{
  "reportId": "27a5d01a-d4f1-476b-aace-7272769e5c47",
  "submissionId": "3f8a5f64-5717-4562-b3fc-2c963f66afa6",
  "workId": "control-1",
  "studentId": "s1",
  "fileId": "0c5c47cd-841c-4650-b704-272aafa0ba48",
  "isPlagiarism": false,
  "similarityPercent": 0,
  "baseSubmissionId": null,
  "createdAtUtc": "2025-12-12T01:11:28.1944388Z",
  "submittedAtUtc": "2025-12-12T01:11:28.1152604Z"
}
```

Gateway просто проксирует запрос в Analysis.

---

### 3. `GET /api/works/{workId}/reports`

Получение всех отчётов по конкретной работе (`workId`).

**Параметры маршрута**:

- `workId` – идентификатор работы (например, `"control-1"`).

**Пример ответа 200 OK** (`WorkReportsResponseDto`):

```json
{
  "workId": "control-1",
  "reports": [
    {
      "reportId": "27a5d01a-d4f1-476b-aace-7272769e5c47",
      "submissionId": "e0a437a2-3134-4bb9-a927-e8c6eba637ba",
      "workId": "control-1",
      "studentId": "s1",
      "fileId": "0c5c47cd-841c-4650-b704-272aafa0ba48",
      "isPlagiarism": false,
      "similarityPercent": 0,
      "baseSubmissionId": null,
      "createdAtUtc": "2025-12-12T01:11:28.1944388Z",
      "submittedAtUtc": "2025-12-12T01:11:28.1152604Z"
    },
    {
      "reportId": "a49aaf9a-105b-4e55-8e8a-e63aa33f8165",
      "submissionId": "4eeac9bb-b13a-40fc-814d-54aac44e13c0",
      "workId": "control-1",
      "studentId": "s2",
      "fileId": "0417a347-8bcc-472f-9f41-c1442df8752e",
      "isPlagiarism": true,
      "similarityPercent": 100,
      "baseSubmissionId": "e0a437a2-3134-4bb9-a927-e8c6eba637ba",
      "createdAtUtc": "2025-12-12T01:13:36.4253079Z",
      "submittedAtUtc": "2025-12-12T01:13:36.4071273Z"
    }
  ]
}
```

---

## Analysis API

**Базовый URL:** `http://localhost:5173`

### 1. `POST /submissions`

Создаёт новую сдачу и выполняет анализ на плагиат.

**Тело запроса** (`CreateSubmissionRequestDto`):

```json
{
  "workId": "control-1",
  "studentId": "s1",
  "fileId": "0c5c47cd-841c-4650-b704-272aafa0ba48",
  "submittedAtUtc": "2025-12-12T01:11:28.1152604Z"
}
```

**Ответ** — объект с информацией о созданной сдаче и связном отчёте.

### 2. `GET /reports/{reportId}`

Выдаёт отчёт по конкретной сдаче.

### 3. `GET /works/{workId}/reports`

Список отчётов по работe.

---

## File Storage API

**Базовый URL:** `http://localhost:5110`

### 1. `POST /files`

Загрузка файла и сохранение метаданных.

**Тело запроса** (`multipart/form-data`):

| Поле      | Тип    | Обязательно | Описание                         |
|-----------|--------|-------------|---------------------------------|
| `file`    | file   | да          | Файл работы                     |
| `fileKind`| string | нет         | Тип файла (Work, Attachment...) |

**Ответ 200 OK** (`UploadFileResponseDto`):

```json
{
  "fileId": "0c5c47cd-841c-4650-b704-272aafa0ba48",
  "fileName": "test1.txt",
  "sizeBytes": 123,
  "contentType": "text/plain",
  "fileKind": "Work",
  "uploadedAtUtc": "2025-12-12T01:11:28.1152604Z"
}
```

### 2. `GET /files/{id}/metadata`

Возвращает метаданные файла (`StoredFileMetadata`) по `fileId`.

---
## Проверка через test-gateway.html

Помимо Swagger, для ручной проверки есть простой HTML-файл  
`test-gateway.html`, который отправляет форму напрямую в Gateway.

### Где лежит файл

```text
AntiPlagiarism-KPO-DZ3/
│
├─ AntiPlagiarism.sln
├─ AntiPlagiarism/
├─ AntiPlagiarism.Analysis.Api/
├─ AntiPlagiarism.Gateway.Api/
├─ test-gateway.html      # HTML-форма для проверки через браузер
└─ README.md
```

## Алгоритм проверки на плагиат

Упрощённая учебная логика:

1. Все сдачи группируются по `workId` .
2. При поступлении новой работы:
   - Analysis сравнивает её содержимое (или хэш) с уже существующими.
   - Если найдено полное совпадение **с любой прошлой сдачей**:
     - `isPlagiarism = true`;
     - `similarityPercent = 100`;
     - `baseSubmissionId` = `submissionId` «оригинала».
   - Иначе:
     - `isPlagiarism = false`;
     - `similarityPercent = 0`;
     - `baseSubmissionId = null`.

3. Для демонстрации достаточно:
   - сначала отправить файл как студент `s1` → не плагиат;
   - потом тот же файл как `s2` → плагиат, указывающий на работу `s1`.

---

## Типичный сценарий использования

1. **Запуск сервисов** 
2. **Отправка первой работы** (`studentId = s1`):
   - В Gateway Swagger: `POST /api/works/control-1/submit`.
   - Прикрепить `test1.txt`.
   - Ответ: `isPlagiarism = false`.
3. **Отправка второй работы** (`studentId = s2`, тот же `test1.txt`):
   - Тот же эндпоинт.
   - Ответ: `isPlagiarism = true`, `similarityPercent = 100`,
     `baseSubmissionId` = id первой сдачи.
4. **Просмотр отчётов**:
   - `GET /api/works/control-1/reports` → список всех отчётов по работе.
   - `GET /api/reports/{reportId}` → детальный отчёт по конкретной сдаче.

Этот сценарий демонстрирует полную цепочку:
**Gateway → File Storage → Analysis → Gateway**.

---


