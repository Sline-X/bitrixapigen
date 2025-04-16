# Bitrixapigen ❤️ ContractFirst

__Bitrixapigen__ — пакет для генерации серверной части приложения (контроллеры + дто + роутер) на основе OpenApi контракта на битриксе.

---

## ⚙️ Установка

*composer require webpractik/bitrixapigen --dev*

---

## 🔧 Настройка роутинга Bitrix D7

Если на проекте еще настроен роутинг, то сделайте это.

### Шаг 1: Настройте роутинг по документации
▶️ Официальная документация Bitrix:  
[https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&CHAPTER_ID=013764](https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=43&CHAPTER_ID=013764)

### Шаг 2: *local/routes/api.php*

Обратите внимание на подключение файлов routes.php. Это кастомный файл, который будет присутствовать в модуле сгенерированном пакетом,
поэтому нужно реализовать подключение роутов из routes.php.

```php
<?php

use Bitrix\Main\ModuleManager;
use Bitrix\Main\Routing\RoutingConfigurator;

require_once $_SERVER['DOCUMENT_ROOT'] . '/../vendor/autoload.php';

$getRoutePaths = static function (): array {
    foreach (ModuleManager::getInstalledModules() as $module) {
        $route = $_SERVER['DOCUMENT_ROOT'] . '/local/modules/' . $module['ID'] . '/routes.php';
        if (file_exists($route)) {
            $routes[] = $route;
        }
    }

    return $routes ?? [];
};

return static function (RoutingConfigurator $configurator) use ($getRoutePaths) {
    foreach ($getRoutePaths() as $route) {
        $callback = include $route;
        if ($callback instanceof Closure) {
            $callback($configurator);
        }
    }
};
```

### Шаг 3: Проверьте, что файл `local/routes/api.php` подключен в `.settings.php`

```php
'routing' => [
    'value' => [
        'config' => [
             'api.php',
        ],
    ],
],
```

---

## 🚀 Быстрый старт

1. Подготовьте OpenAPI-спецификацию (JSON или YAML)
> 📝 Все данные передаваемые в телах запросов должны быть описаны через схемы (`schema`) в OpenAPI-спецификации. Именно на их основе происходит генерация соответствующих DTO, коллекций и корректная передача аргументов в UseCase.
2. Выполните генерацию:

*php vendor/bin/bitrixapigen generate --openapi-file path/to/openapi.json*  
или кратко:  
*php vendor/bin/bitrixapigen generate -o path/to/openapi.json*

> 🟡 Параметр *--openapi-file* (или *-o*) — **обязателен**

3. Установите модуль:
    - через административную панель Bitrix (`/bitrix/admin/module_admin.php`)
    - или через миграцию/скрипт

---

## 📁 Структура сгенерированного модуля

> ⚠️ После генерации необходимо обязательно подключить модуль `webpractik.bitrixgen` в самом конце файла `local/php_interface/init.php`, чтобы он корректно зарегистрировал свои контроллеры и роуты.

```
local/modules/webpractik.bitrixgen/
├── lib/
│   ├── Controllers/
│   ├── Dto/
│   │   └── Collection/
│   ├── Exception/
│   ├── Interfaces/            ← интерфейсы, которые имплементируются в UseCase-классах (один роут - один интерфейс)
│   ├── Response/
│   └── UseCase/               ← UseCase-классы (один роут - один интерфейс - UseCase класс)
├── .settings.php              ← регистрация сервисов в DI Bitrix
├── include.php                ← точка подключения модуля
├── routes.php                 ← кастомный файл с роутами
```

---

## 🧠 Архитектура

Для каждого роута генерируется интерфейс, например, `Interfaces/IUploadPetFormWithFiles.php` и
класс-заглушка `UseCase/UploadPetFormWithFiles.php`, который реализует интерфейс `IUploadPetFormWithFiles`.

В файле `.settings.php` UploadPetFormWithFiles регистрируется как реализация для интерфейса.

Контроллеры получают входные данные, инициализируют DTO, коллекции или другие переменные и вызывают реализацию интерфейса UseCase через `ServiceLocator`:

```php
$useCase = \Bitrix\Main\DI\ServiceLocator::getInstance()->get('webpractik.bitrixgen.uploadPetFormWithFiles');
return new \Bitrix\Main\Engine\Response\Json(
    $useCase->process($petId, $dto)
);
```
Модуль `webpractik.bitrixgen` **не должен редактироваться вручную**. Любые правки в нем будут утеряны после перегенерации этого модуля.

---

## 🧩 Как реализовать функционал роутов

Предполагается, что вся логика роута будет размещена в соответствующем UseCase.

1. Создайте свой модуль, например, my.module

2. Создайте в нем реализацию интерфейса соответствующего роута:

```php
namespace My\Module\UseCase;

use Webpractik\Bitrixgen\Interfaces\IUploadPetFormWithFiles;

class UploadPetFormWithFiles implements IUploadPetFormWithFiles
{
    public function process(int $petId, \Webpractik\Bitrixgen\Dto\PetFormUpload $dto): ?\Webpractik\Bitrixgen\Dto\Pet
    {
        return new \Webpractik\Bitrixgen\Dto\Pet();
    }
}
```

2. Зарегистрируйте реализацию в `.settings.php` модуля `my.module`:

```php
<?php

namespace Webpractik\Bitrixgen;

use Bitrix\Main\DI\ServiceLocator;

$serviceLocator = ServiceLocator::getInstance();
$serviceValue = [];

if ($serviceLocator->has('webpractik.bitrixgen.uploadPetFormWithFiles')) {
    if (!in_array(\Webpractik\Bitrixgen\Interfaces\IUploadPetFormWithFiles::class, class_implements($serviceLocator->get('webpractik.bitrixgen.uploadPetFormWithFiles')))) {
        $serviceValue['webpractik.bitrixgen.uploadPetFormWithFiles'] = [\My\Module\UseCase\UploadPetFormWithFiles::class];
    }
}

if (!$serviceLocator->has('webpractik.bitrixgen.uploadPetFormWithFiles')) {
    $serviceValue['webpractik.bitrixgen.uploadPetFormWithFiles'] = ['className' => \My\Module\UseCase\UploadPetFormWithFiles::class];
}

return ['services' => ['value' => $serviceValue]];
```

4. Подключите свой модуль `my.module` в файле `local/php_interface/init.php` перед подключением модуля `webpractik.bitrixgen`:

```php
\Bitrix\Main\Loader::includeModule('my.module');

\Bitrix\Main\Loader::includeModule('webpractik.bitrixgen'); // строго в конце!
```

---

## 📥 Что передаёт контроллер в UseCase

Контроллер автоматически передаёт в метод `process()`:

- **`$dto` или `$collection`** — если `requestBody` с типом `application/json` или `multipart/form-data`
- **`string $octetStreamRawContent`** — если тип `requestBody` — `application/octet-stream`
- **`array $queryParameters`** — если в OpenAPI-спецификации заданы query-параметры
- **path-параметры** — передаются как отдельные переменные (например, `int $petId`)

---

## 📤 Формат ответа: JSON и Bitrix-структура

По умолчанию роуты возвращают **"чистый" JSON** без обёртки Bitrix.

Чтобы включить формат Bitrix, в OpenAPI-документации у роута нужно задать флаг:

```yaml
x-bitrix-format: true
```

### Пример

```yaml
/pet/findByStatus:
  get:
    x-bitrix-format: true
    tags:
      - pet
```

### Поведение:

- Без флага:

```json
{ "id": 123, "name": "doggie" }
```

- С `x-bitrix-format: true`:

```json
{ "status": "success", "data": { "id": 123, "name": "doggie" } }
```

Ошибки также будут обёрнуты в формат Bitrix.

---

## 🛠 Требования

- PHP 8.1+
- Bitrix Framework (D7)
- OpenAPI 3.0+

---

## Roadmap

- [ ] Генерация валидаций
- [ ] Генерация ошибок и логика их обработки в контроллере=
- [ ] Генерация тестов
- [ ] Авторизация в роутах

