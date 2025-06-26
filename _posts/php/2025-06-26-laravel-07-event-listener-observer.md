---
title: Laravel 07 - Event & Listener, Observer 패턴
description: 라라벨의 이벤트 시스템과 옵저버 패턴의 원리와 예제
author: holymason
categories: [php, laravel, architecture]
tags: [php, laravel, event, listener, observer, design-pattern]
---

이 포스트에서는 Laravel의 **이벤트(Event) & 리스너(Listener)**,  
그리고 **옵저버(Observer) 패턴**에 대해 알아보겠습니다.

# Laravel Event & Listener, Observer 패턴 완벽 가이드

---

## 목차

1. [Event & Listener란?](#1-event--listener란)
2. [Observer 패턴이란?](#2-observer-패턴이란)
3. [Event와 Observer의 차이점](#3-event와-observer의-차이점)
4. [Event & Listener 사용 예제](#4-event--listener-사용-예제)
5. [Observer 패턴 실전 예제](#5-observer-패턴-실전-예제)
6. [팁 & 주의할 점](#6-팁--주의할-점)
7. [결론](#결론)

---

## 1. Event & Listener란?

- **Event(이벤트)**:  
  “특정 상황이 발생했다”는 **신호**를 애플리케이션 전체에 알리는 객체입니다.  
  (ex: 회원 가입, 주문 생성, 게시글 작성 등)
- **Listener(리스너)**:  
  해당 Event가 발생했을 때, **실제 동작(업무 처리)** 을 담당하는 객체(핸들러)입니다.

> **예시:**  
> “회원이 가입하면(이벤트), 환영 이메일을 보내거나 포인트를 지급(리스너)”  
>  
> 여러 개의 Listener가 하나의 Event를 구독할 수 있고,  
> 하나의 Listener가 여러 Event에 반응할 수도 있습니다.

---

## 2. Observer 패턴이란?

- **Observer(옵저버) 패턴**은  
  **특정 모델의 상태 변화(생성, 수정, 삭제 등)를 관찰**하다가,  
  변화가 감지되면 자동으로 정해진 동작을 실행하는 패턴입니다.
- Laravel에서는 주로 **Eloquent 모델과 연계**해 사용합니다.

> **예시:**  
> 게시글(Post)이 생성될 때마다, 자동으로 로그를 남기거나 관리자에게 Push 알림 전송  
> 게시글이 삭제될 때, 게시글에 포함된 댓글(Comment)도 함께 삭제처리 
---

## 3. Event와 Observer의 차이점

|              | Event & Listener   | Observer                    |
|--------------|--------------------|-----------------------------|
| 대상         | 상황/행위 발생  | Model 상태의 변화 (생성, 수정, 삭제 등) |
| 확장성       | 여러 리스너/큐/브로드캐스트 지원 | 하나의 Model과 직접 연결            |
| 사용 위치    | 서비스/비즈니스/프로젝트 전체   | Eloquent 모델 주변(생명주기)        |
| 예시         | 주문 생성시 SMS, 이메일 발송 등 | 게시글 생성시 자동 푸시 알림 등          |

---

## 4. Event & Listener 사용 예제

- **User**가 **Post**를 작성할 때,  
  - Post 작성 성공 시 Comment 모델에 첫 댓글을 자동으로 생성하는 예시입니다.
  - 즉, `PostCreated` 이벤트 → `CreateFirstComment` 리스너가 동작하는 구조입니다.

### 4.1. Event/Listener 클래스 생성

`php artisan make` 명령어로 생생하면,  
각각 `App\Events`, `App\Listeners` 경로에 이벤트와 리스너 클래스가 생성됩니다

```bash
php artisan make:event PostCreated
php artisan make:listener CreateFirstComment
````

---

### 4.2. Event/Listener 등록

`app/Providers/EventServiceProvider.php`
> EventServiceProvider는
Event가 발생했을 때, 어떤 Listener(핸들러)들이 실행되어야 하는지를 정의하는 서비스 프로바이더입니다.
> [Laravel Docs](https://laravel.com/docs/11.x/events#registering-events-and-listeners)


```php
protected $listen = [
    \App\Events\PostCreated::class => [
        \App\Listeners\CreateFirstComment::class,
    ],
];
```

---

### 4.3. Event Dispatch & Listener 구현

**1) PostController 등에서 Post 생성 후 Event 발생시키기**

`event(new PostCreated($post));` 또는 `PostCreated::dispatch($post);` 메소드로 Event를 발생시키면,  
EventServiceProvider에 등록된 Listener들이 자동 실행됩니다.

```php
use App\Events\PostCreated;

public function store(Request $request)
{
    $post = Post::create([
        'user_id' => auth()->id(),
        'title' => $request->input('title'),
        'content' => $request->input('content'),
    ]);

    // PostCreated 이벤트를 발생시킬 때 post 모델을 페이로드로 넘겨줍니다
    event(new PostCreated($post));
    // PostCreated::dispatch($post); 

    return redirect()->route('posts.show', $post);
}
```

**2) Event 클래스 구현**

`app/Events/PostCreated.php`

```php
namespace App\Events;

use App\Models\Post;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class PostCreated
{
    use Dispatchable, SerializesModels;

    public $post;

    public function __construct(Post $post)
    {
        $this->post = $post;
    }
}
```

**3) Listener 클래스 구현**

`app/Listeners/CreateFirstComment.php`

```php
namespace App\Listeners;

use App\Events\PostCreated;
use App\Models\Comment;

class CreateFirstComment
{
    public function handle(PostCreated $event)
    {
        // 게시글에 자동으로 첫 댓글 작성
        Comment::create([
            'post_id' => $event->post->id,
            'user_id' => $event->post->user_id, // 댓글작성자를 게시글 작성자로 설정
            'content' => '게시글에 관련한 문의는 댓글로 남겨주세요!',
        ]);
    }
}
```

---

### 4.5. 하나의 Listener에서 여러 개의 Event를 처리

하나의 Listener 클래스가 여러 Event에 매핑되어 있을 때, 각각의 이벤트에서 Listener의 handle() 메서드로 이벤트 객체가 주입됩니다.  
이 때 handle() 내부에서 이벤트 객체의 타입(클래스명) 또는 전달된 데이터로 분기 처리할 수 있습니다.

**1) EventServiceProvider에 등록**

```php
protected $listen = [
    \App\Events\PostCreated::class => [
        \App\Listeners\MultiEventListener::class,
    ],
    \App\Events\CommentCreated::class => [
        \App\Listeners\MultiEventListener::class,
    ],
];
```

**2) Listener 클래스 예시**
```php
namespace App\Listeners;

use App\Events\PostCreated;
use App\Events\CommentCreated;

class MultiEventListener
{
    public function handle($event)
    {
        if ($event instanceof PostCreated) {
            // PostCreated 이벤트 처리
            \Log::info('[Event] 게시글 생성: ' . $event->post->id);
        } elseif ($event instanceof CommentCreated) {
            // CommentCreated 이벤트 처리
            \Log::info('[Event] 댓글 생성: ' . $event->comment->id);
        }
    }
}
```


### 4.6. 이벤트 끄기(Muting Events)

Eloquent 모델에서 어떤 동작(ex: save(), update(), delete() 등)을 할 때,  
이벤트(created, updated, deleting 등)가 자동으로 발생하는 것을 일시적으로 **비활성화** 하는 기능입니다.

**Post 모델의 withoutEvents 예시**

```php
$post->saveQuietly();
$post->updateQuietly(['title' => '변경']);
$post->deleteQuietly();

Post::withoutEvents(function () {
    // 이 블록 안에서 Post 모델의 이벤트만 꺼집니다.
    // ex: 대량 게시글 업데이트, 삭제, 생성
    Post::where('status', 'draft')->update(['status' => 'published']);
    Post::create([
        'title' => '이벤트 없이 생성',
        'content' => 'Observer 또는 Listener가 실행되지 않습니다.',
    ]);
});
```


**모든 모델의 withoutEvents 예시**

```php
use Illuminate\Support\Facades\Event;

Event::withoutEvents(function () {
    // 이 블록 안에서는 모든 Eloquent 이벤트가 발생하지 않기 때문에
    // 대량 데이터 이관, 마이그레이션 등에 유용
    Post::create([...]);
    User::all()->each->delete();
});

```

---

## 5. Observer 패턴 실전 예제

### 5.1. Observer 란?
Observer는 "관찰자"라는 의미 그대로 특정 Model의 상태 변화(생성, 수정, 삭제 등)를 자동으로 관찰하여, 
상태가 변화할 때마다 미리 지정한 작업을 자동으로 실행하는 구조입니다.

Laravel에서는 Eloquent Model과 통합되어 Model의 라이프사이클 이벤트를 간편하게 감지하고 처리할 수 있습니다.

### 5.1. 옵저버 클래스 생성

`php artisan make` 명령어로 생생하면 `App\Observers` 경로에 옵저버 클래스가 생성됩니다

```bash
php artisan make:observer PostObserver --model=Post
```

### 5.2. 옵저버 구현

`app/Observers/PostObserver.php`

```php
namespace App\Observers;

use App\Models\Post;

// artisan 명령어로 생성하면 아래의 5개의 메소드가 자동으로 생성됩니다
// created, updated, deleted, restored, forceDeleted

class PostObserver
{
    public function created(Post $post)
    {
        // 게시글 생성 시 자동 slug 생성
        $post->slug = Str::slug($post->title) . '-' . $post->id;
        $post->saveQuietly();
        // saveQuietly()는 Laravel Eloquent 모델에서 이벤트(ex: saving, saved 등)를 발생시키지 않고, 단순히 값만 저장하는 메서드입니다.

    }

    public function deleting(Post $post)
    {
        // 게시글 삭제 시, 게시글에 달린 댓글도 삭제
        $post->comments()->delete();
    }
}
```

기본 메소드 외에 활용할 수 있는 이벤트 메소드도 있습니다

| 이벤트 메소드          | 실행 시점/설명              | 대표적인 이벤트 사용 예시                |
|------------------| --------------------- |-------------------------------|
| **creating**     | 생성 **직전**             | 필드 자동 값 세팅, UUID/slug 생성      |
| **created**      | 생성 **직후**             | 알림, 통계, 로그 기록, 연관 모델 자동 생성    |
| **updating**     | 수정 **직전**             | 수정 전 값 체크/변환, 변경사항 감지         |
| **updated**      | 수정 **직후**             | 수정 후 알림, 로그, 외부 API 연동        |
| **saving**       | 저장(생성/수정) **직전**      | 데이터 일괄 전처리, 필드값 자동 보정         |
| **saved**        | 저장(생성/수정) **직후**      | 저장 후 알림, 로그 기록, 캐시 갱신         |
| **deleting**     | 삭제 **직전**             | 연관 데이터 삭제, soft delete 조건 체크  |
| **deleted**      | 삭제 **직후**             | 삭제 후 로그, 통계 처리, 파일/첨부 외부 삭제   |
| **restoring**    | soft delete 복구 **직전** | 복구 전 유효성 체크, 복구 제한 로직         |
| **restored**     | soft delete 복구 **직후** | 복구 후 알림, 복구 관련 후처리, 연관 데이터 복구 |
| **forceDeleting** | 영구 삭제 **직전**          | 완전 삭제 전 데이터 백업, 외부 시스템 동기화    |
| **forceDeleted** | 영구 삭제 **직후**          | 파일 등 실제 물리적 데이터 제거, 최종 로그 기록  |



### 5.3. 옵저버 등록

`AppServiceProvider` 또는 전용 ServiceProvider의 `boot()`에 추가하면,  
이제 Post 모델의 created, deleting 등 라이프사이클 이벤트에서 자동으로 PostObserver의 메서드가 실행됩니다.

```php
use App\Models\Post;
use App\Observers\PostObserver;

public function boot()
{
    Post::observe(PostObserver::class);
}
```

---

## 6. 팁 & 주의할 점

### 6.1. Event & Listener 활용할 때의 팁

* **비즈니스 이벤트(주문 생성, 결제 완료 등)는 Event & Listener 에 분리**
  * 여러 후처리를 동시에 확장, 분리, 비동기(Queue)로 쉽게 관리
  * ex: 주문 생성 → 적립금 지급, 이메일 발송, 알림 등 각각의 리스너로 처리
* **Listener에선 단일 책임 원칙(SRP)을 준수할 것**
  * 한 리스너가 너무 많은 역할을 가지지 않도록
  * 이메일 발송용, 슬랙 알림용 등 명확히 분리
* **비동기(Queue) 리스너 적극 활용**
  * 리스너 클래스에서 `implements ShouldQueue`만 추가하면, 별도 Queue 작업으로 대용량/지연 없는 이벤트 처리 가능
* **이벤트 페이로드(전달 데이터)는 최소화, 꼭 필요한 데이터만 넘기기**
  * 너무 많은 모델/컬렉션을 통째로 넘기면 메모리·성능 저하 가능

### 6.2. Event & Listener 활용할 때 주의할 점 

* **Listener 내부에서 예외/에러가 발생하면 전체 이벤트 실행에 영향 줄 수 있음**
  * try-catch, 실패 로깅 등 리스너 내부에서 적절한 예외처리 필요
* **Event를 과도하게 남용하는 것은 지양**
  * 단순한 후처리는 컨트롤러/서비스에서, 복잡한/여러 작업이 필요한 경우만 이벤트로 분리하여 관리

### 6.3. Observer 활용할 때의 팁

* **반복되거나 자동화가 필요한 Model 후처리는 Observer에 위임**
  * slug 생성, 연관 데이터 자동 삭제 등*
* **Observer에서 비즈니스 로직 대신, 모델 관련 후처리만 구현**
  * 복잡한 업무 규칙은 Service/Event/Listener로 분리하여 관리하고, Observer는 자동화된 데이터 처리를 중심으로 관리
* **생성·삭제 직전(creating, deleting)과 직후(created, deleted) 활용 구분**
  * DB의 외래키 등을 고려하여 연관 데이터 삭제 등은 deleting, 로그/알림 등은 deleted에 작성

### 6.4. Observer 활용할 때의 주의할 점

* **Observer에서 예외 발생시 전체 트랜잭션이 롤백될 수 있음**
  * 예외 처리/로깅 필수, 외부 API 호출은 Observer보단 이벤트로 분리 권장
* **Observer에서도 단일 책임 원칙(SRP)을 준수할 것**
  * 하나의 옵저버에 1~2개의 역할만 권장
* **무한루프 주의**
  * Observer 내에서 save(), update() 등을 호출하면 재귀호출로 무한 반복될 수 있으니, 반드시 saveQuietly() 또는 적절한 조건문으로 처리 필요

---

### 참고 문서

* [Laravel 공식: Events & Listeners](https://laravel.com/docs/events)
* [Laravel 공식: Eloquent Observers](https://laravel.com/docs/eloquent#observers)
* [라라벨코리아: 옵저버 활용 팁](https://laravel.kr/docs/8.x/eloquent#observers)

---

**실무적으로 언제 Event/Listener와 Observer를 쓰는 게 좋은지,
상호 활용/조합 전략**이 궁금하다면 추가로 설명해드릴 수 있습니다!


---

## 결론

* **Event & Listener**:
  애플리케이션 전반의 "비즈니스 이벤트" 확장/분리/비동기화에 매우 강력한 기능
* **Observer**:
  Eloquent 모델의 상태 변화에 자동으로 동작하기 때문에, 코드 간결화와 자동화에 효율적

### 참고 링크
* [공식 문서 - Laravel Events](https://laravel.com/docs/events)
* [공식 문서 - Laravel Observers](https://laravel.com/docs/eloquent#observers)

