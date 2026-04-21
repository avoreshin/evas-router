# evas-router

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Packagist](https://img.shields.io/badge/packagist-evas--php%2Fevas--router-blue)](https://packagist.org/packages/evas-php/evas-router)
[![Version](https://img.shields.io/badge/version-1.3-brightgreen)](composer.json)

PHP-маршрутизатор из экосистемы **evas-php**. Поддерживает явный маппинг маршрутов (Map routing), автоматическую маршрутизацию по файлам, классам, методам класса и кастомной функции, middleware-цепочки, вложенные группы маршрутов, алиасы путей и REST-синтаксис.

Подробное руководство: [router.evas-php.com](https://router.evas-php.com/)

---

## Быстрый старт

### 1. Установите пакет

```bash
composer require evas-php/evas-router
```

### 2. Создайте роутер и обработайте запрос

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use Evas\Router\Router;

$router = (new Router)
    ->viewsDir(__DIR__ . '/views/')
    ->default('404.php')
    ->get('/', function () {
        return 'Hello, world!';
    })
    ->get('/users/(:id)', function (int $id) {
        return "user $id";
    });

$result = $router->routing($_SERVER['REQUEST_URI'], $_SERVER['REQUEST_METHOD']);
echo $result->returned;
```

Этого достаточно, чтобы получить работающее приложение с двумя маршрутами и fallback-страницей.

---

## Содержание

- [Возможности](#возможности)
- [Требования](#требования)
- [Установка](#установка)
- [Использование](#использование)
  - [Маршруты и REST-методы](#маршруты-и-rest-методы)
  - [Алиасы путей](#алиасы-путей)
  - [Middleware](#middleware)
  - [Группы и вложенные роутеры](#группы-и-вложенные-роутеры)
  - [Авто-роутинг по файлам](#авто-роутинг-по-файлам)
  - [Авто-роутинг по классам](#авто-роутинг-по-классам)
  - [Авто-роутинг по методам класса](#авто-роутинг-по-методам-класса)
  - [Авто-роутинг по кастомной функции](#авто-роутинг-по-кастомной-функции)
  - [Fallback и обработка 404](#fallback-и-обработка-404)
- [Архитектура](#архитектура)
- [Справочник API](#справочник-api)
- [Константы](#константы)
- [Тестирование](#тестирование)
- [Наблюдения по коду](#наблюдения-по-коду)
- [Лицензия](#лицензия)
- [Автор](#автор)

---

## Возможности

- **Map routing** — явный маппинг `method → path → handler`.
- **Auto routing** — четыре стратегии автоматической генерации обработчика: по файлу, по классу, по методу класса, по пользовательской функции.
- **Middleware** — цепочки, наследуемые вверх по дереву вложенных роутеров.
- **Группы маршрутов** — вложенные роутеры с отложенной сборкой (lazy).
- **Алиасы путей** — встроенные `:any`, `:int`, `:id` плюс пользовательские.
- **REST-синтаксис** — методы `get()`, `post()`, `put()`, `delete()`, `patch()`, `options()`, `all()` через магический `__call`.
- **Кастомные контроллеры и view-директория** — наследуются вниз по группам.

---

## Требования

Из `composer.json`:

```json
"require": {
    "evas-php/evas-base": "*",
    "evas-php/evas-http": "*"
}
```

- PHP ≥ 7.1 (typed properties и nullable return types в коде).
- [`evas-php/evas-base`](https://github.com/evas-php/evas-base) — используются `App`, `PhpHelp`, `FileNotFoundException`.
- [`evas-php/evas-http`](https://github.com/evas-php/evas-http) — используются `HttpRequest`, `RequestInterface`.

---

## Установка

### Composer (рекомендуется)

```bash
composer require evas-php/evas-router
```

### Вручную

```bash
git clone https://github.com/evas-php/evas-router.git
```

Зарегистрируйте PSR-4 namespace `Evas\Router` на директорию `src/` в `composer.json` проекта:

```json
{
    "autoload": {
        "psr-4": {
            "Evas\\Router\\": "path/to/evas-router/src/"
        }
    }
}
```

Либо через Codeception-autoloader (пример из `tests/RouterTest.php:22-23`):

```php
use Codeception\Util\Autoload;

Autoload::addNamespace('Evas\\Router', 'vendor/evas-php/evas-router/src');
```

Зависимости `evas-php/evas-base` и `evas-php/evas-http` необходимо установить тем же способом.

---

## Использование

### Маршруты и REST-методы

```php
$router
    ->get('/items', $listHandler)
    ->post('/items', $createHandler)
    ->put('/items/(:id)', $updateHandler)
    ->delete('/items/(:id)', $deleteHandler)
    ->all('/ping', fn() => 'pong');
```

Поддерживаемые методы (константа `EVAS_ROUTER_REST_METHODS`): `GET`, `POST`, `PUT`, `DELETE`, `PATCH`, `OPTIONS` + псевдо-метод `all`.

В качестве обработчика принимаются `string` (путь к view-файлу), `\Closure` или `array` вида `['Class' => 'method']`.

### Алиасы путей

Встроенные:

| Алиас | Regex |
|-------|-------|
| `:any` | `.*` |
| `:int` | `[0-9]{1,}` |
| `:id`  | `[1-9]+\d*` |

Добавить свой:

```php
$router->alias(':slug', '[a-z0-9-]+');
$router->get('/posts/(:slug)', function (string $slug) { /* ... */ });
```

### Middleware

Middleware получает тот же формат, что и обработчики (`string | Closure | ['Class' => 'method']`). Если middleware возвращает `false` — цепочка прерывается и выполняется fallback (`default`).

```php
use App\Middlewares\Access;

$router
    ->middleware([Access::class => 'isLogin'])
    ->get('/profile', $profileHandler);
```

Middleware собираются **каскадом вверх** по дереву вложенных роутеров; глубина ограничена `EVAS_ROUTER_MIDDLEWARES_DEEP_LIMIT` (по умолчанию 5).

### Группы и вложенные роутеры

```php
$router->map('/admin/', function () {
    $this->middleware([Access::class => 'isAdmin'])
        ->get('', fn() => 'admin panel')
        ->autoByMethod('list/', function () {
            $this->classCustom(ListController::class)
                ->methodPostfix('List');
        });
});
```

Третий аргумент — либо HTTP-метод/массив методов, либо сам `callable` (тогда метод выставляется в `'all'`).

Доступные группирующие методы: `map()`, `autoByFile()`, `autoByClass()`, `autoByMethod()`, `autoByFunc()`, общий `bindChild()`.

### Авто-роутинг по файлам

```php
$router->autoByFile('/', function () {
    $this->filePrefix(__DIR__ . '/views/')
        ->filePostfix('.php');
});
```

Правила преобразования пути (`AutoRouterByFile::generateHandler`):
- пустой путь → `/index`;
- путь, заканчивающийся на `/` → добавляется `index`;
- итог: `filePrefix . path . filePostfix`.

### Авто-роутинг по классам

```php
$router->autoByClass('/auth/', 'POST', function () {
    $this->classPrefix('App\\Auth\\')
        ->classPostfix('Action')
        ->classMethod('auto');
});
```

`POST /auth/login` → `App\Auth\LoginAction::auto()`.

### Авто-роутинг по методам класса

```php
$router->autoByMethod('/profile/', function () {
    $this->classCustom(ProfileController::class)
        ->methodPostfix('Action');
});
```

`GET /profile/edit` → `ProfileController::editAction()`. Без `classCustom` последний сегмент пути трактуется как метод, остальные — как имя класса (`\`-склейка).

Автоматическое разрешение класса и метода из пути без явного `classCustom`:

```php
$router->autoByMethod('/api/v1', function () {
    $this->classPrefix('Controllers\\Api\\')
        ->classPostfix('Controller')
        ->methodPostfix('Action');
});
```

`GET /api/v1/user/list` → `Controllers\Api\UserController::listAction()`.

### Авто-роутинг по кастомной функции

```php
$router->autoByFunc('/', function () {
    $this->routingFunc(function (string $path) {
        return ['App\\Dispatcher' => 'dispatch'];
    });
});
```

### Fallback и обработка 404

```php
$router->default('404.php');
```

`default` вызывается при отсутствии совпадения **и** при `RouterResultException` (включая возврат `false` из middleware/обработчика). Без `default` корневой `routing()` бросает `RouterException('404. Not Found')`.

---

## Архитектура

```
Router (final)
  └── Routers\MapRouter                          implements RouterInterface
        └── Routers\AbstractAutoRouter (abstract)
              ├── Routers\AutoRouterByFile
              ├── Routers\AutoRouterByClass
              ├── Routers\AutoRouterByClassMethod
              └── Routers\AutoRouterByFunc

Routers\NestedRouterWrap   — обёртка отложенной сборки вложенного роутера
RouterResult               — объект результата роутинга
Controller                 — базовый контроллер для view-файлов и Closure
```

`MapRouter` подмешивает шесть трейтов: `RouterRoutesTrait`, `RouterAliasesTrait`, `RouterMiddlewaresTrait`, `RouterControllerTrait`, `RouterGroupTrait`, `RouterRestTrait`.

Поток выполнения `routing()` (`src/Routers/MapRouter.php:122`):

1. Метод по умолчанию `GET`; если не установлен `request` — создаётся `HttpRequest`.
2. `mapRouting()` сливает маршруты метода с `ALL` и сопоставляет их regex-ом из `preparePath()`.
3. При совпадении: если handler — `NestedRouterWrap`, вызывается `build()` и рекурсивный `routing()` на собранном роутере; иначе — `newResult()`.
4. Если маппинг пуст и есть `autoRouting()` (у наследников `AbstractAutoRouter`) — выполняется авто-роутинг.
5. Если всё пусто — `default`, иначе у корневого `RouterException('404. Not Found')`, у вложенного — `null`.

---

## Справочник API

Все сигнатуры ниже взяты из исходников без изменений.

### `Evas\Router\Router`

`final class Router extends MapRouter`. Точка входа без собственных членов (`src/Router.php`).

### `Evas\Router\Routers\MapRouter`

| Сигнатура | Описание |
|-----------|----------|
| `__construct(RouterInterface &$parent = null)` | При наличии родителя наследует `aliases`, `controllerClass`, `viewsDir`. |
| `default($handler): RouterInterface` | Обработчик по умолчанию. |
| `newResult($handler, array $args = null): RouterResultInterface` | Создаёт `RouterResult`, прокидывает `controllerClass`/`request`/`viewsDir` и вызывает `resolve()`. |
| `routing(string $path, string $method = null, array $args = null): ?RouterResultInterface` | Основной метод. |
| `requestRouting(RequestInterface $request): ?RouterResultInterface` | Сокращение: берёт `path`/`method` из запроса. |
| `protected mapRouting(string $path, string $method, array $args = null): ?RouterResultInterface` | Поиск маршрута в маппинге. |

Поля: `protected $parent`, `protected $default`.

### `Evas\Router\Routers\AbstractAutoRouter`

| Сигнатура | Описание |
|-----------|----------|
| `abstract generateHandler(string $path)` | Возвращает обработчик, сгенерированный по пути. |
| `autoRouting(string $path, array $args = null): ?RouterResultInterface` | Вызывает `generateHandler` → `newResult`. При `RouterResultException` — `null`. |

### `Evas\Router\Routers\AutoRouterByFile`

| Член | Значение/сигнатура |
|------|--------------------|
| `public $filePrefix` | `EVAS_AUTOROUTER_FILE_PREFIX` (`''`). |
| `public $filePostfix` | `EVAS_AUTOROUTER_FILE_POSTFIX` (`'.php'`). |
| `filePrefix(string $value = null)` | Сеттер. |
| `filePostfix(string $value = null)` | Сеттер. |
| `generateHandler(string $path): string` | См. [Авто-роутинг по файлам](#авто-роутинг-по-файлам). |

### `Evas\Router\Routers\AutoRouterByClass`

| Член | Значение/сигнатура |
|------|--------------------|
| `public $classMethod` | `EVAS_AUTOROUTER_CLASS_METHOD` (`'auto'`). |
| `public $classPrefix` | `EVAS_AUTOROUTER_CLASS_PREFIX` (`''`). |
| `public $classPostfix` | `EVAS_AUTOROUTER_CLASS_POSTFIX` (`''`). |
| `classMethod / classPrefix / classPostfix(string $value = null)` | Сеттеры. |
| `generateHandler(string $path): array` | Сегменты пути → `ucfirst`, склейка через `\\`, оборачивание в prefix/postfix; возвращает `[$className => $classMethod]`. |

### `Evas\Router\Routers\AutoRouterByClassMethod`

| Член | Значение/сигнатура |
|------|--------------------|
| `public $classPrefix / $classPostfix` | `''`. |
| `public $classCustom` | `null` (фиксированный класс; тогда все сегменты пути → метод). |
| `public $methodPrefix / $methodPostfix` | `''`. |
| `public $useIndexClass` | `false` (использовать класс `Index` для пустого пути). |
| `classPrefix / classPostfix / classCustom / methodPrefix / methodPostfix(string $value = null)` | Сеттеры. |
| `useIndexClass(bool $use = true)` | Сеттер. |
| `generateHandler(string $path): array` | См. [Авто-роутинг по методам класса](#авто-роутинг-по-методам-класса). |

### `Evas\Router\Routers\AutoRouterByFunc`

| Сигнатура | Описание |
|-----------|----------|
| `routingFunc(callable $callback): AutoRouterByFunc` | Устанавливает функцию генерации. |
| `generateHandler(string $path)` | `call_user_func($this->routingFunc, $path)`. Без колбэка — `RouterException('Undefined autorouting function')`. |

### `Evas\Router\Routers\NestedRouterWrap`

| Сигнатура | Описание |
|-----------|----------|
| `__construct(\Closure &$callback, RouterInterface &$router)` | Сохраняет колбэк и будущий вложенный роутер. |
| `build()` | `($callback)->bindTo($router)()`; возвращает собранный роутер. |

### `Evas\Router\RouterResult`

Публичные поля: `$handler`, `$args`, `$middlewares`, `$returned`.

| Сигнатура | Описание |
|-----------|----------|
| `__construct($handler = null, array $args = null, array $middlewares = null)` | Всё по ссылке. |
| `controllerClass(string $controllerClass): RouterResult` | Локальное переопределение метода трейта. |
| `newController(string $controllerClass = null): object` | `new $controllerClass($this->request, $this->viewsDir)`; кэшируется. Класс не найден → `FileNotFoundException`. |
| `prepare(): self` | Формирует `preparedHandlers` (сначала middleware, затем handler). Ошибки → `RouterResultException`. |
| `resolve()` | Последовательный вызов всех подготовленных обработчиков. `false` → `RouterResultException('Route handler returned false')`. Результат последнего вызова → `$returned`. |

### `Evas\Router\Controller`

Поля: `public $viewsDir = EVAS_VIEWS_DIR` (`'views/'`), `public $request`.

| Сигнатура | Описание |
|-----------|----------|
| `__construct(RequestInterface &$request, string $viewsDir = null)` | Если у наследника есть `_before()` — вызывается. |
| `resolveViewPath(string $filename): string` | Разрешает абсолютный путь через `App::*`. |
| `view(string $filename, array $args = null, object &$context = null)` | Инклюд файла; контекст по умолчанию — сам контроллер. |
| `canView(string $filename): bool` | Проверка через `App::canInclude`. |
| `throwIfNotCanView(string $filename)` | Делегат в `App::throwIfNotCanInclude`. |

### Трейты

**`RouterRoutesTrait`** — `route()`, `mergeRoute()`, `getRoutes()`, `getRoutesByMethodWithAll()`, `isCorrectHandlerType()`.

**`RouterAliasesTrait`** — `alias()`, `aliases()`, `getAliases()`, `applyAliases()`, `preparePath()`.

**`RouterMiddlewaresTrait`** — `middleware(...)`, `getMiddlewares()` (каскад вверх до `EVAS_ROUTER_MIDDLEWARES_DEEP_LIMIT`).

**`RouterControllerTrait`** — `controllerClass()`, `getControllerClass()`, `withRequest()`, `viewsDir()`, `getViewsDir()`.

**`RouterGroupTrait`** — `map()`, `autoByFile()`, `autoByFunc()`, `autoByClass()`, `autoByMethod()`, `bindChild()`.

**`RouterRestTrait`** — `static getRestMethods()`, `static isSupportRestMethod()`, магический `__call()` для `get/post/put/delete/patch/options/all`.

### Интерфейсы

- `Interfaces\RouterInterface` — контракт `MapRouter` (конструктор, `default`, `routing`, `requestRouting`, методы трейтов routes/aliases/middleware).
- `Interfaces\RouterResultInterface` — только `__construct($handler, $args, $middlewares)`.
- `Interfaces\ControllerInterface` — `view()`, `throwIfNotCanView()`.

### Исключения

- `Exceptions\RouterException extends \Exception`
- `Exceptions\RouterResultException extends RouterException`

Выбрасываются в случаях: отсутствие маршрута без `default`, превышение глубины middleware, незаданный колбэк `AutoRouterByFunc`, несуществующий контроллер/класс/метод, возврат `false` из обработчика.

---

## Константы

Все переопределяемы до подключения класса (определяются через `if (!defined(...))`).

| Константа | По умолчанию |
|-----------|--------------|
| `EVAS_VIEWS_DIR` | `'views/'` |
| `EVAS_CONTROLLER_CLASS` | `Controller::class` |
| `EVAS_AUTOROUTER_FILE_PREFIX` | `''` |
| `EVAS_AUTOROUTER_FILE_POSTFIX` | `'.php'` |
| `EVAS_AUTOROUTER_CLASS_METHOD` | `'auto'` |
| `EVAS_AUTOROUTER_CLASS_PREFIX` | `''` |
| `EVAS_AUTOROUTER_CLASS_POSTFIX` | `''` |
| `EVAS_AUTOROUTER_METHOD_PREFIX` | `''` |
| `EVAS_AUTOROUTER_METHOD_POSTFIX` | `''` |
| `EVAS_ROUTER_REST_METHODS` | `['GET','POST','PUT','DELETE','PATCH','OPTIONS']` |
| `EVAS_ROUTER_MIDDLEWARES_DEEP_LIMIT` | `5` |

---

## Тестирование

Тесты в `tests/RouterTest.php` написаны на Codeception (`\Codeception\Test\Unit`). Сам пакет Codeception не декларирует — установите в проекте-хосте как dev-зависимость:

```bash
composer require --dev codeception/codeception
vendor/bin/codecept run
```

Вспомогательные фикстуры — `tests/help/` (контроллеры, middlewares, модели, страницы).

---

## Наблюдения по коду

Эти факты обнаружены при анализе исходников — для ясности, а не как критика:

- В `tests/RouterTest.php:72` используется `->viewDir(...)`, фактический метод — `viewsDir()` (`src/Traits/RouterControllerTrait.php:65`). Вызов `viewDir` попадёт в `RouterRestTrait::__call` и бросит `BadMethodCallException`.
- В `src/Traits/RouterMiddlewaresTrait.php:31` присутствует отладочный `var_dump($middlewares);`, выводящийся на каждый вызов `middleware(...)`.
- `RouterResultInterface::resolve()` в файле закомментирован — интерфейс гарантирует только конструктор.
- В `composer.json` зависимости объявлены как `*` — для воспроизводимой сборки зафиксируйте версии в проекте-хосте.

---

## Лицензия

[CC-BY-4.0](LICENSE)

## Автор

Egor Vasyakin — <egor@evas-php.com>
