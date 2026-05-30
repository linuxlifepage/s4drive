Привет! Ниже — архитектурный план S4Drive как **надежного Google Drive/Dropbox‑подобного клиента поверх S3**, а не как GUI над `rclone`. Главная идея: **S3-бакет = диск/хранилище**, но надежность достигается не прямым “путь файла = S3 key”, а собственным слоем метаданных, версий, журналом операций и аккуратной синхронизацией.

## 1. Главные архитектурные решения

### 1.1. Базовый стек

**Рекомендованный стек для S4Drive v1:**

* **Rust Core** — вся критичная логика: S3 API, синхронизация, локальная база, очередь операций, конфликты, версионирование, шифрование, проверка целостности, лимиты ресурсов.
* **Tauri v2 Desktop UI** — Windows/Linux/macOS: окно, трей, настройки, файловый браузер, статус синхронизации. Tauri официально поддерживает desktop и mobile, использует Rust + web frontend, а для глубокой интеграции с платформами — Swift/Kotlin. ([Tauri][1])
* **Tauri Mobile или native shell поверх Rust Core** — Android/iOS. На первом этапе можно использовать Tauri mobile, но для production‑качества файловой интеграции все равно понадобятся native Kotlin/Swift плагины.
* **SQLite локально** — локальный индекс, очередь операций, статусы файлов, журнал синхронизации, кэш метаданных.
* **S3-compatible backend** — твое самописное S3-приложение, но с обязательным набором возможностей: strong consistency, conditional writes, multipart upload, checksums, HEAD, LIST, Range, корректные ETag/version-id.

Почему Rust Core обязателен: синхронизация — это не UI-задача. Ее нельзя надежно держать в JS/frontend. Нужно, чтобы движок переживал закрытие окна, сбои сети, sleep/wake, частичную загрузку, параллельные изменения и рестарты.

### 1.2. Tauri — да, но не как “вся платформа одним махом”

Для desktop Tauri выглядит очень подходящим: есть system tray API, tray menu, управление окном, возможность держать приложение в фоне, а также механизм sidecar-бинарников, если временно понадобится запускать внешний инструмент. ([Tauri][2])

Но мобильные платформы нельзя воспринимать как “desktop в телефоне”. На iOS и Android фоновые задачи, доступ к файловой системе, интеграция с Files/SAF и длительные синхронизации ограничены ОС. Поэтому правильная модель такая:

* UI можно переиспользовать через Tauri.
* Core можно переиспользовать через Rust.
* Но файловая интеграция и background execution должны быть native: Swift на iOS, Kotlin на Android. Tauri mobile plugins как раз позволяют вызывать native mobile code из Rust/приложения. ([Tauri][3])

### 1.3. Rclone/s3cmd — не как основа

**Не рекомендую делать rclone/s3cmd главным sync engine.**

Причины:

1. S4Drive должен управлять собственными метаданными, file_id, версиями, конфликтами, locks, журналом операций и UX-статусами.
2. `rclone sync` хорош как CLI-инструмент, но он не знает продуктовую модель S4Drive: “этот файл переименован”, “эта версия конфликтная”, “этот файл открыт на другом устройстве”, “это локальное изменение еще не закоммичено”.
3. На mobile внешний бинарник — плохая основа: ограничения ОС, упаковка, фоновые задачи, App Store/Play Store, контроль ресурсов.
4. Управление памятью и concurrency для S3 multipart uploads должно быть твоим, предсказуемым и наблюдаемым. Даже в rclone есть настройки, влияющие на память и concurrency, особенно для S3 multipart upload. ([Rclone][4])
5. Tauri умеет встраивать sidecar-бинарники, но это лучше оставить для миграции/диагностики, а не для главного пути синхронизации. ([Tauri][5])

**Итог:** S4Drive Sync Engine — свой, на Rust. Rclone можно использовать только как временный инструмент для импорта, экспорта, диагностики или “advanced compatibility mode”.

---

## 2. Целевая архитектура S4Drive

Логически приложение должно состоять из таких слоев:

1. **S4Drive UI**

   * Desktop: Tauri window + tray.
   * Mobile: mobile UI + native file provider integration.
   * Показывает файлы, статусы, настройки, конфликты, версии, аккаунты.

2. **App Shell**

   * Управление окном.
   * Трей.
   * Autostart.
   * Notifications.
   * Native context menu / file manager integration.
   * Update flow.

3. **Rust Core**

   * Sync engine.
   * Metadata engine.
   * Transfer engine.
   * Conflict resolver.
   * Local index.
   * Credential manager abstraction.
   * Background scheduler.
   * Diagnostics.

4. **Local Storage Layer**

   * Локальная папка синхронизации.
   * SQLite база.
   * Staging directory.
   * Cache directory.
   * Trash/recovery area.

5. **S3 Adapter**

   * AWS-compatible S3 API.
   * Твой custom S3 backend.
   * Проверка совместимости.
   * Поддержка conditional writes, multipart, checksums, HEAD, LIST, Range.

6. **S4 Metadata Protocol**

   * `.s4drive/` hidden prefix в бакете.
   * Журнал операций.
   * Snapshots.
   * File IDs.
   * Device IDs.
   * Version graph.
   * Locks/leases.
   * Tombstones.
   * Garbage collection.

7. **Platform Adapters**

   * Windows: Cloud Files API для placeholder/files-on-demand и интеграции с Explorer. Microsoft описывает Cloud Files API как набор Win32/WinRT API для sync providers, включая placeholder files и интеграцию с File Explorer. ([Microsoft Learn][6])
   * macOS/iOS: File Provider extension для доступа к удаленным файлам из Finder/Files и других приложений. Apple описывает File Provider как extension для файлов и папок, которыми управляет приложение и которые синхронизируются с remote storage. ([Apple Developer][7])
   * Android: DocumentsProvider/Storage Access Framework, потому что Android позволяет облачным/локальным провайдерам представлять документы через стандартный системный picker. ([Android Developers][8])
   * Linux: обычная sync folder v1, затем optional FUSE/desktop integrations.

---

## 3. Самое важное: модель хранения данных и метаданных

### 3.1. Почему нельзя полагаться только на S3 object metadata

S3 object metadata полезна, но она не должна быть главным источником правды для S4Drive.

Причины:

* S3 user-defined metadata задается при upload; после upload ее нельзя просто изменить — нужно копировать объект и задавать metadata заново. ([AWS Documentation][9])
* Metadata headers имеют ограничения по размеру.
* Переименование в обычном S3 — это copy + delete, а не файловая операция. AWS прямо описывает rename/move как копирование объекта и удаление исходного. Исключение — `RenameObject` для S3 Express One Zone directory buckets, но это не универсальная S3-совместимая возможность. ([AWS Documentation][10])
* Путь объекта — плохой идентификатор файла. Файл может быть переименован, перемещен, иметь конфликт имен, отличаться регистром на Windows/macOS/Linux.

**Вывод:** главный источник правды — не S3 metadata и не key path, а **S4Drive metadata log** внутри бакета.

### 3.2. Режим хранения: S4 Native Mode

Для максимальной надежности основной режим должен быть **S4 Native Mode**.

В бакете создается скрытый префикс:

* `.s4drive/system/` — bucket descriptor, schema version, capabilities.
* `.s4drive/devices/` — зарегистрированные устройства.
* `.s4drive/meta/heads/` — указатели на текущие состояния.
* `.s4drive/meta/ops/` — append-only журнал операций.
* `.s4drive/meta/snapshots/` — периодические снимки дерева файлов.
* `.s4drive/content/blobs/` — immutable content blobs.
* `.s4drive/locks/` — soft/hard leases.
* `.s4drive/trash/` — tombstones и удаленные версии.
* `.s4drive/gc/` — состояние сборщика мусора.

Пользователь видит “диск” как дерево папок и файлов, но внутри S3 это управляемая структура. Это ближе к тому, как должен работать Drive-клиент, а не к тупой зеркалке S3-префикса.

### 3.3. Compatibility Mode

Отдельно можно сделать **Compatibility Mode**:

* Файлы лежат в бакете обычными ключами: `Documents/report.docx`, `Photos/a.jpg`.
* `.s4drive/` хранит дополнительную информацию.
* Пользователь может открыть bucket через S3 browser и увидеть обычные файлы.

Но этот режим хуже:

* rename/move дороже и рискованнее;
* сложнее сохранить стабильный `file_id`;
* внешние инструменты могут менять объекты без S4Drive metadata;
* сложнее гарантировать отсутствие silent overwrite.

**Рекомендация:** v1 делать в S4 Native Mode. Compatibility Mode — позже как отдельный профиль с честным предупреждением: “удобнее для совместимости, менее надежно для сложной синхронизации”.

---

## 4. S4 Metadata Protocol v1

### 4.1. Главные сущности

**Bucket**

* `bucket_id`
* `schema_version`
* `created_at`
* `owner`
* `capabilities`
* `encryption_policy`
* `sync_policy`
* `gc_policy`

**Device**

* `device_id`
* `device_name`
* `platform`
* `public_key`
* `last_seen`
* `capabilities`
* `client_version`

**FileEntry**

* `file_id` — стабильный ID, не зависит от пути.
* `parent_id`
* `name`
* `normalized_name`
* `type`: file/folder/symlink-like-placeholder.
* `current_revision_id`
* `content_ref`
* `size`
* `content_hash`
* `mime`
* `created_at`
* `updated_at`
* `deleted_at`
* `version_history`
* `attributes`
* `lock_state`

**ContentBlob**

* `blob_id`
* `hash`
* `size`
* `checksum`
* `storage_key`
* `encryption_info`
* `created_by`
* `created_at`
* `ref_count` logically, not necessarily stored as mutable counter.

**Operation**

* `op_id`
* `device_id`
* `actor_id`
* `logical_clock`
* `base_head`
* `target_file_id`
* `op_type`
* `preconditions`
* `effects`
* `timestamp`
* `signature`

**Revision**

* `revision_id`
* `file_id`
* `parent_revision_id`
* `content_ref`
* `author_device_id`
* `created_at`
* `base_revision_id`
* `merge_state`

**Tombstone**

* `file_id`
* `path_at_delete`
* `deleted_by`
* `deleted_at`
* `retention_until`

**Lease/Lock**

* `file_id`
* `owner_device_id`
* `actor`
* `mode`: soft/hard/read/write.
* `expires_at`
* `heartbeat_at`
* `reason`

### 4.2. Почему file_id критичен

Без `file_id` приложение будет путать:

* rename с delete+create;
* move с новым файлом;
* case-only rename;
* edit после rename;
* concurrent rename + edit.

С `file_id` логика становится нормальной:

* Файл `A/report.docx` переименовали в `B/final.docx`.
* На другом устройстве его одновременно отредактировали.
* S4Drive видит, что это один и тот же `file_id`, и применяет edit к новой локации.
* Конфликта нет.

Это один из главных способов резко уменьшить количество конфликтов.

---

## 5. S3 backend contract: что обязан поддерживать твой S3

Чтобы S4Drive был реально надежным, твое S3-приложение должно пройти отдельный **S4Drive Compatibility Test Suite**.

### 5.1. Обязательный уровень

Нужно поддерживать:

1. **Strong read-after-write consistency**

   * После PUT/DELETE/LIST клиент должен видеть актуальное состояние.
   * AWS S3 сейчас обеспечивает strong read-after-write consistency для GET, PUT, LIST, metadata/tag/ACL changes. ([Amazon Web Services, Inc.][11])

2. **Conditional writes**

   * `If-None-Match` для создания объекта только если его еще нет.
   * `If-Match` для commit только если ETag совпадает.
   * AWS S3 поддерживает conditional writes через `If-None-Match` и `If-Match`; при несовпадении ETag операция падает, а concurrent requests могут дать conflict. ([AWS Documentation][12])

3. **HEAD Object**

   * Быстро получать размер, ETag, checksum, metadata, version-id.

4. **List Objects V2**

   * Стабильная пагинация.
   * Корректная работа prefix/delimiter.

5. **Range GET**

   * Для resume download, preview, streaming.

6. **Multipart Upload**

   * Для больших файлов.
   * AWS рекомендует multipart upload для объектов от 100 MB, а также подчеркивает retry только неудачных parts и параллельную загрузку. ([AWS Documentation][13])

7. **Checksum support**

   * Проверка целостности на upload/download.
   * Для multipart важно не считать ETag простым MD5 всего файла: AWS прямо указывает, что итоговый ETag после multipart не обязательно MD5 всего объекта. ([AWS Documentation][13])

8. **Корректные ошибки**

   * 412 Precondition Failed.
   * 409 Conflict.
   * 404 Not Found.
   * 403 Forbidden.
   * Retryable 5xx.

### 5.2. Очень желательный уровень

1. **Versioning**

   * Для восстановления и audit.
   * Не использовать как единственный version graph, но использовать как дополнительную страховку.

2. **Object Lock**

   * Для защиты системных metadata/snapshots от случайного удаления.
   * S3 Object Lock защищает версии объектов от удаления/перезаписи на retention period или indefinitely и работает поверх S3 Versioning. ([AWS Documentation][14])

3. **Bucket notifications / change feed**

   * Для мгновенной синхронизации без агрессивного polling.
   * В твоем custom S3 лучше добавить отдельный S4Drive change feed API: WebSocket/SSE/long polling.

4. **Server-side encryption**

   * Для данных at rest.

5. **Short-lived credentials**

   * Чтобы desktop/mobile не хранили долгоживущие root/API ключи.

---

## 6. Sync engine: логика синхронизации

### 6.1. Основной принцип

S4Drive должен синхронизировать не “файлы”, а **операции над версиями файлов**.

Не так:

* “локальный файл новее — загрузить”
* “удаленного нет — удалить”
* “mtime больше — победил”

А так:

* “у файла `file_id` была revision `R1`”
* “устройство A создало revision `R2` от `R1`”
* “устройство B создало revision `R3` от `R1`”
* “это sibling revisions, нужен merge/conflict flow”
* “rename — отдельная операция над `file_id`, не новая сущность”

### 6.2. Локальный цикл изменений

1. File watcher ловит изменение.
2. События debounce’ятся.
3. Core проверяет, что файл больше не пишется:

   * размер стабилен;
   * mtime стабилен;
   * нет временного имени приложения;
   * файл не залочен приложением, если ОС позволяет проверить.
4. Core считает быстрый fingerprint.
5. Если файл реально изменился — считает content hash.
6. Создает локальную pending revision в SQLite.
7. Кладет upload job в persistent queue.
8. Загружает content blob в S3 staging/content area.
9. Проверяет checksum.
10. Делает metadata commit через conditional write.
11. Если commit успешен — файл становится clean.
12. Если commit неуспешен — начинается rebase/conflict resolution.

### 6.3. Удаленный цикл изменений

1. Core получает remote changes:

   * через change feed, если есть;
   * через polling `head`;
   * через list ops после последнего известного checkpoint.
2. Загружает новые ops.
3. Проверяет подписи/валидность.
4. Применяет к локальному metadata graph.
5. Для нужных файлов скачивает content blobs.
6. Пишет в staging-файл.
7. Проверяет checksum.
8. Атомарно заменяет локальный файл, только если локальная base revision не изменилась.
9. Если локальный файл изменился параллельно — создает конфликт, не перетирает данные.

### 6.4. Commit через CAS

Для metadata head нужен compare-and-swap-подобный механизм:

1. Клиент читает текущий `head`.
2. Получает его ETag.
3. Готовит новый `head`.
4. Пишет новый `head` с `If-Match: previous_etag`.
5. Если другой клиент уже обновил head — операция падает.
6. Клиент перечитывает head, применяет чужие ops, пересчитывает результат и повторяет commit.

Это ключевой механизм, без которого “идеальная надежность” невозможна на обычном object storage.

### 6.5. Append-only operation log

Даже если `head` не обновился из-за конфликта, операция не должна теряться.

Поэтому:

* Каждый op пишется как immutable object.
* Идентификатор op уникален: device_id + logical_clock + uuid.
* Создание op делается через `If-None-Match`.
* `head` — это только ускоритель и checkpoint.
* Истинная история — append-only log + snapshots.

### 6.6. Snapshots

Журнал операций будет расти. Нужны snapshots:

* snapshot дерева файлов каждые N операций или M мегабайт ops;
* snapshot содержит compacted state;
* старые ops можно оставить до retention period;
* GC удаляет только то, что точно не нужно active devices.

### 6.7. Локальная база SQLite

SQLite должна хранить:

* file_id ↔ local path;
* remote revision;
* local revision;
* dirty state;
* upload/download queue;
* pending ops;
* failed jobs;
* conflict records;
* locks;
* device state;
* last applied remote op;
* last snapshot;
* content cache index;
* diagnostics.

Важно: SQLite — локальный индекс и очередь, а не единственный источник правды. После повреждения локальной базы клиент должен уметь восстановиться из bucket metadata.

---

## 7. Конфликты: как минимизировать и как честно обрабатывать

### 7.1. Ноль конфликтов для произвольных файлов невозможен без ограничений

Для обычных бинарных файлов нельзя гарантировать, что два независимых одновременных изменения будут автоматически слиты в один “правильный” результат. Но можно гарантировать другое:

* **никогда не терять данные;**
* **никогда молча не перетирать чужую версию;**
* **делать конфликты редкими;**
* **делать конфликты понятными;**
* **для текстовых форматов пробовать auto-merge;**
* **для бинарных файлов создавать ясные sibling versions.**

### 7.2. Главные правила conflict model

1. **Last writer wins запрещен для содержимого файла.**

   * Можно использовать только для неопасных UI-атрибутов, например “последняя выбранная сортировка”.

2. **Каждая локальная правка имеет base revision.**

   * Если remote revision не изменилась — upload обычный.
   * Если remote revision изменилась — это merge/rebase.

3. **Rename/move не должны конфликтовать с edit.**

   * Это достигается через `file_id`.

4. **Delete — это tombstone, не немедленное уничтожение.**

   * Удаление хранится как операция.
   * Восстановление возможно из Trash/Version History.

5. **Conflict copy — допустимый fallback.**

   * Например: `report (conflict from MacBook, 2026-05-30).docx`.

6. **UI обязан показать причину.**

   * “Этот файл был изменен на MacBook и Windows PC до завершения синхронизации.”

### 7.3. Матрица конфликтов

| Ситуация                                                                  | Желаемое поведение                                                                                                 |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Два устройства отредактировали один текстовый файл от одной base revision | Попробовать 3-way merge. Если merge чистый — создать новую merged revision. Если нет — показать conflict resolver. |
| Два устройства отредактировали binary/Office/PDF                          | Создать sibling revisions. Пользователь выбирает: оставить обе, заменить, открыть обе версии.                      |
| Одно устройство переименовало файл, другое отредактировало                | Не конфликт. Edit применяется к тому же `file_id` уже под новым именем.                                            |
| Одно устройство удалило файл, другое отредактировало                      | Не удалять локальную правку. Создать “deleted remotely, edited locally” conflict.                                  |
| Два устройства создали файл с одинаковым именем в одной папке             | Один получает исходное имя, второй — auto-suffix conflict name.                                                    |
| Два устройства переименовали один файл по-разному                         | Детерминированный winner + UI conflict для второго имени. Содержимое не теряется.                                  |
| Case-only rename на Windows/macOS/Linux                                   | Нормализовать имена и заранее предупреждать, если имя опасно для другой ОС.                                        |
| Внешний S3-клиент поменял объект без S4 metadata                          | Импортировать как external change, пометить как untrusted/external revision.                                       |

### 7.4. Locks/leases

Чтобы конфликтов было меньше, нужен механизм locks.

**Soft lock:**

* Когда пользователь открывает файл, S4Drive пытается поставить lease.
* Lease имеет TTL.
* Клиент обновляет heartbeat.
* Другие устройства видят “файл редактируется на MacBook”.
* Пользователь может открыть read-only или создать копию.

**Hard lock:**

* Возможен только если твой custom S3 backend будет enforce’ить lock policy.
* Обычный S3 не умеет сам проверять “есть ли lock object на другом key” перед записью произвольного object key.
* Поэтому hard lock лучше делать как S4Drive server extension.

**Для LLM-агентов:**

* Не давать агентам “просто писать в папку” как единственный путь.
* Сделать локальный S4Drive Agent API:

  * получить lock;
  * прочитать current revision;
  * записать новую revision;
  * commit с expected base revision;
  * подписаться на изменения.
* Тогда агенты будут работать через тот же concurrency protocol, что и приложение.

---

## 8. Versioning и восстановление

### 8.1. Два уровня версий

**Уровень 1: S4Drive revisions**

Это главный уровень:

* каждая версия файла — revision;
* revision входит в graph;
* conflicts — sibling revisions;
* restore работает через S4 metadata.

**Уровень 2: S3 Versioning**

Это дополнительная страховка:

* accidental overwrite;
* manual recovery;
* audit;
* защита от бага клиента.

Не надо строить всю продуктовую логику только на S3 Versioning. S3 Versioning не знает про file_id, rename, конфликтные ветки, UI-логику и локальное состояние.

### 8.2. Trash вместо мгновенного удаления

Удаление в S4Drive:

1. Создает tombstone.
2. Убирает файл из видимого дерева.
3. Хранит content refs до retention period.
4. Показывает файл в Trash.
5. Позволяет восстановить.
6. Только GC после retention удаляет unreferenced blobs.

### 8.3. Защита системных метаданных

Для `.s4drive/meta/snapshots`, critical heads и audit ops желательно:

* включить S3 Versioning;
* optionally Object Lock на retention window;
* ограничить delete permissions для обычного клиента.

S3 Object Lock может защищать object versions от удаления или перезаписи на заданный срок или indefinitely, но работает с versioned buckets. ([AWS Documentation][14])

---

## 9. Desktop behavior: фон, трей, окно, выход

### 9.1. Desktop UX contract

На Windows/Linux/macOS:

* Приложение стартует при логине, если пользователь включил autostart.
* В трее всегда есть иконка.
* Левый клик по иконке:

  * открывает главное окно;
  * если окно открыто — фокусирует;
  * если идет sync — показывает статус.
* Правый клик:

  * Open S4Drive;
  * Sync now;
  * Pause sync;
  * Recent activity;
  * Settings;
  * Help/Diagnostics;
  * Exit.
* Закрытие окна на крестик:

  * не завершает приложение;
  * скрывает окно;
  * sync продолжает работать.
* “Exit” из tray:

  * останавливает sync queue безопасно;
  * завершает background core;
  * закрывает приложение полностью.

Tauri system tray официально поддерживает создание и настройку tray icon, меню и события tray. ([Tauri][2])

### 9.2. Два процесса или один?

**MVP:** один процесс Tauri + Rust Core.

Плюсы:

* проще;
* меньше IPC;
* быстрее разработка.

Минусы:

* если UI процесс упал, sync тоже упал;
* сложнее держать core без UI.

**Production:** два процесса.

1. `s4drive-agent`

   * Rust daemon;
   * sync;
   * local DB;
   * network;
   * file watcher;
   * transfer queue.

2. `s4drive-ui`

   * Tauri window;
   * tray;
   * settings;
   * communicates with agent through local IPC.

Для v1 можно начать с одного процесса, но архитектуру сразу проектировать так, будто core — отдельный сервис.

---

## 10. Platform integration roadmap

### 10.1. Windows

Фазы:

1. Обычная sync folder.
2. Explorer badges/status.
3. Context menu.
4. Cloud Files API / placeholders / files-on-demand.
5. Smart hydration/dehydration.

Cloud Files API — правильный путь для OneDrive-like интеграции: Windows API предназначен для sync providers, placeholder files и File Explorer integration. ([Microsoft Learn][6])

### 10.2. macOS

Фазы:

1. Обычная sync folder.
2. Menu bar/tray.
3. Finder integration.
4. File Provider extension.
5. Files-on-demand.
6. Spotlight/Quick Look позже.

Apple File Provider — правильный путь для доступа к документам, управляемым приложением и синхронизируемым с remote storage. ([Apple Developer][7])

### 10.3. Linux

Фазы:

1. Обычная sync folder.
2. AppIndicator/tray.
3. File manager extensions:

   * Nautilus;
   * Dolphin;
   * Thunar.
4. Optional FUSE mount.
5. Optional xdg-desktop-portal integration.

На Linux не стоит обещать одинаковый OneDrive-like UX во всех DE на первом этапе. Нужно поддержать “хорошо работает в папке” и затем добавлять интеграции.

### 10.4. Android

Фазы:

1. Обычное приложение:

   * browse;
   * upload;
   * download;
   * offline files;
   * camera upload позже.
2. Background sync через WorkManager.
3. DocumentsProvider/SAF.
4. Share sheet integration.
5. Selective offline.

Android SAF позволяет provider’ам представлять remote/local документы в стандартном picker, а WorkManager — стандартный путь для persistent background work на Android. ([Android Developers][8])

### 10.5. iOS

Фазы:

1. Обычное приложение:

   * browse;
   * preview;
   * upload/download;
   * offline.
2. Files app integration через File Provider.
3. Background tasks для ограниченной фоновой синхронизации.
4. Push/change notifications от backend.
5. Selective offline.

На iOS нельзя проектировать “вечный background sync как на desktop”. Нужно проектировать “система дает окна выполнения, приложение их эффективно использует”. Apple background tasks предназначены для обновления приложения в фоне с учетом системных ограничений энергии и времени. ([Apple Developer][15])

---

## 11. UI/UX: структура приложения

### 11.1. Основной принцип UX

S4Drive не должен показывать обычному пользователю “S3-бакет, prefix, ETag, multipart, consistency”.

Для обычного пользователя:

* “Диск”
* “Папки”
* “Файлы”
* “Синхронизировано”
* “Доступно офлайн”
* “Конфликт”
* “История версий”
* “Корзина”
* “Аккаунт”
* “Настройки”

S3-термины — только в Advanced Settings.

### 11.2. Главное окно desktop

Размер:

* среднее окно;
* не fullscreen по умолчанию;
* remember last size/position;
* responsive layout.

Структура:

**Левая навигация**

* My Drive
* Recent
* Offline
* Shared / Links — позже
* Transfers
* Conflicts
* Trash
* Activity
* Settings

**Верхняя панель**

* breadcrumbs;
* search;
* sync status;
* account avatar;
* quick upload/new folder.

**Центр**

* list/grid view;
* file icons;
* status badges;
* size;
* modified;
* version/conflict marker;
* lock marker.

**Нижняя/статусная зона**

* “Synced”
* “Syncing 12 files”
* “Paused”
* “Offline”
* “Action needed”
* “Storage full”
* “Credential expired”

### 11.3. Файловые статусы

Иконки/бейджи:

* Cloud only
* Downloading
* Uploading
* Synced
* Pinned offline
* Local only pending upload
* Conflict
* Locked by another device
* Error
* Ignored
* Deleted in remote trash

### 11.4. Контекстное меню файла

Внутри S4Drive:

* Open
* Open with…
* Show in Explorer/Finder
* Download now
* Keep offline
* Free up local space
* Rename
* Move
* Copy
* Delete
* Version history
* Resolve conflict
* Copy S4Drive link — позже
* Share — позже
* File details
* Lock / Unlock — если включен lock flow

В системном файловом менеджере:

* Windows: shell/Cloud Files integration.
* macOS: Finder/File Provider actions.
* Linux: extensions per DE.
* Android/iOS: long press + share sheet/context actions.

### 11.5. Conflict Resolver UX

Экран “Conflicts” должен быть очень простым:

* Файл.
* Где конфликт.
* Какие устройства.
* Какие времена изменения.
* Размеры версий.
* Preview, если возможно.
* Кнопки:

  * Keep both
  * Use this version
  * Use latest
  * Merge text
  * Open both
  * Restore previous

Важно: конфликт — не ошибка пользователя. Тон интерфейса должен быть спокойным: “Нужно выбрать версию”, а не “Sync failed”.

### 11.6. Onboarding

Пошаговый flow:

1. Welcome.
2. Выбор режима:

   * Connect to existing S4Drive bucket.
   * Create new S4Drive bucket.
   * Advanced S3-compatible storage.
3. Endpoint:

   * URL;
   * region;
   * access key / secret / token;
   * TLS options.
4. Bucket selection.
5. Compatibility test.
6. Local folder selection.
7. Selective sync:

   * everything;
   * selected folders;
   * online-only default.
8. Security:

   * remember credentials;
   * optional app lock;
   * optional encryption.
9. Finish:

   * show tray behavior;
   * show local folder;
   * start initial sync.

---

## 12. Security architecture

### 12.1. Credential storage

Не хранить access key/secret в plain text.

Абстракция `CredentialStore`:

* Windows: Credential Manager/DPAPI.
* macOS: Keychain.
* Linux: Secret Service/libsecret, fallback с предупреждением.
* iOS: Keychain.
* Android: Android Keystore.

### 12.2. Credentials model

Лучше не выдавать desktop/mobile полный root-доступ.

Рекомендуемая модель:

* short-lived tokens;
* refresh token;
* scoped credentials на конкретный bucket/prefix;
* read/write/delete только нужных областей;
* отдельные permissions для `.s4drive/meta`, `.s4drive/content`, `.s4drive/trash`.

### 12.3. Подписи операций

Для защиты от битого/старого клиента:

* каждый device имеет keypair;
* ops подписываются;
* bucket registry хранит public keys устройств;
* revoked devices больше не могут создавать trusted ops;
* server может enforce’ить это в custom S3/S4 API.

### 12.4. Encryption

Три уровня:

1. **TLS in transit** — обязательно.
2. **Server-side encryption** — желательно.
3. **Client-side E2EE** — отдельный режим.

E2EE сильно усложняет:

* search;
* previews;
* web links;
* thumbnails;
* server-side dedup;
* conflict resolver.

Поэтому для v1 лучше:

* server-side encryption default;
* local credential protection;
* E2EE как advanced roadmap.

### 12.5. Updates

Desktop update должен быть подписан. Tauri updater требует signature для проверки доверенного обновления и не позволяет отключить эту проверку. ([Tauri][16])

---

## 13. Performance/resource optimization

### 13.1. Целевые принципы

S4Drive должен быть “тихим”:

* почти нулевой CPU в idle;
* минимум wakeups;
* bounded memory;
* bounded concurrency;
* streaming upload/download;
* no full-file-in-memory;
* no full rescan без причины.

### 13.2. File watching

* Использовать native file events.
* Debounce.
* Batch.
* После sleep/wake делать partial rescan.
* Периодически делать low-priority reconciliation scan.
* Не доверять watcher на 100%: события могут теряться.

### 13.3. Transfer engine

Настройки:

* max concurrent uploads;
* max concurrent downloads;
* max multipart parts per file;
* global memory budget;
* bandwidth limit;
* pause on battery;
* pause on metered network;
* Wi‑Fi only на mobile;
* priority queue.

### 13.4. Large files

Для больших файлов:

* multipart upload;
* resume;
* staged upload;
* checksum validation;
* local partial state;
* avoid duplicate upload if content hash already exists;
* background upload progress.

На v1 не надо делать сложный rsync/delta для всех файлов. Лучше сделать надежный full-object/multipart sync. Delta/chunked sync можно добавить позже.

### 13.5. Selective sync и online-only

v1:

* selective folders;
* keep offline;
* free local space;
* cache limits.

v1.5/v2:

* Windows Cloud Files API;
* macOS File Provider;
* mobile provider integration;
* placeholder hydration.

---

## 14. Reliability rules: “никогда не терять данные”

Это должно быть прямо в engineering constitution проекта.

### 14.1. Золотые правила

1. Не перезаписывать локальный файл напрямую.

   * Сначала staging.
   * Checksum.
   * Atomic replace.
   * Только если base revision совпадает.

2. Не удалять content blob сразу.

   * Tombstone.
   * Retention.
   * GC later.

3. Не считать upload успешным до metadata commit.

4. Не считать metadata commit успешным до successful conditional write.

5. Не удалять local pending changes при ошибке.

6. При любом сомнении — conflict, а не overwrite.

7. Все transfer jobs persistent.

8. Все sync decisions explainable:

   * почему загрузили;
   * почему скачали;
   * почему конфликт;
   * почему удалили.

9. Любой crash должен восстанавливаться:

   * незавершенный upload можно продолжить или отменить;
   * temp files очищаются;
   * pending ops остаются;
   * local DB проходит integrity check.

### 14.2. Sync Doctor

В приложении нужен раздел диагностики:

* Last successful sync.
* Pending uploads/downloads.
* Failed operations.
* Conflicts.
* Credential status.
* Bucket compatibility status.
* Local DB health.
* Storage usage.
* Logs export.
* “Repair local index”.
* “Rebuild from remote metadata”.
* “Verify checksums”.
* “Rescan local folder”.

---

## 15. Подробный план реализации по фазам

## Фаза 0. Product Definition и техническая конституция

**Цель:** зафиксировать, что именно строится.

Решения:

* S4Drive — Drive-like app over S3, а не GUI для rclone.
* Bucket = drive.
* S4 Native Mode — default.
* Compatibility Mode — позже.
* Rust Core — обязательный.
* Tauri — desktop shell.
* Native mobile plugins — обязательны для production mobile.
* No silent overwrite.
* No immediate hard delete.
* Conflicts are preserved versions.

Deliverables:

* Product spec.
* Reliability spec.
* Sync semantics spec.
* Naming rules.
* Conflict policy.
* UX principles.
* Supported platform matrix.
* S3 compatibility matrix.

Definition of Done:

* Понятно, какие функции входят в v0.1, v0.5, v1.0, v1.5.
* Зафиксировано, какие S3 capabilities обязательны.
* Зафиксировано, что без conditional writes production sync не считается надежным.

---

## Фаза 1. S3 Compatibility Layer

**Цель:** убедиться, что твой S3 backend может быть надежной основой.

Компоненты:

* S3 adapter abstraction.
* Compatibility test suite.
* Endpoint testing UI.
* Capability detector.
* Error classifier.
* Retry policy.

Проверки:

* PUT/GET/HEAD/DELETE.
* LIST pagination.
* Prefix/delimiter.
* Range GET.
* Multipart upload.
* Abort multipart.
* Checksums.
* Conditional write `If-None-Match`.
* Conditional write `If-Match`.
* Concurrent writes.
* Strong consistency after PUT.
* Strong consistency after DELETE.
* Unicode keys.
* Very long paths.
* Case-sensitive names.
* Large files.
* Network interruption.
* 5xx retry.
* Clock skew tolerance.

Deliverables:

* “This bucket is S4Drive compatible” report.
* Compatibility levels:

  * Level 0: not supported.
  * Level 1: basic storage only.
  * Level 2: safe sync.
  * Level 3: versioned reliable sync.
  * Level 4: full S4 enhanced backend.

Definition of Done:

* Custom S3 passes Level 2 minimum.
* Level 3/4 planned for production.

---

## Фаза 2. Rust Core Skeleton

**Цель:** создать ядро без UI-зависимости.

Модули:

* Core runtime.
* Config manager.
* Local SQLite store.
* Credential abstraction.
* S3 adapter.
* File watcher abstraction.
* Transfer queue.
* Metadata engine.
* Sync scheduler.
* Diagnostics.

Требования:

* Core можно запускать отдельно от UI.
* Core имеет стабильный internal API.
* UI не знает деталей S3.
* Все операции async, cancellable, retryable.
* Все важные состояния persist’ятся.

Deliverables:

* Rust library/core.
* Local DB schema v1.
* Internal event bus.
* Basic logs.
* Basic health status.

Definition of Done:

* Core может подключиться к S3.
* Core может создать `.s4drive/`.
* Core может записать bucket descriptor.
* Core может сохранить локальную конфигурацию.
* Core может восстановиться после restart.

---

## Фаза 3. Metadata Protocol v1

**Цель:** реализовать основу дерева файлов и операций.

Реализовать:

* bucket descriptor;
* device registration;
* file_id generation;
* folder/file entries;
* content blobs;
* operation log;
* head pointer;
* snapshots;
* tombstones;
* schema migrations.

Операции v1:

* create folder;
* create file;
* upload new revision;
* rename;
* move;
* delete;
* restore;
* update metadata;
* pin offline;
* unpin/free space.

Definition of Done:

* Два клиента могут читать один metadata graph.
* Клиент может восстановить состояние из snapshot + ops.
* Повторное применение ops идемпотентно.
* Duplicate op не ломает состояние.
* Старый клиент видит schema incompatibility и не портит bucket.

---

## Фаза 4. Basic Sync MVP

**Цель:** синхронизация одной локальной папки с одним bucket на desktop.

Функции:

* Подключение аккаунта.
* Выбор local sync folder.
* Initial upload.
* Initial download.
* Incremental local changes.
* Incremental remote changes.
* Persistent upload/download queue.
* Pause/resume.
* Retry.
* Basic conflicts.
* Basic activity log.

MVP sync loop:

1. Local scan.
2. Remote metadata load.
3. Diff.
4. Upload missing blobs.
5. Commit metadata.
6. Download missing blobs.
7. Apply local changes.

Definition of Done:

* Windows ↔ macOS ↔ Linux синхронизируют файлы в обе стороны.
* Kill app during upload не приводит к потере данных.
* Disconnect network не ломает queue.
* Restart продолжает sync.
* Delete переходит в trash/tombstone.
* Rename сохраняет file_id.

---

## Фаза 5. Conflict Engine и Version History

**Цель:** сделать надежность заметной пользователю.

Функции:

* Revision graph.
* Sibling revisions.
* Conflict records.
* Conflict naming policy.
* Text 3-way merge.
* Binary conflict copy.
* Delete/edit conflict.
* Rename/rename conflict.
* Version history screen.
* Restore version.
* Keep both.
* Use selected version.
* Explain conflict reason.

Definition of Done:

* 100+ сценариев конфликтов покрыты тестами.
* Нет silent overwrite.
* Пользователь может восстановить любую конфликтную версию.
* Conflict UI понятен обычному пользователю.

---

## Фаза 6. Desktop App Shell: Tauri + Tray

**Цель:** сделать полноценное desktop-приложение.

Функции:

* Main window.
* Medium default size.
* Hide on close.
* Tray icon.
* Tray context menu.
* Open/focus from tray.
* Pause/resume sync.
* Exit.
* Autostart.
* Notifications.
* Settings.
* Account screen.
* Transfers screen.
* Conflicts screen.
* Activity screen.
* App version/update screen.

Особенности:

* UI не должен блокировать sync.
* Если окно закрыто — sync работает.
* Если пользователь нажал Exit — sync останавливается безопасно.
* При shutdown/sleep — core сохраняет состояние.

Definition of Done:

* Поведение соответствует Google Drive/Dropbox-like desktop app.
* Пользователь не может случайно закрыть sync крестиком.
* Полный выход доступен через tray.

---

## Фаза 7. UI/UX Design System

**Цель:** сделать продукт красивым, логичным и простым.

Разработать:

* дизайн-систему;
* компоненты файлового списка;
* empty states;
* error states;
* sync status language;
* icons/badges;
* onboarding;
* settings IA;
* conflict resolver;
* version history;
* activity timeline.

Принципы:

* минимум S3-терминов;
* понятные статусы;
* красивые transitions;
* доступность;
* keyboard navigation;
* dark/light mode;
* responsive desktop/mobile layouts.

Definition of Done:

* Обычный пользователь может подключить S3 bucket без чтения документации.
* Ошибки объясняются человеческим языком.
* Все destructive actions имеют undo/restore.

---

## Фаза 8. Resource Optimization

**Цель:** сделать S4Drive реально легким.

Оптимизации:

* streaming IO;
* bounded transfer queue;
* adaptive concurrency;
* idle polling backoff;
* change feed вместо частого polling, если backend поддерживает;
* no full scan на каждый запуск;
* file watcher + periodic reconciliation;
* SQLite indexes;
* thumbnail lazy loading;
* virtualized file list;
* cache eviction.

Целевые бюджеты:

* Desktop idle CPU: около нуля.
* Core memory отдельно: малый постоянный footprint.
* UI memory: зависит от webview, но без тяжелого Electron-подхода.
* No unbounded RAM on large uploads.
* No unbounded queue in memory.

Definition of Done:

* 100k файлов не ломают UI.
* 1M metadata entries проектно поддерживаемы.
* Большой файл не загружается в RAM целиком.
* Ограничения bandwidth/concurrency работают.

---

## Фаза 9. Desktop OS Integrations

**Цель:** приблизиться к Google Drive/OneDrive UX.

Windows:

* Explorer context menu.
* Status badges.
* Cloud Files API placeholders.
* Online-only files.
* Hydration/dehydration.
* Explorer “free up space”.

macOS:

* Finder integration.
* File Provider extension.
* Status badges.
* Online-only.
* Quick Look позже.

Linux:

* file manager extensions;
* AppIndicator;
* optional FUSE.

Definition of Done:

* На Windows/macOS можно пользоваться S4Drive из системного файлового менеджера.
* Online-only не выглядит как костыль.
* Контекстные действия работают вне главного окна.

---

## Фаза 10. Mobile v1

**Цель:** мобильный клиент как нормальное Drive-приложение, не просто webview.

Android:

* Browse files.
* Upload/download.
* Offline files.
* Share to S4Drive.
* Open from S4Drive.
* WorkManager sync.
* DocumentsProvider.
* Notifications.
* Battery/network constraints.

iOS:

* Browse files.
* Preview.
* Upload/download.
* Offline files.
* Share extension.
* File Provider.
* Background tasks.
* Push/change notifications.

Mobile sync policy:

* Не пытаться всегда синхронизировать все как desktop.
* По умолчанию cloud-first.
* Offline только выбранные файлы/папки.
* Upload camera/media — отдельная фича.
* Background sync best-effort.
* Важные uploads показывать как user-initiated progress.

Definition of Done:

* Пользователь может открыть файл из Files/SAF.
* Пользователь может сохранить файл в S4Drive из другого приложения.
* Offline files работают без сети.
* Background sync не убивает батарею.

---

## Фаза 11. Security, Auth, Sharing

**Цель:** сделать безопасную основу.

Функции:

* OS keychain integration.
* Short-lived credentials.
* Device registration.
* Device revocation.
* Signed metadata ops.
* Optional app lock.
* Secure logs redaction.
* TLS validation.
* Proxy settings.
* Remote wipe token/session.
* Bucket policy recommendations.

Sharing позже:

* Private links.
* Public links.
* Expiring links.
* Password-protected links.
* Per-folder permissions.
* Audit log.

Definition of Done:

* Секреты не лежат в plain text.
* Устройство можно отозвать.
* Логи можно отправить в поддержку без утечки ключей.
* Updates подписаны.

---

## Фаза 12. Observability и Diagnostics

**Цель:** чтобы проблемы решались, а не превращались в “у меня не синкается”.

Функции:

* structured logs;
* activity timeline;
* sync health;
* failed jobs;
* retry reasons;
* bucket compatibility status;
* local DB health;
* export diagnostic bundle;
* privacy scrubber;
* repair tools;
* remote metadata verifier;
* checksum verifier.

Definition of Done:

* Пользователь видит, почему файл не синхронизируется.
* Поддержка может диагностировать проблему по bundle.
* Core может восстановить локальный индекс из remote metadata.

---

## Фаза 13. QA, Chaos Testing, Beta

**Цель:** доказать надежность.

Тестовые сценарии:

* kill during upload;
* kill during metadata commit;
* network drop;
* sleep/wake;
* disk full;
* permission denied;
* file locked by editor;
* 2 devices edit same file;
* 3 devices rename/move/delete;
* delete while upload pending;
* clock skew;
* custom S3 returns 409/412/500;
* corrupt local DB;
* corrupt local file;
* missing remote blob;
* duplicated op;
* old client with new schema;
* Unicode filenames;
* reserved Windows names;
* case conflicts;
* huge directories;
* millions of small files;
* very large files;
* mobile background interruption.

Definition of Done:

* No data loss in chaos suite.
* Conflict scenarios deterministic.
* Sync recovery documented.
* Beta telemetry shows low conflict/error rates.

---

## Фаза 14. Production Release

**Цель:** стабильный v1.

v1 должен включать:

* S3 connection.
* S4 Native bucket mode.
* Desktop Windows/macOS/Linux.
* Tray/background behavior.
* Two-way sync.
* Local folder.
* Version history.
* Trash.
* Conflict resolver.
* Safe deletes.
* Persistent queue.
* Basic mobile browse/upload/download/offline.
* Signed updates.
* Diagnostics.
* Compatibility tester.

v1.5:

* Windows Cloud Files.
* macOS File Provider.
* Android DocumentsProvider.
* iOS File Provider.
* Online-only.
* Better context menu integration.

v2:

* Sharing.
* Team/multi-user permissions.
* E2EE mode.
* Delta sync/chunked files.
* Rich previews.
* Full-text search.
* Agent API.
* Web app/admin console.
* Advanced collaboration.

---

## 16. Что обязательно сделать в самом начале

Самые важные первые решения:

1. **Не строить sync на rclone.**
2. **Сразу ввести `file_id`, revision graph и operation log.**
3. **Сразу требовать conditional writes от S3 backend.**
4. **Не использовать S3 object metadata как главный источник правды.**
5. **Сразу проектировать no-silent-overwrite.**
6. **Сразу отделить Rust Core от UI.**
7. **Сразу сделать Compatibility Test Suite для твоего S3.**
8. **Сразу признать mobile ограничения и проектировать native adapters.**
9. **Сразу сделать conflict UX частью продукта, а не “потом”.**
10. **Сразу заложить `.s4drive/` protocol versioning и migrations.**

Самая правильная формула для S4Drive:

**S4Drive = Rust sync engine + S4 metadata protocol + S3 bucket storage + красивый Drive-like UI + native OS integrations.**

Не “rclone с красивой мордой”, не “Syncthing поверх S3”, а именно **новый Drive-клиент, где S3 — надежный backend storage layer**.

[1]: https://v2.tauri.app/?utm_source=chatgpt.com "Tauri 2.0 | Tauri"
[2]: https://v2.tauri.app/learn/system-tray/ "System Tray | Tauri"
[3]: https://v2.tauri.app/develop/plugins/develop-mobile/ "Mobile Plugin Development | Tauri"
[4]: https://rclone.org/docs/ "Documentation"
[5]: https://v2.tauri.app/develop/sidecar/ "Embedding External Binaries | Tauri"
[6]: https://learn.microsoft.com/en-us/windows/win32/cfapi/cloud-files-api-portal "Cloud Sync Engines - Win32 apps | Microsoft Learn"
[7]: https://developer.apple.com/documentation/fileprovider?utm_source=chatgpt.com "File Provider | Apple Developer Documentation"
[8]: https://developer.android.com/guide/topics/providers/document-provider "Open files using the Storage Access Framework  |  App data and files  |  Android Developers"
[9]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingMetadata.html "Working with object metadata - Amazon Simple Storage Service"
[10]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/copy-object.html?utm_source=chatgpt.com "Copying, moving, and renaming objects - docs.aws.amazon.com"
[11]: https://aws.amazon.com/s3/consistency/ "consistency"
[12]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html "How to prevent object overwrites with conditional writes - Amazon Simple Storage Service"
[13]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html "Uploading and copying objects using multipart upload in Amazon S3 - Amazon Simple Storage Service"
[14]: https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html "Locking objects with Object Lock - Amazon Simple Storage Service"
[15]: https://developer.apple.com/documentation/uikit/using-background-tasks-to-update-your-app?utm_source=chatgpt.com "Using background tasks to update your app - Apple Developer"
[16]: https://v2.tauri.app/plugin/updater/?utm_source=chatgpt.com "Updater - Tauri"

