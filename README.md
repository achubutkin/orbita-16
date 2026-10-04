# Орбита‑16

Статическая веб‑страница с эмулятором 16‑битной приставки Sega Mega Drive / Genesis. Консоль нарисована целиком на SVG и CSS, вид сверху, без единой растровой картинки. Игра идёт на мониторе, который стоит поверх корпуса. При загрузке ROM проигрывается анимация: картридж опускается в слот, шторки разъезжаются, щёлкает тумблер питания, экран включается вспышкой, как ЭЛТ.

Корпус оригинальный и не копирует фирменный дизайн Sega. Есть чёрный и белый варианты.

**▶ Поиграть прямо в браузере: [achubutkin.github.io/orbita-16](https://achubutkin.github.io/orbita-16/)**

## Возможности

- Эмуляция Mega Drive / Genesis, Master System и Game Gear на ядре Genesis Plus GX (WebAssembly).
- Загрузка ROM кнопкой, кликом по слоту или перетаскиванием файла на страницу. Форматы: `.md`, `.bin`, `.gen`, `.smd`, `.sms`, `.gg`, `.sg`.
- Чтение заголовка ROM: название, система, регион, серийный номер, объём. Эти данные попадают на наклейку картриджа, а её цвета зависят от названия игры.
- Анимация вставки и извлечения картриджа (Web Animations API).
- Чёрный и белый корпус с плавной сменой цвета. Выбор сохраняется в `localStorage`.
- Питание, сброс, пауза, звук, полноэкранный режим, извлечение картриджа.
- Клавиатура, геймпад (подхватывается автоматически) и сенсорный джойстик на телефонах.
- Учитывает `prefers-reduced-motion`.
- Всё работает в браузере: ROM‑файлы никуда не отправляются.

## Управление

| Клавиша | Кнопка Mega Drive |
|---|---|
| `←` `↑` `↓` `→` | Крестовина |
| `Z` | A |
| `X` | B |
| `C` | C |
| `Enter` | Start |

## Запуск

Сборка не нужна, но страницу надо открывать через HTTP‑сервер. При открытии по `file://` браузер не загрузит `core.wasm` и ES‑модуль ядра.

```bash
git clone <url-репозитория> orbita-16
cd orbita-16
python3 -m http.server 8000
# откройте http://localhost:8000
```

Подойдёт любой статический сервер: `npx serve`, nginx, GitHub Pages, Netlify и т. п. Сервер должен отдавать `core.wasm` с типом `application/wasm`, а `core.js` — с JavaScript‑типом (`text/javascript`). Большинство серверов делают это сами.

Звук в браузерах включается только после действия пользователя, поэтому до первого клика или нажатия клавиши эмулятор может молчать.

## Структура

```
index.html          Страница: разметка, SVG‑консоль, стили и логика интерфейса
nostalgist.umd.js   Библиотека Nostalgist 0.22.0 (обёртка над RetroArch), с одной правкой — см. ниже
core.js             JS‑часть ядра Genesis Plus GX (RetroArch 1.21.0, Emscripten), заранее пропатченная
core.wasm           Ядро Genesis Plus GX, WebAssembly (~6.8 МБ)
```

## Как устроено

**Интерфейс.** Корпус — один встроенный `<svg>` с градиентами, узором вентиляции и шумом через `feTurbulence`. Цвета корпуса заданы CSS‑переменными (`--k-*`) на элементе `.stage`. Белый вариант переопределяет их через `.stage[data-skin="white"]`, а градиенты подхватывают переменные через CSS‑свойство `stop-color`. Картридж — отдельный SVG в слое `.cart-zone`, который обрезается `clip-path` по линии слота, поэтому картридж визуально уходит внутрь корпуса.

**Эмулятор.** Страница сама скачивает `core.wasm` с индикатором прогресса и передаёт ядро в `Nostalgist.launch()` как `Blob`. Nostalgist запускает RetroArch с ядром Genesis Plus GX на отдельном `<canvas>` внутри «стекла» монитора. Кнопки A/B/C переназначены через `retroarchConfig`.

**Почему ядро пропатчено заранее.** Обычно Nostalgist патчит JS ядра на лету и импортирует его через `blob:` URL. В песочницах со строгой Content Security Policy (например, в артефактах Claude) такой импорт запрещён. Поэтому:

1. `core.js` — это результат функции `patchCoreJs` из Nostalgist, применённой к `genesis_plus_gx_libretro.js` и сохранённой как обычный ES‑модуль.
2. В `nostalgist.umd.js` изменена одна строка: если задан `globalThis.__ndCoreModule`, ядро импортируется по этому URL вместо `blob:`.

```js
// было
const { getEmscripten } = await importCoreJsAsESM(core);
// стало
const { getEmscripten } = globalThis.__ndCoreModule
  ? await import(globalThis.__ndCoreModule)
  : await importCoreJsAsESM(core);
```

Если модуль по адресу `core.js` не загрузится, страница вернётся к стандартному пути Nostalgist.

**Заголовок ROM.** Для Mega Drive читаются поля по смещениям `0x100` (система), `0x150` (название), `0x180` (серийный номер) и `0x1F0` (регион). Файлы `.smd` перед чтением деинтерливятся. Для Master System проверяется подпись `TMR SEGA` по адресу `0x7FF0`.

## Обновление ядра или библиотеки

Ядро взято из npm‑пакета `@rebitplay/retroarch-emscripten` (каталог `1.21.0/`), библиотека — из пакета `nostalgist@0.22.0`. Если обновляете ядро:

```bash
npm pack @rebitplay/retroarch-emscripten nostalgist
# распакуйте genesis_plus_gx_libretro.{js,wasm} и dist/nostalgist.umd.js,
# прогоните JS ядра через patchCoreJs({ js, name: 'genesis_plus_gx' }) из Nostalgist,
# сохраните результат как core.js, а .wasm — как core.wasm,
# и заново внесите правку с __ndCoreModule в nostalgist.umd.js
```

## Лицензии и ROM

- [Genesis Plus GX](https://github.com/ekeeke/Genesis-Plus-GX) распространяется под собственной некоммерческой лицензией. Коммерческое использование ядра запрещено.
- [RetroArch](https://github.com/libretro/RetroArch) — GPLv3.
- [Nostalgist](https://github.com/arianrhodsandlot/nostalgist) — MIT.

ROM‑файлы в репозиторий не входят. Запускайте только образы картриджей, которыми владеете. Sega, Mega Drive и Genesis — товарные знаки Sega Corporation. Проект с Sega не связан.
