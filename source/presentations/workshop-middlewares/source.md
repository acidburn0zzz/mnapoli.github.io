class: title

# Les middlewares en PHP

---

Les middle-what ?

---

Framework vs Library

---

.center[ ![](img/middleware.png) ]

---

```php
$response = middleware($request);
```

---

## Symfony

```php
interface HttpKernelInterface
{
    /**
     * @return Response
     */
    public function handle(Request $request, ...);
}
```

---

## PSR-7

```
composer require psr/http-message
```

- `RequestInterface`
- `ServerRequestInterface`
- `ResponseInterface`
- ...

---

```php
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;

function (ServerRequestInterface $request) : ResponseInterface {
    return new Response('Hello world!');
}
```

.small[
*[Zend Diactoros](https://github.com/zendframework/zend-diactoros)*
]

---
class: title

# Step 1

## write and run a middleware

---

.center[ ![](img/step-1.png) ]

---

## Step 1: write and run a middleware

Write your first middleware.

The web application (in `index.php`) should show "Hello world!" at [http://localhost:8000](http://localhost:8000/).

---

```php
use Zend\Diactoros\Response\TextResponse;

$application = function (ServerRequestInterface $request) {
    return new TextResponse('Hello world!');
};
```

---
class: title

# Step 2

## use the request

---

## Step 2: use the request

Use the request so that [http://localhost:8000/?name=Bob](http://localhost:8000/?name=Bob) shows "Hello Bob!"

.small[
*Bonus: [http://localhost:8000/](http://localhost:8000/) should still show "Hello world!"*
]

---

```php
$application = function (ServerRequestInterface $request) {
    $queryParams = $request->getQueryParams();
    
    $name = $queryParams['name'] ?? 'world';
    
    return new TextResponse('Hello ' . $name . '!');
};
```

---
class: title

# Step 3

## compose middlewares to handle errors nicely

---

```
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

.center[ ![](img/step-1.png) ]

---

.center[ ![](img/step-3.png) ]

---

# TODO: intro to layers

![](img/layers.png)

---

## Step 3: compose middlewares to handle errors

Assemble multiple middlewares with a `Pipe`.

```php
$pipe = new Pipe([
    function (...) { ... },
    function (...) { ... },
]);
```

Write an error handler middleware. Place it first in the pipe. It should catch exceptions thrown in next middlewares and show an error page.

.small[
*Bonus: write the error handler middleware as a class.*
]

---

```php
use Psr\Http\Message\ServerRequestInterface as Request;

$application = new Pipe([

    // Error handler
    function (Request $request, callable $next) {
        try {
            return $next($request);
        } catch (\Exception $e) {
            $m = $e->getMessage();
            return new TextResponse('Error: '.$m, 500);
        }
    },

    // Application
    function (Request $request, callable $next) {
        throw new Exception('Test');
    },

]);
```

---

```php
class ErrorHandler implements Middleware
{
    public function __invoke(Request $request, callable $next)
    {
        try {
            return $next($request);
        } catch (\Exception $e) {
            $whoops = $this->createWhoops();
            $output = $whoops->handleException($e);
            return new HtmlResponse($output, 500);
        }
    }

    private function createWhoops()
    {
        return ...;
    }
}
```

---
class: title

# Step 4

## split the flow with a router

---

# TODO: intro

---

.center[ ![](img/step-3.png) ]

---

.center[ ![](img/step-4.png) ]

---

```php
function (ServerRequestInterface $request, callable $next) {

    $url = $request->getUri()->getPath();
    
    if ($url === '/login') {
        return /* login page */;
    } elseif ($url === '/dashboard') {
        return /* dashboard page */;
    }
    
    return $next($request);
}
```

---

![](img/fastroute.png)

---

## Step 4: split the flow with a router

Use the router to map URLs to handlers (aka controllers).

```php
$router = new Router([
    '/' => function () { ... },
    '/blog/' => function () { ... },
    '/article/{name}' => function () { ... },
]);
```

---

```php
$application = new Pipe([
  new ErrorHandler(),
  new Router([
  
    '/' => function () use ($container) {
        $articles = $container->articleRepository()->getArticles();
        $html = $container->twig()->render('home.html.twig', [
            'articles' => $articles,
        ]);
        return new HtmlResponse($html);
    },
    
    '/about' => function () use ($container) {
        return new HtmlResponse(
            $container->twig()->render('about.html.twig')
        );
    },
      
  ]),
]);
```

---

# Controller == Middleware

---
class: title

# Step 5

## Authentication middleware

---

.center[ ![](img/step-5.png) 

---

## HTTP Basic authentication

```
Authorization: Basic QWxhZGRpbjpPcGVuU2VzYW1l
```

---

```php
$header = $request->getHeaderLine('Authorization');
if (strpos($header, 'Basic') !== 0) {
    // No authentication found: 401
    ...
}

// Decode the username and password from the HTTP header
$header = explode(':', base64_decode(substr($header, 6)), 2);
$username = $header[0];
$password = isset($header[1]) ? $header[1] : null;

if (/* $username and $password are valid */) {
    // Authenticated
    ...
}

// Authentication failed: 403
...
```

---

## Step 5: authentication middleware

Write a middleware that checks for a valid HTTP "Basic" authentication before calling the next middleware.

Complete the existing `HttpBasicAuthentication` class.

Run tests with: `composer tests`

.small[
*Bonus: use the middleware in your application to prevent access to the whole website.*
]

---

```php
$header = $request->getHeaderLine('Authorization');
if (strpos($header, 'Basic') !== 0) {
    // No authentication found
    return new EmptyResponse(401, [
        'WWW-Authenticate' => 'Basic realm="Superpress"',
    ]);
}

// [...]

if (/* $username and $password are valid */) {
    return $next($request);
}

// Authentication failed
return new EmptyResponse(403);
```

---

```php
$application = new Pipe([
    new ErrorHandler(),
    new HttpBasicAuthentication([
        'user' => 'password',
    ]),
    new Router([
        ...
    ]),
]);
```

---
class: title

# Step 6

## nesting middlewares

---

## Step 6: nesting middlewares

Add an API to your application (JSON responses):

- `/api/articles` should return the list of articles
- `/api/time` should return the current `time()`

The API must require authentication (HTTP basic auth), but the website must be publicly accessible (no authentication anymore).

Remember the router or the middleware pipe are like any other middleware: you can nest them and use them several times.

---

```php
$application = new Pipe([
    new ErrorHandler(),
    
    new Router([
        '/' => function () { ... },
        '/about' => function () { ... },
        
        '/api/{path:.*}' => new Pipe([
            new HttpBasicAuthentication(['user' => 'password']),
            
            new Router([
                '/api/articles' => function () {
                    return new JsonResponse(...);
                },
                '/api/time' => function () {
                    return new JsonResponse(time());
                },
            ]),
        ]),
    ]),
]);
```

---

```php
$application = new Pipe([
    new ErrorHandler(),
    
    new Router([
        '/' => function () { ... },
        '/about' => function () { ... },
    ]),
    
    new Pipe([
        new HttpBasicAuthentication(['user' => 'password']),
        
        new Router([
            '/api/articles' => function () {
                return new JsonResponse(...);
            },
            '/api/time' => function () {
                return new JsonResponse(time());
            },
        ]),
    ]),
]);
```

---

## Middlewares

Architecture:

- pipe
- router
- prefix router

---

## Middlewares

Request/response pre/post-processors:

- authentication
- firewall/authorization
- HTTP cache headers
- response cache
- content/language negotiation
- logging
- CRSF protection
- rate limiting for APIs
- exception handler/error page
- force HTTPS, redirect to www., add trailing /, ...
- robots (X-Robots-Tag)
- IP restriction

---

## Middlewares

Applications or part of applications:

- maintenance page
- debug toolbar & pages
- assets & medias ([Glide](http://glide.thephpleague.com/))
- modules…

---

## Slim

```php
$app = new \Slim\App();

// Global middleware
$app->add(function ($request, $response, $next) {
	// ...
});

// Route middleware
$app->get('/', function ($request, $response, $args) {
	// controller
})->add(function ($request, $response, $next) {
    // middleware
});
```

---

## Zend Expressive/ZF3

```php
$app->pipe('/', function ($req, $res, $next) {
    // ...
});
```

---

## Silex

```php
$app->before(function (Request $request, Application $app) {
    // ...
});

$app->after(function (Request $request, Response $response) {
    // ...
});
```

---

## Laravel

```php
class MyMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        // ...

        return $next($request);
    }

}
```