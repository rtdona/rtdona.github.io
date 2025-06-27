---
title: Laravel 09 - Queue & Job 
description: 라라벨 큐와 잡(Queue & Job) 시스템의 원리와 예시
author: holymason
categories: [php, laravel, architecture]
tags: [php, laravel, queue, job, 비동기, 대기열, 이벤트]
---

이번 포스트에서는 Laravel의 **Queue & Job 시스템** 을 통해  
애플리케이션에서의 **비동기 처리**와 **대량 작업 처리 예시** 를 알아보겠습니다.

앞선 포스트에서 알아보았던 이벤트(Event), 리스너(Listener), 옵저버(Observer)에서  
후처리 로직을 즉시(동기)로 처리할지, 또는 큐(Job)를 활용해 비동기 처리할지  
서비스의 규모와 요구사항에 따라 유연하게 선택할 수 있습니다.


---

# Laravel Queue & Job

---

## 목차

1. [Queue & Job이란?](#1-queue--job이란)
2. [Queue가 필요한 이유](#2-queue가-필요한-이유)
3. [Queue의 동작 순서](#3-queue의-동작-순서)
4. [User/Post/Comment 모델 활용 실전 예제](#4-userpostcomment-모델-활용-실전-예제)
5. [팁 & 주의할 점](#5-팁--주의할-점)
6. [결론](#결론)

---

## 1. Queue & Job이란?

- **Queue(큐, 대기열)**:
  - Job들을 저장하고 순차적/비동기적으로 처리하는 대기열
  - 즉시 실행하지 않고, 일단 대기열에 쌓아두었다가 워커(worker)가 하나씩 꺼내 실행하는 구조
- **Job(잡, 작업)**:
  - 비동기적으로 실행할 작업을 정의한 클래스
  - 큐에 쌓이는 실행 단위 (ex: 메일 발송, 푸시 알림, 대량 데이터 가공, 외부 API 호출 등)

즉시 처리(동기)하면 서비스가 느려질 작업을 일단 대기열에 올려두고   
백그라운드에서 별도의 워커가 순차적으로 처리하는 방식

> 웹 애플리케이션을 구축하는 동안 업로드된 CSV 파일을 구문 분석하고 저장하는 등 **일반적인 웹 요청 중에 수행하는 데 시간이 너무 오래 걸리는 작업**이 있을 수 있습니다
> 다행히도 Laravel을 사용하면 백그라운드에서 처리할 수 있는 대기 중인 작업을 쉽게 만들 수 있습니다.  
> 시간 집약적인 작업을 대기열(Queue)로 이동하면 **애플리케이션이 웹 요청에 빠른 속도로 응답**하고   
> 고객에게 더 나은 사용자 경험을 제공할 수 있습니다. [출처: Laravel Docs](https://laravel.com/docs/11.x/queues)


---

## 2. Queue가 필요한 이유

- **대량의 데이터 처리**
  - ex: 수천 명에게 메일/푸시 발송, 업로드된 이미지의 썸네일 파싱, 대용량 파일의 분석/변환 등
- **외부 서비스/네트워크 요청**
  - ex: 내부 로직에서 외부 API와 통신할 때, API의 응답 대기 때문에 메인 트래픽이 막히지 않도록
- **성능, 확장성 보장**
  - 애플리케이션의 규모와 트래픽이 커질수록 대용량 작업과 데이터의 후처리를 별도 워커로 처리하여 서버의 부하를 안정적으로 분산

**사용 예시**
- 게시글(Post) 작성 시, 관련 사용자(User)에게 알림(푸시) 보내기
- 댓글(Comment) 작성 시, 게시글 작성자에게 알림
- 사용자가 게시글에 업로드한 이미지의 후처리 (CDN 업로드, 리사이징, 썸네일 추출, 유해성 검증 등) 

---

## 3. Queue의 동작 순서

**동작 순서**

1. **Job 생성 및 Queue 로 보냄**  
   - 컨트롤러, 이벤트, 리스너, 혹은 서비스 내부에서  
     `Job::dispatch($data)` 식으로 Job 객체를 생성해서 큐에 저장
   - 이때 **Job 클래스**는 반드시 `ShouldQueue` 인터페이스를 구현해야 큐에 쌓임

2. **큐 대기열(Queue)에 저장**  
   - `.env`에서 설정한 큐 드라이버(database/redis/sqs 등)에 따라  
     Job이 저장됨  
   - DB 드라이버의 경우 `jobs` 테이블에 레코드로,  
     Redis라면 리스트 형태로 저장

3. **워커(Worker) 실행**  
   - `php artisan queue:work` 명령어나 Supervisor 등 프로세스 관리자가 큐에 새 Job이 쌓였는지 실시간 감시
   - 새 Job이 발견되면 워커가 Job을 하나 꺼내어 실행

4. **Job 실행 및 결과 처리**
   - 워커는 Job의 `handle()` 메소드를 실행
   - 성공 시: 큐에서 해당 Job이 삭제됨
     - Queue 드라이버를 database로 사용하면 `jobs` 테이블에 Job이 추가됨
   - 실패 시:  
     - 정해진 횟수만큼 재시도(`attempts`) 후  
       그래도 실패하면 `failed_jobs` 테이블(DB 기준)에 기록  
     - 필요시 `queue:retry`로 재시도하거나, 로깅/알림 처리

5. **모니터링 및 관리**
   - 라라벨 공식 [Horizon](https://laravel.com/docs/11.x/horizon) 등으로 큐 상태, 처리 속도, 실패 Job 모니터링 가능

---

## 4. User/Post/Comment 모델 활용 실전 예제

### 4.1. 게시글(Post) 작성 시, 팔로워들에게 알림을 큐로 비동기 발송
 
유저가 게시글을 새로 작성하면, 작성자를 팔로우하는 모든 유저에게 "새 게시글 알림"을 푸시 또는 메일로 발송

#### 1) Job 클래스 생성

```bash
php artisan make:job NotifyFollowersOfNewPost
````

#### 2) Job 클래스 구현

```php
// app/Jobs/NotifyFollowersOfNewPost.php

namespace App\Jobs;

use App\Models\Post;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class NotifyFollowersOfNewPost implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $post_id; // serialize 안전하게 Post ID만 저장

    public function __construct($post_id)
    {
        $this->post_id = $post_id;
    }

    public function handle()
    {
        $post = Post::find($this->post_id);
        if (!$post) return;

        // 예시: 작성자의 팔로워 모두에게 알림 전송
        $followers = $post->user->followers; // User 모델에 followers 관계(hasMany)
        foreach ($followers as $follower) {
            // 각 팔로워마다 알림 생성
            $follower->notify(new \App\Notifications\NewPostNotification($post));
        }
    }
}
```

`\App\Notifications\NewPostNotification` 클래스는 라라벨의 Notification 시스템을 이용해  
메일, SMS, 슬랙, 푸시알림 등 다양한 방법으로 유저에게 메시지를 보내는 알림(Notify) 클래스입니다.  
Notification에 대해서는 다음 포스트에서 알아보겠습니다.

#### 3) 컨트롤러에서 Job 디스패치

```php
use App\Jobs\NotifyFollowersOfNewPost;

public function store(Request $request)
{
    $post = Post::create([
        'user_id' => auth()->id(),
        'title'   => $request->input('title'),
        'content' => $request->input('content'),
    ]);

    // Job을 비동기로 큐에 등록
    NotifyFollowersOfNewPost::dispatch($post->id);

    return redirect()->route('posts.show', $post);
}
```

### 4.2. 대량 데이터 후처리 (예: 유저별 게시글/댓글 통계 집계)

전체 유저의 게시글/댓글 수를 정기적으로 집계하여 Users 테이블의 데이터와 동기화

#### 1) Job 클래스 생성

```bash
php artisan make:job CalculateUserPostStats
```

#### 2) Job 클래스 구현

```php
// app/Jobs/CalculateUserPostStats.php

namespace App\Jobs;

use App\Models\User;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class CalculateUserPostStats implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function handle()
    {
        // User 전체 데이터를 100개 레코드씩 나눠서 읽어오는 메소드
        User::chunk(100, function ($users) {
            foreach ($users as $user) {
                $postCount = $user->posts()->count();
                $commentCount = $user->comments()->count();

                // 예: 통계 필드에 결과 저장
                $user->update([
                    'post_count' => $postCount,
                    'comment_count' => $commentCount,
                ]);
            }
        });
    }
}
```

주기적인 스케줄링(crontab)으로 이 Job을 생성하여, 정기적으로 User 모델을 업데이트

> `chunk()`는 Eloquent ORM 에서 대량의 데이터를 조금씩 나눠서 읽어오는 메소드입니다.  
> 만약 User::all() 처럼 한 번에 전체 유저를 가져오게되면, 한 번에 모든 레코드를 메모리에 올리게되므로   
> 메모리 부족, 서버 과부하 에러가 발생할 수 있습니다.  
> 따라서, 대용량 데이터를 처리할 때에는 chunk() 사용이 필수적입니다.


### 4.3. 큐/워커 설정 & 실행

**1) .env 설정**

```
QUEUE_CONNECTION=database
```

**2) 큐 테이블 생성**  

```bash
php artisan queue:table
php artisan migrate
```

**3) 워커 실행**  
워커를 실행한 프로세스가 유지될 때에만 동작합니다
```bash
php artisan queue:work
```

**4) 백그라운드에서 상시 실행**  
Supervisor, systemd 등으로 백그라운드에서 워커가 계속 동작하도록 설정할 수 있습니다

---

## 5. 팁 & 주의할 점

### 5.1. Job 설계 및 데이터 관리

* **Eloquent 모델 전체보다 ID만 저장**
  Job에 모델 인스턴스 전체를 직접 넘기기보다는, 꼭 필요한 식별자(예: user_id, post_id 등)만 저장하고 handle()에서 모델을 다시 조회하여 처리하는 것이 안전
  (serialize/unserialize 에러 방지, DB 상태 동기화)

### 5.2. 실패/재시도/모니터링

* **실패 기록/재시도**

  * `php artisan queue:failed-table`로 failed_jobs 테이블 생성
  * Job 실패 시 자동 기록, `php artisan queue:retry`로 재시도 가능
  * Job 클래스에 `failed()` 메소드를 오버라이드해 후처리(알림 등) 구현

* **Larvel Horizon 패키지로 관리/모니터링**

  * [Laravel Horizon](https://laravel.com/docs/11.x/horizon)

### 5.3. 성능/실행 전략

* **대량 데이터 작업은 `chunk()` 활용**
  * 수천~ 이상의 레코드를 처리 할 때 `chunk()`로 메모리/속도 최적화
* **Job 중복 실행 방지**
  * 동일한 Job이 여러 번 쌓이지 않도록 Lock/unique 구현 필요 (ex. cache, Redis lock 등 활용)
* **적절한 큐 드라이버 선택**
  * 단일 서버/테스트: database
  * 대규모/분산/실시간: redis, sqs 등 추천

## 결론
* Queue & Job 시스템은 **시간이 오래 걸리는 작업**을 메인 트래픽과 분리해서**백그라운드에서 안정적으로 처리**할 수 있게 해줍니다.
* 이를 통해 사용자에게는 빠르고 즉각적인 응답을 제공하고, 서버는 과부하 없이 **확장성**과 **안정성**을 유지할 수 있습니다.

### 참고 링크
* [공식 문서 - Laravel Queue](https://laravel.com/docs/11.x/queues)

