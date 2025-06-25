---
title: Laravel 06 - Repository 패턴
description: 서비스 계층과 데이터 접근의 분리, 코드 구조 개선을 위한 Repository 패턴
author: holymason
categories: [php, laravel, architecture]
tags: [php, laravel, repository, service, design-pattern]
---

이 포스트에서는 Laravel 프로젝트에서 Repository 패턴을 도입하는 실무적인 이유와 구현 방법,  
그리고 Service Layer와의 연계까지 알아보겠습니다.

# Repository 패턴

---

## 목차

1. [Repository 패턴이란?](#1-repository-패턴이란)
2. [왜 필요한가? (장점 및 실전 필요성)](#2-왜-필요한가-장점-및-실전-필요성)
3. [기본 구조와 예제](#3-기본-구조와-예제)
    - 3.1 [인터페이스/구현체](#31-인터페이스구현체)
    - 3.2 [서비스 계층과의 연계](#32-서비스-계층과의-연계)
4. [Service Provider/DI로 연결하기](#4-service-providerdi로-연결하기)
5. [Repository 패턴의 단점/주의점](#5-repository-패턴의-단점주의점)
6. [결론](#6-결론)

---

## 1. Repository 패턴이란?

Repository 패턴은 데이터베이스 접근 로직(쿼리, Eloquent 등)을 한 곳에 모으고  
비즈니스 로직(서비스 계층)과 분리하는 설계 디자인 패턴(Design Pattern)입니다.  
컨트롤러/서비스 계층에서는 Repository만을 통해 데이터를 CRUD 하며, 구현(ORM, Raw Query 등)은 Repository 내부에만 존재합니다.  
따라서, 유지보수성 향상과 Repository를 통한 코드 테스트에 용이합니다. 

> **데이터 추상화(Data Abstraction)**   
> 데이터가 어디서, 어떻게, 어떤 방식으로 저장·조회되는지 **감추고**, 그 데이터에 접근할 수 있는 **공통된 인터페이스(메서드)** 만 노출하는 것을 말합니다.   
> 즉, 데이터 소스의 세부 구현을 숨기고(추상화), 사용하는 로직(서비스, 컨트롤러 등)은 인터페이스(계약)만 알면 됩니다.

> **Design Pattern**  
> 소프트웨어 설계에서 반복적으로 나타나는 문제에 대해, 이미 검증된 “코드 구조와 설계 해법”을 정리해놓은 것을 말합니다.   
> Repository 외에 여러 Design Pattern 은 아래 참고 사이트에서 자세한 설명을 확인할 수 있습니다.  
> [Design Patterns in PHP](https://refactoring.guru/design-patterns/php)

---

## 3. 기본 구조와 예제

### 3.1 인터페이스/구현체

**예시: PostRepository**

- `app/Repositories/PostRepositoryInterface.php`
- `app/Repositories/PostRepository.php`

```php
// PostRepositoryInterface.php
namespace App\Repositories;

// 컨트롤러, 서비스 등에서 구현체가 어떤 방식으로 동작하는지 몰라도
// 이 메서드만 사용하면 Post 데이터를 다룰 수 있게 해줍니다.
interface PostRepositoryInterface
{
    public function all();
    public function find($id);
    public function create(array $data);
    public function update($id, array $data);
    public function delete($id);
}
````

```php
// PostRepository.php
namespace App\Repositories;

use App\Models\Post;

// 실제 데이터 접근과 처리(Eloquent, DB 쿼리 등)은 이 곳에서 처리
// 컨트롤러/서비스 등은 내부 동작을 몰라도 됨
class PostRepository implements PostRepositoryInterface
{
    public function all()
    {
        return Post::all();
    }
    public function find($id)
    {
        return Post::find($id);
    }
    public function create(array $data)
    {
        return Post::create($data);
    }
    public function update($id, array $data)
    {
        $post = Post::find($id);
        if ($post) {
            $post->update($data);
            return $post;
        }
        return null;
    }
    public function delete($id)
    {
        return Post::destroy($id);
    }
}
```

---

### 3.2 서비스 계층과의 연계
* 서비스에 관한 자세한 내용은 [서비스 Service 알아보기](/posts/laravel-04-eloquent-orm/#7-postservice를-활용한-orm-예제) 포스트를 참고해주세요.

```php
// app/Services/PostService.php
namespace App\Services;

use App\Repositories\PostRepositoryInterface;

class PostService
{
    protected $postRepo;

    public function __construct(PostRepositoryInterface $postRepo)
    {
        $this->postRepo = $postRepo;
    }

    public function createPost(array $data)
    {
        // 비즈니스 로직 가능
        return $this->postRepo->create($data);
    }
}
```

---

## 4. Service Provider/DI로 연결하기

> **DI (Dependency injection, 의존성 주입)**  
> 소프트웨어 엔지니어링에서 의존성 주입은 하나의 객체가 다른 객체의 의존성을 제공하는 기술이다.

```php
// app/Providers/RepositoryServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Repositories\PostRepositoryInterface;
use App\Repositories\PostRepository;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(PostRepositoryInterface::class, PostRepository::class);
    }
}
```

* 그리고 `config/app.php` providers 배열에 등록
* 이제 서비스 컨테이너가 **PostRepositoryInterface** 요청 시, **PostRepository** 인스턴스를 주입해줍니다.

---

## 5. Repository 패턴의 단점/주의점

* 너무 단순한 프로젝트에서는 오히려 오버엔지니어링이 될 수 있습니다.
* Eloquent의 기능들을 Repository에서 다루다 보면 코드가 복잡해질 수 있고, 과도하게 추상화하면 오히려 코드 복잡도 증가할 수 있습니다.

---

## 6. 결론

Repository 패턴은 규모가 큰 프로젝트, 테스트/유지보수/협업이 중요한 개발 환경에서 유용하게 사용할 수 있습니다.
Laravel의 서비스 컨테이너, 의존성 주입, 그리고 서비스 계층과 결합하여 프로젝트의 실제 서비스 구조를 더 견고하게 만들 수 있습니다.

### 참고자료

* [Design Patterns in PHP](https://refactoring.guru/design-patterns/php)
* [Laravel Docs: Service Container](https://laravel.com/docs/11.x/container)
* [Laravel Docs: Service Provider](https://laravel.com/docs/11.x/provider)

