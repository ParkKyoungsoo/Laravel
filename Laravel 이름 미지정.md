#### Model

Laravel에서는 artisan 명령어를 통해 비교적 쉽게 모델을 생성 할 수 있다.

```
php artisan make:model User
```



모델을 생성할 때 데이터 마이그레이션을 생성하고 싶다면 `--migration` 또는 `-m`옵션을 사용하면 된다.

```
php artisan make:model User --migration
// 또는
php artisan make:model User -m
```



Laravel Model에서는 별도의 테이블 이름을 지정하지 않으면, 기본적으로 Model 클래스의 이름을 스네이크 케이스로 변환한 뒤, 복수형을 붙여 사용한다.

만약 Model 이름이 User인경우 users 테이블로, MyBestFriend인 경우 my_best_friends 테이블이 된다.



