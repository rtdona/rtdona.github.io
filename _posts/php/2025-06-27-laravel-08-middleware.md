---
title: Laravel 08 - Middleware 요청/응답의 필터
description: 라라벨의 미들웨어 개념과 역할, 실무 적용 예제 및 활용 팁 정리
author: holymason
categories: [php, laravel, architecture]
tags: [php, laravel, middleware, http, architecture]
---

이 포스트에서 Laravel의 **미들웨어(Middleware)** 에 대해서 알아보겠습니다.   
이전 [포스트](/posts/laravel-07-event-listener-observer/)에서 알아보았던 **Event/Listener, Observer**가 데이터와 로직의 흐름을 잡았다면,  
**미들웨어(Middleware)** 는 요청(Request)과 응답(Response)의 **입구와 출구** 를 통제하는 핵심 역할입니다.

---

# Laravel Middleware – 요청/응답의 필터

---

## 목차

1. [Middleware란?](#1-middleware란)
2. [Middleware의 주요 역할](#2-middleware의-주요-역할)
3. [Event/Observer와 Middleware의 차이](#3-eventobserver와-middleware의-차이)
4. [Middleware 예제](#4-middleware-예제)
5. [Middleware 등록과 적용 방법](#5-middleware-등록과-적용-방법)
6. [팁 & 주의할 점](#6-팁--주의할-점)
7. [결론](#결론)

---

## 1. Middleware란?

- **Middleware**는 HTTP **요청(Request)** 과 **응답(Response)** 사이에 위치하여,  
  모든 트래픽이 반드시 통과하는 **필터** 역할을 합니다.
- 컨트롤러(혹은 라우트)에 도달하기 전에, 또는 응답이 사용자에게 전달되기 전에  
  **요청/응답을 검사하거나 가공하고 필터링**할 수 있습니다.

> 미들웨어는 애플리케이션에 들어오는 HTTP 요청을 검사하고 필터링하는 편리한 메커니즘을 제공합니다. 예를 들어, Laravel에는 애플리케이션의 사용자가 인증되었는지 확인하는 미들웨어가 포함되어 있습니다. 사용자가 인증되지 않은 경우 미들웨어는 사용자를 애플리케이션의 로그인 화면으로 리디렉션합니다. 그러나 사용자가 인증된 경우 미들웨어는 요청을 애플리케이션으로 더 진행할 수 있도록 허용합니다.  
> https://laravel.com/docs/11.x/middleware#main-content
---

## 2. Middleware의 주요 역할

- **인증/권한 체크**  
  ex) 로그인하지 않은 사용자는 로그인 페이지로 리다이렉트
- **요청 데이터 가공/전처리**  
  ex) input trimming, XSS 필터, request 데이터 치환
- **API Key, Token 등 인증 검증**  
  ex) `X-API-KEY` 헤더 검증, JWT 토큰 파싱
- **로깅/모니터링**  
  ex) 요청/응답 로그 기록, 응답 시간 측정
- **Rate Limiting, Throttling**  
  ex) 과도한 요청 제한
- **CORS/헤더 조작 차단**  
  ex) cross-origin 요청 허용/차단, custom header 삽입

---

## 3. Event/Observer와 Middleware의 차이

| 구분           | Middleware                 | Event/Listener & Observer            |
|----------------|---------------------------|--------------------------------------|
| **대상**       | HTTP 요청/응답            | 데이터/비즈니스 로직                 |
| **실행시점**   | 컨트롤러(또는 응답) 전후  | 모델의 상태 변화, 특정 로직 발생 시  |
| **적용 범위**  | 전역/라우트/그룹 단위      | 특정 로직, 모델 단위                |
| **역할**       | 트래픽 흐름 제어, 필터링   | 데이터/업무 처리 로직 분리           |
| **예시**       | 인증, 로깅, 키검증, CORS   | 주문 생성 알림, 게시글 생성 로그     |

---


## 4. Middleware 예제

### 4.1. 커스텀 미들웨어 생성

`php artisan make:middleware` 명령어로 미들웨어를 생생하면,  
`App\Middelware` 경로에 미들웨어 클래스가 생성됩니다

```bash
php artisan make:middleware CheckApiKey
```

### 4.2. 미들웨어 예시 01 - 요청 헤더 검증

```php
// app/Http/Middleware/CheckApiKey.php
// 요청 헤더의 X-API-KEY를 검증하는 미들웨어
public function handle($request, Closure $next)
{
    $apiKey = $request->header('X-API-KEY');
    if ($apiKey !== config('services.my_api.key')) {
        return response()->json(['message' => 'Invalid API Key'], 401);
    }
    
    // 다음 단계(컨트롤러 등)로 요청 전달
    return $next($request);
}
```

### 4.3. 미들웨어 예시 02 - 게시글 작성/수정/삭제 전 로그인 여부/권한 확인

사용자가 로그인하지 않은 상태에서 게시글(Post) 또는 댓글(Comment) 관련 API로 요청을 할 때,  
로그인 페이지 또는 401 에러로 리다이렉트/응답

```php
// app/Http/Middleware/EnsureOwner.php
// 게시글 수정/삭제 요청이 들어올 때, 게시글 작성자인지 검증하는 미들웨어
public function handle($request, Closure $next)
{
    $user = auth()->user();
    $route = $request->route();

    // 인증되지 않은 사용자 체크
    if (!$user) {
        return response()->json(['message' => '로그인이 필요합니다.'], 401);
    }
        
    // 게시글의 경우: {post}
    if ($route->parameter('post')) {
        $post = $route->parameter('post');
          if ($post->user_id !== $user->id) {
            return response()->json(['message' => '수정/삭제 권한이 없습니다.'], 403);
        }
    }

    // 댓글의 경우: {comment}
    if ($route->parameter('comment')) {
        $comment = $route->parameter('comment');
        if ($comment->user_id !== $user->id) {
            return response()->json(['message' => '수정/삭제 권한이 없습니다.'], 403);
        }
    }

    return $next($request);
}

```

### 4.3. 미들웨어 예시 03 - Response 래핑 

API 응답을 일관된 형태로 래핑(wrapping)하거나  
응답 직전에 API 요청과 응답을 로깅하는 미들웨어로 사용할 수 있습니다

```php
// app/Http/Middleware/ApiResponseWrapper.php
public function handle($request, Closure $next)
{
    // 1. 요청 정보 로깅하기
    $this->logRequest($request);
        
    // 2. 요청을 다음 미들웨어/컨트롤러로 전달하고 응답 받기
    $response = $next($request);

    // 3. JSON 응답인 경우에만 처리
    if ($response instanceof \Illuminate\Http\JsonResponse) {
        // 4. 기존 응답 데이터 가져오기
        $original = $response->getData();
         
        // 5. 일관된 형태로 래핑
        $wrapped = [
            'status'  => $response->getStatusCode(),
            'data'    => $original->data ?? $original,
            'message' => $original->message ?? null,
        ];
        
        // 6. 래핑된 데이터로 응답 재설정
        $response->setData($wrapped);
    }
    
    // 7. 응답 정보 로깅
    $this->logResponse($request, $response);

    return $response;
}
private function logRequest(Request $request){...}
private function logResponse(Request $request){...}

```

**`ApiResponseWrapper` 미들웨어 적용 전**

```php
// PostController
public function show($id) 
{
    $user = Post::find($id);
    return response()->json($user);
}
// Response
{
    "id": 1,
    "title": "라라벨 미들웨어란?",
    "content": "미들웨어란 HTTP 요청과 응답사이에 위치하여, 모든 트래픽이 반드시 통과하는 필터 역할을 합니다."
}
```

**`ApiResponseWrapper` 미들웨어 적용 후**

```php
{
    "status": 200,
    "data": {
        "id": 1,
        "title": "라라벨 미들웨어란?",
        "content": "미들웨어란 HTTP 요청과 응답사이에 위치하여, 모든 트래픽이 반드시 통과하는 필터 역할을 합니다."
    },
    "message": null
}
```


---

## 5. Middleware 등록과 적용 방법

### 5.1. 미들웨어 등록

**Laravel 10 버전 이하**

`app/Http/Kernel.php` 파일에 등록하여 사용
* `protected $middleware = [...];` : 전역(글로벌) 미들웨어
* `protected $middlewareGroups = [...];` : 라우트 그룹별 미들웨어
* `protected $middlewareAliases = [...];` : 라우트별 미들웨어 별칭

```php
// app/Http/Kernel.php
protected $routeMiddleware = [
    // ...
    'check_api_key' => \App\Http\Middleware\CheckApiKey::class,
    'owner.only'     => \App\Http\Middleware\EnsureOwner::class,
];
```

**Laravel 11 버전 이상**

`bootstrap/app.php` 파일에 등록하여 사용
```php
// bootstrap/app.php
return Application::configure()
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // 전역(글로벌) 미들웨어
        $middleware->append([
            \App\Http\Middleware\LogRequests::class,
        ]);

        // 라우트 web 그룹 미들웨어
        $middleware->group('web', [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Session\Middleware\StartSession::class,
        ]);
        
        // 라우트 api 그룹 미들웨어
        $middleware->group('api', [
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            'throttle:api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ]);

        // 미들웨어 별칭 등록
        $middleware->alias([
            'check_api_key' => \App\Http\Middleware\CheckApiKey::class,
            'ensure_owner' => \App\Http\Middleware\EnsureOwner::class,
            'api.wrapper' => \App\Http\Middleware\ResponseWrapperWithLogging::class,
        ]);

        // 특정 조건부 미들웨어
        $middleware->web(append: [
            \App\Http\Middleware\HandleInertiaRequests::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })->create();
```

```php
// app/Http/Kernel.php
protected $routeMiddleware = [
    // ...
    'check_api_key' => \App\Http\Middleware\CheckApiKey::class,
    'owner.only'     => \App\Http\Middleware\EnsureOwner::class,
];
```

### 5.2. 미들웨어 적용

- **라우트에 직접 적용하는 방법**

```php
Route::middleware(['auth.user'])->group(function () {
    // 게시글 생성, 수정, 삭제
    Route::post('/posts', 'PostController@store');
    Route::put('/posts/{post}', 'PostController@update')->middleware('owner.only');
    Route::delete('/posts/{post}', 'PostController@destroy')->middleware('owner.only');
    
    // 댓글 작성 (게시글 조건 체크)
    Route::post('/posts/{post}/comments', 'CommentController@store')->middleware('post.check');
    
    // 댓글 수정/삭제
    Route::put('/comments/{comment}', 'CommentController@update')->middleware('owner.only');
    Route::delete('/comments/{comment}', 'CommentController@destroy')->middleware('owner.only');
});
```

`->middleware(['owner.only', 'check_api_key']);` 와 같이 미들웨어를 여러개 적용할 수도 있습니다.

- **컨트롤러에 적용하는 방법**

```php
class ApiController extends Controller
{
    public function __construct()
    {
        // ->middleware('check_api_key') : 이 컨트롤러에 적용할 미들웨어
        // ->only('profile'); : 미들웨어를 특정 메소드에만 적용하기 
        $this->middleware('check_api_key')->only('profile');
    }
}
```

---

## 6. 팁 & 주의할 점

### 6.1. 미들웨어 활용 팁

* **전역, 그룹, 라우트별로 유연하게 적용**
  - 인증/로깅/언어설정 등은 전역/그룹에 적용, 특정 요청을 검증하는 미들웨어는 라우트별로 적용
* **미들웨어 체이닝 가능**
  - 여러 미들웨어를 배열로 연결하여 **순차적 실행** 가능
* **API와 Web, 사용자/관리자 등 분리**
  - 그룹 미들웨어를 활용해서 구역별 정책 차등 적용
* **커스텀 Exception 처리/로깅**
  - 미들웨어 내에서 예외 throw/로깅하여 이슈 빠르게 추적

### 6.2. 주의할 점

* **미들웨어가 너무 복잡해지지 않도록, 책임 분리**
  - 하나의 미들웨어에는 한 가지 역할만, SRP(단일 책임 원칙) 지키기
* **request 데이터 직접 수정 주의**
  - input 가공은 가급적 clone 후 수정, 혹은 FormRequest와 조합하여 사용

---

## 결론

* **미들웨어는 모든 HTTP 요청/응답의 입구와 출구를 통제**하는 핵심 기능
* 인증, 로깅, 키 검증, 응답 가공 등 트래픽 흐름에 필요한 거의 모든 처리를 미들웨어에 집중하여 처리 가능

### 참고 링크

* [공식 문서 - Laravel Middleware](https://laravel.com/docs/middleware)
* [공식 문서 - Request Lifecycle](https://laravel.com/docs/lifecycle)

---
