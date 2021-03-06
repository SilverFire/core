# Core

**Core**

[![Latest Stable Version](https://poser.pugx.org/hiqsol/core/v/stable)](https://packagist.org/packages/hiqsol/core)
[![Total Downloads](https://poser.pugx.org/hiqsol/core/downloads)](https://packagist.org/packages/hiqsol/core)
[![Build Status](https://img.shields.io/travis/hiqsol/core.svg)](https://travis-ci.org/hiqsol/core)
[![Scrutinizer Code Coverage](https://img.shields.io/scrutinizer/coverage/g/hiqsol/core.svg)](https://scrutinizer-ci.com/g/hiqsol/core/)
[![Scrutinizer Code Quality](https://img.shields.io/scrutinizer/g/hiqsol/core.svg)](https://scrutinizer-ci.com/g/hiqsol/core/)
[![Dependency Status](https://www.versioneye.com/php/hiqsol:core/dev-master/badge.svg)](https://www.versioneye.com/php/hiqsol:core/dev-master)

## Installation

The preferred way to install this library is through [composer](http://getcomposer.org/download/).

Either run

```sh
php composer.phar require "hiqsol/core"
```

or add

```json
"hiqsol/core": "*"
```

to the require section of your composer.json.

## Idea

- разделён на части, НЕ радикально, а конструктивно:
    - yiisoft/core - всё кроме выделенных нижеперечисленных частей
    - yiisoft/di - работает как есть без переделки, нашёл пару багов
    - yiisfot/web - всё из папки web
    - yiisfot/console - всё из папки console, но надо ещё смотреть, не делал ещё
    - yiisoft/log - всё из папки log, с прицелом на заменяемость любым PSR логгером
    - yiisoft/cache - всё из папки cache, с прицелом на заменяемость любым PSR кешом
    - что ещё?
- в каждой части своя конфигурация, для понимания см. примеры ниже
    - собирается с помощью composer-config-plugin, можно подумать о другом
      собирателе конфигов, но этот уже оттесченый
    - понимаю, спорный момент с первого взгляда, но на самом делал это может
      стать главной фишкой фреймфорка
    - выкидываются нахер всякие coreComponents
      и костыли в виде мержа кусков конфига раскиданные по коду.
    - конфиг превращается из конфига приложения в конфиг контейнера и в нём уже
      есть конфиг приложения и сервисов.
- DI
    - выпиливается `ServiceLocator`, везде юзается DI
    - выпиливаются компоненты из `Application` и модулей
        - из моего опыта DI вполне достаточно
    - `Application` сииильно сокращается
- выпиливается `Configurable` и `Yii::configure(),` с ними выпиливается `init()`
    - Configurable объекты работали так:
        - в конструктор передаётся массив конфиг
        - конфиг применяется с помощью `Yii::configure()`
        - конструктор вызывает `init()`
        - это всё не работает с новым DI так как он:
            - вызывает конструктор, а потом уже устанавливает свойства
            - теоретически можно вызывать `init()` через конфиг DI, но надо заботится чтоб он вызывался после свойств
    - нужно фиксить все классы где есть `init()`
    - это серьёезный BC-break, но всё становится понятнее
    - не нужно пилить yiisoft/di под `Configurable`
- Yii становится чистым хелпером без статических глобальных свойств
    - НЕ НУЖНО реквайрить Yii в entry script'е
    - переносится к остальным хелперам -> `yii\helpers\Yii`
    - выпиливаются "глобальные переменные" `Yii::$app`, `Yii::$logger` и др.
    - алиасы переносятся в `Application`
    - остаются: `Yii::t()`, `Yii::createObject()` + функции логгинга и профайлинга
    - стопку define'ов перенёс в `yii\base\Application`
    - XXX: не придумал как лучше: чтоб оно работало надо делать `Yii::setContainer($container)`
- cleanup
    - папку web заменить на public и алиас `@webroot` заменить на `@public` ?
    - хочу сделать меньше файлов в нагруженых папках (base, web)- чуть больше папок, но не выращивая глубину
        - эксепшены в папку exceptions и в base и в web
        - web/formatters
        - подумать: url, action

Entry script будет такой:

```php
<?php
use hiqdev\composer\config\Builder;
use yii\di\Container;
use yii\helpers\Yii;

(function () {
    require_once __DIR__ . '/../vendor/autoload.php';
    require_once Builder::path('defines');

    $container = new Container(require Builder::path('web'));
    Yii::setContainer($container);

    $container->get('application')->run();
})();
```

Часть конфига из `yiisoft/core`:

```php
<?php

return [
    \yii\base\Application::class => Reference::to('application'),
    'application' => [
        'aliases' => [
            '@root'     => dirname(__DIR__, 5),
            '@vendor'   => dirname(__DIR__, 4),
        ],
        'params' => $params,
    ],

    \yii\base\Request::class => Reference::to('request'),
    \yii\base\View::class => Reference::to('view'),
];
```

Часть конфига из `yiisoft/web`:

```php
<?php

return [
    'application' => [
        '__class' => \yii\web\Application::class,
        'id' => 'web',
        'name' => 'web',
    ],

    'request' => [
        '__class' => \yii\web\Request::class,
    ],
    'view' => [
        '__class' => \yii\web\View::class,
    ],
];
```

Часть конфига из `yiisoft/console`:

```php
<?php

return [
    'application' => [
        '__class' => \yii\console\Application::class,
        'id' => 'console',
        'name' => 'console',
    ],

    'request' => [
        '__class' => \yii\console\Request::class,
    ],
    'view' => [
        '__class' => \yii\console\View::class,
    ],
];
```

Часть конфига из `yiisoft/log`:

```php
<?php

return [
    'logger' => [
        '__class' => \yii\log\Logger::class,
    ],

    \Psr\Log\LoggerInterface::class => \yii\di\Reference::to('logger'),
];
```

## License

This project is released under the terms of the BSD-3-Clause [license](LICENSE).
Read more [here](http://choosealicense.com/licenses/bsd-3-clause).

Copyright © 2018, sol (http://hiqdev.com/)
