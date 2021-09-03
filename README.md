# Sunflower

Sunflower is a super small application kernel that is used to build a dependency
injection container. This kernel is useful for microservices and applications that
dont use HTTP. Say; reading from a queue or application invoked by AWS Lambda.

With this kernel you can use normal Symfony service definition with auto wiring
and all!

## Install

```
composer require nyholm/sunflower
```

## Use

```php
// src/Kernel.php
namespace App;

use Nyholm\SunflowerKernel;

class Kernel extends SunflowerKernel
{
   /**
    * Optionally override the configureContainer()
    */
   protected function configureContainer(ContainerConfigurator $container): void
    {
        $container->import('../config/{packages}/*.yaml');
        $container->import('../config/{packages}/'.$this->environment.'/*.yaml');
        $container->import('../config/{packages}/'.$this->environment.'/*.php');

        if (\is_file(\dirname(__DIR__).'/config/services.yaml')) {
            $container->import('../config/services.yaml');
            $container->import('../config/{services}_'.$this->environment.'.yaml');
        } else {
            $container->import('../config/{services}.php');
        }
    }
}
```

```php

use App\Kernel;
use App\Service\MyService;

require_once dirname(__DIR__).'/vendor/autoload.php';

$kernel =  new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
$kernel->getContainer()->get(MyService::class)->handle();
```

## Use with HTTP

```php

use Nyholm\Psr7;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context) {
    $kernel =  new \App\Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);
    $container = $kernel->getContainer();

    // This is an example router
    $urlPath = $context['REQUEST_URI'];
    switch ($urlPath) {
        case '/':
        case '':
            // This is an RequestHandlerInterface
            return $container->get(\App\Controller\Startpage::class);
        case '/foobar':
            return $container->get(\App\Controller\Foobar::class);
        default:
            return new Psr7\Response(404, [], 'The route does not exist');
    }
};
```

## Use with Bref

This works perfectly with Symfony 5.3+ and the Runtime component. Read more at
https://github.com/php-runtime/bref

```php
// src/Kernel.php

namespace App;

use Nyholm\SunflowerKernel;

class Kernel extends SunflowerKernel
{
    public function isLambda(): bool
    {
        return false !== \getenv('LAMBDA_TASK_ROOT');
    }

    public function getCacheDir(): string
    {
        if ($this->isLambda()) {
            return '/tmp/cache/'.$this->environment;
        }

        return parent::getCacheDir();
    }

    public function getLogDir(): string
    {
        if ($this->isLambda()) {
            return '/tmp/log/';
        }

        return parent::getLogDir();
    }

    public function getProjectDir(): string
    {
        return \dirname(__DIR__);
    }
}
```

```php
// bin/container.php

use App\Kernel;

require_once dirname(__DIR__).'/vendor/autoload_runtime.php';

return function (array $context) {
    $kernel =  new Kernel($context['APP_ENV'], (bool) $context['APP_DEBUG']);

    return $kernel->getContainer();
};
```

```yaml
// config/services.yaml

services:
    _defaults:
        autowire: true
        autoconfigure: true

    _instanceof:
        Bref\Event\Handler:
            public: true
```
