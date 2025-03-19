---
title: Laravel 03 - Service Container, Service Provider
description: 서비스컨테이너, 서비스프로바이더 알아보기
author: holymason
categories: [php, laravel]
tags: [php, laravel, 라라벨, mvc, di, service]
---

이 포스트에서는 Laravel의 서비스 컨테이너와 서비스 프로바이더 대해 간단하게 소개합니다.

# Laravel 서비스 컨테이너와 서비스 프로바이더

Laravel은 강력한 의존성 관리 시스템인 **서비스 컨테이너**와 **서비스 프로바이더**를 제공합니다. 이 두 기능은 효율적인 의존성 주입(Dependency Injection) 패턴을 구현하는 데 핵심적인 역할을 합니다.
> **의존성 주입(Dependency Injection)**  
의존성 주입이란 객체 A가 객체 B에 의존해야 할 때, A 내부에서 B를 직접 생성하는 것이 아니라, 외부(IoC 컨테이너 등)에서 B의 인스턴스를 생성하고 이를 A에 주입하는 설계 패턴입니다. 이 방식은 객체 간 결합도를 낮춰 **코드의 유연성과 유지보수성**을 높이며, 코드의 변경이 발생할 경우 최소한의 코드 수정만으로 대응할 수 있습니다.

## 목차
1. [서비스 컨테이너(Service Container)](#1-서비스-컨테이너service-container)
2. [서비스 프로바이더(Service Provider)](#2-서비스-프로바이더service-provider)
3. [예제](#3-예제)
4. [결론](#결론)

## 1. 서비스 컨테이너(Service Container)

서비스 컨테이너는 Laravel 애플리케이션의 핵심으로, 클래스 의존성을 관리하고 의존성 주입을 수행하는 강력한 도구입니다.   
쉽게 말해서 애플리케이션에서 필요한 객체를 자동으로 생성하고 관리해주는 컨테이너라고 볼 수 있습니다.

### 주요 기능

1. **의존성 해결**: 클래스가 필요로 하는 의존성을 자동으로 주입합니다.
2. **인스턴스 바인딩**: 인터페이스와 클래스를 연결하여 느슨한 결합을 유지합니다.
3. **싱글톤 관리**: 애플리케이션 전체에서 단일 인스턴스만 사용하도록 제어합니다.
4. **컨텍스트 바인딩**: 특정 상황에서만 사용할 클래스를 지정할 수 있습니다.

## 2. 서비스 프로바이더(Service Provider)

서비스 프로바이더는 Laravel 애플리케이션의 부팅(bootstrap) 과정에서 중요한 역할을 합니다. 서비스 컨테이너에 바인딩을 등록하고 이벤트 리스너, 미들웨어, 라우트 등을 설정하는 역할을 합니다.

### 주요 기능

1. **서비스 등록**: 서비스 컨테이너에 바인딩을 등록합니다.
2. **부팅 로직**: 애플리케이션이 시작될 때 실행되어야 하는 설정을 처리합니다.
3. **패키지 통합**: 외부 패키지들을 Laravel 애플리케이션에 통합합니다.

### 예시) Laravel 부트스트랩에서의 서비스 컨테이너 
```php
// /bootstrap/app.php (Laravel 10)
$app = new Illuminate\Foundation\Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);
$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);
$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);
$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);
return $app;
```

### 

## 3. 예제
### 3.1. 기본 바인딩

```php
// AppServiceProvider.php 에 기본 바인딩하고,
public function register()
{
    $this->app->bind(\App\Services\PostService::class, function ($app) {
        return new \App\Services\PostService();
    });
}

// App\Http\Controllers\PostController.php 에서 이를 의존성 주입받습니다.
class PostController extends Controller
{
    protected $postService;

    public function __construct(PostService $postService)
    {
        $this->postService = $postService;
    }
}
```
* `bind()`는 요청할 때마다 새 `PostService` 인스턴스를 반환합니다.
* `PostController`에서 `PostService`를 요청하면 컨테이너가 이를 생성해 전달합니다.

### 3.2. 인터페이스와 클래스 바인딩
* 인터페이스와 클래스 바인딩을 사용하면 구체적인 클래스에 직접 의존하지 않고, 인터페이스를 통해 의존성을 주입받을 수 있습니다.
```php
// 인터페이스 생성
namespace App\Contracts;
use App\Models\Post;

interface PostServiceInterface
{
    // PostServiceInterface는 서비스가 제공해야 할 기능을 정의하지만, 구체적인 구현은 없습니다
    public function createPost(array $data): Post;
}

// 인터페이스를 구현하는 서비스 클래스
namespace App\Services;
use App\Contracts\PostServiceInterface;
use App\Models\Post;

class PostService implements PostServiceInterface 
{
    // PostServiceInterface를 따르는 모든 클래스는 같은 방식으로 사용할 수 있습니다.
    public function createPost(array $data): Post
    {
        return Post::create($data);
    }
}

// 서비스 프로바이더(AppServiceProvider.php)에서 바인딩
public function register()
{
    // PostServiceInterface를 요청하면 PostService의 인스턴스를 반환하도록 바인딩합니다.  
    $this->app->bind(\App\Contracts\PostServiceInterface::class, \App\Services\PostService::class);
}

// 컨트롤러에서 사용
class PostController extends Controller
{
    protected $postService;
    public function __construct(PostServiceInterface $postService)
    {
        // 이제 컨트롤러에서 구체적인 클래스가 아닌 인터페이스를 주입받습니다.
        $this->postService = $postService;
    }
}
```
* `bind(인터페이스::class, 구현 클래스::class)`를 사용하여 인터페이스와 클래스를 연결합니다.
* `PostController`에서는 구체적인 클래스가 아닌 `PostServiceInterface`를 의존성으로 받습니다.
* 따라서 나중에 `PostService`를 다른 클래스로 대체해도 컨트롤러 코드를 수정할 필요가 없습니다.

> **구체적인 클래스가 아닌 인터페이스를 바인딩할 때의 장점**  
> * 서비스 교체하기 편함 : `PostService`를 다른 서비스(`NewPostService`)로 변경할 때도 컨트롤러 수정 없이 `AppServiceProvider`에서 바인딩만 변경하면 됩니다.
> * 테스트하기 편함 : `PostServiceInterface`를 `MockPostService` 같은 더미 서비스로 교체하면 테스트가 쉬워집니다.
> * 코드의 유연성과 확장성이 증가 : 다양한 서비스 구현체를 만들어도 `PostController`는 변경할 필요 없어서, 보다 유연하게 작업할 수 있습니다.

### 3.3. 싱글톤 바인딩
* 싱글톤 바인딩을 사용하면 애플리케이션이 실행되는 동안 단 하나의 인스턴스만 유지됩니다.
```php
public function register()
{
    $this->app->singleton(\App\Services\PostService::class, function ($app) {
        return new \App\Services\PostService();
    });
}
```
* `singleton()`을 사용하면 `PostService` 인스턴스가 애플리케이션이 종료될 때까지 유지됩니다.
* 즉, 여러 번 호출해도 같은 인스턴스를 재사용합니다.

### 3.4. 컨텍스트 바인딩
* 특정 클래스에서만 특정 구현을 사용하도록 컨텍스트 바인딩을 설정할 수 있습니다.
* 예를 들어, `PostController`에서는 `PostService`를 사용하고, `AdminPostController`에서는 `AdminPostService`를 사용하도록 설정할 수 있습니다.
```php
namespace App\Services;
use App\Contracts\PostServiceInterface;
use App\Models\Post;

class AdminPostService implements PostServiceInterface
{
    public function createPost(array $data): Post
    {
        return Post::create(array_merge($data, ['is_admin' => true]));
    }
}

// 서비스 프로바이더에서 컨텍스트 바인딩
public function register()
{
    $this->app->when(PostController::class)
              ->needs(PostServiceInterface::class)
              ->give(PostService::class);

    $this->app->when(AdminPostController::class)
              ->needs(PostServiceInterface::class)
              ->give(AdminPostService::class);
}

// 컨트롤러에서 사용
// PostController, AdminPostController에서 같은 인터페이스를 사용하지만,
// 서비스 컨테이너에서 컨트롤러에 맞는 서비스로 바인딩 해줍니다.
class PostController extends Controller
{
    protected $postService;

    public function __construct(PostServiceInterface $postService)
    {
        $this->postService = $postService; // PostService
    }
}

class AdminPostController extends Controller
{
    protected $postService;

    public function __construct(PostServiceInterface $postService)
    {
        $this->postService = $postService; // AdminPostService
    }
}
```

## 결론

Laravel의 서비스 컨테이너와 서비스 프로바이더는 애플리케이션의 코드 구조를 유연하고 확장성 있게하고, 테스트를 편리하게 할 수 있도록 만드는 강력한 도구입니다.  
의존성 주입을 통해 코드의 결합도를 낮추고, 인터페이스를 통한 추상화를 쉽게 구현할 수 있습니다.

1. **코드 재사용성 향상**: 컴포넌트들을 쉽게 재사용할 수 있습니다.
2. **테스트 용이성**: 의존성을 쉽게 모킹(mocking)하여 단위 테스트를 수행할 수 있습니다.
3. **유지보수성 향상**: 코드 간의 결합도를 낮춰 변경 사항이 시스템 전체에 미치는 영향을 최소화합니다.
4. **확장성**: 기존 코드를 수정하지 않고도 새로운 기능을 추가할 수 있습니다.

Laravel의 서비스 컨테이너와 서비스 프로바이더를 효과적으로 활용하면, 유지보수가 용이하고 확장성 있는 코드베이스를 구축할 수 있습니다.
