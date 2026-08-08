# Playwright — конспект для собеседований

## Фикстуры: три уровня (низкоуровневые, GUI, API)

В Playwright фикстуры делятся на три уровня — от встроенных «сырых» до кастомных, которые ты строишь поверх них.

### 1. Низкоуровневые (встроенные) фикстуры

Даются Playwright **из коробки** — ты их не создаёшь, только запрашиваешь. Работают на уровне браузера / протокола / HTTP.

```typescript
test('встроенные фикстуры', async ({ page, context, browser, request }) => {
  // page    — одна вкладка браузера (главная рабочая лошадка UI)
  // context — изолированный браузерный контекст (куки, storage, сессия)
  // browser — сам инстанс браузера (общий на воркер)
  // request — API-контекст для HTTP-запросов БЕЗ браузера
});
```

- **`browser`** — запущенный браузер, один на воркер, тяжёлый.
- **`context`** — изолированное «окно инкогнито»: свои cookies, localStorage, сессия. Внутри может быть несколько `page`.
- **`page`** — одна вкладка. Основной инструмент UI: `page.goto`, `page.click`, локаторы.
- **`request`** — HTTP-клиент для API-тестов, работает **без UI**: `request.get/post/...`.

Их не создают — только используют. Всё остальное строится поверх них.

### 2. GUI (UI) фикстуры — поверх `page`

Кастомные фикстуры, оборачивающие `page` в Page Object'ы. Уровень абстракции — **бизнес-действия в браузере**.

```typescript
import { test as base, Page } from '@playwright/test';

class LoginPage {
  constructor(private page: Page) {}
  async login(user: string, pass: string) {
    await this.page.goto('/login');
    await this.page.fill('#username', user);
    await this.page.fill('#password', pass);
    await this.page.click('#submit');
  }
}
class InventoryPage {
  constructor(private page: Page) {}
  async addToCart(item: string) {
    await this.page.click(`[data-testid="add-${item}"]`);
  }
}

type UIFixtures = { loginPage: LoginPage; inventoryPage: InventoryPage };

export const test = base.extend<UIFixtures>({
  loginPage: async ({ page }, use) => {   // ← зависит от низкоуровневой page
    await use(new LoginPage(page));
  },
  inventoryPage: async ({ page }, use) => {
    await use(new InventoryPage(page));
  },
});
```

Тест говорит на языке UI-действий, а не кликов:

```typescript
test('добавление товара', async ({ loginPage, inventoryPage }) => {
  await loginPage.login('user', 'pass');
  await inventoryPage.addToCart('backpack');
});
```

**Суть:** берёт `page` → отдаёт готовый Page Object.

### 3. API фикстуры — поверх `request`

Кастомные фикстуры, оборачивающие `request` в API-клиент. Уровень абстракции — **операции на бэкенде, без браузера**.

```typescript
import { test as base, APIRequestContext } from '@playwright/test';

class ApiClient {
  constructor(private request: APIRequestContext) {}
  async createUser(role: string) {
    const resp = await this.request.post('/api/users', { data: { role } });
    return resp.json();
  }
  async deleteUser(id: number) {
    await this.request.delete(`/api/users/${id}`);
  }
  async getOperationStatus(opId: string) {
    const resp = await this.request.get(`/api/operations/${opId}`);
    return (await resp.json()).status;
  }
}

type ApiFixtures = { api: ApiClient };

export const test = base.extend<ApiFixtures>({
  api: async ({ request }, use) => {   // ← зависит от низкоуровневой request
    await use(new ApiClient(request));
  },
});
```

Использование — API-тесты или подготовка данных:

```typescript
test('создание пользователя через API', async ({ api }) => {
  const user = await api.createUser('admin');
  expect(user.role).toBe('admin');
});
```

**Суть:** берёт `request` → отдаёт готовый клиент.

### Как они сочетаются (реальный проект)

API-фикстура готовит данные (быстро), GUI-фикстура проверяет через интерфейс. Фикстуры зависят друг от друга — Playwright выстраивает порядок как DI-контейнер.

```typescript
type Fixtures = {
  api: ApiClient;
  loginPage: LoginPage;
  seededUser: { id: number; login: string; password: string };
};

export const test = base.extend<Fixtures>({
  api: async ({ request }, use) => { await use(new ApiClient(request)); },
  loginPage: async ({ page }, use) => { await use(new LoginPage(page)); },

  // цепочка зависимостей: request → api → seededUser
  seededUser: async ({ api }, use) => {
    const user = await api.createUser('vip');   // setup через API (быстро)
    await use(user);
    await api.deleteUser(user.id);              // teardown
  },
});

test('vip видит спец-предложение', async ({ loginPage, seededUser }) => {
  await loginPage.login(seededUser.login, seededUser.password); // данные из API, проверка через UI
});
```

### Сводная таблица

| Уровень | Фикстура | На чём построена | Уровень абстракции | Пример |
|---|---|---|---|---|
| **Низкоуровневые** | `page`, `context`, `browser`, `request` | встроены в Playwright | браузер / протокол / HTTP | `page.click()`, `request.post()` |
| **GUI** | `loginPage`, `inventoryPage` | поверх **`page`** | бизнес-действия в UI | `loginPage.login()` |
| **API** | `api`, `apiClient` | поверх **`request`** | операции на бэкенде | `api.createUser()` |

### Ключевые различия

- **Низкоуровневые** — даёт Playwright, не создаёшь; сырой уровень браузера/HTTP.
- **GUI-фикстуры** — твои, оборачивают `page` в Page Object'ы; тест говорит языком пользователя, а не селекторов.
- **API-фикстуры** — твои, оборачивают `request` в клиент; работают с бэкендом без браузера — быстрее, для подготовки данных и API-проверок.

**Ответ для собеса:**
> «Низкоуровневые фикстуры (`page`, `request`, `context`) даёт Playwright из коробки — это сырой уровень браузера и HTTP. Поверх них строю свои: GUI-фикстуры оборачивают `page` в Page Object'ы, чтобы тест говорил бизнес-действиями; API-фикстуры оборачивают `request` в клиент для работы с бэкендом без UI. Всё связывается через `test.extend`, где фикстуры зависят друг от друга, а Playwright работает как DI-контейнер».

---

## Зачем `test.extend` вместо `new PageObject(page)` в тесте

Можно и напрямую — `const loginPage = new LoginPage(page)` в тесте работает. Но `test.extend` даёт:

1. **DRY** — `new ...` пишется один раз в фикстуре, а не в каждом тесте.
2. **Ленивое создание** — фикстура инстанцируется, только если тест её запросил; неиспользуемые Page Object'ы не создаются.
3. **Чистое тело теста** — только сценарий, без технической сборки.
4. **setup/teardown** — код до `await use()` это setup, после — teardown (логин до теста, очистка после).
5. **Композиция/DI** — фикстуры зависят друг от друга, Playwright сам выстраивает порядок.
6. **Единая точка изменений** — поменялся конструктор Page Object → правка в одной фикстуре.

Аналогия: можно в каждой функции руками открывать и закрывать файл, а можно завести обёртку, которая открывает, отдаёт готовый и сама закрывает.

---

## Два generic-параметра `test.extend`

```typescript
base.extend< TestFixtures , WorkerFixtures >(...)
//           ^scope "test"   ^scope "worker"
```

- **1-й параметр** — фикстуры scope **`test`** (на каждый тест, аналог pytest `function`).
- **2-й параметр** — фикстуры scope **`worker`** (раз на воркер, аналог `session`).

```typescript
base.extend<{ loginPage: LoginPage }>({...})            // только test-scoped
base.extend<{}, { apiClient: ApiClient }>({...})        // {} = нет test-scoped, apiClient — worker-scoped
```

`{}` — пустой объектный тип (заглушка), когда добавляешь только worker-фикстуру. Фактический scope в рантайме задаётся в реализации: `[async (...) => {...}, { scope: 'worker' }]`.

---

## `storageState` — авторизация один раз

Логинишься один раз (лучше через API), сохраняешь сессию (cookies + localStorage) в файл, все тесты стартуют уже авторизованными:

```typescript
// глобальный setup: логин → сохранить состояние
await page.context().storageState({ path: 'auth.json' });

// в конфиге или тесте
test.use({ storageState: 'auth.json' });
```

Снимает повторный логин в каждом тесте (быстрее, меньше флака на форме). Для разных ролей — отдельный `storageState` на роль.

---

## Локаторы: приоритет надёжности

1. `getByTestId` (`data-testid`) — самый стабильный, не зависит от вёрстки и языка.
2. `getByRole('button', { name: 'Submit' })` — по роли и имени (доступность).
3. Сужение по стабильному **родительскому контейнеру** — если элемент дублируется.
4. ❌ Никогда: индекс/`first`, привязка к тексту на мультиязычном UI, хрупкий CSS/xpath.

`getByTestId` под капотом ищет атрибут `data-testid` (настраивается `testIdAttribute` в конфиге). Locator — не элемент, а «рецепт поиска»: ищется заново при каждом действии, поэтому нет проблемы stale-элементов (в отличие от Selenium `WebElement`).

---

## Ожидания: авто-вейтинг и polling, а не `sleep`

- Playwright имеет **встроенные авто-ожидания**: `expect(locator).toBeVisible({ timeout })` перепроверяет условие до таймаута.
- Для API-состояний (eventual consistency) — **polling**: опрашивать статус, пока не придёт нужный, с таймаутом и понятным сообщением; **не `sleep`** (фиксированная пауза = и медленно, и флак).

---

## Отчётность и артефакты

- Старт — встроенный **HTML-репортер**; при зрелости CI/CD — **Allure** (история, тренды, привязка к кейсам).
- Артефакты на падение: **screenshot, video, trace**. `trace` — полная запись прогона (шаги, DOM, сеть) → разбор без локального перезапуска.

---

## Матрица браузеров/окружений — `projects`

```typescript
// playwright.config.ts
projects: [
  { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
  { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
  { name: 'mobile',   use: { ...devices['iPhone 13'] } },   // мобильный адаптив
]
```

Один набор тестов гоняется под разные браузеры/вьюпорты без дублирования кода.

---

## Пакетный менеджер и версии

- Пакет: `npm i -D @playwright/test` (или pnpm/yarn/bun). pnpm быстрее в CI, npm — из коробки.
- Браузеры отдельно: `npx playwright install`.
- **Версия Chromium жёстко привязана к версии Playwright**; узнать: `browser.version()`, из release notes, `npx playwright --version`. Для системного Chrome — `channel: 'chrome'`.

---

# Погружение: селекторы и работа с DOM (от простого к сложному)

> Всё, что нужно для уверенного лайв-кодинга на собесе. От философии локаторов до продвинутого сужения скоупа и разбора дублей.

## Уровень 0. Философия: что такое Locator

**Locator — это не элемент, а "рецепт поиска".** Ты не находишь элемент один раз, а описываешь, **как** его найти, и Playwright ищет его **заново перед каждым действием**.

```typescript
const btn = page.getByRole('button', { name: 'Submit' }); // это рецепт, не элемент
await btn.click();   // сейчас Playwright идёт в актуальный DOM, находит и кликает
await btn.click();   // и здесь снова ищет заново
```

Два ключевых следствия:
1. **Нет stale-элементов.** DOM перерисовался (SPA, React ре-рендер) — не важно, локатор ищет по живому DOM. В Selenium `WebElement` кэширует ссылку → `StaleElementReferenceException`; в Playwright этой проблемы нет by design.
2. **Встроенный auto-waiting.** Перед действием Playwright сам ждёт, пока элемент станет **actionable** (виден, в DOM, не перекрыт, включён). Не нужны ручные `wait`.

**ElementHandle** (`page.$()`) — единственное, что держит ссылку на элемент и может протухнуть. Считается устаревшим — используй Locator.

### Что такое stale-элемент (подробно)

**Stale-элемент** (от *stale* — «протухший») — устаревшая ссылка на узел DOM: сам элемент перерисовался, был пересоздан или удалён, а ссылка указывает на старый, уже не существующий узел. Обращение к ней падает.

**Механизм (на примере Selenium, где это боль):**
1. Нашёл элемент: `WebElement button = driver.findElement(...)` — получил ссылку на конкретный узел.
2. Страница перерисовалась — SPA/React ре-рендер, AJAX подгрузил данные; старый узел заменён новым.
3. Ссылка `button` указывает на старый, уже несуществующий узел.
4. `button.click()` → **`StaleElementReferenceException`** — «элемент больше не привязан к DOM».

*Аналогия:* сфотографировал кнопку и тыкаешь в фото; экран перерисовался — фото устарело.

**Где особенно часто:** SPA (React/Vue/Angular), динамические таблицы/списки, модалки, действия после AJAX.

**Почему в Playwright проблемы нет:** Locator не кэширует ссылку, а **ищет элемент заново перед каждым действием** по актуальному DOM — протухать нечему.

```typescript
const btn = page.getByRole('button', { name: 'Submit' });
await btn.click();   // ищет заново
// ...DOM перерисовался...
await btn.click();   // снова ищет заново — работает
```

Протухнуть может только **`ElementHandle`** (держит ссылку), поэтому его и не рекомендуют — используй Locator.

**Ответ для собеса:**
> «Stale-элемент — устаревшая ссылка на узел DOM: элемент перерисовался, а ссылка указывает на старый узел, действие падает со `StaleElementReferenceException`. Это боль Selenium, где `WebElement` кэширует ссылку. В Playwright проблемы нет by design: Locator не хранит элемент, а ищет заново перед каждым действием; протухнуть может только `ElementHandle`».

## Уровень 1. Как Playwright «видит» DOM

`page` — пульт к вкладке; DOM живёт в браузере, Playwright общается с ним по протоколу. Локатор при действии выполняет поиск в актуальном дереве. Проверить, сколько элементов находит локатор, можно в **DevTools → Elements → Ctrl+F** (вбить CSS/XPath — покажет количество совпадений) или в консоли `$$('селектор')`.

**Главное правило локатора:** он должен находить **ровно один** элемент. Если находит несколько — Playwright в strict mode кинет ошибку на действии (об этом ниже).

## Уровень 2. Рекомендуемые user-facing локаторы

Playwright рекомендует искать элементы так, **как их видит и воспринимает пользователь** — по роли, тексту, лейблу. Это устойчиво к рефакторингу вёрстки.

```typescript
// по РОЛИ (accessibility) + доступному имени — приоритет №1 после data-testid
page.getByRole('button', { name: 'Купить' });
page.getByRole('link',   { name: 'Тарифы' });
page.getByRole('checkbox', { name: 'Согласен', checked: true });
page.getByRole('heading', { level: 1 });

// по ТЕКСТУ (видимому)
page.getByText('Добро пожаловать');
page.getByText('Точное', { exact: true });   // точное совпадение

// по LABEL формы (то, что связано с input через <label>)
page.getByLabel('Email');

// по PLACEHOLDER
page.getByPlaceholder('Введите email');

// по alt изображения
page.getByAltText('Логотип компании');

// по title-атрибуту
page.getByTitle('Закрыть');

// по data-testid — самый стабильный, спец-атрибут для тестов
page.getByTestId('submit-button');
```

Опции `getByRole`: `name`, `exact`, `checked`, `disabled`, `expanded`, `pressed`, `selected`, `level` (для heading). Имя можно задавать строкой или регуляркой: `{ name: /куп/i }`.

## Уровень 3. CSS-селекторы

Когда user-facing локаторов не хватает — CSS через `page.locator()`:

```typescript
page.locator('#submit');                    // по id
page.locator('.btn-primary');               // по классу
page.locator('button[type="submit"]');      // по атрибуту
page.locator('[data-test="add-to-cart"]');  // по любому атрибуту
page.locator('div.card > button');          // по структуре (прямой потомок)
page.locator('ul li:first-child');          // псевдоклассы CSS
```

Playwright добавляет **свои CSS-псевдоклассы** поверх стандартных:

```typescript
page.locator('button:has-text("Купить")');   // содержит текст (подстрока)
page.locator('button:text("Купить")');        // точный текст элемента
page.locator('.item:visible');                // только видимые
page.locator('article:has(button.buy)');      // содержит внутри элемент по селектору
page.locator(':nth-match(button, 3)');        // 3-е совпадение по всей странице
```

## Уровень 4. XPath (и почему по возможности избегать)

```typescript
page.locator('xpath=//button[text()="Купить"]');
page.locator('//div[@class="card"]//button');   // // = автоматически xpath
```

XPath мощнее CSS (умеет ходить вверх по дереву, искать по тексту), но **хрупкий и нечитаемый**, завязан на структуру. Использовать в крайнем случае, когда ни роль, ни CSS не помогают.

## Уровень 5. ИЕРАРХИЯ выбора селекторов (это спрашивают на собесе!)

Приоритет от самого надёжного к худшему:

1. **`getByTestId`** (`data-testid`) — спец-атрибут для тестов, не меняется при рефакторинге вёрстки и не зависит от языка. **Самый стабильный.**
2. **`getByRole` + name** — по доступности; устойчиво и проверяет заодно a11y.
3. **`getByLabel` / `getByPlaceholder` / `getByText`** — user-facing, но текст завязан на язык (осторожно на мультиязычном UI).
4. **CSS по стабильным атрибутам** (`[data-*]`, устойчивый id).
5. **Сужение по стабильному родительскому контейнеру** — когда элемент дублируется.
6. ❌ **Никогда:** индекс/`nth`/`first` как основной способ; привязка к тексту на мультиязычном сайте; хрупкий CSS по вёрстке (`div > div > span`); длинный XPath.

**Формулировка для собеса:**
> «Иерархия: `data-testid` → роль с именем → label/placeholder/text → стабильный CSS → сужение по родителю. Индекс и текст на мультиязычном UI — антипаттерн: сегодня элемент первый, завтра второй; текст меняется со сменой языка».

## Уровень 6. Ограничение скоупа (chaining) — ключевой навык

Когда элемент не уникален глобально, но уникален **внутри контейнера** — сужаем поиск. Это главный приём против дублей.

```typescript
// локатор внутри локатора — поиск ТОЛЬКО внутри найденного контейнера
const card = page.getByTestId('product-card');
await card.getByRole('button', { name: 'Купить' }).click();

// цепочка
page.locator('nav.header').getByRole('link', { name: 'Тарифы' });

// пример из жизни: дубль кнопки в десктопном и мобильном хедере
page.locator('header.desktop').getByRole('button', { name: 'Premium' });
// → находит ровно один элемент за счёт уникального родителя
```

**Правило:** если селектор находит несколько — не бери индекс, **сужай по стабильному родителю**.

## Уровень 7. `filter()` — фильтрация набора

Когда есть список одинаковых элементов, фильтруем нужный по содержимому:

```typescript
// строка таблицы, содержащая нужный текст
const row = page.getByRole('row').filter({ hasText: 'Иванов' });
await row.getByRole('button', { name: 'Удалить' }).click();

// фильтр по наличию вложенного элемента
page.getByRole('listitem').filter({ has: page.getByRole('button', { name: 'Купить' }) });

// фильтр по ОТСУТСТВИЮ
page.getByRole('listitem').filter({ hasNot: page.getByText('Нет в наличии') });

// по регулярке
page.getByRole('listitem').filter({ hasText: /\d+ ₽/ });
```

`filter` возвращает новый локатор — можно чейнить дальше.

## Уровень 8. Работа с несколькими элементами

```typescript
const items = page.getByRole('listitem');

await items.count();        // сколько элементов (число)
await items.first();        // первый
await items.last();         // последний
await items.nth(2);         // третий (индекс с 0)

// перебор всех
for (const item of await items.all()) {
  console.log(await item.textContent());
}
```

⚠️ **`first/last/nth` — только когда порядок гарантирован** (например, «первая строка результатов»). Как способ обойти дубль — **антипаттерн**: порядок может измениться. Лучше `filter` или сужение по родителю.

## Уровень 9. Комбинирование локаторов — `and()`, `or()`

```typescript
// элемент, удовлетворяющий ОБОИМ условиям
page.getByRole('button').and(page.getByTitle('Подписаться'));

// ИЛИ — один из двух (например, разные состояния)
const dialog = page.getByRole('button', { name: 'OK' })
  .or(page.getByRole('button', { name: 'Продолжить' }));
await dialog.click();
```

`or()` полезен, когда UI может быть в одном из состояний (появилась модалка или нет).

## Уровень 10. Strict mode — почему локатор должен быть уникальным

По умолчанию локаторы **строгие**: если действие вызывается на локаторе, который находит **больше одного** элемента, Playwright **бросает ошибку** (`strict mode violation`), а не берёт молча первый (как делал Selenium). Это защита: тест не должен «угадывать».

```typescript
await page.getByRole('button').click();
// ❌ ошибка, если кнопок несколько: "strict mode violation: resolved to 2 elements"
```

Решение — **сделать локатор уникальным** (сужением/фильтром), а не глушить `first()`.

Проверка на уникальность в лайв-кодинге: `await expect(locator).toHaveCount(1)` или посмотреть в DevTools через Ctrl+F.

## Уровень 11. Действия (actions) — все с auto-waiting

```typescript
await locator.click();
await locator.dblclick();
await locator.fill('текст');            // очистить и ввести (для input)
await locator.clear();
await locator.pressSequentially('abc'); // посимвольный ввод (бывший type)
await locator.press('Enter');           // клавиша
await locator.check();                  // чекбокс/радио
await locator.uncheck();
await locator.selectOption('value');    // <select>
await locator.hover();
await locator.setInputFiles('file.pdf');// загрузка файла
await locator.scrollIntoViewIfNeeded();
```

Перед каждым действием Playwright сам ждёт actionable-состояние — не нужны `waitForSelector`.

## Уровень 12. Проверки (web-first assertions) — с ретраями

Ассерты через `await expect(locator)` **автоматически перепроверяются** до таймаута — это встроенный «умный wait».

```typescript
await expect(locator).toBeVisible();
await expect(locator).toBeHidden();
await expect(locator).toBeEnabled();
await expect(locator).toBeChecked();
await expect(locator).toHaveText('Готово');       // точный текст
await expect(locator).toContainText('готов');     // подстрока
await expect(locator).toHaveValue('user@mail.ru');// значение input
await expect(locator).toHaveAttribute('href', /\/plans/);
await expect(locator).toHaveCount(3);             // столько элементов
await expect(page).toHaveURL(/\/plans/);
await expect(page).toHaveTitle(/Тарифы/);
```

Важно: `await expect(locator).toBeVisible()` **ждёт** появления, а `await locator.isVisible()` — проверяет **мгновенно** (без ретраев, вернёт true/false сейчас). Для проверок используй `expect`, для условной логики — `isVisible()`.

## Уровень 13. Ожидания — когда всё же нужны явные

Auto-waiting покрывает 95%. Явные ожидания — для нестандартного:

```typescript
await page.waitForURL('**/inventory');           // смена URL
await page.waitForLoadState('networkidle');       // сеть утихла (осторожно, флак)
await page.waitForRequest(u => u.includes('/collect')); // ждать запрос (аналитика!)
await page.waitForResponse(r => r.url().includes('/api/order') && r.status() === 200);
await expect(locator).toBeVisible({ timeout: 10000 }); // кастомный таймаут
```

**`sleep` / `waitForTimeout` — антипаттерн.** Для eventual consistency — polling через `expect.poll`:

```typescript
await expect.poll(async () => {
  const r = await request.get(`/api/operations/${id}`);
  return (await r.json()).status;
}, { timeout: 15000, intervals: [500] }).toBe('completed');
```

## Уровень 14. Перехват сети (route/mock) — для лайв-кодинга полезно

```typescript
// перехват и подмена ответа
await page.route('**/api/products', route => {
  route.fulfill({ json: [{ id: 1, name: 'Test' }] });
});

// заблокировать (например, картинки/аналитику)
await page.route('**/*.png', route => route.abort());

// проверить отправку запроса (аналитика GA/AppMetrica)
const reqPromise = page.waitForRequest(u => u.includes('/collect'));
await page.getByRole('button', { name: 'Купить' }).click();
const req = await reqPromise;
expect(req.postData()).toContain('event=purchase');
```

## Уровень 15. Как рассуждать вслух на лайв-кодинге (скелет)

Интервьюер оценивает **ход мысли**, не скорость набора. Проговаривай:

1. **«Разобью задачу на шаги»** — например: найти ссылку → проверить `href` → кликнуть → проверить, что попали.
2. **«Начну с локатора — это главное»** — назови иерархию: `data-testid`? нет → `getByRole` с именем → если дубль, сужу по родителю. К тексту не привязываюсь (мультиязычность), индекс не беру.
3. **«Проверю, что локатор уникален»** — в DevTools Ctrl+F или `toHaveCount(1)`.
4. **Пиши web-first assertions** — `await expect(...)`, не `if (isVisible)`.
5. **Комментируй выбор** — «проверяю href до клика, чтобы падение указывало на причину».

**Эталонный мини-тест (типовое задание):**

```typescript
import { test, expect } from '@playwright/test';

test('ссылка Plans ведёт на страницу тарифов', async ({ page }) => {
  await page.goto('/');

  const plans = page.getByRole('link', { name: 'Plans' }); // роль+имя, не текст/индекс

  await expect(plans).toHaveAttribute('href', /\/plans/);   // 1. href до клика
  await plans.click();                                       // 2. клик
  await expect(page).toHaveURL(/\/plans/);                   // 3. дошли по адресу
  expect(await page.title()).not.toContain('Page Not Found'); // 4. страница живая, не 404
});
```

## Уровень 16. Частые кейсы для лайв-кодинга

**Таблица — найти строку и действие в ней:**
```typescript
const row = page.getByRole('row').filter({ hasText: 'ORDER-123' });
await expect(row.getByRole('cell').nth(2)).toHaveText('Оплачен');
await row.getByRole('button', { name: 'Отменить' }).click();
```

**Модалка — дождаться и закрыть:**
```typescript
const modal = page.getByRole('dialog');
await expect(modal).toBeVisible();
await modal.getByRole('button', { name: 'Закрыть' }).click();
await expect(modal).toBeHidden();
```

**Дубль десктоп/мобайл (классика):**
```typescript
// НЕ page.getByRole('button', { name: 'Premium' }).first()  ❌
await page.locator('header.desktop-nav')
  .getByRole('button', { name: 'Premium' }).click();          // ✅ сужение по родителю
```

**Список — проверить количество и содержимое:**
```typescript
const items = page.getByTestId('product-item');
await expect(items).toHaveCount(6);
await expect(items.first()).toContainText('₽');
```

**Ввод формы:**
```typescript
await page.getByLabel('Email').fill('user@mail.ru');
await page.getByLabel('Пароль').fill('secret');
await page.getByRole('button', { name: 'Войти' }).click();
await expect(page.getByText('Личный кабинет')).toBeVisible();
```

## Уровень 17. Дебаг-инструменты (упомянуть на собесе)

- **`npx playwright codegen <url>`** — генератор: кликаешь, получаешь код (для быстрой основы, не как замена навыка).
- **`npx playwright test --debug`** / **`--ui`** — пошаговый прогон с инспектором, подсветкой локаторов.
- **`await page.pause()`** — остановка и открытие Inspector в нужном месте.
- **Trace Viewer** — `trace: 'on'` в конфиге, потом `npx playwright show-trace` — полная запись прогона (шаги, DOM, сеть) для разбора падений.
- **`--headed`** — видеть браузер во время прогона.

## Шпаргалка приоритетов (запомнить)

```
Локатор:  getByTestId → getByRole(name) → getByLabel/Text → CSS[data-*] → сужение по родителю
Никогда:  nth/first как основа · текст на мультиязыке · div>div>span · длинный xpath
Дубль:    сужай по родителю или filter(), НЕ индексом
Ожидание: auto-wait + expect().toBeVisible() ; для async — expect.poll ; НЕ sleep
Проверка: await expect(locator).toXxx() (с ретраями), не isVisible() в ассерте
Уникальность: strict mode ловит дубли → делай локатор однозначным
```

---

# Жизненный цикл фикстуры: setup → тест → teardown (`use` / `yield`)

> Как гарантируется, что данные создаются ДО теста и удаляются ТОЛЬКО ПОСЛЕ него — с полными рабочими тестами на TypeScript и Python.

## Механизм в одной картинке

```
create(user)  →  use(user) / yield user  ⏸ [ ТЕСТ ВЫПОЛНЯЕТСЯ ЗДЕСЬ ]  →  delete(user)
   setup              точка паузы                                          teardown
```

Ключевая точка — **`await use(value)`** (в Playwright) или **`yield value`** (в pytest). Это **точка приостановки** фикстуры:
- всё, что **до** неё — setup, выполняется **до** теста;
- на ней фикстура **замирает**, отдаёт объект в тест и **ждёт**, пока тест полностью завершится;
- всё, что **после** неё — teardown, выполняется **только после** теста.

Пользователь физически не может удалиться раньше, потому что строка удаления стоит **за** `await use()` / `yield`, который держит фикстуру на паузе всё время теста. Даже если тест **упал** — Playwright/pytest всё равно снимают паузу, поэтому teardown отработает и данные не «протекут» (аналог `finally`).

---

## Вариант 1. TypeScript (Playwright)

### Фикстура + API-клиент

```typescript
// fixtures.ts
import { test as base, APIRequestContext } from '@playwright/test';

type User = { id: number; login: string; password: string; role: string };

class ApiClient {
  constructor(private request: APIRequestContext) {}

  async createUser(role: string): Promise<User> {
    const resp = await this.request.post('/api/users', { data: { role } });
    return resp.json();
  }
  async deleteUser(id: number): Promise<void> {
    await this.request.delete(`/api/users/${id}`);
  }
}

type Fixtures = { api: ApiClient; seededUser: User };

export const test = base.extend<Fixtures>({
  api: async ({ request }, use) => {
    await use(new ApiClient(request));
  },

  seededUser: async ({ api }, use) => {
    // 1. SETUP — выполняется ДО теста
    const user = await api.createUser('vip');
    console.log('создан пользователь', user.id);

    // 2. ПАУЗА — управление уходит в тест, фикстура ждёт здесь
    await use(user);

    // 3. TEARDOWN — выполняется ТОЛЬКО ПОСЛЕ теста (даже если тест упал)
    await api.deleteUser(user.id);
    console.log('удалён пользователь', user.id);
  },
});

export { expect } from '@playwright/test';
```

### Тест, использующий фикстуру

```typescript
// test.spec.ts
import { test, expect } from './fixtures';

test('vip видит спец-предложение', async ({ page, seededUser }) => {
  // на этот момент пользователь УЖЕ создан (setup отработал)
  await page.goto('/login');
  await page.getByLabel('Логин').fill(seededUser.login);
  await page.getByLabel('Пароль').fill(seededUser.password);
  await page.getByRole('button', { name: 'Войти' }).click();

  // целевое действие/проверка — пользователь ещё жив, удаление не произошло
  await expect(page.getByTestId('vip-banner')).toBeVisible();

  // ...тест закончился ЗДЕСЬ...
  // и ТОЛЬКО ТЕПЕРЬ фикстура разблокируется и выполнит api.deleteUser()
});
```

**Порядок в консоли:**
```
создан пользователь 42        ← setup (до теста)
[тест выполняется: логин, проверка баннера]
удалён пользователь 42        ← teardown (после теста)
```

---

## Вариант 2. Python (pytest-playwright)

В Python та же логика, только вместо `await use(value)` — **`yield value`**.

### Фикстура + API-клиент

```python
# conftest.py
import pytest
from playwright.sync_api import APIRequestContext

class ApiClient:
    def __init__(self, request: APIRequestContext):
        self.request = request

    def create_user(self, role: str) -> dict:
        resp = self.request.post("/api/users", data={"role": role})
        return resp.json()

    def delete_user(self, user_id: int) -> None:
        self.request.delete(f"/api/users/{user_id}")


@pytest.fixture
def api(playwright):
    request = playwright.request.new_context(base_url="https://example.com")
    yield ApiClient(request)
    request.dispose()


@pytest.fixture
def seeded_user(api):
    # 1. SETUP — выполняется ДО теста
    user = api.create_user("vip")
    print("создан пользователь", user["id"])

    # 2. ПАУЗА — управление уходит в тест, фикстура ждёт здесь
    yield user

    # 3. TEARDOWN — выполняется ТОЛЬКО ПОСЛЕ теста (даже если тест упал)
    api.delete_user(user["id"])
    print("удалён пользователь", user["id"])
```

### Тест, использующий фикстуру

```python
# test_vip.py
from playwright.sync_api import Page, expect

def test_vip_banner(page: Page, seeded_user):
    # на этот момент пользователь УЖЕ создан (setup отработал)
    page.goto("/login")
    page.get_by_label("Логин").fill(seeded_user["login"])
    page.get_by_label("Пароль").fill(seeded_user["password"])
    page.get_by_role("button", name="Войти").click()

    # целевое действие/проверка — пользователь ещё жив
    expect(page.get_by_test_id("vip-banner")).to_be_visible()

    # ...тест закончился ЗДЕСЬ...
    # и ТОЛЬКО ТЕПЕРЬ pytest вернётся в фикстуру и выполнит delete_user()
```

`yield user` = «отдать пользователя в тест и заморозить фикстуру, пока тест не закончится».

---

## Почему нельзя, чтобы удалилось раньше (детально)

- **`await use(user)` возвращает Promise**, который Playwright резолвит **только после окончания теста**. `await` заставляет фикстуру ждать этот Promise → строки ниже (teardown) не выполняются, пока тест идёт.
- **`yield user`** в pytest работает по тому же принципу через генератор: pytest забирает значение по `yield`, отдаёт в тест, а когда тест завершился — возобновляет генератор с места после `yield`, выполняя teardown.
- **При падении теста** teardown всё равно выполнится — фреймворк снимает паузу в любом случае (как `finally`), поэтому уборка гарантирована и данные не остаются в системе.

## Аналогия

`use` / `yield` — как «одолжить вещь»:
```
создал → одолжил ⏸ [тест пользуется] → забрал обратно → выбросил
```
Нельзя выбросить (`delete`) до того, как забрал обратно, а «забрал» происходит только когда тест отпустил вещь.

## Порядок при нескольких фикстурах (teardown в обратном порядке)

Если фикстуры зависят друг от друга, setup идёт по цепочке зависимостей, а **teardown — в обратном порядке** (как стек):

```
setup:     api → seededUser → [тест] 
teardown:  seededUser → api
```

Сначала удаляется пользователь (последний созданный), потом закрывается api-контекст — «что создано последним, убирается первым».

## Ответ для собеса

> «`await use(user)` (в pytest — `yield user`) — это точка приостановки фикстуры. Код до неё выполняется как setup до теста, потом фикстура замирает, отдаёт объект в тест и ждёт его завершения. Удаление стоит за `await use()`, поэтому выполнится только после того, как тест полностью прошёл. Даже при падении теста фреймворк снимает паузу, так что teardown отработает и данные всегда чистятся. Это прямой аналог `yield` в pytest, а при нескольких фикстурах teardown идёт в обратном порядке — как стек».

---

# Сужение скоупа локатора и поиск по атрибутам

> Главный приём против дублей: искать элемент не по всей странице, а внутри родительского контейнера. Плюс — как читать CSS-поиск по атрибутам (`button[type="submit"]` и т.п.).

## `.locator()` на локаторе = сужение скоупа

Когда вызываешь `.locator()` **на другом локаторе** (а не на `page`), поиск ограничивается **поддеревом DOM этого локатора**:

```typescript
page.locator('button[type="submit"]')       // ищет по ВСЕЙ странице
this.form.locator('button[type="submit"]')  // ищет ТОЛЬКО внутри this.form
```

Пример в Page Object:
```typescript
class PaymentPage {
  private form: Locator;
  private payButton: Locator;

  constructor(private page: Page) {
    this.form = page.getByTestId('payment-form');                 // родитель
    this.payButton = this.form.locator('button[type="submit"]');  // кнопка ВНУТРИ формы
  }

  async pay() {
    await this.payButton.click();
  }
}
```

**Зачем:** на странице может быть несколько кнопок `submit` (форма оплаты, поиск, модалка). Поиск по всей странице найдёт несколько → **strict mode violation**. Сужение по `this.form` даёт **ровно одну** — правильное решение дублей **без индексов и `.first()`**.

**Ленивость:** `this.form.locator(...)` в конструкторе ничего не ищет сразу — это по-прежнему **рецепт**. Реальный поиск (и формы, и кнопки внутри) произойдёт при действии (`payButton.click()`). Поэтому в конструкторе так писать безопасно, даже если формы ещё нет в DOM.

## Приоритет: сужение по родителю vs альтернативы

- ✅ **сужение по стабильному родителю** — `parent.locator(child)` / `parent.getByRole(...)`;
- ✅ **`filter()`** — отфильтровать нужный из списка (`.filter({ hasText })`);
- ❌ **`.first()` / `.nth()` / индекс** — антипаттерн (сегодня первый, завтра второй);
- ❌ **привязка к тексту** на мультиязычном UI.

## Поиск по атрибутам (CSS)

Синтаксис `[атрибут="значение"]` — CSS-поиск по атрибуту:
```typescript
page.locator('button[type="submit"]')   // кнопка с type=submit
page.locator('input[name="email"]')     // поле с name=email
page.locator('[data-testid="pay"]')      // любой элемент с data-testid=pay
page.locator('a[href="/plans"]')         // ссылка с конкретным href
```

## Что такое `type="submit"` (частый вопрос)

`type` — атрибут кнопки/`<input>`, задаёт поведение:
- **`type="submit"`** — при клике **отправляет форму** (кнопка «Оплатить/Войти»);
- **`type="reset"`** — сбрасывает поля формы;
- **`type="button"`** — обычная кнопка, действие только через JS.

Нюанс: если у `<button>` внутри формы `type` не указан — по умолчанию это **`submit`**. Поэтому `button[type="submit"]` ищет именно кнопку отправки формы.

## Ответ для собеса

> «`this.form.locator('button[type="submit"]')` — сужение скоупа: кнопка ищется не по всей странице, а внутри локатора формы. Так решаются дубли — когда submit-кнопок несколько, поиск по родителю даёт ровно одну, без индексов и `.first()`. `button[type="submit"]` — это CSS-поиск по атрибуту: кнопка с `type=submit`, то есть кнопка отправки формы (у `<button>` type бывает submit/reset/button). И это ленивый локатор — реальный поиск при действии, а не при создании».
