### (추가중)

# Service Container

>The Laravel service container is a powerful tool for managing class dependencies and performing dependency injection. Dependency injection is a fancy phrase that essentially means this : class dependencies are "injected" into the class via the constructor or, in some cases, "setter" methods.

The Service Container is a `container` that holds classes you'd like to `resolve(instantiate)` programmatically later in your application.

[Laravel Service Containers for dummies. - Agyenim Boateng](https://medium.com/@barfiagyenim/laravel-service-containers-for-dummies-c29ab315683b)



Service Container is a Dependency Injection Container and a Registry for the application.

The advantages of using a Service Container over creating manually your objects are : **Ability to manage class dependencies on object creation**

[stack overflow - What is the concept of Service Container in Laravel?](https://stackoverflow.com/questions/37038830/what-is-the-concept-of-service-container-in-laravel)



Service Container is a container that holds all the bindings that need to run Laravel application smoothly. you can bind almost everything you'd like to instantiate programmatically later in your application when needed. That means Service Container holds a single object of all of our various bindings.

[DEV community - Laravel Service Container](https://dev.to/patelparixit07/laravel-service-container-3gaj)



Laravel 공식문서와 많은 사람들이 설명하고 있는 Service Container 의 정의이다. 

가장 와닿는 말은 '**Service Container는 인스턴스화 하고자하는 클래스들을 담고있는 컨테이너 이다.**' 인거 같다.

우리는 `bind` 또는 `singleton`메소드를 통해 클래스들을 **Service Container**에 담을 수 있다. 이러한 과정은 `App -> Providers -> AppServiceProvider.php` 또는 Artisan 명령어인 `php artisan make:provider {프로바이더명}` 을 통해 할 수 있다.



간단한 예제를 통해 알아보자

우선 덧셈과 곱셈을 하는 서비스 클래스를 만들어보자. App -> Services 폴더를 만든 뒤, `MathService.php`를 생성하자

![](https://user-images.githubusercontent.com/44697835/114497627-85771e80-9c5d-11eb-8a93-05bf0da6bcf6.png)

```php
// MathService.php
<?php
namespace App\Services;

class MathService
{
    public function doAddition($number)
    {
        return array_sum($number);
    }

    public function doMultiplication($number)
    {
        return array_product($number);
    }
}
```



`MathService` 클래스에 있는 `doAddition` 메소드를 실행하기 위해 라우터를 이용하자. `자신의주소/add`로 접속을 하면 `doAddition` 함수가 실행되도록 한다.

```php
// web.php
<?php
use App\Services\MathService;

Route::get('/add', function () {
    $mathService = new MathService;
    $sum = $mathService->doAddition([40, 20, 10]);
    return $sum;
});
```

우리는 `MathService` 클래스를 이용하기 위해 `use App\Services\MathService;`를 선언하고 `new MathService` 한 뒤 이용하였다. 



여기서 **Service Container**의 역할을 다시 생각해보자.

> The Service Container is a `container` that holds classes you'd like to `resolve(instantiate)` programmatically later in your application.

그렇다면 우리는 Service Container를 통해 인스턴스화 하고자 하는 클래스(여기서는 `MathService`)를 바인드 할 수 있다.



우선적으로는 `App->Providers->AppServiceProvider.php`를 이용해 `MathService`클래스를 바인드 해볼것이다.

```php
// AppServiceProvider.php
<?php

namespace App\Providers;

use App\Services\MathService;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->bind('MathService', MathService::class);
        //$this->app->singleton('MathService', MathService::class);
    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        //
    }
}
```

`$this->app->bind('MathService', MathService::class);` 구문을 통해 'MathService' 라는 이름으로 `MathService` 클래스가 Service Container에 바인드 되었다.



이제 라우터(`web.php`)를 다음과 같이 쓸 수 있다.

```php
// web.php
<?php

Route::get('/add', function () {
    $mathService = app()->make('MathService');
    $sum = $mathService->doAddition([40, 20, 10]);
    return $sum;
});
```

위에서 `use App\Services\MathService;` 를 선언하고 `new MathService;`의 부분이 빠졌다.

Service Container에 바인드 함으로써, `app()->make()`를 통해 `MathService` 클래스를 인스턴스화 할 수 있게 되었다. 



~~사실 여기까지 봐서는 왜 Service Container가 왜 필요한지를 못 느낄수도 있다.(실제로 나는 이해 할 수 없었다.)~~



#### Interface를 Implements로 바인딩

아래와 같이 인사를 할 수 있는 클래스를 만들었다고 하자.(App->Services->HelloService.php)

```php
<?php
namespace App\Services;

class HelloService
{
    public static function sayHello()
    {
        return '안녕!';
    }
}
```
```php
// AppServiceProvider.php

<?php

namespace App\Providers;

use App\Services\HelloService;
use Illuminate\Support\Facades\App;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register any application services.
     *
     * @return void
     */
    public function register()
    {
        App::bind('HelloService', function () {
            return new HelloService;
        });

    }

    /**
     * Bootstrap any application services.
     *
     * @return void
     */
    public function boot()
    {
        //
    }
}
```

```php
// web.php
use App\Services\HelloService;

Route::get('/hello', function (HelloService $helloService) {
    return $helloService->sayHello();
});
```



`자신의 주소/hello`로 접속을 하면 `안녕!`이라고 인사를 받을 수 있다.

이제 이 서비스를 외국인을 상대로 한다고 가정하자. 미국의 경우 `Hello!`, 프랑스의 경우 `Bonjour!`, 인도의 경우`नमस्ते` 라고 해야 한다. 그렇다면 서비스하는 나라가 바뀔때마다 모든 Hello 메소드, 라우터, 바인드를 바꿔야 할까?



우리는 Interface와 Interface to Implements 바인드를 통해 이러한 번거로움을 줄일 수 있다. 먼저 HelloService라는 이름으로 인터페이스를 생성한다.(`App->Services->HelloService.php`)

```php
<?php
namespace App\Services;

interface HelloService
{
    public static function sayHello();
}

```

지금까지 구현해왔던 클래스와는 차이가 있다. 가장 큰 차이는 함수를 선언만 하였고, 구체화 하지 않았다.



이제 `HelloService` 인터페이스를 구현(Implement)하는 클래스를 만들어보자.  우선 한국어로 인사하는 클래스이다. (`App->Services->HelloServiceKorImpl`)

```php
<?php
namespace App\Services;

class HelloServiceKorImpl implements HelloService
{
    public static function sayHello()
    {
        return '안녕!';
    }
}
```

Class 뒤에 `implements HelloService`라는게 붙어있다.

**interface**를 *implements* 한 클래스는 반드시 **interface**에서 선언된 함수들을 구체화 하여하 한다. 즉, 클래스를 구현할때 가이드라인을 준수한다고 보면 된다.

우리는 `HelloService`라는 **Interface**를 만들고, `HelloServiceKorImpl`을 구현함으로써 `HelloServiceKorImpl`에는 반드시 `sayHello 메소드`가 구현되어야 한다는것을 알 수 있고, 해당 메소드 내의 상세한 사항은 개발자가 정할 수 있다는 것을 알 수 있다.



각 나라별 인사 서비스를 개발한다고 할때, `HelloService`인터페이스를 통해 개발자들은 `sayHello`메소드를 각 언어별로 추가 할 수 있다.(ex : `HelloServiceEngImpl`등등...)

```php
// HelloServiceEngImpl.php
<?php
namespace App\Services;

class HelloServiceEngImpl implements HelloService
{
    public static function sayHello()
    {
        return 'Hello!';
    }
}
```



그렇다면 이렇게 만든 클래스들을 바인드 하는 부분또한 매번 수정되야 하는걸까?

Laravel에서는 인터페이스를 각각의 implements들로 바인드 할 수 있다. 이러한 과정을 통해 번거로움을 해결 할 수 있다.

```php
// AppServiceProvider.php

use App\Services\HelloService;
use App\Services\HelloServiceKorImpl;
// use App\Services\HelloServiceEngImpl;	// 영어의 경우
// use App\Services\HelloServiceFraImpl;	// 프랑스어의 경우

//..
public function register () {
		App::bind(HelloService::class, function () {
		return new HelloServiceKorImpl();
        // return new HelloServiceEngImpl();	// 영어의 경우
        // return new HelloServiceFraImpl();	// 프랑스어의 경우
	});
}
```



라우터(실제 `sayHello` 메소드를 호출할 부분)에서는 어떤 서비스의 메소드를 호출하는지 알 필요가 없다.

```php
// web.php
use App\Services\HelloService;

Route::get('/hello', function (HelloService $helloService) {
    return $helloService->sayHello();
});
```



#### Contextual Binding

#### Binding Primitives

#### Binding Typed Variadics

#### Variadic Tag Dependecies

#### Tagging

#### Extending Bindings





# Service Provider

Service Provider는 Laravel 어플리케이션 **부트스트래핑**의 집합소이다. 부트스트래핑(Bootstrapping)이란 **registering** 하는것이다. 여기에는 Service Container 바인딩 등록, 이벤트 리스너 등록, 미들웨어, 라우터까지 포함되어 있다.

영단어를 번역 할 수 없어 그대로 썼기때문에 말이 이해가 안되지만, 간단하게는 Laravel 어플리케이션 구동에 필요한 것들이 모여있는 곳이라고 보면 되겠다.

모든 Service Provider는 `Illuminate\Support\ServiceProvider`클래스를 상속받고 있다. 그리고 대부분의 Service Provider는 `register`메소드와 `boot`메소드를 가지고 있다.

`register`메소드에서는 **반드시** Service Container로의 바인드만 존재해야 한다. 

Service Provider는 artisan 명령어로 생성이 가능하다.

```
php artisan make:provider MyServiceProvider
```



이렇게 생성된 Provider는 `App -> Providers`에서 확인이 가능 하다. 그리고 `config -> app.php`에서 providers 배열에 등록하여 사용할 수 있다.

```php
// app.php
<?php

return [
	// 생략

    'providers' => [

        /*
         * Laravel Framework Service Providers...
         */
		// 생략
        /*
         * Package Service Providers...
         */

        /*
         * Application Service Providers...
         */
        App\Providers\AppServiceProvider::class,
        App\Providers\AuthServiceProvider::class,
        // App\Providers\BroadcastServiceProvider::class,
        App\Providers\EventServiceProvider::class,
        App\Providers\RouteServiceProvider::class,
        
        // 여기에 Proivder를 등록 할 수 있다. 

    ],

    /*
    |--------------------------------------------------------------------------
    | Class Aliases
    |--------------------------------------------------------------------------
    |
    | This array of class aliases will be registered when this application
    | is started. However, feel free to register as many as you wish as
    | the aliases are "lazy" loaded so they don't hinder performance.
    |
    */

    'aliases' => [
		//...
    ],

];

```





