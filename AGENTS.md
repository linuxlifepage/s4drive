# AGENTS.md — S4Drive Project Guide

**S4Drive** — кросс-платформенный (Windows/Linux/macOS/Android/iOS) GUI-клиент поверх S3. Rust Core + Tauri UI. Альтернатива Google Drive.

## Ключевые решения (не менять без ADR)
- Rust Core — вся синхронизация на Rust, не rclone
- S4 Native Mode — `.s4drive/` metadata protocol, не Compatibility Mode
- file_id (UUID v7) — стабильный идентификатор, не S3 key path
- Conditional writes (If-Match/If-None-Match) — основа CAS commit
- Desktop tray-first: close→hide, exit только из tray menu

## Структура проекта
```
s4drive/
├── src-core/          # Rust библиотека (sync engine)
│   ├── src/core/      # Lifecycle, runtime
│   ├── src/s3/        # S3 adapter + compatibility test suite
│   ├── src/db/        # SQLite local index + transfer queue
│   ├── src/metadata/  # FileEntry, Revision, Operation, ContentBlob
│   ├── src/sync/      # Sync engine (two-way)
│   ├── src/watcher/   # File system watcher (notify)
│   ├── src/transfer/  # Persistent upload/download queue
│   ├── src/config/    # Config (TOML)
│   ├── src/diagnostics/ # Health, logs, repair
│   └── src/error/     # CoreError enum
├── src-cli/           # CLI бинарник (compatibility check)
├── src-tauri/         # Tauri app (Phase 6+)
├── docs/spec/         # Spec-документация
└── MAIN.md            # Главный план реализации
```

## Code Quality — обязательные проверки

### Rust (всегда перед commit)
```bash
cargo fmt --check      # Проверка форматирования
cargo clippy -- -D warnings  # Линтер + ошибки
cargo test             # Все тесты (unit + integration)
cargo check            # Компиляция без ошибок
```

### Rust Code Style — жёсткие правила
- **NO `.unwrap()` в production-коде** — только в тестах (`#[cfg(test)] mod`).
  - В production: `?` (Try), `.context()/with_context(|| ...)`, `.unwrap_or_else()`, `.expect("message")` с человеко-читаемым сообщением.
  - Единственное исключение: `.expect("infallible: ...")` на гарантированно-успешных операциях (типа взятие Mutex lock) — с комментарием почему.
- **NO `unsafe`** — zero unsafe code.

### Frontend (Solid.js / Svelte, Phase 6+)
```bash
npm run lint           # ESLint + TypeScript strict
npm run format:check   # Prettier
npm run typecheck      # tsc --noEmit
npm run test           # Vitest / Playwright
```

### Pre-commit hook (Phase 7+)
```json
{
  "hooks": {
    "pre-commit": "cargo fmt --check && cargo clippy && cargo test"
  }
}
```

## Тестирование — обязательное на каждую фазу

### Unit tests
- Каждый Rust модуль: `#[cfg(test)] mod tests { ... }`
- Покрытие: все публичные функции, граничные случаи, error paths
- Isolated: не требуют S3, сети, внешних зависимостей

### Smoke tests
- Основной сценарий фазы (happy path)
- Запускается на реальном или mock-бекенде (MinIO для S3)
- Пример: `cargo test --test smoke_phase_1` — проверка S3 connection + conditional write

### E2E tests
- Полный сценарий: init → sync → conflict → resolve
- MinIO + несколько процессов-клиентов
- ```bash
  cargo test --test e2e_basic_sync
  cargo test --test e2e_concurrent_conflict
  cargo test --test e2e_crash_recovery
  ```
- Каждый тест чистит за собой (cleanup)

### Chaos testing (Phase 13)
- kill during upload, network drop, sleep/wake, corrupt DB
- Property-based testing для random sequences of operations
- ```bash
  cargo test --test chaos -- --ignored
  ```

## Важные команды по фазам

```bash
# Текущая: Фаза 2 — Rust Core Skeleton ✅
# S3 adapter, Config, DB, Metadata types, Credential store,
# File watcher (notify), Transfer queue (SQLite), Sync engine (tokio loop),
# Diagnostics, Graceful shutdown, Bucket initialization (.s4drive/)

# Фаза 3 — Metadata Protocol v1
## CLI: инициализация бакета
cargo run -p s4drive-cli -- init-bucket \
  --endpoint http://127.0.0.1:9000 \
  --bucket s4drive-test \
  --access-key minioadmin --secret minioadmin

## Тесты метадаты
cargo run -p s4drive-cli -- init-bucket \
  --endpoint ... --bucket ...

# Фаза 4 — Basic sync
cargo test -p s4drive-core sync
minio server /tmp/s4drive-test &  # mock S3
cargo test --test e2e_basic_sync

# CLI help
cargo run -p s4drive-cli -- --help
```

## Документация
- [MAIN.md](MAIN.md) — главный план (2255 строк)
- [docs/spec/product-spec.md](docs/spec/product-spec.md)
- [docs/spec/reliability-spec.md](docs/spec/reliability-spec.md)
- [docs/spec/sync-semantics.md](docs/spec/sync-semantics.md)
- [docs/spec/conflict-policy.md](docs/spec/conflict-policy.md)
- [docs/spec/tech-constitution.md](docs/spec/tech-constitution.md)

## Git workflow
- Частые коммиты (1 на фазу/подфазу)
- `git add -A && git commit -m "Phase N: ..."` + push
- Branch: `main` (release), feature branches для крупных фич
- Коммиты подписанные (в идеале)
