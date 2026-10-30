# Конвейер ИИ-агентов для разработки

Строю и веду **«Штаб»** — конвейер агентов Claude, который сам ведёт цикл разработки: задача текстом → агент-исполнитель пишет код и открывает PR → автопроверки → агент-ревьюер даёт вердикт → человек сливает с телефона. Моё участие — три вещи: ставлю задачи, принимаю решения, нажимаю «слить».

**Цифры (10.2026):** 7 приложений на конвейере, 170 слитых PR агентов. Задача — 1–5 $ и 3–15 минут работы агента.

> **Откуда доллары.** Конвейер работает по подписке Claude, отдельного счёта за задачу нет. «1–5 $» — оценка самого Claude Code: после каждого прогона он считает, сколько стоили бы потраченные токены по ценам API (`total_cost_usd`), и конвейер пишет это в «Замер прогона» вместе с ходами и минутами. Это мера объёма работы — по ней сравниваются задачи, модели и недели.

## Как устроен «Штаб»

- **Исполнитель** — берёт задачу из очереди, пишет код и тесты, открывает PR. Класс задачи (простая, обычная, крупная) задаёт модель, усилие и предел ходов.
- **Ревьюер** — читает PR после зелёных проверок: «можно сливать» или «не сливать» с причинами и списком того, что проверить руками.
- **Развязчик** — сам разрешает конфликт, когда два PR задели одно место.
- **Спецификатор** — превращает просьбу в задачу: проверяемые признаки «готово», контекст для исполнителя со ссылками на код, оценка объёма.
- **Ретро** — раз в неделю: сводка всех прогонов, ошибки агентов, правки их инструкций. Каждую правку сначала проверяет отдельный агент-«судья» без доводов автора.
- **Аналитик** — раз в неделю читает код и документы приложений и предлагает задачи; решает человек.
- **Консультант** — раз в месяц смотрит на всё сразу и предлагает до пяти изменений, у каждого доказательство (файл и строка) и цена.
- **Очередь и предохранители** — задача «после #N» стартует сама, когда #N слита; суточный предел PR; пороги лимитов подписки перед запуском; стоп-кнопка; аварийный план на случай потери аккаунта или сервиса.
- **Вход задач с телефона** — сообщение боту в Telegram → Cloudflare Worker → задача во входящих.

Инструкции агентов и workflow открыты: **[Skills_Claude](https://github.com/goigolden88/Skills_Claude)** — с гайдом, как подключить агентов к своему репо.

## Работа конвейера по неделям

По еженедельным отчётам ретро. «Оценка, $» — сумма `total_cost_usd` исполнителя за неделю (см. выше).

| Неделя | Что сделали агенты | PR агентов / слито | Ходов на прогон | Оценка, $ |
|---|---|---|---|---|
| 01–07.10 | первая неделя конвейера: функции «Делу Время» (особые дни, распорядок, итоги месяца), «Дневников», «Злотых»; подбор модели — одна задача двумя моделями, лучший PR сливался, 7 проигравших закрыты | 48 / 41 | 71 | — (замер с конца недели) |
| 08–12.10 | новое приложение «Фишки» от пустого репо; демо-режим в трёх приложениях — пачкой за вечер по образцу пилота | 25 / 24 | 71 | 52 |
| 13–14.10 | три задачи, все с первого раза | 3 / 3 | 52 | 3 |
| 15–21.10 | «Авто» с нуля — 15 слитых PR за неделю; справки и доработки остальных | 44 / 44 | 68 | 92 |
| 22–28.10 | онлайн-покер в «Фишках»: комната по ссылке, карты на телефонах, турниры, омаха и шорт-дек, стол против ботов | 35 / 35 | 88 | 101 |

## Что находили ретро, аналитик и консультант

- **72 отказа в правах за первую неделю → 0.** Исполнитель упирался в запреты на команды (`gh pr create`, перенаправление вывода, составные команды). Ретро переписало инструкцию — отчёт PR файлом, список разрешённых команд в промпте; со второй недели отказов 0–3 на неделю.
- **8 прогонов без PR из-за одного пробела в YAML.** Выражение `${{ … - 1 }}` в workflow очереди ломалось молча, разбор YAML этого не видел. Ретро добавило проверку `actionlint` на PR с workflow и доказало пробой: на старой ошибке проверка красная, после правки — зелёная.
- **Задачи оказывались больше своей оценки — две недели подряд, в четырёх проектах.** Ретро добавило спецификатору правило: сверять «Оценку» со слитым PR по тем же файлам и поднимать число, если там файлов больше.
- **Двойной тап удваивал капитал.** Аналитик нашёл в «Злотых»: быстрый двойной тап по «Сохранить» записывал операцию дважды — на «Долгах», в «Комнате» и в «Капитале». Задача → PR → слито в тот же день.
- **Одни тренировки вносятся в два приложения.** Консультант заметил, что «Дневники» и «Делу Время» просят одно и то же, и показал, где связь между ними уже решена, но не сделана.
- **«Проверь на телефоне» — оговорка, а не проверка.** Консультант нашёл дефекты, прошедшие ревью с этой фразой, и предложил прогон в браузере на ширине телефона в проверках PR.
- **Старт сессии — скрипт вместо ручной починки.** Консультант посчитал, что каждая облачная сессия тратила ходы на починку клонов, — теперь это делает `start.sh` одним вызовом.

**Стек конвейера:** Claude Code, GitHub Actions (`claude-code-action`), bash, Python, GitHub REST API, Cloudflare Workers, Telegram Bot API.

## Что построено конвейером — примеры

| Проект | Что это | Чем интересен |
|---|---|---|
| [Фишки](https://github.com/goigolden88/Magic_Fishki) · [открыть](https://goigolden88.github.io/Magic_Fishki/) | учёт покерных вечеров и онлайн-покер с друзьями | первый проект целиком агентами от пустого репо; онлайн-покер (сервер на Cloudflare Workers, холдем, омаха, шорт-дек, турниры, боты) — 35 задач и 41 PR за неделю |
| Семья local-first приложений: [Делу Время](https://github.com/goigolden88/DeluVremya), [Дневники](https://github.com/goigolden88/Dnevniki), [Трапеза](https://github.com/goigolden88/Trapeza), [Злотые](https://github.com/goigolden88/Zloty), [Авто](https://github.com/goigolden88/Avto), [Тотальный Учёт](https://github.com/goigolden88/TotalUchet) | время, обслуживание и здоровье, еда, финансы, машина, сводка всего | работают без сервера и сети (PWA); общее ядро [FamilyCore](https://github.com/goigolden88/FamilyCore) — синхронизация через GitHub, новые версии ядра расходятся по приложениям сами; демо-режим без регистрации |

**Стек приложений:** Vite, React, TypeScript (strict), IndexedDB, Vitest, GitHub Pages, Cloudflare Workers.

## Беру заказы

Небольшие проекты на 1–5 дней: веб-приложения и PWA на React/TypeScript, Telegram-боты, скрипты и парсеры на Python, Cloudflare Workers — и настройка агентов Claude под ваш репозиторий.

Связаться: [goigolden88@gmail.com](mailto:goigolden88@gmail.com)

---

# AI agent pipeline for software development

I build and run **Shtab**, a pipeline of Claude agents that runs the development cycle on its own: a task in plain text → an executor agent writes code and opens a PR → automated checks → a reviewer agent gives a verdict → I merge from my phone. My part is three things: set tasks, make decisions, press "merge".

**Numbers (Oct 2026):** 7 apps on the pipeline, 170 merged agent PRs. A task costs $1–5 and 3–15 minutes of agent time.

> **Where the dollars come from.** The pipeline runs on a Claude subscription, so there is no per-task bill. "$1–5" is Claude Code's own estimate: after each run it reports what the tokens would cost at API prices (`total_cost_usd`), and the pipeline logs it with turns and minutes. It is a measure of work, used to compare tasks, models and weeks.

**Agents:** executor (code, tests, PR; task class sets model, effort and turn limit) · reviewer ("merge" / "don't merge" with reasons and manual checks) · resolver (merge conflicts) · specifier (turns a request into a task with testable acceptance criteria) · weekly retro (reviews every run, fixes agent instructions; each fix is checked first by a separate "judge" agent) · weekly analyst (proposes tasks) · monthly consultant (up to five process changes, each with evidence and cost) · queue with daily PR limits, usage thresholds, a stop switch and a disaster plan · tasks from the phone via a Telegram bot and a Cloudflare Worker.

**Week by week** (from retro reports; $ = executor `total_cost_usd`):

| Week | What agents built | Agent PRs / merged | Turns per run | Estimate, $ |
|---|---|---|---|---|
| Oct 1–7 | first week: features in time-tracking, journals and finance apps; model selection by running one task on two models | 48 / 41 | 71 | — |
| Oct 8–12 | a new app from an empty repo; demo mode in three apps in one evening | 25 / 24 | 71 | 52 |
| Oct 13–14 | three tasks, all first try | 3 / 3 | 52 | 3 |
| Oct 15–21 | a car-log app from scratch — 15 merged PRs in a week | 44 / 44 | 68 | 92 |
| Oct 22–28 | online poker: rooms by link, cards on phones, tournaments, Omaha and short deck, bots | 35 / 35 | 88 | 101 |

**What retro, analyst and consultant found:** 72 permission denials in week one dropped to 0–3 a week after the retro rewrote the executor's instructions · 8 runs without a PR traced to one space in a YAML expression — the retro added `actionlint` and proved it red on the old bug · tasks were bigger than their estimates two weeks running in four projects — the specifier now checks estimates against the last merged PR on the same files · the analyst found that a fast double tap saved a finance entry twice · the consultant noticed workouts were entered in two apps and that "check on your phone" in reviews was a caveat, not a check.

**Built by the pipeline:** [Fishki](https://github.com/goigolden88/Magic_Fishki) — poker nights and online poker with friends, built entirely by agents from an empty repo (35 tasks, 41 PRs in a week for online poker) · a family of local-first PWAs (time, journals, food, finance, car, dashboard) on a shared core [FamilyCore](https://github.com/goigolden88/FamilyCore) with sync through GitHub. Agent instructions are open: [Skills_Claude](https://github.com/goigolden88/Skills_Claude).

**Open to small projects (1–5 days):** React/TypeScript web apps and PWAs, Telegram bots, Python scripts and scrapers, Cloudflare Workers, Claude agent setup for your repo. Contact: [goigolden88@gmail.com](mailto:goigolden88@gmail.com)
