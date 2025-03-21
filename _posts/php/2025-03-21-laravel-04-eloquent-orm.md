---
title: Laravel 04 - Eloquent ORM
description: Eloquent ORM 알아보기
author: holymason
published: false
categories: [php, laravel]
tags: [php, laravel, 라라벨, eloquent-orm, mvc]
---

이번 포스트는 Laravel 프레임워크의 강력한 기능 중 하나인 Eloquent ORM에 대한 포스트입니다.
Post 모델과 PostService를 활용한 실제 예제를 통해 Eloquent ORM의 기본 개념과 활용법을 알아보겠습니다.

## 목차



## Eloquent ORM이란?

Eloquent는 Laravel에서 제공하는 ORM(Object-Relational Mapping) 시스템입니다.  
데이터베이스 테이블을 PHP 객체로 매핑하여 **SQL 쿼리를 직접 작성하지 않고도** 데이터베이스 작업을 수행할 수 있게 해줍니다.   
개발자는 Eloquent ORM을 통해서 더 **직관적이고 단순한 코드**로 데이터베이스 작업을 수행할 수 있습니다. 
Eloquent ORM의 주요 특징과 기능을 중점으로 알아보겠습니다.

> _The Eloquent ORM included with Laravel provides a beautiful, simple ActiveRecord implementation for working with your database. Each database table has a corresponding "Model" which is used to interact with that table. Models allow you to query for data in your tables, as well as insert new records into the table._  
> Laravel에 포함된 Eloquent ORM은 데이터베이스 작업을 위한 아름답고 간단한 ActiveRecord 구현을 제공합니다. 각 데이터베이스 테이블에는 해당 테이블과 상호 작용하는 데 사용되는 해당 "모델"이 있습니다. 모델을 사용하면 테이블의 데이터를 쿼리하고 테이블에 새 레코드를 삽입할 수 있습니다.  
> 출처 : https://laravel.com/docs/11.x/structure#the-models-directory


## 1. Active Record 패턴
* 객체와 테이블의 일대일 매핑: 각 Eloquent 모델 클래스는 DB의 한 테이블에 대응됩니다
* 객체에 내장된 CRUD 기능: 모델 인스턴스가 직접 자신의 데이터를 DB에 저장, 수정, 삭제할 수 있습니다.
* 캡슐화된 데이터베이스 액세스: SQL 쿼리를 직접 작성하지 않고 객체지향적 방식으로 DB에 접근합니다.
   
```php
// CRUD, 데이터베이스 액세스
$post = new Post();
$post->title = '제목';
$post->content = '내용';
$post->save(); // CREATE : DB에 저장

$post->title = '수정된 제목';
$post->save(); // UPDATE : DB에 업데이트

$post->delete(); // DELETE : DB에서 삭제
```

## 2. 직관적인 데이터베이스 상호작용
* Eloquent는 데이터베이스 작업을 PHP 코드로 자연스럽게 표현합니다
* 메서드 체이닝을 통해 복잡한 쿼리를 가독성 있게 작성할 수 있습니다.
  * 예시 : `Post::where('is_published', true)->where('view_count', '>', 100)->get();`

```php
// 직관적인 쿼리
$recentPosts = Post::where('is_published', true)
    ->orderBy('created_at', 'desc')
    ->take(5)
    ->get();

// 직관적인 메소드
$userPostCount = Post::where('user_id', $userId)->count();
$averageViewCount = Post::where('is_published', true)->avg('view_count');
```

## 3. 관계(Relationships) 관리 용이
* 명시적 관계 정의: 모델 클래스 내에 메서드로 관계를 정의해 코드의 가독성과 유지보수성을 높입니다.
* 다양한 관계 유형 지원
  * 1:1 - hasOne / belongsTo 
  * 1:N / N:1 - hasMany / belongsToMany
  * 다형성 - morphTo / morphMany / morphToMany 
* 관계 지연 로딩과 즉시 로딩: 필요에 따라 관련 데이터를 함께 또는 나중에 로드할 수 있어 성능 최적화가 가능합니다.

```php
// 관계 정의 
class Post extends Model {
    // hasOne() 또는 hasMany() 관계를 정의하면, 현재 모델의 단수형 이름 + _id 를 외래 키로 찾습니다.
    public function comments() {
        return $this->hasMany(Comment::class);
    }
    
    public function tags() {
        return $this->belongsToMany(Tag::class);
    }
    
    // 외래키를 따로 지정해줄 수도 있습니다.
    public function user() {
        return $this->belongsTo(User::class, 'writer_user_id');
    }
}

// 관계 사용 예시
// 지연 로딩
$post = Post::find(1); // SELECT * FROM posts WHERE id = 1 LIMIT 1;
$comments = $post->comments; // 이 시점에 쿼리 실행 : SELECT * FROM comments WHERE post_id = 1;

// 즉시 로딩
$posts = Post::with(['comments', 'user'])->get(); // 한 번에 모든 데이터 로드
// 총 3번의 쿼리가 실행되어 결과가 $posts 에 반환됩니다
// SELECT * FROM posts;
// SELECT * FROM comments WHERE post_id IN (1, 2, 3, ...);
// SELECT * FROM users WHERE id IN (10, 20, 30, ...);

// 관계를 통한 쿼리
$posts = Post::whereHas('comments', function($query) {
    $query->where('is_approved', true);
})->get();

```

## 4. 모델 이벤트 및 옵저버 지원
* Eloquent는 모델의 생명주기 동안 다양한 이벤트를 발생시켜 비즈니스 로직을 확장할 수 있게 합니다
* 옵저버 패턴: 별도의 클래스로 이벤트 리스너를 분리해 코드를 더 깔끔하게 관리할 수 있습니다.
* 생명주기 이벤트: 모델이 생성, 업데이트, 삭제 등의 작업 전후에 특정 코드를 실행할 수 있습니다.
  * `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restoring`, `restored` 등
* 옵저버 생성 명령어 예제 : `php artisan make:observer PostObserver --model=Post`

```php
// 모델 이벤트 예시
class Post extends Model {
    // 각각 이벤트가 발생할 때 자동으로 처리되는 로직을 추가할 수 있습니다.
    protected static function booted() {
        // 생성 시 어드민에게 푸시 발송
        static::creating(function ($post) {
            // App Push | Web Push
        });
        
        // 캐시 관리
        static::updated(function ($post) {
            Cache::forget('post_' . $post->id);
        });
        
        // 연관 데이터 처리
        static::deleting(function ($post) {
            $post->comments()->delete();
        });
    }
}

// 옵저버 클래스 예시
class PostObserver {
    public function creating(Post $post) {
       // App Push | Web Push
    }
    
    public function updating(Post $post) {
        // 게시물이 수정될 때마다 last_modified_at 필드 업데이트
        $post->last_modified_at = now();
    }
}

// AppServiceProvider에서 옵저버 등록
public function boot() {
    Post::observe(PostObserver::class);
}
```

## 5. 모델과 DB 마이그레이션
* Eloquent ORM는 Laravel의 마이그레이션 시스템과 완벽하게 통합되어 있습니다.
  * https://laravel.com/docs/11.x/migrations
* 모델과 마이그레이션 동시 생성: `php artisan make:model Post -m` 명령어로 모델과 마이그레이션 파일을 동시에 생성할 수 있습니다.
* 팩토리와 시더: 모델과 함께 팩토리(Factory)와 시더(Seeder)를 생성해 테스트 데이터를 쉽게 만들 수 있습니다.
* 마이그레이션 명령어
  * 실행 : `php artisan migrate`
  * 롤백 : `php artisan migrate:rollback`

```php
// 마이그레이션 예시 : /database/migrations/2025_03_21_create_posts_table.php
use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('content');
            $table->foreignId('user_id')->constrained();
            $table->boolean('is_published')->default(false);
            $table->timestamps();
            $table->softDeletes(); // 소프트 삭제 지원
        });
    }

    public function down()
    {
        Schema::dropIfExists('posts');
    }
};
```

## 6. PostService를 활용한 ORM 예제
* 앞선 포스트에서 다루었던 내용처럼, **유지보수성과 확장성**을 위해 Laravel에서는 비즈니스 로직을 서비스 클래스로 분리하는 것이 좋습니다.    
* 비즈니스 로직들을 Post 모델 내에서 처리하지 않고, PostService 클래스를 생성하는 예제입니다.

```php
// PostService
namespace App\Services;

use App\Models\Post;
use Illuminate\Database\Eloquent\Collection;
use Illuminate\Pagination\LengthAwarePaginator;

class PostService
{
    public function getAllPosts(): Collection
    {
        return Post::all();
    }
    
    public function getPaginatedPosts(int $perPage = 15): LengthAwarePaginator
    {
        return Post::paginate($perPage);
    }
    
    public function getPostById(int $id): ?Post
    {
        return Post::find($id);
    }
    
    public function createPost(array $data): Post
    {
        return Post::create($data);
    }
    
    public function updatePost(int $id, array $data): bool
    {
        $post = Post::find($id);
        if (!$post) {
            return false;
        }
        return $post->update($data);
    }
    
    public function deletePost(int $id): bool
    {
        $post = Post::find($id);
        if (!$post) {
            return false;
        }
        return $post->delete();
    }
    
    public function getPublishedPosts(): Collection
    {
        return Post::where('is_published', true)->get();
    }
    
    public function publishPost(int $id): bool
    {
        return $this->updatePost($id, ['is_published' => true]);
    }
}
```

### 6.1. 생성(Create)

```php
// 모델의 create 메소드 사용
$post = Post::create([
    'title' => '포스트 생성 01',
    'content' => '모델 메소드를 통해 생성된 포스트입니다',
    'user_id' => 1,
    'is_published' => true
]);

// PostService 를 통한 생성
$postService = new PostService();
$post = $postService->createPost([
    'title' => ' 포스트 생성 02',
    'content' => 'PostService를 통해 생성된 포스트입니다',
    'user_id' => 1
]);
$postService->publishPost($post->id); // 서비스로 생성된 포스트를 publish 
```

### 6.2. 조회(Read)

```php
// 모든 레코드 가져오기
$posts = Post::all();

// 특정 ID로 레코드 가져오기
$post = Post::find($postId);

// 조건에 맞는 첫 번째 레코드 가져오기
$post = Post::where('is_published', true)->first();

// 조건에 맞는 모든 레코드 가져오기
$publishedPosts = Post::where('is_published', true)->get();

// 페이지네이션
$paginatedPosts = Post::paginate(20);

// 서비스를 통한 조회
$postService = new PostService();
$posts = $postService->getAllPosts();
$publishedPosts = $postService->getPublishedPosts();
$post = $postService->getPostById($postId);
```

### 6.3. 수정(Update)

```php
// 모델의 update 메소드 사용
$post = Post::find($postId);
$post->title = '수정된 제목';
$post->content = '이 내용은 수정되었습니다.';
$post->save();

// PostService 를 통한 업데이트
$postService = new PostService();
$postService->updatePost($postId, [
    'title' => 'PostService를 통해 업데이트된 제목',
    'content' => 'PostService를 통한 업데이트된 내용'
]);
```

### 6.4. 삭제(Delete)

```php
// 첫 번째 방법: 모델에서 삭제
$post = Post::find($postId);
$post->delete();

// PostService를 통한 삭제
$postService = new PostService();
$postService->deletePost($postId);
```


## 7. 쿼리 스코프 사용하기
* 쿼리 스코프를 사용하면 자주 사용하는 쿼리 조건을 캡슐화하여, 재사용성과 유지보수성을 높일 수 있습니다.
* 모델에 `scope` 접두사를 붙인 메소드를 정의하면, `where` 처럼 메소드 이름에서 `scope` 를 생략하고 직접 체이닝할 수 있습니다.

```php
class Post extends Model
{
    // 전역 스코프: 항상 적용되는 조건
    protected static function booted()
    {
        static::addGlobalScope('latest', function ($query) {
            $query->latest(); // created_at 기준으로 내림차순 정렬
        });
    }
    
    // 로컬 스코프 : 필요할 때 체이닝하여 사용
    // scopePublished → published() 메소드로 사용 가능
    public function scopePublished($query)
    {
        return $query->where('is_published', true);
    }
    
    public function scopeNew($query)
    {
        return $query->where('created_at', '>', Carbon::now()->subWeek());
    }
    
    public function scopeHot($query)
    {
        return $query->where('view_count', '>', 500);
    }
    
    public function scopeFromUser($query, $userId)
    {
        return $query->where('user_id', $userId);
    }
}
```

### 스코프 사용 예제

```php
// published 스코프 사용
$publishedPosts = Post::published()->get();

// 여러 스코프 체이닝
$hotNewPublishedPosts = Post::published()->hot()->new()->get();

// 파라미터가 있는 스코프 사용
$userPosts = Post::fromUser($userId)->get();

// 서비스에서 스코프 활용
class PostService
{
    public function getHotPublishedPosts(): Collection
    {
        return Post::published()->hot()->get();
    }
    
    public function getUserPublishedPosts(int $userId): Collection
    {
        return Post::published()->fromUser($userId)->get();
    }
}
```

## 8. 쿼리 빌더 
* **쿼리 빌더 Query Builder**는 SQL을 객체지향적으로 작성할 수 있도록 도와주는 강력한 도구입니다.
* 기본적인 `where`, `orderBy`, `groupBy`, `get` 등의 메소드 외에도, 더 복잡한 쿼리를 효율적으로 작성할 수도 있습니다.

```php
// 여러 조건을 사용한 쿼리
$posts = Post::where('is_published', true)
    ->where(function ($query) {
        $query->where('view_count', '>', 100)
              ->orWhere('comment_count', '>', 5);
    })
    ->whereHas('comments', function ($query) {
        $query->where('is_approved', true);
    })
    ->orderBy('created_at', 'desc')
    ->take(5)
    ->get();

// Sub Query
$posts = Post::whereIn('id', function ($query) {
    $query->select('post_id')
          ->from('post_views')
          ->where('viewed_at', '>=', now()->subDays(7));
})
->get();

// Raw Query
$posts = Post::select(DB::raw('COUNT(*) as post_count, user_id'))
    ->where('is_published', true)
    ->groupBy('user_id')
    ->having('post_count', '>', 3)
    ->get();
```

## 9. Eloquent Collection 활용
* Eloquent에서 반환되는 결과는 Collection 객체입니다. 이를 활용하여 필요에 따라 데이터를 처리할 수 있습니다.

```php
// Collection 필터링
$publishedPosts = $posts->filter(function ($post) {
    return $post->is_published;
});

// Collection 매핑
$titles = $posts->map(function ($post) {
    return $post->title;
});

// 특정 컬럼만 추출
$writer_ids = $posts->pluck('writer_user_id');

// 그룹화
$postsByUser = $posts->groupBy('user_id');

// 조건 검사
$isHot = $posts->contains(function ($post) {
    return $post->view_count > 1000;
});
```

## 결론

이번 포스트에서는 Post 모델과 PostService 서비스 클래스 예제를 통해서 Eloquent ORM의 기본 개념과 활용법에 대해 알아보았습니다.

Eloquent ORM을 활용하면 복잡한 데이터베이스 작업을 직관적이고 효율적으로 처리할 수 있고, 특히 서비스 레이어와 함께 사용하면 코드의 가독성과 유지보수성을 크게 향상시킬 수 있습니다.  

Eloquent ORM에 대해서 포스트에서도 다루지 못한 더 많은 기능과 활용방법이 있으니  
Eloquent ORM이 제공하는 더 많은 고급 기능은 공식 문서를 참조하시면 더 깊이 있는 내용을 확인하실 수 있습니다. 
