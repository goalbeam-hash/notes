### Общий обзор происходящего в коде (do_push.py)

Этот скрипт — это модуль для автоматизированной синхронизации (пуша) изменений из локальной папки (WATCHED_FOLDER) в GitHub-репозиторий с использованием **только REST API GitHub**, без полноценного локального Git-репозитория (хотя pygit2 используется для анализа diff). Цель — "умная" синхронизация только изменённых файлов, с обработкой ошибок, восстановлением и force-push'ем. Скрипт ориентирован на файлы с расширениями .md и .json, игнорируя другие.

#### Основная логика работы функции do_push() (шаг за шагом):

1. **Блокировка concurrent вызовов**: Использует глобальную переменную push_lock (из config). Если уже true, пропускает вызов (как в логе: "do_push уже выполняется → пропуск"). Это предотвращает параллельные пушы, которые могли бы привести к конфликтам.
2. **Инициализация обработчика восстановления (PushRecoveryHandler)**:
    - Создаёт экземпляр класса для обработки ошибок (retries, backups, delays).
    - Параметры: temp_repo_path (FAKE_PUSH_GIT, обычно C:\pythonProject3\fake_git_temp), backup_dir (VERSIONS_DIR), delay=25 сек, max_retries=3.
    - В случае ошибки (например, subprocess errors, "reference not found", FileNotFoundError) — делает бэкап temp-папки, очищает её, паузу и retry (вызывая do_push рекурсивно через callback).
3. **Сброс temp-папки к состоянию remote main (reset_temp_to_main)**:
    - Удаляет существующую temp_repo_path (shutil.rmtree).
    - Клонирует весь репозиторий с GitHub в temp (subprocess git clone).
    - Пауза 15 сек после клона (чтобы избежать race conditions с GitHub?).
    - Это обеспечивает, что локальная temp всегда начинается с актуального состояния remote, перед наложением изменений.
4. **Синхронизация изменений (sync_changed_files)**:
    - Импортируется из copy_item.py (не показан, но предположительно сравнивает файлы по hash/modtime и копирует только изменённые/новые/удалённые из WATCHED_FOLDER в temp_repo_path).
    - Логирует before/after кол-во файлов.
    - Если нет изменений (has_changes=False), отменяет push.
    - Это "умная" часть: не копирует всё заново, только delta.
5. **Подготовка staged изменений**:
    - git add . в temp (subprocess).
    - Инициализирует pygit2.Repository для анализа.
    - Генерирует описание коммита (generate_commit_description):
        - Анализирует diff (index.diff_to_tree) для staged файлов.
        - Если head не существует (unborn repo) — fallback на перечисление staged как "Создан".
        - Парсит diff строки (DiffParser.parse_diff_lines): Игнорирует headers, берёт только +/-/значимые строки (не пустые, не только whitespace).
        - Для каждого файла: статус (Added/Modified/Deleted), путь, parsed diff (обрезает до MAX_BLOCK_LENGTH=1300 символов).
        - Сохраняет в файл comments/{commit_sha}.txt (header с timestamp, stats, blocks по файлам).
        - Поддерживает только SUPPORTED_EXTENSIONS (.md, .json).
6. **Создание tree через GitHub API (github_api_create_tree_from_folder)**:
    - Перебирает все файлы в temp (rglob).
    - Для каждого: rel_path = relative_to(temp).as_posix() (например, "folder/file.md").
    - Создаёт blob (POST /git/blobs с base64 content).
    - Собирает tree_entries: {path: rel_path, mode:100644, type:blob, sha:blob_sha}.
    - POST /git/trees с {"tree": entries}.
    - Возвращает tree_sha (если успешно).
7. **Проверка на конфликты (handle_conflict_force_push)**:
    - Получает current main SHA (github_api_get_current_head).
    - Получает server_tree_sha из commit/main.
    - Если local_tree_sha (из шага 6) == server_tree_sha — OK, продолжаем.
    - Иначе: Расхождение → создаёт новую ветку "Conflict_{timestamp}" из current main (POST /git/refs).
        - Это бэкап remote состояния перед force-push'ем.
    - Если бэкап не удался — fallback в локальный backup_dir с info.txt.
    - Возвращает 'ok' / 'force_needed' / False.
8. **Force-push через API (push_with_retry + github_api_force_push_from_tree)**:
    - Генерирует message: "AutoSync {timestamp} — изменено файлов".
    - Несколько попыток (methods: основной + повторный, с sleep 10 сек).
    - Создаёт commit (POST /git/commits: {message, tree: tree_sha, parents: [current_sha]}).
    - Обновляет ref (PATCH /git/refs/heads/main: {sha: new_commit_sha, force: true}).
    - Проверяет final HEAD.
9. **Пост-обработка**:
    - Если успех: Переименовывает comment_file на new_commit_sha.txt.
    - Запускает timer (30 сек) для постинга комментария к коммиту (POST /commits/{sha}/comments с body из файла).
    - Очищает логгеры (parser_logger, run_logger_clean).
10. **Finally**:
    - Temp-папка остаётся (не удаляется, как указано в комментарии: "FAKE_PUSH_GIT не очищается и не удаляется").
    - Снимает lock.
    - Если ошибка — recovery_handler.handle_error (с retry).
11. **Тестовый режим (if __name__ == "__main__")**: Инициализирует logger и вызывает do_push.

Это делает скрипт устойчивым: retries, backups, API-only (минимизирует локальный git), фокус на delta-синхронизации и детальных commit-описаниях.