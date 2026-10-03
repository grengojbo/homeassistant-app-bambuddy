# homeassistant-app-bambuddy — форк для власної збірки

Форк репозиторію add-on BamBuddy. Мета: **збирати образ у GitHub Actions**, а не на Raspberry Pi,
і тримати виправлення, потрібні для домашньої інсталяції.
Стан на 03.10.2026 — перед висновками перевіряй живий репозиторій.

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
| `acf7fc6` | `iproute2` в обох Dockerfile | **Прийнято в апстрім**: `Spegeli/homeassistant-app-bambuddy#5` (merged 04.09.2026). Гілку PR видалено 26.09.2026. |
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

Обидва job (`Update STABLE` і `Update DAILY`) пушать у `main` паралельно. 02.10.2026 реліз 1.2.5.7
і daily вийшли в одну годину, пуш STABLE програв (`cannot lock ref`), і образ з'явився лише з
наступним запуском, через ~5 год. Тому пуш в обох job — це `git pull --rebase && git push` до
трьох спроб. Конфліктів не буде: job змінюють різні теки.

`Update DAILY` бере digest тегу `ghcr.io/maziggy/bambuddy:daily` запитом до маніфесту. З 03.10.2026
апстрім публікує його як **OCI image index**. Коли в `Accept` лише Docker-формати, GHCR відповідає
`404` (наче тегу немає), заголовка `docker-content-digest` немає, і job падає з
`ERROR: Digest could not be fetched!`. Тому в запиті є обидва набори типів: Docker
(`manifest.v2`, `manifest.list.v2`) і OCI (`image.index.v1`, `image.manifest.v1`). Якщо крок знову
впаде з цією помилкою, спершу перевір формат вручну:

```bash
t=$(curl -s "https://ghcr.io/token?scope=repository:maziggy/bambuddy:pull" | jq -r .token)
curl -sI -H "Authorization: Bearer $t" \
  -H "Accept: application/vnd.oci.image.index.v1+json" \
  -H "Accept: application/vnd.docker.distribution.manifest.list.v2+json" \
  https://ghcr.io/v2/maziggy/bambuddy/manifests/daily | grep -iE '^HTTP|content-type|digest'
```

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

- **`maziggy/bambuddy#3009`** — SD-помилка `0500-C010` після вимкнення принтера. Issue закрита,
  але 29.09.2026 додано коментар з новими даними — чекаємо відповіді мейнтейнера.
  - **Початкова версія хибна.** Ми вважали, що FTP-сесії після друку не закриваються. Мейнтейнер
    показав, що вони закриваються, просто `disconnect()` нічого не логував; у `10f0900` (є в
    1.2.5.6) додано рядки `FTP session … closed after QUIT, held …`.
  - **Тест 27.09.2026 на 1.2.5.6** (P1S і A1 Mini одночасно, друк → пауза → вимкнення → увімкнення):
    FTP виключено — кожне підключення закрилось через `QUIT`, останнє за 11 хв до вимкнення.
    Помилка з'явилась на обох принтерах.
  - **Знайдений механізм (сильна кореляція, не доведено):** після друку post-print cleanup
    (`backend/app/main.py`, `Deleted /<job>.3mf from printer … SD card`) видаляє файл завдання
    з кореня картки. Прошивка пам'ятає останнє завдання — після ввімкнення повідомляє
    `gcode_file: <job>.3mf` — і через 3–4 с видає `0500_C010`. Cleanup існує, бо P1S і A1
    після ввімкнення самі запускають файли з кореня (ghost prints, #374, #1542). Тобто це дві
    сторони однієї поведінки прошивки: файл лишився — примарний друк, файл видалено — помилка.
  - Cleanup **безумовний**, налаштування для нього немає. Помилка — підказка `0xC0xx`, картку не
    псує, достатньо натиснути Confirm.
  - Сценарій користувача: Bambu Studio → **Send** на віртуальний принтер → черга BamBuddy →
    BamBuddy завантажує файл на принтер. До BamBuddy друк ішов із Bambu Studio напряму, файл
    лишався на картці, помилки не було.
  - Остаточна перевірка — повернути файл на картку перед вимкненням. Ризикована: принтер може
    сам почати друк, тож лише з порожнім столом.
- `Spegeli/homeassistant-app-bambuddy#5` — merged, `iproute2` тепер в апстрімі.

## Уроки

- **Відсутність рядка в лозі не доводить, що дії не було.** Саме на цьому я побудував хибний
  висновок про «незакриті FTP-сесії». Перш ніж стверджувати про відсутність дії, перевір, чи
  взагалі ця дія щось логує.
- **Як знімати debug-лог для таких тестів.** `debug: true` дає сотні тисяч рядків на годину,
  тому пиши у файл лише потрібне: `docker logs -f --since 0s app_cdf6b064_bambuddy | awk '…'`
  з фільтром на `bambu_ftp`, `FTP session`, `print_error`, `HMS`, `MQTT disconnected`,
  `PRINT COMPLETE`, `gcode_state:` **і `SD card`**. Останнє пише логер `backend.app.main`, а не
  `bambu_ftp` — без нього видалення файлу з картки не потрапить у запис (так і сталось 27.09;
  довелось добирати з повного логу). Після тесту вимкни `debug` і зупини запис:
  `sudo pkill -f "docker logs -f --since 0s app_cdf6b06[4]"`.
- **Workflow впав, хоча ми нічого не міняли, — шукай зміну в апстрімі.** Так було з OCI-форматом
  `daily` 03.10.2026. Спершу відтвори запит вручну, а тоді вже правь workflow.
- Стан цього форку перевіряй фактами: `gh run list`, теги в GHCR, `version_latest` у Supervisor.
  Локальний `main` легко відстає — там щодня з'являються автоматичні коміти.
