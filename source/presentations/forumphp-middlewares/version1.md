class: main-title

### Conclusion :

# Un *middleware* est quelque chose qui prend une *requête* et retourne une *réponse*.

---
class: profile

.profile-picture[
    ![](img/profile.jpeg)
]

## Matthieu Napoli [github.com/mnapoli](https://github.com/mnapoli)

.company-logo[ [![](img/wizaplace.png)](https://wizaplace.com) ]

---

# middle-what ?

---

# Singleton

---
class: main-title

# Un *singleton* est une classe qui a *une seule instance*.

---
class: main-title

# Un *middleware* est quelque chose qui prend une *requête* et retourne une *réponse*.

---
class: title

# Concrètement

---

```php
function middleware($request) {
    return new Response('Hello');
}

$response = middleware($request);
```

---

```php
$middleware = function ($request) {
    return new Response('Hello');
}

$response = $middleware($request);
```

---

```php
class Middleware
{
    public function __invoke($request) {
        return new Response('Hello');
    }
}

$middleware = new Middleware();

$response = $middleware($request);
$response = $middleware->__invoke($request);
```

[PHP callables](http://php.net/manual/en/language.types.callable.php)

---
class: title

# PSR-7

---

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

function middleware(ServerRequestInterface $request) : ResponseInterface {
    return new Response();
}
```

---

## Zend Diactoros [github.com/zendframework/zend-diactoros](https://github.com/zendframework/zend-diactoros)

![](img/diactoros.png)

---

## Request

```php
$request = new ServerRequest(
    $_SERVER,
    $_FILES,
    new Uri(...),
    $_SERVER['REQUEST_METHOD'],
    'php://input',
    $headers,
    $_COOKIE,
    $_GET,
    $_POST,
    $_SERVER['SERVER_PROTOCOL']
);
```

```php
$request = ServerRequestFactory::fromGlobals();
```

---

## Response

```php
foreach ($response->getHeaders() as $header => $values) {
    foreach ($values as $value) {
        header("$header: $value");
    }
}
echo $response->getBody();
```

```php
(new SapiEmitter)->emit($response);
```

---
class: title

# Application

---
class: main-title

# Un *middleware* est quelque chose qui prend une *requête* et retourne une *réponse*.

---

```php
// index.php

$application = function (ServerRequestInterface $request) {
    return new TextResponse('The date is ' . date('d M Y'));
};

$response = $application(…);
…
```

---

```php
// index.php

$application = function (ServerRequestInterface $request) {
    return new TextResponse('The date is ' . date('d M Y'));
};

$response = $application(ServerRequestFactory::fromGlobals());
…
```

---

```php
// index.php

$application = function (ServerRequestInterface $request) {
    return new TextResponse('The date is ' . date('d M Y'));
};

$response = $application(ServerRequestFactory::fromGlobals());
(new SapiEmitter)->emit($response);
```

---

.browser-mockup[ The date is 28 Oct 2016 ]

---
class: title

# Error handling

---

```php
$application = function (ServerRequestInterface $request) {
    try {
        return new TextResponse('The date is ' . date('d M Y'));
    } catch (\Throwable $e) {
        return new EmptyResponse('Oops!', 500);
    }
};

$response = $application(ServerRequestFactory::fromGlobals());
(new SapiEmitter)->emit($response);
```

---

```php
$application = function (ServerRequestInterface $request) {
    return new TextResponse('The date is ' . date('d M Y'));
};

$errorHandler = function (ServerRequestInterface $request) use ($application) {
    try {
        return $application($request);
    } catch (\Throwable $e) {
        return new EmptyResponse('Oops!', 500);
    }
};

$response = $errorHandler(ServerRequestFactory::fromGlobals());
(new SapiEmitter)->emit($response);
```

---

.browser-mockup.error[ Oops! ]

---
class: title

# Force HTTPS

---

```php
$forceHttps = function (ServerRequestInterface $request) use ($application) {
    $uri = $request->getUri();
    if (strtolower($uri->getScheme()) !== 'https') {
        $uri = $uri->withScheme('https')->withPort(443);
        return new RedirectResponse($uri);
    }
    return $application($request);
};
```

---

```php
$application = function (ServerRequestInterface $request) {
    return new TextResponse('The date is ' . date('d M Y'));
};

$forceHttps = function (ServerRequestInterface $request) use ($application) {
    ...
};

$errorHandler = function (ServerRequestInterface $request) use ($forceHttps) {
    ...
};

$response = $errorHandler(ServerRequestFactory::fromGlobals());
(new SapiEmitter)->emit($response);
```

---
class: title

# Pipe

---

```ruby
$ cat /var/log/apache2/access.log \
        | grep 404 \
        | awk '{ print $7 }' \
        | sort \
        | uniq -c \
        | sort

   1 /blog/wp-content/uploads/2012/12/favicon.ico
   1 /favicon.ico
   1 /login?code=auie&state=auie
  10 /dreams/wp-content/uploads/2016/03/header-bg.png
  33 /description.xml
```

---

## Unix philosophy

- Write programs that do one thing and do it well.
- Write programs to handle text streams, because that is a universal interface.
- Write programs to work together.

.small[ Peter H. Salus ]

---

- Write **middlewares** that do one thing and do it well.
- Write **middlewares** to handle **PSR-7 objects** because that is a universal interface.
- Write **middlewares** to work together.

---

```ruby
$ cat access.log | grep 404 | awk '{ print $7 }' | sort
```

---

.left-block[
```php
$pipe = new Pipe([
    function ($request, $next) {
        ...
    },
    function ($request, $next) {
        ...
    },
    function ($request, $next) {
        ...
    },
]);
```
]

--

.right-block[
```php
$pipe = new Pipe();

$pipe->pipe(function ($request, $next) {
    ...
});
$pipe->pipe(function ($request, $next) {
    ...
});
$pipe->pipe(function ($request, $next) {
    ...
});
```
]

---

```php
$pipe = new Pipe([

    function ($request, $next) {
        try {
            return $next($request);
        } catch (\Throwable $e) {
            return new EmptyResponse('Oops!', 500);
        }
    },
    
    function ($request, $next) {
        // force https
        // ...
    },
    
    function ($request, $next) {
        return new TextResponse('The date is ' . date('d M Y'));
    },
    
]);
```

---

```php
$pipe = new Pipe([
    new ErrorHandler(),
    new ForceHttps(),
    function ($request, $next) {
        return new TextResponse('The date is ' . date('d M Y'));
    },
]);
```

---
class: main-title

# Un *middleware* est quelque chose qui prend une *requête* **(et $next)** et retourne une *réponse*.

---

```php
$application = new Pipe([
    ...
]);

$next = function () {
    throw new Exception('No middleware handled the response');
};
$response = $application(ServerRequestFactory::fromGlobals(), $next);
(new SapiEmitter)->emit($response);
```

---
class: title

# Routing

---

```php
$router = new Router();
$router->addRoute('/', function () {
    return new TextResponse('The date is ' . date('d M Y'));
});
$router->addRoute('/about', function () {
    return new TextResponse('This super website is sponsored by AFUP!');
});
```
---

```php
$router = new Router(...

$application = new Pipe([
    new ErrorHandler(),
    new ForceHttps(),
]);
```

---
class: main-title

# Un *middleware* est quelque chose qui prend une *requête* et retourne une *réponse*.

---

```php
$router = new Router(...

$application = new Pipe([
    new ErrorHandler(),
    new ForceHttps(),
    $router,
]);
```

---
class: title

# Frameworks

---

## Zend Expressive/ZF3

```php
$app = Zend\Expressive\AppFactory::create();

$app->pipe(function (...) {
    // middleware
});
$app->pipe(new MyMiddleware());
$app->pipe('nom-de-service');

$app->get('/', function () {
    // controller
});

// ...
$app->run();
```

---

## Slim

```php
$app = new Slim\App();

$app->add(function (...) {
	// middleware
});

$app->get('/', function () {
	// controller
})->add(function (...) {
    // route middleware
});
```

---

## Laravel

```php
class MyMiddleware
{
    public function handle(Request $request, $next)
    {
        // ...
    }
}

class Kernel extends HttpKernel
{
    protected $middleware = [
        MyMiddleware::class,
    ];
    
    ...
}
```

---
class: title

# Frameworks et Architecture

---

```php
$application = new Pipe([
    new ErrorHandler(),
    new ForceHttps(),
    new MaintenanceMode(),
    new SessionMiddleware(),
    new DebugBar(),
    new Authentication(),
    
    new Router([
        '/' => function () { ... },
        '/article/{id}' => function () { ... },
        '/api/articles' => function () { ... },
        '/api/articles/{id}' => function () { ... },
    ]),
]);
```

---

```php
$application = new Pipe([
    new ErrorHandler(),
    new ForceHttps(),
    new PrefixRouter([
        '/api/' => new Pipe([
            new Authentication(),
            new Router([
                '/api/articles' => function () { ... },
                '/api/articles/{id}' => function () { ... },
            ]),
        ]),
        '/' => new Pipe([
            new MaintenanceMode(),
            new SessionMiddleware(),
            new DebugBar(),
            new Router([
                '/' => function () { ... },
                '/article/{id}' => function () { ... },
            ]),
        ]),
    ]),
]);
```

---

# TODO :

- avantages
- inconvénients
- attributes
- frameworks
- PSR-15
- middlewares PSR-7