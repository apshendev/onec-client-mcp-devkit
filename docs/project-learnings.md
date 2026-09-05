# Project Learnings

Operating-rule corrections on agent behaviour. Subject-matter rules belong in the spec for the feature they cover. Append a concrete one-line rule **in the same turn** the user corrects you ("Always X for Y", not "be careful with Y"). Tighten an existing line if one already covers it; remove lines when the underlying issue goes away.

<!-- append one-line rules below as they happen -->
- В ChildObjects Configuration.xml (Designer-XML) перечисляй объекты только именем без uuid-атрибута; uuid с атрибутом на элементе ChildObjects зацикливает XML-импорт платформы (бесконтрольный рост памяти до падения и в ibcmd, и в DESIGNER).
- В v8project.mcp-instruments.yaml держи builder: IBCMD; полный 1cv8 DESIGNER для локальной загрузки расширений не запускать (патологический рост памяти), .cfe выгружать только в CI.
- Перед каждым повтором сборки после таймаута/падения удаляй stale-лок v8-runner: build/.v8-runner.workspace.lock и .lock.json (после проверки, что pid из lock.json мёртв).
- Сборку/импорт контура mcp_instruments запускай только по прямой команде пользователя.
2026-09-05: Для генерации XML-исходников 1С (Configuration.xml и т.п.) использовать инструменты unica (cfe_init/cf_init с параметром cwd, outputPath игнорируется), а не писать файлы вручную.
- Любые команды по базам devkit (в т.ч. build/ib-live) запускать с явным таймаутом не более 2 минут; дольше - считать блокером, останавливаться и отчитываться, не запускать без таймаута.
2026-09-05: build/ib-live — авторизация «Администратор»/пустой пароль: ibcmd всегда с `--user=Администратор --password=`; тонкий клиент — только `/N Администратор` (ключа /User у 1cv8c нет); ibcmd резолвит относительные пути от своего каталога standalone-server — передавать абсолютные.
