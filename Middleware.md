# Middleware

![](https://user-images.githubusercontent.com/44697835/114823240-32889d00-9dfe-11eb-927b-11807720e5ad.jpeg)

<center>출처 - <a href="https://medium.com/@devamplify/deep-dive-into-middlewares-in-laravel-a41921ffa42">Deep Dive Into Middlewares In Laravel</a><div style="font-size:11px;">Middleware를 가장 잘 표현한 그림인거 같다.</div></center>



Laravel Middleware는 어플리케이션으로 들어오는 HTTP Request를 검사하고 필터링하는 기능을 제공한다.



미들웨어는 Artisan 명령어로 생성 할 수 있다.

```
php artisan make:middleware {미들웨어명}
```



이렇게 생성한 미들웨어를 특정 라우트에 적용시킬 수 있다.

```php
    <?php

    namespace App\Http;

    use Illuminate\Foundation\Http\Kernel as HttpKernel;

    class Kernel extends HttpKernel
    {
        /**
         * The application's global HTTP middleware stack.
         *
         * These middleware are run during every request to your application.
         *
         * @var array
         */
        protected $middleware = [
            // ...생략
        ];

        /**
         * The application's route middleware groups.
         *
         * @var array
         */
        protected $middlewareGroups = [
            // ...생략
        ];

        /**
         * The application's route middleware.
         *
         * These middleware may be assigned to groups or used individually.
         *
         * @var array
         */
        protected $routeMiddleware = [
            'auth' => \App\Http\Middleware\Authenticate::class,
            'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
            'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class,
            'can' => \Illuminate\Auth\Middleware\Authorize::class,
            'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
            'password.confirm' => \Illuminate\Auth\Middleware\RequirePassword::class,
            'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,
            'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
            'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,

            //
            '등록할 이름' => '미들웨어 경로'
        ];
    }

```



라우터

```php
// web.php
<?php

Route::post('/middletest', function () {
    // 미들웨어 통과 후 할 일
})->middleware('등록한 이름');
```







간단한 예제를 만들어보자. 우선 TestMiddleware 를 만든다.

```
php aritsan make:middleware TestMiddleware
```



이후 `TestMiddleware.php`(`App -> Http -> Middleware -> TestMiddleware.php`)를 열고 다음과 같이 작성한다.

```php
<?php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class TestMiddleware
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle(Request $request, Closure $next)
    {
        if ($request->input('access') !== 'OK') {
            return redirect('/');
        }

        return $next($request);

    }
}
```

`if` 에서는 들어오는 `request`중 `access `라는 키값의 `value`를 확인하여 그 값이 `OK`이면 `return $next($request)`을, 아닌경우 `post`라는 주소로 redirect 시킨다.



이렇게 생성한 미들웨어를  `App -> HTTP -> Kernel.php` 의 `routeMiddleware`에 `test`라는 이름으로 등록하자.

```php
<?php

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    /**
     * The application's global HTTP middleware stack.
     *
     * These middleware are run during every request to your application.
     *
     * @var array
     */
    protected $middleware = [
		// ...생략
    ];

    /**
     * The application's route middleware groups.
     *
     * @var array
     */
    protected $middlewareGroups = [
		// ...생략
    ];

    /**
     * The application's route middleware.
     *
     * These middleware may be assigned to groups or used individually.
     *
     * @var array
     */
    protected $routeMiddleware = [
        'auth' => \App\Http\Middleware\Authenticate::class,
        'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
        'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class,
        'can' => \Illuminate\Auth\Middleware\Authorize::class,
        'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
        'password.confirm' => \Illuminate\Auth\Middleware\RequirePassword::class,
        'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,
        'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
        'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
        
        //
        'test' => \App\Http\Middleware\TestMiddleware::class,
    ];
}

```



마지막으로 라우터에서 다음과 같이 작성한다.

```php
// web.php
<?php

use App\Http\Controllers\PostController;
use Illuminate\Support\Facades\Route;

Route::resource('post', PostController::class);

Route::post('/middletest', function () {
    return 'Access Success';
})->middleware('test');
```

`/middletest`라는 경로의 requset들은 `test`미들웨어를 통해 검사가 가능해졌다.





postman으로 테스트를 하기 위해 csrf 토큰검사에 예외경로`/middletest`를 추가해준다.

```php
// VerifyCsrfToken
<?php

namespace App\Http\Middleware;

use Illuminate\Foundation\Http\Middleware\VerifyCsrfToken as Middleware;

class VerifyCsrfToken extends Middleware
{
    /**
     * The URIs that should be excluded from CSRF verification.
     *
     * @var array
     */
    protected $except = [
        //
        'post/*',
        'post',
        'middletest',
    ];
}
```





`/middletest` 의 경로로 body에 `key = access`, `value = OK`를 넣어주면 `Access Success`가 나타나고,

![](https://user-images.githubusercontent.com/44697835/115186572-ed75ab00-a11c-11eb-8c90-31f8311070e5.png)



`value` 값의 `OK`가 아닌 다른게 들어 갈 경우 원래 게시판 페이지가 나타난다.

![](https://user-images.githubusercontent.com/44697835/115186720-2c0b6580-a11d-11eb-88be-1973a36529e8.png)

