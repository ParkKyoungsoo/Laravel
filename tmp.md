기존에 만들었던 게시판의 경우 컨트롤러에 너무 많은 기능이 구현되어 있다. 지금부터 만들어볼 게시판은 Controller에 집중되었던 기능들을 Service, Repository로 분리시켜 DB와의 통신은 Repository, 비즈니스 로직은 Service에 집중시킬것이다.



# Model의 생성

처음과는 다르게 이번엔 Laravel에서 모델을 생성 후 이를 DB에 마이그레이션 할 것이다.

```
php artisan make:model Post -mcr
```

먼저 Post 이름으로 Model을 생성한다.

`-m`,`--migration` : 모델로부터 새 마이그레이션파일을 생성
`-c`,`--controller` : 모델로부터 새 컨트롤러 생성
`-r`,`--resource` : 생성된 컨트롤러가 리소스 컨트롤러여야하는지를 나타낸다. 

이렇게 하면 Model과 migartion, 기본 리소스를 가진 Controller가 생성이 된다.

┌ **app**
│	├ **Http**
│	│	└ **Controller**
│	│		└ PostController.php
│	└ **Models**
│		└ Post.php
│
└ **database**
	└ **migrations**
		└ _create_posts_table.php



`App->Models->Post.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    protected $fillable = [
        'title',
        'description',
    ];

    protected $hidden = [
        'created_at',
        'updated_at',
    ];
}
```

모델을 다음과 같이 작성한다.

`protected $hidded = ['created_at', 'updated_at'];` 구문의 경우 Model을 json이나 배열의 형태로 변환했을때 포함이 되지 않는다.



이제 CreatePostsTable을 보자

`database->migrations->_create_posts_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class CreatePostsTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->bigIncrements('id');
            $table->string('title');
            $table->string('description');
            $table->timestamps();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('posts');
    }
}
```

`bigIncrements()` 메소드는 `auto_increaments` 형식의 `unsign big int`(*primaey key*) 컬럼을 생성한다.

Model에서 별도의 table 명을 명시하지 않았기 때문에 table의 이름은 자동으로 `posts`가 된다.

(스네이크 케이스로 변환 한 뒤 복수형)



이제 아티즌 명령어 `php artisan migrate`를  통해 마이그레이션을 진행하자.(마이그레이션을 진행하기 전 Laravel에서 DB설정이 모두 완료되어야 한다.)

![](https://user-images.githubusercontent.com/44697835/114660243-ebcc7180-9d2f-11eb-8c10-aa34d86d4ed3.png)







# Repository 생성

Repository에서는 Eloquent ORM을 사용하여 DB와 통신하고, 결과 값을 Service 로 넘겨 줄 것이다.

먼저 `App`폴더 아래 `Repositories`폴더를 생성하고, `Repositories`내에 `IPostRepository.php`를 생성한다. 이때 생성된 `IPostRepository.php.`는 **Interface**이다. 우선적으로 글을 저장하는 `save`메소드만 명시하자.

**App**
​	└ **Repositories**
		└ IPostRepository.php

`App -> Repositories->IPostRepository.php`

```php
<?php
namespace App\Repositories;

interface IPostRepository
{
    public function save($data);
}
```





그리고 `PostRepositoryImpl.php`를 만든 뒤 `IPostRepository` **Interface**를 **구현(implements)**한다.

`App->Repositories->PostRepositoryImpl.php`

```php
<?php
namespace App\Repositories;

use App\Models\Post;

class PostRepositoryImpl implements IPostRepository
{
    protected $post;

    public function __construct(Post $post)
    {
        $this->post = $post;
    }

    public function save($data)
    {
        $post = new $this->post;

        $post->title = $data['title'];
        $post->description = $data['description'];

        $post->save();

        return $post->fresh();
    }
}
```

`PostRepositoryImpl`는 Model을 사용해야 하기 때문에 생성자`__construct()`를 통해 DI(의존성 주입)를 한다. 그리고 **interface**에서 명시하였던 `save()`메소드를 구현한다. 

`fresh()`메소드는 DB에 에서 컬렉션에 있는 각 Model의 새 인스턴스를 검색한다. 즉, 새롭게 추가된 데이터를 검색한다고 생각하면 된다. 이렇게 검색된 데이터를 return 함으로써 Service에 새로 추가된 데이터를 넘겨주는 것이다.





# Service 생성

RepositoryImpl을 호출하고, 리턴값을 받아서 비즈니스 로직을 실행하는 Service를 만든다. `App`폴더 아래에 `Services`폴더를 만든 뒤, `IPostService.php`**Interface**를 생성한다. 여기서도 글을 저장하는 `savePostData()`메소드를 명시하자.

**App**
	└ **Services**
		└ IPostService.php

`App-Services->IPostService.php`

```php
<?php
namespace App\Services;

interface IPostService
{
    public function savePostData($data);
}
```



이후 `PostRepositoryImpl.php`와 마찬가지로 `PostServiceImpl.php`를 생성하고 `IPostService`를 **구현(Implements)**한다.

```php
<?php
namespace App\Services;

use App\Repositories\IPostRepository;
use Illuminate\Support\Facades\Validator;
use InvalidArgumentException;

class PostServiceImpl implements IPostService
{
    protected $postRepository;

    public function __construct(IPostRepository $postRepository)
    {
        $this->postRepository = $postRepository;
    }

    public function savePostData($data)
    {
        $validator = Validator::make($data, [
            'title' => 'required',
            'description' => 'required',
        ]);

        if ($validator->fails()) {
            throw new InvalidArgumentException($validator->errors()->first());
        }

        $result = $this->postRepository->save($data);

        return $result;
    }
}
```



`PostServiceImpl`에서 `IPostRepositoy`를 사용하기 때문에, `__construct()`메소드를 이용하여 의존성을 주입한다.



`use Illuminate\Support\Facades\Validator;`, `$vaildator = Validator::make()`를 통해 validator 인스턴스를 생성할 수 있다. 

```
use Illuminate\Support\Facades\Validator;
...
$vaildator = Validator::make();
```



유효성 검사에 실패했을경우(`$validator->fails() 가 참이되는 경우`) `$validator->errors()->first()`내용을 담고 있는 `InvalidArgumentException`를 생성 한 뒤 `throw `한다. 예외를 던질(`throw`)경우 `throw`아래 코드부터는 실행이 되지 않고 `catch`문을 찾아 실행한다.

`$validator->errors()`는 validator결과 나타나는 모든 에러사항을 json형식으로 표시를 해준다. 만약 게시물을 작성할 때 제목과 내용 모두 빈값으로 작성한다면 `$validator->errors()`의 내용은 아래와 같다.

```json
{
	"title": [
		"The title field is required."
	],
	"description":
		"The description field is required."
	]
}
```

이중 우리는 우선적으로 첫번째 에러사항만을 보여주기 때문에 `first()`메소드를 사용하였다.



유효성 검사에 성공한 경우에는`PostRepositoryImpl`클래스의 `save()`메소드를 호출한 뒤, 받은 결과값을 컨트롤러에 return 해준다.





# Controller 구현

controller의 경우 아티즌 명령어에 의해 resoure가 구현되어있는 Controller가 생성되어 있을 것이다.

그 중에서 글을 저장하는 `store` 메소드를 구현하자.

컨트롤러또한 `IPostService`를 사용하기 때문에 생성자(`__construct()`)를 통해 의존성을 주입한다.

```php
<?php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Services\IPostService;
use Exception;
use Illuminate\Http\Request;

class PostController extends Controller
{
    protected $postService;

    public function __construct(IPostService $postService)
    {
        $this->postService = $postService;
    }
	
    // 생략
    
    public function store(Request $request)
    {
        $data = $request->only([
            'title',
            'description',
        ]);

        $result = ['status' => 200];

        try {
            $result['data'] = $this->postService->savePostData($data);
        } catch (Exception $e) {
            $result = [
                'status' => 500,
                'error' => $e->getMessage(),
            ];
        }

        return response()->json($result, $result['status']);

    }
	// 생략
}
```

`$request->only()`를 사용하여 requset중 key값이 title, description인 값들을 가져 올 수 있다.

`only()`메소드는 `key / value` 를 모두 리턴해준다.



이후 `postService->savePostData()`에서 에러를 던졌(`throw`)다면 catch문을, 아니면 try문을 실행한다.

마지막으로 `response()->json()` 을 통해 response body에 json data를 담아서 return 해준다.



# 라우터 설정

`routes->web.php`에서 라우터를 설정해준다.

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\PostController;


Route::resource('post', [PostController::class]);
```

처음 컨트롤러를 생성할 때 리소스 컨트롤러를 생성하였기 때문에 `Route::resource('post', [PostController::class]);` 이 한줄로 CRUD가 할당이 된다.







# 테스트

이제 게시글을 등록해보자. 우리는 view 단을 만들지 않았기 때문에 웹상에서는 테스트를 할 수 없다. 그렇기 때문에 지금부터는 `postman`을 통해 테스트를 진행할 것이다.

[postman - https://www.postman.com/](https://www.postman.com/)

포스트맨으로 요청을 보내기 전에 `post`요청의 경우 Laravel에서 지원하는 csrf 미들워어에 의해 토큰이 없으면 요청 자체를 할 수 없다. 우선 `사이트주소/post` 요청은 예외 처리를 해두자

`App->Middleware->VerifyCsrfToken.php`

```php
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
        '/post',
    ];
}
```

`protected $except` 항목에 `/post`를 추가하여 `사이트주소/post`의 요청은 csrf 토큰검사 예외처리를 할 수 있다.





다시 postman으로 돌아가서 글쓰기 요청을 보내보자.

![image](https://user-images.githubusercontent.com/44697835/114677006-d6157700-9d44-11eb-9d72-2bd7b4951e29.png)



무수한 에러와 함께 글이 등록이 되지 않는다. 에러 내용을 확인해보자

> Illuminate\Contracts\Container\BindingResolutionException: Target [App\Services\IPostService] is not instantiable while building [App\Http\Controllers\PostController]. in file 

**Target is not instantiable while building** 타겟(IPostService)을 인스턴스화 할 수 없다고 한다.



우리가 구현했던 `IPostService`는 **Interface**이다. **Interface**는 인스턴스화(`$postService = new IPostService`) 할 수 없다. 그렇기 때문에 우리는 이를 **구현(implements)**하였다. 



그렇다면 `Repository`, `Service`, `Controller`에서 주입(DI)했던 **Interface**들을 **Implements**들로 바꾸면 될까?



여기서 생각을 해보자. 만약 `PostServiceImpl.php`코드에 어떠한 로직이 추가되었다고 하자.

```php
<?php
namespace App\Services;

use App\Repositories\IPostRepository;
use Illuminate\Support\Facades\Validator;
use InvalidArgumentException;

class PostServiceImpl implements IPostService
{
    protected $postRepository;

    public function __construct(IPostRepository $postRepository)
    {
        $this->postRepository = $postRepository;
    }

    public function savePostData($data)
    {
        $validator = Validator::make($data, [
            'title' => 'required',
            'description' => 'required',
        ]);
		
        // 특별한 로직이 추가되었다고 가정한다. 이 로직은 매우 복잡함
        //
        //
        
        if ($validator->fails()) {
            throw new InvalidArgumentException($validator->errors()->first());
        }

        $result = $this->postRepository->save($data);

        return $result;
    }
}
```

이렇게 개발을 한 뒤 A에서 이 게시판 프로그램을 쓴다고 가정한다.



또한 B에서도 이 게시판 프로그램을 쓰지만 추가되었던`특별한 로직`말고 `정밀한 로직` 이 필요하다고 하자.

```php
<?php
namespace App\Services;

use App\Repositories\IPostRepository;
use Illuminate\Support\Facades\Validator;
use InvalidArgumentException;

class PostServiceImpl implements IPostService
{
    protected $postRepository;

    public function __construct(IPostRepository $postRepository)
    {
        $this->postRepository = $postRepository;
    }

    public function savePostData($data)
    {
        $validator = Validator::make($data, [
            'title' => 'required',
            'description' => 'required',
        ]);
		
        // 정밀한 로직이 추가되었다고 가정한다. 이 로직은 매우 복잡함하며 정밀하다.
        //
        //
        
        if ($validator->fails()) {
            throw new InvalidArgumentException($validator->errors()->first());
        }

        $result = $this->postRepository->save($data);

        return $result;
    }
}
```

여기까지는 큰 문제가 없다. 하지만 C에서는 `정밀한 로직`과 `특별한 로직`을 번갈아 가며 사용한다고 하자(실제로 이런일이 발생하는지는 모르겠다.)



그렇다면 C에서는 `PostServiceImpl복잡.php`와 `PostServiceImpl정밀.php` 두 코드를 만든 뒤, 이 코드들을 주입하고 있는 모든 Controller 또한 수정해야 할까? Controller의 개수가 적다면 상관없지만 주입받은 Controller가 10개가 넘어간다면 매우 번거롭고 비 효율적인 일이 되버린다.

**Interface**는 규약이다. **Interface**에서 메소드들을 명시하고 이를 다양한 형태로 **구현(Implements)** 할 수 있다. 위의 `복잡`과 `정밀`은 모두 `IPostService`를 **구현(implements)** 한 것이다. 만약 **Interface**를 인스턴스화 할 수 있다면 이러한 문제를 해결 할 수 있지 않을까?



Laravel에서는 `bind`를 통해 위의 문제를 조금이나마 해결 할 수있다. 먼저 `App->Providers->AppServiceProvider.php` 를 열자

```php
<?php

namespace App\Providers;

use App\Repositories\IPostRepository;
use App\Repositories\PostRepositoryImpl;
use App\Services\IPostService;
use App\Services\PostServiceImpl;
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
        //
        $this->app->bind(IPostService::class, PostServiceImpl::class);
        $this->app->bind(IPostRepository::class, PostRepositoryImpl::class);

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

`register()`메소드 내의 `bind()`에 주목한다.

우리는 구현체(`PostServiceImpl`, `PostRepositoryImpl`)들을 인터페이스(`IPostService`,`IPostRepository`)에 바인드 시킴으로써 Controller, Service등에서 인터페이스를 호출 하였을때, 바인딩한 구현체(Implements)들을 인스턴스화 시킬 수 있다. 



C의 상황에 맞추어보면 Controller는 `정밀`인지`특별`인지 알 필요 없이 인터페이스인 `IPostService`만 호출하면 되고, 매번 상황에 맞추어 인터페이스에 바인딩한 구현체(Implements)만 바꿔주면 된다. 



이제 다시 Postman을 통해 요청을 보내보자.

![](https://user-images.githubusercontent.com/44697835/114687473-c4d16800-9d4e-11eb-9da7-4619770cba58.png)



