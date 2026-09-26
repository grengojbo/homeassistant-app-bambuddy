# homeassistant-app-bambuddy — форк для власної збірки

Форк репозиторію add-on BamBuddy. Мета: **збирати образ у GitHub Actions**, а не на Raspberry Pi,
і тримати виправлення, потрібні для домашньої інсталяції.
Стан на 26.09.2026 — перед висновками перевіряй живий репозиторій.

## Ланцюжок репозиторіїв

| Роль | Репозиторій |
|---|---|
| Застосунок BamBuddy | `maziggy/bambuddy` (issues по самій програмі — сюди) |
| Апстрім add-on | `Spegeli/homeassistant-app-bambuddy` (Dockerfile, config.yaml) |
| Цей форк | `grengojbo/homeassistant-app-bambuddy` → `origin` |

`upstream` як git-remote **не налаштований** — є лише `origin`. Синхронізація йде через
власний workflow `update-versions.yml`, який тягне версії з релізів `maziggy/bambuddy`.

## Структура

- `bambuddy/` — stable add-on: `Dockerfile`, `config.yaml`, `CHANGELOG.md`, `translations/`, `rootfs/`.
- `bambuddy-daily/` — щоденна збірка. **Ми її не збираємо**, лише оновлюються версії.
- `repository.json` — назва репозиторію в магазині HA (змінено на «by grengojbo»).
- `.github/workflows/`: `build.yml`, `update-versions.yml`, `test.yaml`.

## Чим форк відрізняється від апстріму

| Коміт | Що | Доля |
|---|---|---|
| `acf7fc6` | `iproute2` в обох Dockerfile | **Прийнято в апстрім**: `Spegeli/homeassistant-app-bambuddy#5` (merged). Гілка `fix/iproute2-for-ip-aliases` більше не потрібна. |
| `bbcf500` | `image:` в `config.yaml`, `arch: [aarch64]`, `build.yml`, `repository.json` | Тільки для форку, в апстрім не пропонувати |
| `3c4f538` | `bambuddy/translations/uk.yaml` | Можна запропонувати в апстрім окремою гілкою |

При наступній синхронізації з апстрімом коміт з `iproute2` стане зайвим — розв'язуй конфлікт
на користь апстріму, але **не втрать** `image:`, `arch:` і `build.yml`.

## Збірка образу

`build.yml`: нативний раннер `ubuntu-24.04-arm`, **лише `linux/arm64`** (цільова машина — CM5),
кеш `type=gha`, теги `:<версія з config.yaml>` і `:latest`, публікація в
`ghcr.io/grengojbo/addon-bambuddy` (пакет **публічний** — інакше Supervisor отримає `401`).
Збірка триває ~1–2 хвилини. PCIe/Gen3 та `amd64` не чіпати.

### Як запускається збірка (автоматично з 26.09.2026)

`update-versions.yml` працює за розкладом (події `schedule` кілька разів на добу) і комітить нову
версію. **Push, зроблений вбудованим `GITHUB_TOKEN`, не тригерить інші workflow** — захист GitHub
від циклів, через нього `build.yml` раніше не запускався. Винятки з цього правила —
`workflow_dispatch` і `repository_dispatch`: вони спрацьовують **завжди**, навіть від
`GITHUB_TOKEN`. Тому в кінець job `Update STABLE` доданий крок `Build the image for the new
version`, який робить `gh workflow run build.yml`. PAT не потрібен.

Умова збірки — **та сама, що й у кроку коміту**: новий не-передрелізний реліз `maziggy/bambuddy`
відрізняється від `ARG BAMBUDDY_VERSION` у `bambuddy/Dockerfile`. Тобто одна збірка на один новий
реліз; коли версія не змінилась, крок пропускається (перевірено 26.09.2026 — холостих запусків
немає). Зміни в `bambuddy-daily/` збірку не запускають.

Job потребує `permissions: actions: write` — без нього виклик `gh workflow run` отримає `403`.

Вручну збірку й далі можна запустити будь-коли:

```bash
gh workflow run build.yml --repo grengojbo/homeassistant-app-bambuddy
gh run watch $(gh run list --repo grengojbo/homeassistant-app-bambuddy --workflow=build.yml --limit 1 --json databaseId --jq '.[0].databaseId') --repo grengojbo/homeassistant-app-bambuddy
```

Історія: 26.09.2026 автооновлення підняло `1.2.5.6`, образу не було (`404`), HA показував
оновлення, яке впало б. Полагоджено ручним запуском, add-on оновлено до `1.2.5.6`, після чого
додано автоматичний виклик. Якщо колись знову побачиш версію без образу — перевір, чи не зламався
цей крок і чи не зникло право `actions: write`.

### Перевірка, чи є образ

```bash
t=$(curl -s "https://ghcr.io/token?scope=repository:grengojbo/addon-bambuddy:pull&service=ghcr.io" \
  | python3 -c "import json,sys;print(json.load(sys.stdin)['token'])")
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer $t" \
  -H "Accept: application/vnd.oci.image.index.v1+json" \
  https://ghcr.io/v2/grengojbo/addon-bambuddy/manifests/<версія>
```

## Правила роботи з репозиторієм

- **`CHANGELOG.md` не редагувати** — його перезаписує `update-versions.yml` текстом релізу апстріму.
- **Версію вручну не міняти** — її піднімає той самий workflow із релізів `maziggy/bambuddy`.
- Зміни для апстріму тримати в окремій гілці **без** `image:`, `arch:` і `build.yml`, інакше PR не приймуть.
- `bambuddy-daily/` не збираємо; якщо колись знадобиться — там свій Dockerfile з тими ж правками.

## Зв'язок з Home Assistant

- Репозиторій доданий у HA; slug add-on — **`cdf6b064_bambuddy`** (префікс — хеш URL репозиторію,
  тому при зміні URL це буде інший add-on із порожньою текою даних).
- Дані: `/addon_configs/cdf6b064_bambuddy/data`.
- **Репозиторій публічний** — адреси, імена хостів і облікові записи домашньої мережі сюди не
  пиши. Вони, разом з описом самої інсталяції HA (SSH, alias для віртуальних принтерів, мережа),
  лежать у локальних нотатках `~/src/homeassistant/CLAUDE.md`, які нікуди не публікуються.
- Перевірити, що встановлено й що бачить стор (`<ha-host>` — адреса з локальних нотаток):

```bash
ssh hassio@<ha-host> 'export SUPERVISOR_TOKEN=$(cat /run/s6/container_environment/SUPERVISOR_TOKEN); \
  curl -s -H "Authorization: Bearer $SUPERVISOR_TOKEN" http://supervisor/addons/cdf6b064_bambuddy/info \
  | python3 -c "import json,sys; d=json.load(sys.stdin)[\"data\"]; print(d[\"version\"], d[\"version_latest\"], d[\"update_available\"])"'
```

## Відкриті питання в апстрімі

- **`maziggy/bambuddy#3009`** — закрито. Ми повідомили, що після друку дві FTP-сесії до принтера
  нібито не закриваються (і пов'язали це з SD-помилкою `0500-C010`). Мейнтейнер перевірив:
  **з'єднання закриваються**, обидві — це post-print cleanup (`/x.gcode` і `/x.3mf`), кожна
  закривається у `finally`. У логах цього не видно, бо `disconnect()` **нічого не пише**.
  У наступній збірці він додав рядок про закриття кожної сесії.
  Причина `0500-C010` **лишається невідомою**. Корисний наступний крок: на версії ≥1.2.5.6
  зняти debug-лог через цикл «друк → вимкнення → вмикання» і надіслати в ту ж issue.
- `Spegeli/homeassistant-app-bambuddy#5` — merged, `iproute2` тепер в апстрімі.

## Уроки

- **Відсутність рядка в лозі не доводить, що дії не було.** Саме на цьому я побудував хибний
  висновок про «незакриті FTP-сесії». Перш ніж стверджувати про відсутність дії, перевір, чи
  взагалі ця дія щось логує.
- Стан цього форку перевіряй фактами: `gh run list`, теги в GHCR, `version_latest` у Supervisor.
  Локальний `main` легко відстає — там щодня з'являються автоматичні коміти.
