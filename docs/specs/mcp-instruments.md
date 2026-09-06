# Расширение mcp_instruments: порт инструментов mcp-1c

Расширение `exts/mcp-instruments` (модули `Мсп_Инструменты*`, NamePrefix `Мспи_`) — провайдер client_mcp: переносит 8 инструментов
и 11 промптов Go-сервера `mcp-1c` (vendor/1c-mcp-HTTP+GO) на платформу `client_mcp` без Go-части и HTTP-обвязки,
плюс REPL-песочница BSL (`repl_create`/`repl_eval`/`repl_close`, контракт — [repl.md](repl.md); заменила дефектный
`validate_bsl_code`, см. [ADR-0004](../decisions/0004-repl-instead-of-validate-bsl-code.md)).

Исходники — Designer XML (`src/`), сборка конфигуратором по `v8project.mcp-instruments.yaml`
(source-set `mcp_instruments`, хост-конфигурация `mci_host` в `.scratch/mcp-instruments/host/src`, база `build/ib-mci`).
Контракт провайдера — [client-mcp-providers.md](client-mcp-providers.md), механика сервера — [client-mcp-internals.md](client-mcp-internals.md).

## Состав

Подсистема `Инструменты_Мсп_Провайдеры` (суффикс `_Мсп_Провайдеры`, `ВключатьВКомандныйИнтерфейс=Ложь`) содержит
5 клиентских модулей — их находит `Мсп_МетаданныеСервер.ПодсистемыПровайдеров()` и регистрирует `Мсп_ПротоколКлиент`.
Серверные модули (`…Сервер`, флаги Server+ServerCall) в состав подсистемы не входят.

| Модуль | Инструменты | Промпты |
| --- | --- | --- |
| `Мсп_ИнструментыМетаданныхКлиент` | get_metadata_tree, get_object_structure, get_form_structure, analyze_subsystems, get_configuration_info | explain_config, explain_object, 1c_metadata_navigation |
| `Мсп_ИнструментыЗапросовКлиент` | execute_query, validate_query | optimize_query, 1c_query_syntax, write_report |
| `Мсп_ИнструментыКодаКлиент` | — | review_module, write_posting, find_duplicates, 1c_development_workflow |
| `Мсп_ИнструментыЖурналаКлиент` | get_event_log | analyze_error |
| `Мсп_ИнструментыREPLКлиент` | repl_create, repl_eval, repl_close | — |

## Порт-матрица (инструмент → модуль → источник)

| Инструмент | Обработчик | Серверная функция | Источник vendor |
| --- | --- | --- | --- |
| get_metadata_tree | Мсп_ИнструментыМетаданныхКлиент | Мсп_ИнструментыМетаданныхСервер.ДеревоМетаданных | HTTPServices/MCPService/Ext/Module.bsl: МетаданныеGET; tools/metadata.go |
| get_object_structure | Мсп_ИнструментыМетаданныхКлиент | Мсп_ИнструментыМетаданныхСервер.СтруктураОбъекта | ОбъектGET; tools/object_structure.go |
| get_form_structure | Мсп_ИнструментыМетаданныхКлиент | Мсп_ИнструментыМетаданныхСервер.СтруктураФормы | ФормаGET; tools/form.go |
| analyze_subsystems | Мсп_ИнструментыМетаданныхКлиент | Мсп_ИнструментыМетаданныхСервер.ДеревоПодсистем | ПодсистемыGET; tools/analyze_subsystems.go |
| get_configuration_info | Мсп_ИнструментыМетаданныхКлиент | Мсп_ИнструментыМетаданныхСервер.ИнформацияОКонфигурации | КонфигурацияGET; tools/config.go |
| execute_query | Мсп_ИнструментыЗапросовКлиент | Мсп_ИнструментыЗапросовСервер.ВыполнитьЗапрос | ЗапросPOST; tools/query.go |
| validate_query | Мсп_ИнструментыЗапросовКлиент | Мсп_ИнструментыЗапросовСервер.ПроверитьЗапрос | ПроверкаЗапросаPOST; tools/query.go |
| get_event_log | Мсп_ИнструментыЖурналаКлиент | Мсп_ИнструментыЖурналаСервер.СобытияЖурнала | ЖурналРегистрацииPOST; tools/event_log.go |

Инструменты REPL (`repl_create`/`repl_eval`/`repl_close`) — не порт: серверная логика `Мсп_ИнструментыREPLСервер`
(включая EPF-движок на макете `Мспи_ШаблонОбработки`), контракт — [repl.md](repl.md).

Имена инструментов и промптов сохранены 1:1. Форма JSON-ответов соответствует структурам `onec/types.go`
(ключи английские, как в Go). Ошибки валидации — тексты Go через `Мсп_Сервер.ОшибкаИнструмента("INVALID_PARAMS"|"NOT_FOUND"|"ACTION_DENIED"|"ACTION_FAILED", …)`.

## Отклонения от Go-источника (сознательные)

- **get_metadata_tree**: вместо Markdown-сводки возвращает JSON `{Категория: МассивИмен, warnings?}`;
  с `filter` — одну категорию; неизвестная категория — INVALID_PARAMS `unknown metadata category: %1`.
  Имена с суффиксами `ПрисоединенныеФайлы`/`ПрисоединённыеФайлы` отфильтрованы (шум).
- **analyze_subsystems**: Go отдавал Markdown; здесь JSON-структуры:
  `orphans` → `{orphans, total, warnings?}`; `containing` → `{object, matched:[{object, subsystems:[{name, fullName, root}]}], total, warnings?}`;
  `intersections` → `{intersections:[{object, subsystems:[…]}], total, warnings?}`. Логика membership/dedupe/фильтров 1:1 с Go.
- **get_form_structure**: параметр `form_name` не перенесён (в Go работал только с `--dump`); основная форма
  выбирается серверной функцией. Состав элементов/команд/обработчиков определяется в режиме Предприятия —
  недоступные части возвращаются пустыми.
- **Промпты**: динамический Description результата (Go `fmt.Sprintf`) не переносится — `Мсп_Результаты.ТекстовыйПромпт`
  описания результата не принимает; промпты возвращают статические тексты (шаги с search_code/bsl_syntax_help/reload_dump удалены).
- **execute_query**: сверх vendor запрещены лексемы `ПОМЕСТИТЬ`/`INTO` (DML-защита, требование плана Q7).
- Аннотация `readOnlyHint` не указывается: в BSL-контракте `Мсп_Сервер.НовыйИнструмент` аннотаций нет.
- Не перенесены (вне объёма): search_code, bsl_syntax_help, reload_dump, ВерсияGET, НастройкиGET, РасширенияGET.

## Сходимость infobase_info / get_configuration_info

`get_configuration_info` порта и встроенный `infobase_info` (Мсп_ИнструментИнфоИБКлиент) пересекаются по полям
`name`/`version`: оба читают `Метаданные` основной конфигурации. `infobase_info` — первичное средство
(идентификация базы при старте), `get_configuration_info` — совместимость с контрактами mcp-1c
(дополнительно `vendor`, `platform_version`, `mode`). Расхождение полей между ними — диагностируемый дефект.

## Семантика warnings

`warnings` — деградация ответа без отказа: коллекции метаданных, упавшие при чтении (МетаданныеGET/ПодсистемыGET),
попадают в массив `{ИмяКоллекции: ОписаниеОшибки()}` и возвращаются рядом с частично собранными данными.
Ключ `warnings` вставляется только когда непуст. Клиентские обработчики сохраняют его без изменений.

## Семантика REPL

Песочница BSL: каждый `repl_eval` исполняется в транзакции с безусловным откатом — база не меняется
никогда, флагов подтверждения нет. Между вызовами сеанса помнятся значения `Состояния`. Полный контракт,
движки и ограничения — [repl.md](repl.md).

## Проверка подключения

По разделу B.8 [client-mcp-providers.md](client-mcp-providers.md): установить оба расширения (client_mcp + mcp_instruments),
запустить сервер, убедиться по форме `Мсп_УправлениеMCP` или `tools/list`/`prompts/list`: 11 инструментов
(8 перенесённых + 3 REPL), 11 промптов; переключатели включённости управляют составом. Живой прогон: `get_metadata_tree`,
`execute_query` (простой SELECT), `validate_query`, `get_event_log`, `repl_eval` (см. [repl.md](repl.md)).

Лайв-прогон 2026-09-05 (демо-конфигурация «Управляемое приложение», прямые вызовы MCP из агента; отчёт
`.scratch/live-test/report.txt`): 16 PASS, включая негативные сценарии (DML отклонён, несуществующий объект → −32601,
неизвестный filter → −32602); `validate_bsl_code` отложен из-за дефекта контекста исполнения (позже инструмент
удалён и заменён на REPL — [ADR-0004](../decisions/0004-repl-instead-of-validate-bsl-code.md)). Ограничения проверки: `prompts/list`
из агента недоступен (сервер не экспортирует ресурсы; состав промптов подтверждается кодом регистрации);
элементы/команды/обработчики форм возвращаются пустыми (см. отклонение get_form_structure);
отказ журнала без права «Администрирование» проверяется вручную сеансом без прав.

## Подключение из opencode

Клиент 1С запускается с ключом `/C"runMcp;mcpPort=9874"` (ADR-0002); endpoint
`http://127.0.0.1:9874/mcp` доступен только при запущенном клиенте. В конфиге opencode сервер объявляется
как remote MCP (например, `"client-mcp": {"type": "remote", "url": "http://127.0.0.1:9874/mcp"}`).
Включение сервера в opencode — ручное, применение — ручной перезапуск opencode: оба действия
осознанно не автоматизируются, чтобы агент не работал с базой без ведома пользователя.
