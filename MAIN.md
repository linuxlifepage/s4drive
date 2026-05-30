# S4Drive — Главный план реализации (Master Plan)

> **Дата:** 30 мая 2026
> **Сводный план на основе трёх инженерных документов:**
> - [`idea-1.md`] — глубокий технический план (61 KB) — metadata protocol, conflict engine, platform integration, 14 фаз
> - [`idea-2.md`] — архитектурный обзор (16 KB) — tray-first, vector clocks, Dropbox-виджет, mobile background
> - [`idea-3.md`] — сбалансированный план (27 KB) — 6 фаз, детальный стек с crate, упаковка/дистрибуция
> - [`описание.txt`] — изначальное ТЗ от автора проекта

---

## Содержание

- [1. Видение продукта](#1-видение-продукта)
- [2. Архитектура приложения](#2-архитектура-приложения)
- [3. Технологический стек](#3-технологический-стек)
- [4. Архитектурные решения (ADRs)](#4-архитектурные-решения-adrs)
- [5. S4 Metadata Protocol v1](#5-s4-metadata-protocol-v1)
- [6. Sync Engine: логика синхронизации](#6-sync-engine-логика-синхронизации)
- [7. Conflict Engine: конфликты и их разрешение](#7-conflict-engine-конфликты-и-их-разрешение)
- [8. Desktop поведение: фон, трей, окно, выход](#8-desktop-поведение-фон-трей-окно-выход)
- [9. UI/UX дизайн и структура приложения](#9-uiux-дизайн-и-структура-приложения)
- [10. Security: безопасность и криптография](#10-security-безопасность-и-криптография)
- [11. Resource Optimization: производительность](#11-resource-optimization-производительность)
- [12. Reliability Rules: никогда не терять данные](#12-reliability-rules-никогда-не-терять-данные)
- [13. Platform Integration Roadmap](#13-platform-integration-roadmap)
- [14. Дорожная карта: 14 фаз реализации](#14-дорожная-карта-14-фаз-реализации)
- [15. Приоритетные первые шаги](#15-приоритетные-первые-шаги)
- [16. Матрица вклада источников](#16-матрица-вклада-источников)

---

## 1. Видение продукта

**S4Drive** — кросс-платформенный (Windows/Linux/macOS/Android/iOS) GUI-клиент, работающий поверх S3-совместимого хранилища. Приложение является **альтернативой Google Drive / Dropbox / OneDrive**, где:

- **S3-бакет = диск** (основное хранилище)
- **Собственный Rust-движок** синхронизации (не rclone с GUI)
- **Фоновый режим** с системным треем на десктопе
- **Минималистичный UI** как у Dropbox-виджета (~400×600 px)
- **Максимальная надёжность** — ни одного потерянного байта

### Ключевая формула S4Drive

```
S4Drive = Rust sync engine + S4 metadata protocol + S3 bucket storage + Drive-like UI + native OS integrations
```

### Чем S4Drive НЕ является

- ❌ **Не GUI для rclone** — sync engine собственный на Rust
- ❌ **Не Syncthing поверх S3** — собственные metadata, file_id, revision graph
- ❌ **Не S3-браузер** — полноценный Drive-клиент, скрывающий сложность S3

---

## 2. Архитектура приложения

Приложение логически разделяется на изолированные слои:

```
┌─────────────────────────────────────────────────────┐
│                    S4Drive UI                         │
│  (Desktop: Tauri window + tray,                       │
│   Mobile: адаптивный UI + native file provider)       │
├─────────────────────────────────────────────────────┤
│                  App Shell                             │
│  • Управление окном • Трей • Autostart                │
│  • Notifications • Нативное контекстное меню          │
│  • Update flow                                        │
├─────────────────────────────────────────────────────┤
│                  Rust Core                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐              │
│  │ Sync     │ │Metadata  │ │Transfer  │              │
│  │ Engine   │ │Engine    │ │Engine    │              │
│  ├──────────┤ ├──────────┤ ├──────────┤              │
│  │Conflict  │ │Local     │ │Credential│              │
│  │Resolver  │ │Index     │ │Manager   │              │
│  ├──────────┤ ├──────────┤ ├──────────┤              │
│  │Background│ │Diagnostics│ │Scheduler │              │
│  └──────────┘ └──────────┘ └──────────┘              │
├─────────────────────────────────────────────────────┤
│              Local Storage Layer                       │
│  • Sync folder • SQLite DB • Staging dir              │
│  • Cache dir • Trash/recovery area                    │
├─────────────────────────────────────────────────────┤
│                S3 Adapter                              │
│  • AWS-compatible S3 API                               │
│  • Custom S3 backend                                   │
│  • Compatibility checks                                │
│  • Conditional writes • Multipart • Checksums          │
│  • HEAD • LIST • Range                                 │
├─────────────────────────────────────────────────────┤
│             S4 Metadata Protocol                       │
│  • .s4drive/ hidden prefix в бакете                    │
│  • Operation log • Snapshots • File IDs                │
│  • Device IDs • Version graph • Locks                  │
│  • Tombstones • GC                                    │
├─────────────────────────────────────────────────────┤
│              Platform Adapters                         │
│  • Windows: Cloud Files API                            │
│  • macOS/iOS: File Provider extension                  │
│  • Android: DocumentsProvider/SAF                      │
│  • Linux: sync folder → FUSE → DE extensions          │
└─────────────────────────────────────────────────────┘
```

### Два процесса или один?

**MVP:** Один процесс Tauri + Rust Core (проще, меньше IPC, быстрее разработка).

**Production:** Два процесса:
1. `s4drive-agent` — Rust daemon (sync, local DB, network, file watcher, transfer queue)
2. `s4drive-ui` — Tauri window (tray, settings, IPC с agent)

Архитектуру проектировать так, будто core — отдельный сервис, даже в MVP.

---

## 3. Технологический стек

| Компонент | Технология | Crate/Библиотека | Обоснование |
|-----------|-----------|------------------|-------------|
| **Core (Rust)** | Async runtime | `tokio` | Производительность, безопасность памяти, единая кодовая база для всех платформ |
| **S3 Client** | AWS SDK Rust | `aws-sdk-s3` (features: `rustls`) | Официальный, multipart, ETag, conditional writes, совместим с любым S3 |
| **S3 Client (альтернатива)** | Apache OpenDAL | `opendal` | Легче, унифицированный API для разных storage backends |
| **Local DB** | SQLite | `rusqlite` или `sqlx` | Транзакционная целостность, встраиваемая, кроссплатформенная |
| **File Watcher** | Кроссплатформенный | `notify` | 5 ОС, рекурсивные события, дедупликация |
| **System Tray** | Tauri API | Tauri v2 system tray | Нативная иконка, контекстное меню, клики |
| **Credentials** | Системный keychain | `keyring` | Windows Credential Manager, macOS Keychain, Linux Secret Service |
| **Logging** | Structured | `tracing` | Асинхронный, структурированные логи |
| **Crypto (хэши)** | BLAKE3 или SHA-256 | `blake3` / `sha2` | BLAKE3 быстрее для streaming, SHA-256 — стандарт |
| **HTTP Client** | Async | `reqwest` с HTTP/2 | Пул соединений, таймауты, retry |
| **Desktop UI** | Tauri v2 | Tauri 2.x | Нативный WebView + Rust-бэкенд на всех десктопах |
| **Frontend** | Solid.js / Svelte | Solid.js / Svelte + TypeScript | Лёгкие, реактивные, без тяжёлого рантайма |
| **Mobile** | Tauri v2 mobile | Tauri + native Swift/Kotlin plugins | Единый проект для Android/iOS |
| **Auto-update** | Tauri updater | `tauri-plugin-updater` | Подписанные обновления, S3-hosted manifest |
| **Notifications** | Tauri plugin | `tauri-plugin-notification` | Desktop + Mobile уведомления |

### Почему не rclone / s3cmd / другие CLI-инструменты?

1. S4Drive управляет собственными metadata, file_id, версиями, конфликтами, locks — rclone не знает эту модель
2. `rclone sync` не понимает "этот файл переименован", "это конфликтная версия", "этот файл открыт на другом устройстве"
3. На mobile внешний бинарник — плохая основа (ограничения ОС, App Store, контроль ресурсов)
4. Управление памятью и concurrency для multipart должно быть предсказуемым и наблюдаемым
5. Tauri sidecar — только для миграции/диагностики, не для главного пути синхронизации

---

## 4. Архитектурные решения (ADRs)

Эти решения приняты и не пересматриваются без нового ADR.

### ADR-001. Язык и среда исполнения

**Решение:** Rust для всей критичной логики, Tauri v2 для UI.

**Обоснование:** Rust даёт безопасность памяти, предсказуемый performance, единую кодовую базу для всех платформ (включая iOS/Android). Tauri — лёгкий desktop shell без Electron.

**Источник:** idea-1 §1.1, idea-2 §Архитектура, idea-3 §1.2

### ADR-002. Собственный sync engine

**Решение:** Собственный Rust-движок синхронизации. Rclone только как инструмент импорта/экспорта/диагностики.

**Обоснование:** Rclone не знает про file_id, revision graph, конфликтные ветки, UI-статусы и локальное состояние устройства. Мобильные ОС не терпят внешние бинарники.

**Источник:** idea-1 §1.3, idea-2 §Архитектура, idea-3 §1.2

### ADR-003. S4 Native Mode для v1

**Решение:** Основной режим — S4 Native Mode с `.s4drive/` префиксом в бакете. Compatibility Mode — позже.

**Обоснование:** Путь файла — плохой идентификатор. S3 user-defined metadata нельзя изменить после upload без copy. Нужен стабильный file_id, operation log, snapshots, tombstones.

**Источник:** idea-1 §3.2–3.3

### ADR-004. file_id — стабильный идентификатор

**Решение:** Каждый файл/папка имеет UUID file_id, не зависящий от пути.

**Обоснование:** Без file_id rename воспринимается как delete+create. Case-only rename ломается на Windows. Edit после rename теряет связь с историей.

**Источник:** idea-1 §4.2, idea-3 §1.4

### ADR-005. Conditional Writes (CAS)

**Решение:** Metadata head пишется через `If-Match` (CAS). Контент пишется через `If-None-Match`.

**Обоснование:** Без CAS невозможна надёжная синхронизация без центрального сервера. При 412/409 — rebase и повтор. Это ключевой механизм надежности.

**Источник:** idea-1 §6.4, idea-3 §1.4

### ADR-006. No silent overwrite

**Решение:** Никогда не перетирать чужие данные. Конфликт = сохранённая версия, не ошибка.

**Обоснование:** Last writer wins ведёт к потерям данных. Всегда sibling revisions или conflict copy. Delete = tombstone, не уничтожение.

**Источник:** idea-1 §7.2, §14

### ADR-007. Desktop tray-first UX

**Решение:** Приложение стартует в фоне/трее. Окно создаётся по клику на иконку. Закрытие → hide. Выход → только tray → Exit.

**Обоснование:** Экономия RAM (<50-70MB в фоне). Пользователь не может случайно закрыть синхронизацию. Поведение = Google Drive/Dropbox.

**Источник:** idea-1 §9, idea-2 §2.1, idea-3 §1.3

### ADR-008. SQLite — локальный индекс, не истина

**Решение:** SQLite хранит кэшированное состояние для быстрого старта. Истинный источник правды — `.s4drive/` в бакете.

**Обоснование:** После повреждения локальной БД клиент восстанавливается из bucket metadata. SQLite — быстрый индекс и очередь, не единственный источник.

**Источник:** idea-1 §6.7, idea-3 §1.4

### ADR-009. Operation Log (append-only)

**Решение:** Каждая операция — immutable object в `.s4drive/meta/ops/`. Head — только ускоритель и checkpoint.

**Обоснование:** Даже при конфликте CAS операция не теряется. Истинная история = append-only log + periodic snapshots. Duplicate op идемпотентен.

**Источник:** idea-1 §6.5

### ADR-010. Системный keychain для секретов

**Решение:** CredentialStore абстракция над OS keychain. Никаких plain-text ключей.

**Обоснование:** Windows DPAPI/Credential Manager, macOS Keychain, Linux Secret Service, Android/iOS Keystore.

**Источник:** idea-1 §12.1, idea-3 §2.7

---

## 5. S4 Metadata Protocol v1

### 5.1. Почему нельзя полагаться только на S3 object metadata

- S3 metadata задаётся при upload, после upload нельзя изменить без copy
- Metadata headers имеют ограничение по размеру
- S3 не знает rename как операцию — это copy+delete
- Путь S3 key — плохой идентификатор (файл может быть переименован, перемещён, иметь case-конфликт)
- S3 Versioning не знает про file_id, rename, ветки, UI-логику

**Вывод:** главный источник правды — `.s4drive/` metadata log, не S3 metadata и не key path.

### 5.2. Структура `.s4drive/` в бакете

```
.s4drive/
├── system/                          # Bucket descriptor
│   ├── descriptor.json              # schema_version, created_at, owner, capabilities
│   └── schema_migrations/           # История миграций протокола
├── devices/
│   ├── <device_id>/                 # Каждое устройство
│   │   ├── registration.json        # device_name, platform, public_key, client_version
│   │   └── capabilities.json        # Что устройство поддерживает
│   └── registry.json                # Все устройства (denormalized index)
├── meta/
│   ├── heads/
│   │   ├── current                  # Pointer к текущему snapshot (CAS через If-Match)
│   │   ├── ops_tail                 # Последняя применённая операция (checkpoint)
│   │   └── per_device/              # Head на устройство (опционально)
│   ├── ops/                          # Append-only журнал операций
│   │   └── <op_id>.json             # Каждая операция — immutable object
│   └── snapshots/
│       ├── <seq_num>/               # Периодические снапшоты состояния
│       │   ├── tree.json            # Полное дерево файлов
│       │   └── metadata.json        # Метаданные снапшота
│       └── LATEST                   # Симлинк на последний снапшот
├── content/
│   └── blobs/                        # Immutable content blobs
│       ├── <hash_prefix>/
│       │   └── <full_hash>          # Контент, хэш = ключ (content-addressable)
│       └── manifest.json            # Блоб → file_id mapping
├── locks/                            # Soft/hard leases
│   ├── <file_id>.lock               # Lock object
│   └── registry.json                # Активные locks
├── trash/                            # Tombstones и удалённые версии
│   ├── tombstones/
│   │   └── <file_id>.json           # Информация об удалении
│   ├── versions/
│   │   └── <file_id>/               # Удалённые версии контента
│   └── gc_mark.json                 # Маркер сборщика мусора
└── gc/
    ├── state.json                    # Текущее состояние GC
    ├── sweep_log.json               # Лог очистки unreferenced blobs
    └── pending_deletion/            # Помеченные к удалению blobs
```

### 5.3. Сущности протокола

#### Bucket

```json
{
  "bucket_id": "uuid",
  "schema_version": 1,
  "created_at": "ISO8601",
  "owner": "user_id",
  "capabilities": [
    "native_sync",
    "conditional_writes",
    "multipart_upload",
    "strong_consistency"
  ],
  "encryption_policy": {
    "mode": "server_side",
    "algorithm": "AES256"
  },
  "sync_policy": {
    "allow_multiple_devices": true,
    "default_conflict_policy": "fork",
    "gc_retention_days": 30
  },
  "gc_policy": {
    "orphan_blob_retention_days": 14,
    "tombstone_retention_days": 90,
    "snapshot_interval_ops": 1000
  }
}
```

#### Device

```json
{
  "device_id": "uuid",
  "device_name": "MacBook Pro Dmitry",
  "platform": "macos",
  "os_version": "15.0",
  "public_key": "base64_ed25519_pubkey",
  "last_seen": "ISO8601",
  "capabilities": {
    "cloud_files_api": false,
    "file_provider": true,
    "fuse": false,
    "background_sync": false,
    "encryption_at_rest": true
  },
  "client_version": "1.0.0"
}
```

#### FileEntry

```json
{
  "file_id": "uuid",
  "parent_id": "uuid | null",
  "name": "report.docx",
  "normalized_name": "report.docx",
  "type": "file",
  "current_revision_id": "uuid",
  "content_ref": {
    "blob_id": "uuid",
    "hash": "sha256:abc123...",
    "size": 1024000,
    "mime": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    "storage_key": ".s4drive/content/blobs/ab/cdef1234..."
  },
  "size": 1024000,
  "content_hash": "sha256:abc123...",
  "mime": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "created_at": "ISO8601",
  "updated_at": "ISO8601",
  "deleted_at": null,
  "version_history": ["rev_id_1", "rev_id_2"],
  "attributes": {
    "favorite": false,
    "shared": false,
    "tags": []
  },
  "lock_state": {
    "locked": false,
    "owner_device_id": null,
    "mode": null,
    "expires_at": null
  }
}
```

#### ContentBlob

```json
{
  "blob_id": "uuid",
  "hash": "sha256:abc123...",
  "hash_algorithm": "sha256",
  "size": 1024000,
  "checksum": "crc32c:...",
  "storage_key": ".s4drive/content/blobs/ab/cdef1234...",
  "encryption_info": {
    "encrypted": false,
    "algorithm": null,
    "key_id": null
  },
  "created_by": "device_id",
  "created_at": "ISO8601",
  "ref_count": 1
}
```

#### Operation

```json
{
  "op_id": "device_id:logical_clock:uuid",
  "device_id": "uuid",
  "actor_id": "user_id",
  "logical_clock": 42,
  "base_head": "seq_num:snapshot_hash",
  "target_file_id": "uuid",
  "op_type": "upload_new_revision",
  "preconditions": {
    "expected_etag": "\"etag_value\"",
    "expected_version_id": "version_id | null",
    "file_exists": true,
    "parent_exists": true
  },
  "effects": {
    "new_revision_id": "uuid",
    "new_content_ref": { ... },
    "new_name": null,
    "new_parent_id": null,
    "deleted": false
  },
  "timestamp": "ISO8601",
  "signature": "base64_ed25519_signature"
}
```

#### Revision

```json
{
  "revision_id": "uuid",
  "file_id": "uuid",
  "parent_revision_id": "uuid | null",
  "content_ref": { ... },
  "author_device_id": "uuid",
  "created_at": "ISO8601",
  "base_revision_id": "uuid",
  "merge_state": "clean | merged | conflicted",
  "conflict_info": {
    "type": "sibling | merge_failed | delete_edit",
    "sibling_revisions": ["uuid1", "uuid2"],
    "resolution": "manual | auto_merge | conflict_copy"
  }
}
```

#### Tombstone

```json
{
  "file_id": "uuid",
  "path_at_delete": "/Documents/report.docx",
  "name_at_delete": "report.docx",
  "deleted_by": "device_id:actor",
  "deleted_at": "ISO8601",
  "retention_until": "ISO8601",
  "content_refs": ["blob_id_1", "blob_id_2"],
  "restorable": true
}
```

#### Lease/Lock

```json
{
  "file_id": "uuid",
  "owner_device_id": "uuid",
  "actor": "user_name | agent_name",
  "mode": "soft",
  "expires_at": "ISO8601",
  "heartbeat_at": "ISO8601",
  "reason": "Editing in Word on MacBook",
  "lease_renewal_count": 5
}
```

### 5.4. Типы операций v1

| Operation | Описание | Preconditions |
|-----------|----------|---------------|
| `create_folder` | Создать папку | Родитель существует, имя не занято |
| `create_file` | Создать файл | Родитель существует, имя не занято |
| `upload_new_revision` | Загрузить новую версию | Файл существует, base revision совпадает |
| `rename` | Переименовать | Файл существует, новое имя свободно |
| `move` | Переместить | Файл существует, новый parent существует |
| `delete` | Удалить (→ tombstone) | Файл существует |
| `restore` | Восстановить из trash | Tombstone существует |
| `update_metadata` | Обновить атрибуты | Файл существует |
| `pin_offline` | Сделать доступным офлайн | Файл существует |
| `unpin` | Убрать офлайн-доступ | Файл существует |

### 5.5. Локальная база SQLite (схема)

```sql
-- Аккаунты: данные подключения к S3
CREATE TABLE accounts (
    id INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    endpoint TEXT NOT NULL,
    region TEXT NOT NULL,
    bucket TEXT NOT NULL,
    access_key_id TEXT NOT NULL,
    encrypted_secret_key BLOB,
    use_tls INTEGER DEFAULT 1,
    created_at TEXT NOT NULL,
    last_used_at TEXT
);

-- Папки синхронизации: локальная папка ↔ bucket (prefix)
CREATE TABLE sync_folders (
    id INTEGER PRIMARY KEY,
    account_id INTEGER NOT NULL,
    local_path TEXT NOT NULL,
    bucket_prefix TEXT DEFAULT '/',
    sync_enabled INTEGER DEFAULT 1,
    polling_interval_sec INTEGER DEFAULT 30,
    bandwidth_limit_kbps INTEGER DEFAULT 0,
    FOREIGN KEY (account_id) REFERENCES accounts(id)
);

-- Объекты (файлы/папки): основной индекс
CREATE TABLE objects (
    id INTEGER PRIMARY KEY,
    file_id TEXT UNIQUE NOT NULL,
    sync_folder_id INTEGER NOT NULL,
    local_path TEXT NOT NULL,
    s3_key TEXT NOT NULL,
    local_mtime TEXT,
    local_hash TEXT,
    size INTEGER DEFAULT 0,
    remote_etag TEXT,
    remote_mtime TEXT,
    state TEXT CHECK(state IN (
        'synced', 'modified_locally', 'modified_remotely',
        'pending_upload', 'pending_download', 'conflicted',
        'deleted_locally', 'deleted_remotely', 'ignored'
    )) DEFAULT 'synced',
    version_vector TEXT,     -- JSON с векторными часами
    current_revision_id TEXT,
    is_folder INTEGER DEFAULT 0,
    parent_file_id TEXT,
    lock_owner TEXT,
    lock_expires_at TEXT,
    offline_pinned INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (sync_folder_id) REFERENCES sync_folders(id)
);

-- Журнал операций (локальный)
CREATE TABLE operation_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    op_id TEXT UNIQUE NOT NULL,
    device_id TEXT NOT NULL,
    file_id TEXT,
    op_type TEXT NOT NULL,
    status TEXT CHECK(status IN (
        'pending', 'committed', 'failed', 'conflicted'
    )) DEFAULT 'pending',
    payload TEXT,           -- JSON с деталями операции
    error_message TEXT,
    created_at TEXT NOT NULL,
    committed_at TEXT,
    retry_count INTEGER DEFAULT 0
);

-- Очередь передачи (upload/download jobs)
CREATE TABLE transfer_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    direction TEXT CHECK(direction IN ('upload', 'download')) NOT NULL,
    file_id TEXT NOT NULL,
    local_path TEXT NOT NULL,
    s3_key TEXT NOT NULL,
    blob_hash TEXT,
    total_bytes INTEGER DEFAULT 0,
    transferred_bytes INTEGER DEFAULT 0,
    status TEXT CHECK(status IN (
        'queued', 'in_progress', 'paused', 'completed', 'failed'
    )) DEFAULT 'queued',
    error_message TEXT,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    priority INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

-- Конфликты
CREATE TABLE conflicts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    file_id TEXT NOT NULL,
    local_revision_id TEXT,
    remote_revision_id TEXT,
    conflict_type TEXT CHECK(conflict_type IN (
        'edit_edit', 'delete_edit', 'rename_rename',
        'create_create', 'external_change'
    )) NOT NULL,
    status TEXT CHECK(status IN (
        'open', 'resolved_keep_local', 'resolved_keep_remote',
        'resolved_keep_both', 'resolved_merged'
    )) DEFAULT 'open',
    details TEXT,           -- JSON с описанием
    created_at TEXT NOT NULL,
    resolved_at TEXT
);

-- Устройство (локальное)
CREATE TABLE device_info (
    device_id TEXT PRIMARY KEY,
    device_name TEXT NOT NULL,
    platform TEXT NOT NULL,
    logical_clock INTEGER DEFAULT 0,
    last_sync_at TEXT,
    last_snapshot_at TEXT,
    db_version INTEGER DEFAULT 1
);

-- Индексы для производительности
CREATE INDEX idx_objects_state ON objects(state);
CREATE INDEX idx_objects_file_id ON objects(file_id);
CREATE INDEX idx_objects_parent ON objects(parent_file_id);
CREATE INDEX idx_transfer_queue_status ON transfer_queue(status);
CREATE INDEX idx_conflicts_file_id ON conflicts(file_id);
CREATE INDEX idx_operation_log_status ON operation_log(status);
```

---

## 6. Sync Engine: логика синхронизации

### 6.1. Основной принцип

S4Drive синхронизирует не "файлы", а **операции над версиями файлов**.

**Не так:**
- "локальный файл новее — загрузить"
- "удалённого нет — удалить"
- "mtime больше — победил"

**А так:**
- "у файла `file_id` была revision `R1`"
- "устройство A создало revision `R2` от `R1`"
- "устройство B создало revision `R3` от `R1`"
- "это sibling revisions — нужен merge/conflict flow"
- "rename — отдельная операция над `file_id`, не новая сущность"

### 6.2. Локальный цикл изменений

```
1. File watcher (notify) ловит изменение
2. Debounce событий (ждут стабильности)
3. Core проверяет, что файл больше не пишется:
   - размер стабилен
   - mtime стабилен
   - нет временного имени приложения (например, .goutputstream-XXX)
   - файл не залочен другим приложением (если ОС позволяет проверить)
4. Быстрый fingerprint (размер + mtime + частичный хэш)
5. Если файл реально изменился → полный content hash (SHA-256 / BLAKE3)
6. Создаёт локальную pending revision в SQLite
7. Кладет upload job в persistent queue
8. Загружает content blob в S3 (.s4drive/content/blobs/) — conditional write If-None-Match
9. Проверяет checksum после загрузки
10. Делает metadata commit (пишет op в .s4drive/meta/ops/ + обновляет head через If-Match)
11. Если commit успешен → файл становится clean (state = synced)
12. Если commit неуспешен (412/409) → rebase / conflict resolution
```

### 6.3. Удалённый цикл изменений

```
1. Core получает remote изменения:
   - через change feed (если backend поддерживает WebSocket/SSE)
   - через polling head ( If-Match на head, каждые N секунд)
   - через list ops после последнего известного checkpoint
2. Загружает новые ops из .s4drive/meta/ops/
3. Проверяет подписи/валидность ops
4. Применяет к локальному metadata graph (SQLite)
5. Для нужных файлов скачивает content blobs
6. Пишет в staging-файл (не сразу в sync folder!)
7. Проверяет checksum
8. Атомарно заменяет локальный файл (rename):
   - только если локальная base revision не изменилась
   - если изменилась → conflict
9. Обновляет local SQLite state
```

### 6.4. Commit через CAS (Compare And Swap)

```rust
// Псевдокод commit
async fn commit_operation(op: Operation) -> Result<(), CommitError> {
    // 1. Читаем текущий head
    let current_head = s3_client.head_object(".s4drive/meta/heads/current").await?;
    let current_etag = current_head.etag;
    
    // 2. Пишем новую операцию (If-None-Match — создаём, только если нет дубликата)
    s3_client.put_object(
        key: format!(".s4drive/meta/ops/{}", op.op_id),
        body: serde_json::to_vec(&op),
        if_none_match: "*",  // Только если не существует
    ).await?;
    
    // 3. Обновляем head (If-Match — CAS)
    match s3_client.put_object(
        key: ".s4drive/meta/heads/current",
        body: serde_json::to_vec(&HeadPointer { ... }),
        if_match: current_etag,  // Только если никто не изменил
    ).await {
        Ok(_) => Ok(()),
        Err(S3Error::PreconditionFailed(412)) => {
            // Конфликт: другой клиент обновил head
            // Перечитываем head, применяем чужие ops, пересчитываем, повторяем
            rebase_and_retry(op).await
        }
        Err(e) => Err(e.into())
    }
}
```

### 6.5. Append-only operation log

- Каждый op пишется как immutable object
- Идентификатор op уникален: `device_id:logical_clock:uuid`
- Создание op делается через `If-None-Match: *`
- `head` — только ускоритель и checkpoint
- Истинная история — append-only log + snapshots
- Duplicate op (при повторе commit) идемпотентен: если op уже существует, S3 вернёт 412

### 6.6. Snapshots

- Snapshot дерева файлов каждые N операций (например, 1000) или M MB ops
- Snapshot содержит compacted state (только текущие файлы, без удалённых)
- Старые ops остаются до retention period (для audit и recovery)
- GC удаляет orphaned content blobs — те, на которые не ссылается ни один entry
- При старте: если есть snapshot → загрузить его + применить ops после него
- При повреждении: можно восстановить из snapshot

### 6.7. Векторные часы (Vector Clocks)

Каждый файл содержит `version_vector` — JSON-массив `{device_id: counter}`. Принцип:

- Устройство A увеличивает свой счётчик при локальном изменении
- Remote sync приносит version_vector с других устройств
- Если `A.counter > B.counter` для всех устройств — A новее
- Если ни один не доминирует — конфликт (параллельные изменения)

**Источник:** idea-2 §1.2

### 6.8. S3 Backend Contract

Чтобы S4Drive был реально надёжным, твоё S3-приложение должно пройти **S4Drive Compatibility Test Suite**.

**Обязательный уровень (Level 2: safe sync):**

1. **Strong read-after-write consistency** — после PUT/DELETE/LIST клиент видит актуальное состояние
2. **Conditional writes** — `If-None-Match` (создание если нет), `If-Match` (commit если ETag совпадает)
3. **HEAD Object** — быстро получать размер, ETag, checksum, metadata, version-id
4. **List Objects V2** — стабильная пагинация, prefix/delimiter
5. **Range GET** — для resume download, preview, streaming
6. **Multipart Upload** — для больших файлов, с retry отдельных parts
7. **Checksum support** — проверка целостности на upload/download
8. **Корректные ошибки**: 412 Precondition Failed, 409 Conflict, 404, 403, retryable 5xx

**Очень желательный уровень (Level 3+):**

1. **Versioning** — для дополнительной страховки
2. **Object Lock** — для защиты системных метаданных от случайного удаления
3. **Change feed** — WebSocket/SSE/long polling для мгновенной синхронизации
4. **Server-side encryption** — для данных at rest
5. **Short-lived credentials** — чтобы клиенты не хранили корневые ключи

---

## 7. Conflict Engine: конфликты и их разрешение

### 7.1. Философия конфликтов

- Ноль конфликтов для произвольных файлов невозможен без ограничений
- Но можно и нужно гарантировать:
  - **Никогда не терять данные**
  - **Никогда молча не перетирать чужую версию**
  - **Делать конфликты редкими** (через file_id, locks, CAS)
  - **Делать конфликты понятными** (человеческим языком)
  - **Для текстовых форматов пробовать auto-merge**
  - **Для бинарных — создавать sibling versions**

### 7.2. Главные правила conflict model

1. **Last writer wins запрещён** для содержимого файла (допустим только для UI-атрибутов)
2. **Каждая локальная правка имеет base revision** — если remote не изменилась, upload обычный; если изменилась — merge/rebase
3. **Rename/move не конфликтуют с edit** — достигается через `file_id`
4. **Delete — это tombstone, не немедленное уничтожение**
5. **Conflict copy — допустимый fallback**: `report (conflict from MacBook, 2026-05-30).docx`
6. **UI обязан показать причину**: "Этот файл был изменён на MacBook и Windows PC до завершения синхронизации."

### 7.3. Полная матрица конфликтов

| Ситуация | Желаемое поведение | Стратегия |
|----------|-------------------|-----------|
| Два устройства отредактировали текстовый файл от одной base revision | Попробовать 3-way merge. Если merge чистый — merged revision. Если нет — conflict resolver | `text_3way_merge` |
| Два устройства отредактировали binary/Office/PDF | Sibling revisions + conflict copy. Пользователь выбирает: оставить обе, заменить, открыть обе | `fork` |
| Одно устройство переименовало файл, другое отредактировало | **Не конфликт.** Edit применяется к тому же `file_id` под новым именем | `apply_to_same_file_id` |
| Одно устройство удалило файл, другое отредактировало | Не удалять локальную правку. "Deleted remotely, edited locally" conflict | `delete_edit_conflict` |
| Два устройства создали файл с одинаковым именем в одной папке | Один — исходное имя, второй — auto-suffix (deterministic по device_id) | `auto_rename_suffix` |
| Два устройства переименовали один файл по-разному | Детерминированный winner (по device_id) + UI conflict для второго имени. Содержимое не теряется | `deterministic_winner` |
| Case-only rename на Windows/macOS/Linux | Нормализовать имена. Предупреждать, если имя опасно для другой ОС (CON, PRN, nul, etc.) | `normalize_and_warn` |
| Внешний S3-клиент поменял объект без S4 metadata | Импортировать как external change, пометить как untrusted/external revision | `external_change` |
| Delete на устройстве A, delete на устройстве B | **Не конфликт.** Второй delete — no-op (идемпотентно). Данные в tombstone | `idempotent_delete` |
| Move на A, move на B (в разные папки) | Детерминированный winner. Второй move запоминается как pending | `deterministic_winner` |
| Файл залочен на A, попытка редактирования на B | Показать "файл редактируется на MacBook". Предложить read-only копию | `lock_block` |

### 7.4. Locks/Leases

**Soft lock:**
- Когда пользователь открывает файл в S4Drive-просмотрщике или редакторе, S4Drive пытается поставить lease
- Lease имеет TTL (например, 10 минут)
- Клиент обновляет heartbeat каждые 2 минуты
- Другие устройства видят "файл редактируется на MacBook"
- Пользователь может открыть read-only или создать копию

**Hard lock:**
- Возможен только если custom S3 backend enforce'ит lock policy
- Обычный S3 не умеет проверять "есть ли lock object на другом key" перед записью
- Hard lock — как extension custom S3 backend

**Для LLM-агентов (v2 roadmap):**
- Не давать агентам "просто писать в папку" как единственный путь
- Сделать Agent API:
  - получить lock
  - прочитать current revision
  - записать новую revision
  - commit с expected base revision
  - подписаться на изменения
- Агенты работают через тот же concurrency protocol, что и приложение

---

## 8. Desktop поведение: фон, трей, окно, выход

### 8.1. Desktop UX Contract

**На Windows/Linux/macOS:**

```
Приложение стартует при логине (если autostart включен)
    ↓
Иконка в трее (всегда)
    ↓
Левый клик → открыть/фокусировать окно
Правый клик → контекстное меню:
  • Open S4Drive
  • Sync Now
  • Pause / Resume Sync
  • Recent Activity
  • Settings
  • Diagnostics
  • Exit
    ↓
Закрытие окна (крестик) → hide (sync продолжается)
Exit из трея → safe shutdown
```

**Подробно:**

| Событие | Поведение |
|---------|-----------|
| Старт | Core запускается. UI НЕ создаётся до первого клика по трею (экономия RAM) |
| Левый клик по трею | Если окна нет → создать (400×600). Если есть и скрыто → показать + фокус. Если открыто → скрыть |
| Правый клик по трею | Нативное контекстное меню |
| CloseRequested (крестик) | `window.hide()`. Не завершать приложение |
| Full Exit (трей → Exit) | Graceful shutdown: дождаться активных загрузок, сохранить состояние, `std::process::exit(0)` |
| Sleep/Wake | Core сохраняет состояние при sleep. После wake — partial rescan |
| Shutdown OS | Система шлёт сигнал → graceful shutdown |

**Источник:** idea-1 §9, idea-2 §2.1

### 8.2. Tauri реализация (псевдокод)

```rust
// Перехват закрытия окна
#[tauri::command]
fn handle_close_request(app_handle: tauri::AppHandle) {
    if let Some(window) = app_handle.get_webview_window("main") {
        let _ = window.hide();
    }
}

// Настройка трея
fn setup_tray(app: &tauri::App) -> Result<(), Box<dyn std::error::Error>> {
    let tray = app.tray_by_id("s4drive-tray")?;
    tray.on_menu_event(|app, event| match event.id.as_ref() {
        "open" => show_window(app),
        "sync_now" => trigger_sync(app),
        "pause" => toggle_pause(app),
        "settings" => open_settings(app),
        "exit" => graceful_shutdown(app),
        _ => {}
    });
    Ok(())
}
```

---

## 9. UI/UX дизайн и структура приложения

### 9.1. Главный принцип

S4Drive не должен показывать обычному пользователю S3-термины. Никаких "bucket", "prefix", "ETag", "multipart", "consistency".

**Для обычного пользователя:**
- "Диск" / "Папки" / "Файлы"
- "Синхронизировано" / "Доступно офлайн"
- "Конфликт" / "История версий" / "Корзина"
- "Аккаунт" / "Настройки"

**S3-термины — только в Advanced Settings.**

### 9.2. Главное окно desktop (layout)

```
┌──────────────────────────────────────────────────┐
│ ☰ My Drive           🔍 Search...    👤 🔔 ⚙️    │ ← Top bar
├──────────┬───────────────────────────────────────┤
│          │ 📁 Documents/                         │
│ My Drive │  ├── report.docx    ✓  2.0 MB   🔒    │
│ Recent   │  ├── photo.jpg     ↻  1.2 MB          │
│ Offline  │  └── notes.txt     ✗  Conflict!       │
│          │                                        │
│ ════════ │ 📁 Projects/          ← папки жирным  │
│          │  └── s4drive/                           │
│ Transfers│     ├── MAIN.md       ✓  39 KB          │
│ Conflicts│     └── idea-1.md    ↻  61 KB          │
│ Trash    │                                        │
│          │ 📁 Photos/                             │
│ Settings │  ├── vacation.jpg    ☁  4.5 MB          │
│          │  └── family.png     ⬇  2.1 MB          │
├──────────┴───────────────────────────────────────┤
│ ✓ Synced              ● 12.3 GB / 50 GB           │ ← Status bar
│ Last sync: just now                               │
└──────────────────────────────────────────────────┘
```

### 9.3. Левая навигация

| Пункт | Описание |
|-------|----------|
| **My Drive** | Основное дерево файлов и папок |
| **Recent** | Недавно изменённые файлы (кроссплатформенно) |
| **Offline** | Файлы, доступные без интернета |
| **Transfers** | Текущие upload/download с прогрессом |
| **Conflicts** | Список конфликтов с вариантами разрешения |
| **Trash** | Удалённые файлы (с возможностью восстановления) |
| **Settings** | Настройки приложения |

### 9.4. Файловые статусы (бейджи/иконки)

| Бейдж | Статус | Описание |
|-------|--------|----------|
| ✓ | Synced | Файл синхронизирован |
| ↻ | Uploading / Downloading | Идёт передача |
| ☁ | Cloud only | Только в облаке (не скачан локально) |
| ⬇ | Pinned offline | Доступен офлайн |
| ✗ | Conflict | Конфликт версий |
| 🔒 | Locked | Редактируется на другом устройстве |
| ⚠ | Error | Ошибка синхронизации |
| ⊘ | Ignored | Игнорируется (системный файл, исключение) |
| 🗑 | Deleted | Удалён (в корзине) |

### 9.5. Контекстное меню файла

```
┌─────────────────────┐
│ Open                │
│ Open with…          │
│─────────────────────│
│ Keep offline        │
│ Free up local space │
│─────────────────────│
│ Rename              │
│ Move                │
│ Copy                │
│ Delete              │
│─────────────────────│
│ Version history     │
│ Resolve conflict    │
│ Copy link           │ (позже)
│ Share               │ (позже)
│─────────────────────│
│ File details        │
│ Lock / Unlock       │ (если lock flow включён)
└─────────────────────┘
```

### 9.6. Conflict Resolver UX

Экран "Conflicts" должен быть очень простым и понятным:

```
┌─── Конфликт ─────────────────────────────────┐
│                                               │
│  📄 report.docx                               │
│                                               │
│  Файл был изменён одновременно на:            │
│    🖥  MacBook Pro  —  сегодня, 14:32         │
│    💻  Windows PC  —  сегодня, 14:30          │
│                                               │
│  ┌─────────────┐  ┌─────────────┐            │
│  │  Версия A   │  │  Версия B   │            │
│  │  24 KB      │  │  26 KB      │            │
│  │  MacBook    │  │  Windows PC │            │
│  └─────────────┘  └─────────────┘            │
│                                               │
│  ○ Keep both (one renamed with date)          │
│  ○ Use version A (MacBook)                    │
│  ○ Use version B (Windows PC)                 │
│  ○ Merge text (for text files)                │
│                                               │
│  [Apply]  [Preview]  [Cancel]                 │
└───────────────────────────────────────────────┘
```

**Важно:** конфликт — не ошибка пользователя. Тон интерфейса — спокойный: "Нужно выбрать версию", а не "Sync failed".

### 9.7. Онбординг (пошаговый flow)

```
Шаг 1: Welcome
  ┌─────────────────────────────┐
  │   🚀 Welcome to S4Drive    │
  │                             │
  │   Your personal cloud       │
  │   storage. Private, fast,   │
  │   reliable.                 │
  │                             │
  │   [Get Started →]           │
  └─────────────────────────────┘

Шаг 2: Выбор режима
  ┌─────────────────────────────┐
  │  How do you want to         │
  │  connect?                   │
  │                             │
  │  ○ Connect to existing      │
  │    S4Drive bucket           │
  │  ○ Create new S4Drive       │
  │    bucket                   │
  │  ○ Advanced S3-compatible   │
  │    storage                  │
  │                             │
  │  [Back]  [Continue →]       │
  └─────────────────────────────┘

Шаг 3: S3 Endpoint
  ┌─────────────────────────────┐
  │  S3 Connection              │
  │                             │
  │  Endpoint:  [______________]│
  │  Region:    [______________]│
  │  Access Key:[______________]│
  │  Secret Key:[______________]│
  │                             │
  │  [Test Connection]          │
  │  ✓ Connection successful!   │
  │   → S4Drive Level 2         │
  │                             │
  │  [Back]  [Continue →]       │
  └─────────────────────────────┘

Шаг 4: Локальная папка
  ┌─────────────────────────────┐
  │  Sync Folder                │
  │                             │
  │  Local path:                │
  │  [~/S4Drive/____________]   │
  │                             │
  │  Select folders to sync:    │
  │  ☑ Everything               │
  │  ☐ Selected folders only    │
  │  ☐ Online-only by default   │
  │                             │
  │  [Back]  [Start Sync →]     │
  └─────────────────────────────┘

Шаг 5: Готово!
  ┌─────────────────────────────┐
  │  ✅ You're all set!         │
  │                             │
  │  • S4Drive runs in tray     │
  │  • Click tray icon to open  │
  │  • Close window hides it    │
  │  • Exit from tray menu      │
  │                             │
  │  📁 Open sync folder        │
  │  [Finish]                   │
  └─────────────────────────────┘
```

### 9.8. Desktop vs Mobile адаптация

| Элемент | Desktop | Mobile |
|---------|---------|--------|
| Окно | Среднее (400×600), не fullscreen | Fullscreen (естественно) |
| Трей | Системная иконка + меню | Нет трея (iOS/Android) |
| Фоновый режим | Всегда работает | WorkManager / Background Tasks |
| Навигация | Левая панель | Bottom Navigation (вкладки) |
| Контекстное меню | Правый клик | Long press + bottom sheet |
| Drag-and-drop | Да (из ОС в окно) | Только upload через share/photo picker |
| Закрытие | В трей | Стандартный app switcher |

**Источник:** idea-1 §11, idea-2 §3, idea-3 §3

---

## 10. Security: безопасность и криптография

### 10.1. Credential Storage

Уровни защиты credentials:

| Уровень | Платформа | Механизм |
|---------|-----------|----------|
| 1 (лучший) | macOS | Keychain |
| 1 (лучший) | Windows | Credential Manager / DPAPI |
| 1 (лучший) | iOS | iOS Keychain |
| 1 (лучший) | Android | Android Keystore |
| 2 (средний) | Linux | Secret Service / libsecret |
| 3 (fallback) | Linux | Encrypted config file с предупреждением |

### 10.2. Credentials Model (рекомендуемая)

- Не выдавать desktop/mobile полный root-доступ
- **Short-lived tokens** + refresh token
- **Scoped credentials** на конкретный bucket/prefix
- Read/write/delete только нужных областей
- Отдельные permissions для `.s4drive/meta`, `.s4drive/content`, `.s4drive/trash`
- Возможность remote wipe token/session

### 10.3. Подписи операций

- Каждое устройство имеет keypair (Ed25519)
- Операции подписываются приватным ключом устройства
- Bucket registry хранит public keys устройств
- Revoked devices больше не создают trusted ops
- Custom S3 может enforce'ить подписи в S4 API

### 10.4. Три уровня шифрования

| Уровень | Когда | Описание |
|---------|-------|----------|
| 1. **TLS in transit** | Обязательно | HTTPS, валидация сертификатов |
| 2. **Server-side encryption** | Желательно | S3 SSE, защита at rest |
| 3. **Client-side E2EE** | v2 roadmap | Шифрование на клиенте до upload |

**E2EE сильно усложняет** (поэтому v1 без него):
- Search / Previews / Thumbnails
- Web links
- Server-side dedup
- Conflict resolver
- 3-way merge для текста

### 10.5. Дополнительная безопасность

- **App lock** (PIN/biometrics на mobile) — опционально
- **Proxy support** — для корпоративных сетей
- **Logs redaction** — секреты не попадают в логи
- **Signed updates** — Tauri updater требует подпись
- **Bucket policy recommendations** — встроенный анализатор

**Источник:** idea-1 §12

---

## 11. Resource Optimization: производительность

### 11.1. Целевые бюджеты

| Метрика | Цель |
|---------|------|
| Desktop idle CPU | ~0% (sleep между polling) |
| Core memory (без UI) | < 30 MB |
| UI memory | Tauri WebView (не Electron!) |
| RAM при загрузке 10GB файла | Streaming — не > 64 MB буфер |
| RAM в фоне | < 50-70 MB |
| Запуск до готовности sync | < 1 секунда |

### 11.2. Оптимизации

**File Watching:**
- Использовать native file events (inotify/FSEvents/ReadDirectoryChanges)
- Debounce + batch (события группируются по времени)
- После sleep/wake — partial rescan (не полный)
- Periodic low-priority reconciliation scan (раз в час)
- Не доверять watcher на 100%: события могут теряться

**Transfer Engine:**
- Max concurrent uploads/downloads (настраивается)
- Max multipart parts per file
- Global memory budget для буферов
- Bandwidth limit (опционально)
- Pause on battery
- Pause on metered network
- Wi-Fi only на mobile
- Приоритетная очередь (малые файлы → быстрее)

**Large Files (>100 MB):**
- Multipart upload с параллельными parts
- Resume при обрыве (не начинать заново)
- Staged upload (пишем part → подтверждаем)
- Checksum validation каждой part
- Локальный partial state (что уже загружено)
- Dedup: если content hash уже существует в `.s4drive/content/blobs/` — не загружать

**Selective Sync & Online-only (v1):**
- Выборочные папки для синхронизации
- Keep offline (пин локально)
- Free local space (удалить локальную копию, оставить cloud-only)
- Cache limits (максимальный размер local cache)

**Adaptive Polling:**
- Если нет изменений → увеличиваем интервал: 30s → 60s → 120s → 300s
- При изменении → сразу back to 30s
- Если backend поддерживает change feed → polling почти не нужен

**UI Оптимизации:**
- Virtual scrolling для списка файлов (тысячи строк)
- Thumbnail lazy loading
- Ленивая загрузка иконок
- No full rescan без причины
- Tree-shaking фронтенда

**Источник:** idea-1 §13

---

## 12. Reliability Rules: никогда не терять данные

Это должен быть engineering constitution проекта.

### 12.1. Золотые правила

1. **Не перезаписывать локальный файл напрямую.** Сначала staging → checksum → atomic replace (rename). Только если base revision совпадает.

2. **Не удалять content blob сразу.** Tombstone → retention period → GC later.

3. **Не считать upload успешным до metadata commit.** Content blob uploaded ≠ файл синхронизирован. Только после успешного CAS commit.

4. **Не считать metadata commit успешным до successful conditional write.** Если `If-Match` упал с 412 — commit не состоялся.

5. **Не удалять local pending changes при ошибке.** Файл остаётся dirty, операция в очереди, retry.

6. **При любом сомнении — conflict, а не overwrite.** Лучше sibling revision, чем потеря данных.

7. **Все transfer jobs persistent.** Очередь в SQLite, переживает crash.

8. **Все sync decisions explainable.** Почему загрузили / скачали / конфликт / удалили — пользователь должен видеть причину.

9. **Любой crash должен восстанавливаться.** Незавершённый upload можно продолжить или отменить. Temp files очищаются. Pending ops остаются. Local DB проходит integrity check.

### 12.2. Versioning (два уровня)

**Уровень 1: S4Drive Revisions (главный)**
- Каждая версия файла — revision
- Revision входит в graph (parent → child → sibling)
- Conflicts = sibling revisions
- Restore работает через S4 metadata

**Уровень 2: S3 Versioning (дополнительная страховка)**
- Accidental overwrite / manual recovery / audit
- Защита от бага клиента
- Не строить продуктовую логику только на S3 Versioning — она не знает про file_id, rename, ветки

### 12.3. Trash (удаление)

1. Delete → создаёт tombstone
2. Файл убирается из видимого дерева
3. Content refs хранятся до retention period (90 дней)
4. Файл показывается в Trash
5. Restore из Trash возможен
6. GC после retention удаляет orphaned blobs

### 12.4. Защита системных метаданных

Для `.s4drive/meta/snapshots`, critical heads и audit ops:
- S3 Versioning (object versioning)
- Object Lock на retention window (опционально)
- Ограниченные delete permissions для обычного клиента

### 12.5. Crash Recovery

- Core периодически сохраняет checkpoint в SQLite (sequence number)
- После restart: проверить integrity → загрузить checkpoint → продолжить pending ops
- Temp files очищаются при старте
- Незавершённые multipart uploads abort'ятся или продолжаются

**Источник:** idea-1 §14

---

## 13. Platform Integration Roadmap

### 13.1. Windows

| Фаза | Функция | Описание |
|------|---------|----------|
| v1 | Sync folder | Обычная папка синхронизации |
| v1.5 | Explorer context menu | Правый клик → действия из Explorer |
| v1.5 | Status badges | Зелёная галочка, синие стрелки на файлах |
| v2 | Cloud Files API | Placeholder files, files-on-demand, hydration |
| v2 | Shell extensions | Глубокая интеграция с Explorer |

**Cloud Files API** — правильный путь для OneDrive-like интеграции: Windows API для sync providers с placeholder files и интеграцией с Explorer.

### 13.2. macOS

| Фаза | Функция | Описание |
|------|---------|----------|
| v1 | Sync folder | Обычная папка синхронизации |
| v1.5 | Menu bar / tray | Системная иконка |
| v1.5 | Finder integration | Статусы в Finder |
| v2 | File Provider extension | Интеграция с Files app, files-on-demand |
| v2.5 | Spotlight / Quick Look | Поиск и быстрый просмотр |

**Apple File Provider** — правильный путь для доступа к документам, управляемым приложением и синхронизируемым с remote storage.

### 13.3. Linux

| Фаза | Функция | Описание |
|------|---------|----------|
| v1 | Sync folder | Обычная папка синхронизации |
| v1.5 | AppIndicator / tray | Иконка в системном лотке |
| v2 | File manager extensions | Nautilus, Dolphin, Thunar |
| v2 | FUSE mount | Виртуальная файловая система |
| v2.5 | xdg-desktop-portal | Интеграция через порталы |

**Важно:** На Linux не обещать одинаковый OneDrive-like UX во всех DE на первом этапе. Сначала "хорошо работает в папке", затем интеграции.

### 13.4. Android

| Фаза | Функция | Описание |
|------|---------|----------|
| v1 | Browse/Upload/Download | Обычное mobile-приложение |
| v1 | Offline files | Ручной выбор для офлайн-доступа |
| v1.5 | WorkManager sync | Периодическая фоновая синхронизация |
| v1.5 | DocumentsProvider | Доступ из Files и других приложений |
| v2 | Share sheet integration | Отправить в S4Drive из любого приложения |
| v2 | Camera upload | Автоматическая загрузка фото |

**Android SAF** позволяет provider'ам представлять remote/local документы в стандартном picker. **WorkManager** — стандартный путь для background work.

### 13.5. iOS

| Фаза | Функция | Описание |
|------|---------|----------|
| v1 | Browse/Preview/Upload/Download | Обычное iOS-приложение |
| v1 | Offline files | Ручной выбор для офлайн-доступа |
| v1.5 | Files app integration | File Provider extension |
| v1.5 | Background tasks | Ограниченная фоновая синхронизация |
| v2 | Push/change notifications | Мгновенное уведомление об изменениях |

**На iOS нельзя проектировать "вечный background sync как на desktop".** Система даёт окна выполнения — приложение должно их эффективно использовать.

**Источник:** idea-1 §10

---

## 14. Дорожная карта: 14 фаз реализации

```
MVP roadmap:
Фаза 0 → Фаза 1 → Фаза 2 → Фаза 3 → Фаза 4 → Фаза 6 → Фаза 7  = v0.1 (alpha)

v1.0:
MVP + Фаза 5 + Фаза 8 + Фаза 11

v1.5:
+ Фаза 9 + Фаза 10

v2.0:
+ Фаза 12 + Фаза 13 + Фаза 14
```

---

### Фаза 0. Product Definition & Техническая конституция

**Цель:** Зафиксировать, что именно строится, на уровне инженерных принципов.

**Решения:**
- S4Drive = Drive-like app over S3, не GUI для rclone
- Bucket = drive
- S4 Native Mode — default, Compatibility Mode — позже
- Rust Core — обязательный
- Tauri — desktop shell
- Native mobile plugins для production
- No silent overwrite
- No immediate hard delete
- Conflicts = preserved versions

**Deliverables:**
- Product spec (1-страничный)
- Reliability spec (правила §12)
- Sync semantics spec
- Conflict policy (§7.3)
- UX principles (§9.1)
- Platform matrix (§13)
- S3 compatibility matrix (§6.8)
- Scope definition: v0.1, v0.5, v1.0, v1.5

**Definition of Done:**
- Понятно, какие функции входят в каждый релиз
- Зафиксированы обязательные S3 capabilities
- Зафиксировано, что без conditional writes sync не считается надёжным

**Источник:** idea-1 §Фаза 0

---

### Фаза 1. S3 Compatibility Layer

**Цель:** Убедиться, что custom S3 backend — надёжная основа для S4Drive.

**Компоненты:**
- S3 adapter abstraction (Rust trait)
- Compatibility test suite (автоматизированный)
- Capability detector (определяет Level S3)
- Error classifier (412, 409, 404, 403, 5xx)
- Retry policy (exponential backoff, jitter)

**Тесты (проверки):**

| # | Тест | Критерий |
|---|------|----------|
| 1 | PUT/GET/HEAD/DELETE объект | Все команды работают |
| 2 | LIST с pagination | ContinuationToken работает |
| 3 | Prefix/delimiter | Имитация папок |
| 4 | Range GET | Частичное скачивание |
| 5 | Multipart upload + abort | Части больших файлов |
| 6 | Abort multipart | Отмена после начала |
| 7 | Checksum на upload | Целостность |
| 8 | Conditional write `If-None-Match` | 412 если объект существует |
| 9 | Conditional write `If-Match` | 412 если ETag не совпал |
| 10 | Concurrent writes (2 клиента) | Один получает 412 |
| 11 | Consistency after PUT | Сразу видно после записи |
| 12 | Consistency after DELETE | Сразу видно после удаления |
| 13 | Unicode keys, long paths, case-sensitive | Корректная обработка |
| 14 | Large files (100 MB, 1 GB, 10 GB) | Multipart работает |
| 15 | Network interruption | retryable 5xx |
| 16 | Clock skew tolerance | Разница во времени не ломает |
| 17 | 5xx retry | Сервер возвращает 500 → retry |

**Deliverables:**
- Compatibility report: "S4Drive Level N"
- Compatibility levels:
  - Level 0: not supported
  - Level 1: basic storage only (без conditional writes)
  - Level 2: safe sync (conditional writes + consistency)
  - Level 3: versioned reliable sync (+ S3 Versioning)
  - Level 4: full S4 enhanced backend (change feed, Object Lock)

**Definition of Done:**
- Custom S3 проходит Level 2 минимум
- S4Drive отказывается подключаться к бакетам ниже Level 2 (с human-readable ошибкой)

**Источник:** idea-1 §Фаза 1

---

### Фаза 2. Rust Core Skeleton

**Цель:** Создать ядро без UI-зависимости, которое можно запускать отдельно.

**Модули:**

| Модуль | Файл | Описание | Зависимости |
|--------|------|----------|-------------|
| `core_runtime` | `src/core/runtime.rs` | Tokio runtime, lifecycle, graceful shutdown | `tokio` |
| `config` | `src/core/config.rs` | Чтение/запись TOML/YAML, миграции | `serde`, `toml` |
| `local_db` | `src/core/db/` | SQLite: schema v1, DAO, миграции | `rusqlite` или `sqlx` |
| `credential_store` | `src/core/credentials.rs` | Абстракция keychain | `keyring` |
| `s3_adapter` | `src/core/s3/` | S3-клиент с conditional writes | `aws-sdk-s3` / `opendal` |
| `file_watcher` | `src/core/watcher.rs` | File system watcher | `notify` |
| `transfer_queue` | `src/core/transfer.rs` | Persistent upload/download очередь | SQLite-based |
| `metadata_engine` | `src/core/metadata/` | file_id, revision graph, snapshots | — |
| `sync_scheduler` | `src/core/sync.rs` | Цикл синхронизации | — |
| `diagnostics` | `src/core/diagnostics.rs` | Health, logs, repair | `tracing` |

**Требования к Core:**
- Core можно запускать отдельно от UI (бинарник `s4drive-agent` со временем)
- Core имеет стабильный internal API (Tauri commands)
- UI не знает деталей S3
- Все операции async, cancellable, retryable
- Все важные состояния persist'ятся

**Definition of Done:**
- Core может подключиться к S3 (любому Level 2+)
- Core может создать `.s4drive/` структуру
- Core может записать bucket descriptor
- Core может сохранить/загрузить локальную конфигурацию
- Core восстанавливается после restart (graceful shutdown)
- Core логирует ключевые события

**Источник:** idea-1 §Фаза 2

---

### Фаза 3. Metadata Protocol v1

**Цель:** Реализовать основу дерева файлов и операций в `.s4drive/`.

**Что реализовать:**

- [ ] Bucket descriptor (`system/descriptor.json`)
- [ ] Device registration (`devices/`)
- [ ] file_id generation (UUID v7 — time-sortable)
- [ ] Folder/file entries
- [ ] Content blobs (content-addressable)
- [ ] Operation log (append-only, в `.s4drive/meta/ops/`)
- [ ] Head pointer (CAS через conditional write)
- [ ] Snapshots (периодические)
- [ ] Tombstones (удалённые файлы)
- [ ] Schema migrations (версионирование протокола)

**Definition of Done:**
- Два клиента могут читать один metadata graph
- Клиент может восстановить состояние из snapshot + ops
- Повторное применение ops идемпотентно
- Duplicate op не ломает состояние
- Старый клиент видит schema incompatibility и не портит bucket
- Миграция схемы протокола возможна без потери данных

**Источник:** idea-1 §Фаза 3, §4

---

### Фаза 4. Basic Sync MVP

**Цель:** Синхронизация одной локальной папки с одним bucket на desktop.

**Функции:**

- [ ] Подключение аккаунта (endpoint, region, bucket, credentials)
- [ ] Выбор local sync folder
- [ ] Initial upload (все локальные файлы → S3)
- [ ] Initial download (все удалённые файлы → локально)
- [ ] Incremental sync: локальные изменения (через `notify`)
- [ ] Incremental sync: удалённые изменения (polling каждые 30s)
- [ ] Persistent upload/download queue (SQLite, переживает crash)
- [ ] Pause/resume sync
- [ ] Retry с exponential backoff
- [ ] Базовый conflict: параллельное редактирование → conflict copy
- [ ] Базовый activity log (лог операций)
- [ ] Rename сохраняет file_id (не delete+create)
- [ ] Delete → tombstone/trash (не hard delete)

**MVP Sync Loop (детально):**

```rust
async fn sync_loop(core: &mut Core) -> Result<()> {
    loop {
        // 1. Сбор локальных изменений
        let local_events = core.file_watcher.drain_events().await;
        for event in local_events {
            let fingerprint = compute_fingerprint(&event.path);
            if fingerprint != core.local_db.get_fingerprint(&event.path)? {
                // Файл изменился
                let hash = compute_content_hash(&event.path).await?;
                let job = UploadJob::new(event.file_id, hash);
                core.transfer_queue.push(job).await;
            }
        }
        
        // 2. Обработка очереди upload
        while let Some(job) = core.transfer_queue.next_upload().await {
            // Upload blob
            let blob_result = core.s3_adapter.put_blob(&job).await?;
            if blob_result.is_conflict() {
                core.conflict_engine.handle(job).await;
                continue;
            }
            // Metadata commit
            let commit = core.metadata_engine.commit_op(job).await?;
            if commit.is_ok() {
                core.local_db.mark_synced(&job.file_id)?;
            }
        }
        
        // 3. Poll удалённых изменений
        let remote_ops = core.s3_adapter.poll_changes().await?;
        for op in remote_ops {
            let download = core.metadata_engine.apply_op(op).await?;
            if let Some(job) = download {
                core.transfer_queue.push(job).await;
            }
        }
        
        // 4. Обработка очереди download
        while let Some(job) = core.transfer_queue.next_download().await {
            core.s3_adapter.download_blob(&job).await?;
            core.replace_local_file_atomically(&job).await?;
            core.local_db.mark_synced(&job.file_id)?;
        }
        
        // 5. Sleep до следующего цикла
        sleep(Duration::from_secs(core.get_poll_interval())).await;
    }
}
```

**Definition of Done:**
- Windows ↔ macOS ↔ Linux синхронизируют файлы в обе стороны
- Kill приложения во время upload → нет потери данных
- Отключение сети → очередь не ломается
- Restart → продолжает sync с того же места
- Переименование сохраняет file_id (история не теряется)
- Удаление → tombstone → видно в корзине

**Источник:** idea-1 §Фаза 4, §6

---

### Фаза 5. Conflict Engine & Version History

**Цель:** Сделать надёжность заметной пользователю.

**Функции:**
- [ ] Revision graph (parent → child → sibling)
- [ ] Sibling revisions (параллельные версии)
- [ ] Conflict records в SQLite
- [ ] Conflict naming policy: `name (conflict from <Device> <Date>).ext`
- [ ] Text 3-way merge (для текстовых форматов)
- [ ] Binary conflict copy (для всего остального)
- [ ] Delete/edit conflict (не терять локальную правку)
- [ ] Rename/rename conflict (детерминированный winner)
- [ ] External change detection (S3 CLI изменил файл без S4Drive)
- [ ] Version history screen (в UI)
- [ ] Restore any version
- [ ] Keep both versions
- [ ] Use selected version
- [ ] Объяснение причины конфликта человеческим языком

**Definition of Done:**
- 100+ сценариев конфликтов покрыты тестами
- Нет silent overwrite ни в одном сценарии
- Пользователь может восстановить любую конфликтную версию
- Conflict UI понятен обычному пользователю

**Источник:** idea-1 §Фаза 5, §7

---

### Фаза 6. Desktop App Shell: Tauri + Tray

**Цель:** Полноценное desktop-приложение с поведением Google Drive/Dropbox.

**Функции:**

- [ ] Main window (средний размер, ~400-500×600 px)
- [ ] Hide on close (перехват CloseRequested → window.hide())
- [ ] Tray icon (левая кнопка → показать/скрыть)
- [ ] Tray context menu:
  - Open S4Drive
  - Sync Now
  - Pause / Resume Sync
  - Recent Activity
  - Settings
  - Diagnostics
  - Exit
- [ ] Autostart (опционально, настройка)
- [ ] Notifications (sync complete, conflict, error)
- [ ] Settings screen (sync folder, polling, bandwidth, excludes, proxy)
- [ ] Account screen (endpoint, bucket, credentials, test connection)
- [ ] Transfers screen (список upload/download с прогресс-барами)
- [ ] Conflicts screen (список конфликтов, resolver)
- [ ] Activity screen (лента изменений)
- [ ] App version / update screen
- [ ] Safe exit: остановить queue → сохранить состояние → закрыть

**Tauri особенности:**
- `window.on_window_event(|event| { if close_requested { window.hide(); } })`
- tray-first: окно не создаётся до первого клика по иконке (экономия RAM)
- `window.set_min_size(Some(Size::Logical(LogicalSize { width: 400.0, height: 500.0 })))`
- IPC: Tauri commands для всех операций к Core
- Tauri events для push-уведомлений из Core в UI (sync status, conflicts)

**Definition of Done:**
- Поведение соответствует Google Drive/Dropbox desktop app
- Пользователь не может случайно закрыть синхронизацию
- Полный выход — только через tray → Exit
- Sync продолжает работать, когда окно скрыто

**Источник:** idea-1 §Фаза 6, §9; idea-2 §2.1

---

### Фаза 7. UI/UX Design System

**Цель:** Сделать продукт красивым, логичным и простым.

**Дизайн-система:**
- [ ] Цветовая палитра (светлая / тёмная темы)
- [ ] Типографика
- [ ] Иконки файлов (по MIME-типам)
- [ ] Sync status badges (§9.4)
- [ ] Компоненты: FileList, FileCard, Breadcrumb, Search, Sidebar
- [ ] Empty states (нет файлов, нет конфликтов, нет активности)
- [ ] Error states (нет соединения, ошибка auth, bucket недоступен)
- [ ] Loading states (skeleton screens)
- [ ] Transitions и анимации

**Экраны (мобильные + desktop):**
- [ ] File Explorer (list/grid view, virtual scrolling)
- [ ] File details panel (size, dates, versions, tags)
- [ ] Transfers (прогресс upload/download)
- [ ] Conflicts (resolver с preview)
- [ ] Version history (timeline)
- [ ] Settings (группированные настройки)
- [ ] Account (credentials, device management)
- [ ] About / Updates
- [ ] Diagnostics / Sync Doctor

**Принципы UX:**
- Минимум S3-терминов в основном UI
- Понятные статусы, а не технические сообщения
- Красивые transitions (но не медленные)
- Keyboard navigation (Tab, Enter, Escape, Arrow keys)
- Dark/light mode (системный + ручной)
- Responsive (desktop → tablet → mobile)

**Definition of Done:**
- Обычный пользователь может подключить S3 bucket без документации
- Ошибки объясняются человеческим языком ("Нет соединения с сервером", а не "HTTP 502")
- Все destructive actions имеют undo/restore
- Дизайн выглядит современно (Linear/Dropbox/Notion уровень)

**Источник:** idea-1 §Фаза 7, §11; idea-2 §3; idea-3 §3

---

### Фаза 8. Resource Optimization

**Цель:** S4Drive реально лёгкий по ресурсам.

**Оптимизации:**
- [ ] Streaming IO (не весь файл в памяти)
- [ ] Bounded transfer queue (max concurrent uploads/downloads)
- [ ] Adaptive concurrency (на основе скорости сети и latency)
- [ ] Idle polling backoff (30s → 60s → 120s → 300s)
- [ ] Change feed вместо polling (если backend поддерживает)
- [ ] No full scan на каждый запуск
- [ ] File watcher + periodic reconciliation (не polling)
- [ ] SQLite индексы и WAL mode
- [ ] Thumbnail lazy loading (только видимые)
- [ ] Virtual scrolling (только видимые строки)
- [ ] Cache eviction policy (LRU, ограниченный размер)
- [ ] Bandwidth limit (опционально, глобально и per-sync)
- [ ] Pause on battery (ноутбук)
- [ ] Pause on metered network (мобильный интернет)
- [ ] Wi-Fi only (mobile)

**Definition of Done:**
- 100k файлов не ломают UI
- 1M metadata entries — проектная поддержка
- Большой файл (10 GB) — streaming, без полной загрузки в RAM
- Ограничения bandwidth/concurrency работают и тестируются
- Core memory < 30 MB в idle, < 70 MB при активной синхронизации

**Источник:** idea-1 §Фаза 8, §13

---

### Фаза 9. Desktop OS Integrations

**Цель:** Приблизиться к OneDrive/Google Drive UX в системном файловом менеджере.

**Windows:**

| Шаг | Что | Когда |
|-----|-----|-------|
| 1 | Explorer context menu (реестр) | v1.5 |
| 2 | Status badges (зелёная галочка на иконках) | v1.5 |
| 3 | **Cloud Files API** placeholders | v2 |
| 4 | Online-only files | v2 |
| 5 | Hydration/dehydration | v2 |
| 6 | "Free up space" из Explorer | v2 |

**macOS:**

| Шаг | Что | Когда |
|-----|-----|-------|
| 1 | Finder integration | v1.5 |
| 2 | **File Provider** extension | v2 |
| 3 | Status badges | v1.5 |
| 4 | Online-only | v2 |
| 5 | Quick Look | v2.5 |

**Linux:**

| Шаг | Что | Когда |
|-----|-----|-------|
| 1 | Sync folder (обычная) | v1 |
| 2 | AppIndicator / tray | v1 |
| 3 | Nautilus/Dolphin/Thunar extensions | v2 |
| 4 | Optional FUSE mount | v2 |
| 5 | xdg-desktop-portal integration | v2.5 |

**Definition of Done:**
- Windows/macOS пользователь работает с S4Drive из системного файлового менеджера
- Online-only файлы отображаются корректно (не скачиваются до открытия)
- Контекстные действия (share link, version history) доступны вне главного окна

**Источник:** idea-1 §Фаза 9, §10

---

### Фаза 10. Mobile v1

**Цель:** Мобильный клиент как нормальное приложение, не webview-обёртка.

**Общий подход:**
- Cloud-first по умолчанию (файлы не скачиваются до открытия)
- Offline — только выбранные файлы/папки
- Background sync — best-effort (ОС убивает фоновые процессы)
- Push-уведомления (FCM/APNS) для уведомления об изменениях

**Android:**

| Компонент | Реализация |
|-----------|-----------|
| Browse files | Tauri mobile + Solid.js/Svelte |
| Upload/download | Rust Core через Tauri commands |
| Offline files | SQLite index + local file cache |
| Share to S4Drive | Intent filter + Tauri plugin |
| Open from S4Drive | Content URI через Tauri plugin |
| Background sync | **WorkManager** (периодическая, с constraints) |
| Files access | **DocumentsProvider** (SAF) |
| Notifications | FCM + local notifications |
| Camera upload | Отдельная фича (не v1) |

**iOS:**

| Компонент | Реализация |
|-----------|-----------|
| Browse/Preview | Tauri mobile |
| Upload/download | Rust Core через Tauri commands |
| Offline files | SQLite + local cache |
| Share extension | Share Sheet + Tauri plugin |
| Files access | **File Provider** extension |
| Background sync | **Background Tasks** (ограниченное время) |
| Push notifications | APNS |

**Mobile sync policy:**
- Не пытаться синхронизировать всё как desktop
- Важные uploads (пользователь выбрал "upload now") — с прогрессом
- Background sync раз в 15-30 минут (WorkManager / BackgroundTasks)
- Только Wi-Fi для больших файлов (настраиваемо)
- Battery-aware (не синхронизировать при низком заряде)

**Definition of Done:**
- Пользователь открывает файл из Files/SAF
- Пользователь сохраняет файл в S4Drive из другого приложения
- Offline files работают без сети
- Background sync не убивает батарею
- Приложение не вылетает при прерывании фоновой задачи ОС

**Источник:** idea-1 §Фаза 10, idea-2 §2.2

---

### Фаза 11. Security, Auth & Device Management

**Цель:** Безопасная основа.

**Функции:**

| Компонент | Статус |
|-----------|--------|
| OS keychain integration (Windows/macOS/Linux/iOS/Android) | v1 |
| Short-lived credentials (token-based, refresh flow) | v1 |
| Device registration (keypair Ed25519, public key в bucket) | v1 |
| Device revocation (отозвать ключ устройства) | v1 |
| Signed metadata ops (подпись каждой op) | v1 |
| Optional app lock (PIN/biometrics) | v1.5 |
| TLS validation (строгая, certificate pinning опционально) | v1 |
| Proxy settings (HTTP/HTTPS/SOCKS) | v1 |
| Secure logs redaction (без ключей и токенов) | v1 |
| Signed updates (Tauri updater, S3-hosted manifest) | v1 |
| Remote wipe (отозвать все токены устройства) | v1.5 |
| Bucket policy recommendations (анализатор) | v1.5 |

**Sharing (v2 roadmap):**
- Private links / Public links / Expiring links
- Password-protected links
- Per-folder permissions
- Audit log

**Definition of Done:**
- Секреты не лежат в plain text нигде
- Устройство можно отозвать, и оно перестаёт синхронизироваться
- Логи можно отправить в поддержку без утечки ключей
- Updates подписаны и проверяются перед установкой

**Источник:** idea-1 §Фаза 11, §12

---

### Фаза 12. Observability & Diagnostics

**Цель:** Чтобы проблемы решались, а не превращались в "у меня не синкается".

**Функции:**
- [ ] Structured logs (tracing + file rotation)
- [ ] Activity timeline (кто, когда, что сделал)
- [ ] Sync health dashboard (статусы всех компонентов)
- [ ] Failed jobs with reason (не просто "error")
- [ ] Retry reasons и история
- [ ] Bucket compatibility status (живой мониторинг)
- [ ] Local DB health check (integrity check)
- [ ] Export diagnostic bundle (privacy-scrubbed, ZIP)
- [ ] Repair tools: "Rebuild local index from remote metadata"
- [ ] Checksum verifier (проверить все файлы)
- [ ] Rescan local folder (force re-scan)
- [ ] Network diagnostics (latency, throughput)

**Sync Doctor screen:**
```
┌─── Sync Doctor ──────────────────────────┐
│                                          │
│  ✅ Sync active                          │
│  ● Last successful sync: just now        │
│  ● Pending uploads: 0                    │
│  ● Pending downloads: 0                  │
│  ● Failed operations: 0                  │
│  ● Conflicts: 0                          │
│                                          │
│  🔑 Credentials: ✅ OK                   │
│  📡 S3 Connection: ✅ OK (Level 2)       │
│  💾 Local DB: ✅ Healthy                 │
│  📁 Sync folder: ✅ Accessible           │
│  🗄 Storage: 12.3 GB / 50 GB             │
│                                          │
│  [Export diagnostics]  [Repair index]    │
│  [Verify checksums]    [Rescan folder]   │
└──────────────────────────────────────────┘
```

**Definition of Done:**
- Пользователь видит, почему файл не синхронизируется
- Поддержка может диагностировать проблему по diagnostic bundle
- Core может восстановить локальный индекс из remote metadata

**Источник:** idea-1 §Фаза 12, §14.2

---

### Фаза 13. QA, Chaos Testing & Beta

**Цель:** Доказать надёжность через стресс-тесты.

**Категории тестов:**

**Reliability (нет потери данных):**

| Сценарий | Описание |
|----------|----------|
| Kill during upload | Принудительное завершение посреди multipart |
| Kill during metadata commit | После upload blob, до commit op |
| Network drop mid-transfer | Разрыв соединения |
| Sleep/wake | Suspend/resume системы |
| Disk full | Нет места для staging |
| Permission denied | Файл заблокирован редактором |
| File locked by editor | Word/Excel держит файл открытым |
| 2 devices edit same file | Параллельное редактирование |
| 3 devices rename/move/delete | Одновременные разные операции |
| Delete while upload pending | Пользователь удалил файл до завершения upload |
| Clock skew | Разница в часах между устройствами |
| Custom S3 errors | Бэкенд возвращает 409/412/500 |

**Corruption recovery:**

| Сценарий | Описание |
|----------|----------|
| Corrupt local DB | SQLite повреждён → восстановление из bucket |
| Corrupt local file | Файл на диске битый → checksum error → re-download |
| Missing remote blob | Блоб удалён из S3 → error recovery |
| Duplicated op | Та же операция применена дважды → идемпотентность |
| Old client + new schema | Старая версия видит новый формат → incompatibility guard |

**Platform-specific:**

| Сценарий | Описание |
|----------|----------|
| Unicode filenames | Японские, арабские, кириллица, эмодзи |
| Reserved Windows names | CON, PRN, NUL, COM1 |
| Case conflicts | MiXeDcAsE на case-insensitive OS |
| Very long paths | > 255 символов |
| Huge directories | 100k файлов в одной папке |
| Millions of small files | 1M файлов по 1 KB |
| Very large files | 10 GB+ |
| Symbolic links | Symlinks, hardlinks, junctions |
| Hidden files | .DS_Store, Thumbs.db, .hidden |

**Инструменты:**
- LocalStack / MinIO — локальный S3 для тестов
- Rust unit tests (cargo test) — модульные тесты
- Integration tests — симуляция нескольких устройств
- Chaos Monkey — случайные kill/network/disk сбои
- Property-based testing (proptest) — random sequences of operations
- GitHub Actions — CI/CD для всех платформ

**Definition of Done:**
- No data loss в chaos suite (ни одного потерянного байта)
- Conflict scenarios детерминированы (предсказуемый результат)
- Sync recovery документирован (runbook)
- Beta telemetry показывает < 1% conflict/error rate

**Источник:** idea-1 §Фаза 13, idea-3 §4

---

### Фаза 14. Production Release & Roadmap

**Цель:** Стабильный v1.

**v1.0 включает:**
- S3 connection (Level 2+)
- S4 Native bucket mode
- Desktop: Windows + macOS + Linux
- Tray/background (close→hide, exit from tray)
- Two-way sync (одна папка ↔ один bucket)
- Local sync folder
- Version history (revision graph + restore)
- Trash (tombstone + restore)
- Conflict resolver (sibling versions + conflict copies)
- Safe deletes (tombstone, not hard delete)
- Persistent queue (переживает crash)
- Basic mobile: browse/upload/download/offline
- Signed updates (Tauri updater)
- Diagnostics page (Sync Doctor)
- Compatibility tester (onboarding test)

**Упаковка:**

| Платформа | Формат | Подпись |
|-----------|--------|---------|
| Windows | .msi + .exe installer | Code signing cert |
| macOS | .dmg | Apple notarization |
| Linux | AppImage + .deb + Flatpak | GPG sign |
| Android | AAB | Google Play sign |
| iOS | IPA | App Store |

**v1.5:**
- Windows Cloud Files API (placeholder files)
- macOS File Provider extension
- Android DocumentsProvider
- iOS File Provider extension
- Online-only files (cloud-only, hydrate on demand)
- Explorer/Finder deeper integration (status badges, context menu)
- WorkManager background sync (Android)
- Background tasks (iOS)

**v2.0:**
- Sharing (links, permissions, expiry)
- Team/multi-user support
- Compatibility Mode (файлы как обычные S3-ключи)
- Client-side E2EE mode
- Delta sync / chunked files (только изменённые части)
- Rich previews (images, PDF, video, audio)
- Full-text search
- Agent API (для LLM-агентов — lock + CAS commit)
- Web app / admin console
- Advanced collaboration (real-time text editing)
- FUSE mount (Linux)
- Camera auto-upload (mobile)

**Definition of Done:**
- Все платформы: Windows, macOS, Linux, Android, iOS
- Все три режима: Native, Compatibility (v2), E2EE (v2)
- Sync надёжен: протестирован на 100+ сценариях, в т.ч. chaos
- Производительность: < 70 MB RAM, ~0% CPU idle
- UX: любой пользователь подключается без документации

**Источник:** idea-1 §Фаза 14, idea-3 §5–6

---

## 15. Приоритетные первые шаги

*Что делать прямо сейчас, не дожидаясь всего плана:*

### Шаг 1. Инициализация проекта

```bash
# Tauri + Solid.js (рекомендую Solid — легче Svelte по сборке)
npm create tauri-app@latest s4drive -- --template solid-ts
cd s4drive

# Rust dependencies
cargo add tokio --features full
cargo add serde serde_json --features serde/derive
cargo add rusqlite --features bundled
cargo add keyring
cargo add aws-sdk-s3 --features rustls
cargo add notify
cargo add tracing tracing-subscriber
cargo add blake3
```

### Шаг 2. S3 Compatibility Test Suite

Написать набор тестов, проверяющих, что custom S3 backend проходит Level 2. **Это фундамент — без conditional writes ничто не работает.**

```rust
#[test]
fn test_conditional_write_if_match() {
    // PUT object с If-Match
    // Обновить и проверить If-Match с новым ETag
    // Должно вернуть 412 для старого ETag
}

#[test]
fn test_conditional_write_if_none_match() {
    // PUT object с If-None-Match: *
    // Повторить — должно вернуть 412
}

#[test]
fn test_strong_consistency_after_put() {
    // PUT object
    // GET object — должно быть доступно сразу
}
```

### Шаг 3. SQLite schema v1

Создать и зафиксировать схему локальной БД (см. §5.5). Написать миграции с versioning.

### Шаг 4. `.s4drive/` protocol v1

```rust
// Псевдокод структуры протокола
struct S4Bucket {
    descriptor: BucketDescriptor,
    devices: HashMap<DeviceId, Device>,
    meta: MetadataStore,
    content: ContentStore,
}

struct MetadataStore {
    heads: HeadsPointer,       // CAS через If-Match
    ops: OperationLog,         // Append-only
    snapshots: SnapshotStore,  // Periodic
}

// Реализовать: создание, чтение head, запись op, создание snapshot
```

### Шаг 5. MVP Sync Loop

Одна локальная папка → один bucket:

```rust
// Минимальный цикл синхронизации
loop {
    // 1. Watch локальные изменения
    let changes = file_watcher.events();
    
    // 2. Upload изменения
    for file in changes.uploads {
        let hash = hash_file(&file);
        let blob_key = upload_blob(&s3, &file).await?;
        commit_operation(&s3, CreateRevision { file_id, hash, blob_key }).await?;
    }
    
    // 3. Poll удалённые изменения
    let remote = poll_remote_changes(&s3).await?;
    
    // 4. Download изменения
    for file in remote.downloads {
        download_blob(&s3, &file).await?;
        atomic_replace(&file).await?;
    }
    
    sleep(30_000).await;
}
```

---

## 16. Матрица вклада источников

| Тема / Раздел | idea-1.md | idea-2.md | idea-3.md |
|---------------|-----------|-----------|-----------|
| **metadata protocol (.s4drive/)** | ★★★ §3–4 | — | — |
| **file_id / revision graph** | ★★★ §4, §7.2 | — | — |
| **conflict matrix (7+ типов)** | ★★★ §7.3 | — | — |
| **locks/leases** | ★★★ §7.4 | — | — |
| **CAS (conditional write)** | ★★★ §6.4 | — | ★ §1.4 |
| **no silent overwrite rules** | ★★★ §14 | — | — |
| **platform integration roadmap** | ★★★ §10 | — | ★ §2.7 |
| **14 фаз с DoD** | ★★★ §15 | — | — |
| **sync engine детали** | ★★★ §6 | — | ★ §2.5 |
| **S3 compatibility contract** | ★★★ §5 | — | — |
| **snapshots + operation log** | ★★★ §6.5–6.6 | — | — |
| **security architecture** | ★★★ §12 | — | ★ §2.7 |
| **UI/UX структура окна** | ★★ §11 | ★ §3 | ★ §3 |
| **desktop tray behavior** | ★★ §9 | ★★ §2.1 | ★ §1.3 |
| **conflict resolver UX** | ★★ §11.5 | — | — |
| **onboarding flow** | ★★ §11.6 | — | — |
| **diagnostics / sync doctor** | ★★ §14.2 | — | — |
| **chaos testing scenarios** | ★★ §13 | — | ★ §4 |
| **tray-first / lazy UI** | — | ★★ §2.1 | — |
| **vector clocks** | — | ★★ §1.2 | — |
| **Dropbox-стиль окна (400-500px)** | — | ★★ §3.1 | — |
| **mobile: push notifications** | — | ★★ §2.2 | — |
| **conflict copy naming** | — | ★★ §1.2 | — |
| **full tech stack (crates)** | — | — | ★★ §1.2 |
| **SQLite schema (accounts/objects/log)** | — | — | ★★ §1.4 |
| **packaging/distribution** | — | — | ★★ §5 |
| **CI/CD / testing framework** | — | — | ★ §4 |
| **SSE/server-side encryption** | ★ §12.4 | — | ★ §2.7 |
| **multipart upload детали** | ★ §13.4 | — | ★ §4.1 |
| **resource budgets** | ★ §13.1 | — | — |

**Легенда:**
- ★★★ = основной вклад — без этого источника раздел был бы пустым
- ★★ = значительный вклад — дополняет и улучшает
- ★ = упоминание — одна из идей подтверждает/дополняет

---

## Приложение: Ссылки

- [Tauri v2](https://v2.tauri.app/)
- [Tauri System Tray](https://v2.tauri.app/learn/system-tray/)
- [Tauri Mobile Plugins](https://v2.tauri.app/develop/plugins/develop-mobile/)
- [Tauri Sidecar](https://v2.tauri.app/develop/sidecar/)
- [Tauri Updater](https://v2.tauri.app/plugin/updater/)
- [AWS S3 Conditional Writes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/conditional-writes.html)
- [AWS S3 Consistency](https://aws.amazon.com/s3/consistency/)
- [AWS S3 Multipart Upload](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Windows Cloud Files API](https://learn.microsoft.com/en-us/windows/win32/cfapi/cloud-files-api-portal)
- [Apple File Provider](https://developer.apple.com/documentation/fileprovider)
- [Android Storage Access Framework](https://developer.android.com/guide/topics/providers/document-provider)
- [notify crate](https://crates.io/crates/notify)
- [keyring crate](https://crates.io/crates/keyring)
- [opendal crate](https://crates.io/crates/opendal)
- [aws-sdk-s3 crate](https://crates.io/crates/aws-sdk-s3)
