# Код-ревью DeBuilder v0.13

## Обзор проекта

**DeBuilder** — одностраничное веб-приложение для создания информационных стендов с настраиваемыми карманами для документов. Весь проект (HTML + CSS + JS) содержится в одном файле `DeBuilder_0.13.html` (~1755 строк).

---

## 1. БАГИ И КРИТИЧЕСКИЕ ПРОБЛЕМЫ

### 1.1 Отсутствие верхней границы при перетаскивании (строка 1489)

```javascript
// Текущий код:
let nx = Math.max(0, dragState.origX + dx), ny = Math.max(0, dragState.origY + dy);
```

**Проблема:** Есть проверка `Math.max(0, ...)`, но нет верхней границы. Карман можно утащить за правый/нижний край стенда.

**Исправление:**
```javascript
const sw = num('standW'), sh = num('standH');
let nx = Math.max(0, Math.min(sw - p.w, dragState.origX + dx));
let ny = Math.max(0, Math.min(sh - p.h, dragState.origY + dy));
```

**Та же проблема** в touch-обработчике (строка 1698-1699).

---

### 1.2 Отсутствие валидации индексов массива

**`removePocket(i)` — строка 1032:**
```javascript
function removePocket(i) { pockets.splice(i, 1); ... }
```
Нет проверки `i >= 0 && i < pockets.length`. При невалидном индексе `splice` выполнится без ошибки, но с непредсказуемым результатом.

**`changePocketType(idx, type)` — строка 989:**
```javascript
const p = pockets[idx], s = getTypeSize(type);
```
Если `idx` невалидный — `p` будет `undefined`, далее крэш на `p.w`.

**`togglePocketOrientation(i)` — строка 1033:**
Та же проблема.

**Исправление:** Добавить guard-check `if (i < 0 || i >= pockets.length) return;` в начало каждой из этих функций.

---

### 1.3 Клавиатурное управление без проверки существования кармана (строка 1633)

```javascript
if (selectedPocket < 0) return;
const p = pockets[selectedPocket], step = e.shiftKey ? 10 : 1;
```

**Проблема:** Проверяется `selectedPocket < 0`, но не проверяется `selectedPocket >= pockets.length`. Если удалить последний карман и `selectedPocket` указывает на несуществующий индекс — крэш.

**Исправление:**
```javascript
if (selectedPocket < 0 || selectedPocket >= pockets.length) return;
```

---

### 1.4 Утечка памяти в exportPNG (строка 1622-1626)

```javascript
const url = URL.createObjectURL(new Blob([svg], { type: 'image/svg+xml;charset=utf-8' }));
img.onload = function () { ...; URL.revokeObjectURL(url) };
img.src = url;
```

**Проблема:** Нет `img.onerror` — если изображение не загрузится, `URL.revokeObjectURL` никогда не вызовется → утечка памяти.

**Исправление:**
```javascript
img.onerror = function() {
    URL.revokeObjectURL(url);
    console.error('Ошибка экспорта PNG: не удалось загрузить SVG');
};
```

---

### 1.5 Возможен null у canvas context (строка 1625)

```javascript
const ctx = canvas.getContext('2d');
```

**Проблема:** `getContext('2d')` может вернуть `null` (например, если WebGL контекст уже создан, на некоторых устройствах). Далее — крэш при `ctx.drawImage(...)`.

**Исправление:**
```javascript
const ctx = canvas.getContext('2d');
if (!ctx) { alert('Не удалось создать контекст Canvas 2D'); return; }
```

---

### 1.6 exportPDF — потенциальный popup-блокировщик (строка 1628)

```javascript
const w = window.open('', '_blank');
w.document.write(...)
```

**Проблема:** `window.open()` может вернуть `null`, если браузер блокирует всплывающие окна. Далее `w.document.write(...)` вызовет крэш.

**Исправление:**
```javascript
const w = window.open('', '_blank');
if (!w) { alert('Всплывающие окна заблокированы. Разрешите их для экспорта PDF.'); return; }
```

---

### 1.7 Утечка Blob URL в download() (строка 1620)

```javascript
function download(c, f, m) {
    const b = new Blob([c], { type: m }), a = document.createElement('a');
    a.href = URL.createObjectURL(b);
    a.download = f;
    a.click();
}
```

**Проблема:** `URL.createObjectURL(b)` создаёт Blob URL, но он никогда не освобождается через `URL.revokeObjectURL()`.

**Исправление:**
```javascript
function download(c, f, m) {
    const b = new Blob([c], { type: m }), a = document.createElement('a');
    const url = URL.createObjectURL(b);
    a.href = url;
    a.download = f;
    a.click();
    setTimeout(() => URL.revokeObjectURL(url), 1000);
}
```

---

### 1.8 Некорректный clipPath ID при множественных SVG (строка 1393)

```javascript
svg += `<clipPath id="hdrClip">...`;
```

**Проблема:** `id="hdrClip"` жёстко прописан. На странице одновременно существует SVG холста и может быть SVG для экспорта — конфликт ID в DOM.

**Исправление:** Использовать уникальные ID с суффиксом (например `hdrClip-${Date.now()}`) или перейти на inline clip-path.

---

### 1.9 Бесконечный цикл при шаге сетки = 0 (строка 1076)

```javascript
if (s > 0) {
    for (let v = ox; v <= sw; v += s) t.x.push(...)
}
```

**Проблема:** Проверка `s > 0` есть — хорошо. Но если `s` — очень малое число (например 0.001), цикл создаст миллионы snap-точек, подвесив браузер.

**Исправление:**
```javascript
if (s >= 1) { ... }  // или хотя бы s >= 0.5
```

**Та же проблема** в отрисовке сетки (строка 1389).

---

### 1.10 Гонка состояний при touchDragState (строки 1690-1725)

```javascript
document.addEventListener('touchmove', function(e) {
    if (!touchDragState) return;
    const p = pockets[touchDragState.idx];
    if (!p) return;
    ...
});
```

**Проблема:** Хоть проверка `if (!p) return` есть, между `touchmove` и `touchend` может произойти удаление кармана (через Delete-клавишу или программно). Состояние `touchDragState` останется "зависшим", и `rebuild()` в `touchend` пересоберёт UI, но `touchDragState.overlay` будет ссылаться на удалённый DOM-элемент.

**Исправление:** В `removePocket` сбрасывать `touchDragState`:
```javascript
function removePocket(i) {
    if (touchDragState && touchDragState.idx === i) touchDragState = null;
    if (i < 0 || i >= pockets.length) return;
    pockets.splice(i, 1);
    ...
}
```

---

### 1.11 Функция relayoutPockets перепозиционирует только при точном совпадении числа (строка 1021)

```javascript
if (pockets.length && pockets.length === lastGridCols * lastGridRows) {
```

**Проблема:** Если пользователь добавил или удалил карман вручную — `relayoutPockets()` тихо ничего не делает (кроме вызова `rebuild()`). Нет обратной связи для пользователя.

---

## 2. ПРОБЛЕМЫ РАСШИРЯЕМОСТИ

### 2.1 Монолитная архитектура — всё в одном файле

1755 строк HTML+CSS+JS в одном файле. Это делает невозможным:
- Модульное тестирование
- Повторное использование компонентов
- Работу нескольких разработчиков

**Рекомендация:** Разделить на модули:
```
src/
├── index.html
├── styles.css
├── config.js          # Константы, настройки по умолчанию
├── models/
│   └── Pocket.js      # Модель данных кармана
├── controllers/
│   ├── SnapController.js
│   ├── DragController.js
│   └── ExportController.js
├── renderers/
│   ├── SVGRenderer.js
│   ├── OverlayRenderer.js
│   └── DimensionRenderer.js
└── utils/
    ├── dom.js          # val(), num(), chk(), escXml()
    └── geometry.js     # computeGridPositions(), snapVal()
```

---

### 2.2 Глобальные переменные как единственное хранилище состояния (строка 930)

```javascript
let pockets = [], zoom = 0.5, selectedPocket = -1, dragState = null, ...
```

**Проблема:** Всё состояние приложения — глобальные переменные. Невозможно:
- Реализовать undo/redo
- Сериализовать/десериализовать проект (save/load)
- Создать несколько независимых экземпляров

**Рекомендация:**
```javascript
class AppState {
    constructor() {
        this.pockets = [];
        this.zoom = 0.5;
        this.selectedPocket = -1;
        this.dragState = null;
        this.panMode = false;
        this.panX = 0;
        this.panY = 0;
    }

    toJSON() { return JSON.stringify(this); }
    static fromJSON(json) { return Object.assign(new AppState(), JSON.parse(json)); }
}
```

---

### 2.3 Нет системы сохранения/загрузки проектов

**Проблема:** При перезагрузке страницы все данные теряются. Нет экспорта/импорта конфигурации проекта.

**Рекомендация:**
```javascript
function saveProject() {
    const state = {
        version: '0.13',
        pockets: pockets,
        settings: {
            standW: num('standW'),
            standH: num('standH'),
            headerH: num('headerH'),
            // ... все настройки
        }
    };
    localStorage.setItem('debuilder_project', JSON.stringify(state));
}

function loadProject() {
    const data = localStorage.getItem('debuilder_project');
    if (!data) return;
    const state = JSON.parse(data);
    pockets = state.pockets;
    // восстановить настройки в UI
}
```

---

### 2.4 Нет системы Undo/Redo

Каждое действие (перемещение, ресайз, удаление) необратимо.

**Рекомендация:** Паттерн Command:
```javascript
const history = [];
let historyIndex = -1;

function saveState() {
    const snapshot = JSON.parse(JSON.stringify(pockets));
    history.splice(historyIndex + 1);
    history.push(snapshot);
    historyIndex++;
}

function undo() {
    if (historyIndex <= 0) return;
    historyIndex--;
    pockets = JSON.parse(JSON.stringify(history[historyIndex]));
    rebuild();
}

function redo() {
    if (historyIndex >= history.length - 1) return;
    historyIndex++;
    pockets = JSON.parse(JSON.stringify(history[historyIndex]));
    rebuild();
}
```

---

### 2.5 Жёстко прописанные магические числа

| Место | Значение | Что означает |
|-------|----------|-------------|
| Строка 933 | `3.7795275591` | MM → PX коэффициент |
| Строка 1231 | `2` | Размер стрелки размерной линии |
| Строка 1273 | `5` | Толеранс группировки по координатам |
| Строка 1501 | `1.2` | Коэффициент увеличения зума (кнопки) |
| Строка 1544 | `1.1` | Коэффициент увеличения зума (колесо) |
| Строка 1199 | `20` | Минимальный размер кармана при ресайзе |
| Строка 1381 | `12` | Множитель отступа для dimension lines |

**Рекомендация:** Вынести в объект конфигурации:
```javascript
const CONFIG = {
    MM_TO_PX: 3.7795275591,
    ARROW_SIZE: 2,
    GROUP_TOLERANCE: 5,
    ZOOM_FACTOR_BUTTON: 1.2,
    ZOOM_FACTOR_WHEEL: 1.1,
    ZOOM_MAX: 3,
    ZOOM_MIN: 0.05,
    MIN_POCKET_SIZE: 20,
    DIM_OFFSET_MULTIPLIER: 12,
};
```

---

### 2.6 rebuild() пересоздаёт ВСЁ при любом изменении

```javascript
function rebuild() {
    // ...
    wrapper.innerHTML = buildSVG(...) + buildOverlays(...);  // полная перестройка DOM
    renderPocketList();    // полная перестройка списка
    attachDragHandlers();  // переподключение всех обработчиков
}
```

**Проблема:** Любое изменение (сдвиг на 1px, изменение цвета) пересоздаёт весь DOM. При 50+ карманах — заметные тормоза.

**Рекомендация по расширению:**
- Использовать event delegation вместо переподключения обработчиков
- Обновлять только изменённые элементы (virtual DOM или diff-подход)
- Разделить rebuild на `rebuildCanvas()`, `rebuildList()`, `rebuildToolbar()`

---

### 2.7 Экспорт жёстко привязан к 3 форматам

```javascript
function exportSVG() { ... }
function exportPNG() { ... }
function exportPDF() { ... }
```

**Проблема:** Добавление нового формата (DXF, CorelDraw CDR, etc.) требует модификации и HTML-модала, и JavaScript-функций.

**Рекомендация:**
```javascript
const exporters = new Map();

exporters.set('svg', {
    label: 'SVG',
    extension: '.svg',
    mimeType: 'image/svg+xml',
    export: () => generateExportSVG()
});

exporters.set('png', {
    label: 'PNG',
    extension: '.png',
    mimeType: 'image/png',
    export: () => { /* ... */ }
});

// Расширение:
exporters.set('dxf', {
    label: 'DXF (AutoCAD)',
    extension: '.dxf',
    mimeType: 'application/dxf',
    export: () => convertToDXF()
});
```

---

### 2.8 Snap-система не расширяема

Все типы привязок (edges, padding, grid, pockets, gapmid, symmetry) жёстко прописаны в `getSnapTargets()` (строки 1059-1157 — почти 100 строк одна функция).

**Рекомендация:** Плагинная архитектура:
```javascript
const snapProviders = [];

function registerSnapProvider(provider) {
    snapProviders.push(provider);
}

registerSnapProvider({
    name: 'edges',
    enabled: () => chk('snapEdges'),
    getTargets: (di, sw, sh) => ({
        x: [{ pos: 0, type: 'edge' }, { pos: sw, type: 'edge' }],
        y: [{ pos: 0, type: 'edge' }, { pos: sh, type: 'edge' }]
    })
});
```

---

### 2.9 Типы карманов — hardcoded enum

```javascript
const sizes = {
    A3: [num('typeA3W'), num('typeA3H')],
    A4: [num('typeA4W'), num('typeA4H')],
    ...
};
```

**Проблема:** Нельзя добавить пользовательские типы (Letter, Legal, евроформат) без правки HTML и JS.

**Рекомендация:** Динамический реестр типов:
```javascript
class PocketTypeRegistry {
    constructor() {
        this.types = new Map();
    }
    register(name, defaultW, defaultH) {
        this.types.set(name, { w: defaultW, h: defaultH });
    }
    get(name) { return this.types.get(name); }
    getAll() { return [...this.types.entries()]; }
}
```

---

### 2.10 Нет валидации пользовательского ввода

HTML-атрибуты `min`/`max` на `<input>` легко обходятся через copy-paste или DevTools. JavaScript нигде не валидирует значения.

**Пример:**
- `standW` = 0 или отрицательное → сломает рендер
- `gridStep` = 0.0001 → бесконечный цикл сетки
- `borderW` = 1000 → рамка больше стенда

**Рекомендация:**
```javascript
function num(id, min, max) {
    let v = parseFloat(document.getElementById(id).value) || 0;
    if (min !== undefined) v = Math.max(min, v);
    if (max !== undefined) v = Math.min(max, v);
    return v;
}
```

---

## 3. ПРОБЛЕМЫ ПРОИЗВОДИТЕЛЬНОСТИ

### 3.1 Пересоздание обработчиков на каждый rebuild()

```javascript
function attachDragHandlers() {
    document.getElementById('canvas-wrapper').querySelectorAll('.pocket-overlay').forEach(ov => {
        ov.addEventListener('mousedown', ...);
        ov.addEventListener('touchstart', ...);
    });
}
```

Вызывается при каждом `rebuild()`. При активном перетаскивании — десятки раз в секунду.

**Исправление:** Event delegation:
```javascript
document.getElementById('canvas-wrapper').addEventListener('mousedown', (e) => {
    const overlay = e.target.closest('.pocket-overlay');
    if (!overlay) return;
    const idx = parseInt(overlay.dataset.idx);
    // обработка
});
```

---

### 3.2 Конкатенация строк в циклах

```javascript
let svg = '';
pockets.forEach(p => {
    svg += `<rect .../>`;  // неэффективно при большом количестве
});
```

**Рекомендация:** Использовать массив:
```javascript
const parts = [];
pockets.forEach(p => {
    parts.push(`<rect .../>`);
});
return parts.join('');
```

---

### 3.3 renderPocketList() пересоздаёт весь HTML-список

При каждом изменении — полная пересборка списка карманов с `el.innerHTML = html`. При 50+ карманах — заметная задержка и потеря фокуса на инпутах.

---

## 4. ПРОБЛЕМЫ БЕЗОПАСНОСТИ

### 4.1 Скрипты Касперского (строки 8-16)

Файл содержит несколько `<script>` тегов, загружающих JS с `me.kis.v2.scr.kaspersky-labs.com`. Это инъекции антивируса Kaspersky в локальную страницу — **их нужно удалить** из репозитория, так как:
- Они специфичны для конкретной машины разработчика
- Содержат длинные уникальные токены
- Не нужны другим пользователям
- Увеличивают размер файла на ~10KB

---

### 4.2 innerHTML с пользовательскими данными

```javascript
html += `... onclick="pockets[${i}].label=this.value;rebuild()" ...`;
```

Хотя прямой XSS-вектор здесь минимален (данные из `<input>`, а не URL), паттерн генерации HTML через конкатенацию строк рискован при расширении функционала.

**Рекомендация:** Использовать DOM API (`document.createElement`) для критичных элементов.

---

## 5. КАЧЕСТВО КОДА

### 5.1 Криптичные имена функций

| Функция | Что делает |
|---------|-----------|
| `val(id)` | Получить значение input |
| `num(id)` | Получить числовое значение |
| `chk(id)` | Проверить checkbox |
| `p`, `t`, `s`, `d` | Одно-буквенные переменные повсюду |

### 5.2 Дублирование кода

- Snap-badge обновляется в 3 местах: `applySnap()`, `applySnapResize()`, `renderTouchSnapLines()`
- Формула `chk('dimEnabled') ? num('dimOffset') * 12 : 0` повторяется 6 раз
- Touch snap-линии дублируют логику рендера из `buildOverlays()`

### 5.3 Нет комментариев к сложной логике

Функция `getSnapTargets()` — 100 строк без единого поясняющего комментария к алгоритму. Раздел «Symmetry» (строки 1122-1155) особенно сложен для понимания.

---

## 6. ПРИОРИТЕТНЫЙ ПЛАН ДЕЙСТВИЙ

### Фаза 1 — Исправление критических багов (быстрые win)
1. Добавить проверки границ массива в `removePocket`, `changePocketType`, `togglePocketOrientation`
2. Добавить `img.onerror` в `exportPNG`
3. Добавить проверку `null` для `window.open` в `exportPDF`
4. Добавить проверку `null` для `canvas.getContext('2d')`
5. Добавить `URL.revokeObjectURL` в `download()`
6. Добавить верхнюю границу при перетаскивании
7. Удалить скрипты Касперского

### Фаза 2 — Стабильность (среднесрочно)
1. Вынести магические числа в объект CONFIG
2. Добавить валидацию числовых полей
3. Перейти на event delegation для drag-обработчиков
4. Освободить `URL.revokeObjectURL` во всех местах

### Фаза 3 — Расширяемость (долгосрочно)
1. Разделить на модули (ES modules)
2. Реализовать save/load проектов (localStorage + файл)
3. Добавить undo/redo
4. Создать плагинную архитектуру для snap-системы и экспортёров
5. Реализовать динамический реестр типов карманов
6. Оптимизировать rebuild() — обновлять только изменённые части
