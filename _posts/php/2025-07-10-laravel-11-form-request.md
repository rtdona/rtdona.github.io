---
title: Laravel 11 - Form Request 입력값 검증
description: FormRequest로 안전한 입력값 검증과 예제
author: holymason
categories: [php, laravel, validation]
tags: [php, laravel, validation, formrequest]
---

Laravel Validator에 [이전 포스트](/posts/laravel-05-validator/)에서는 컨트롤러 내에서 입력값을 검증하는 예제로 알아보았습니다.  

이번 포스트에서는 **라라벨의 FormRequest**를 활용하여 
**유효성 검사**을 더 안전하고 깔끔하게 구현하는 방법을 예제와 함께 알아보겠습니다.  
컨트롤러 내부에서 Validate 코드를 반복하는 대신, **FormRequest로 검증 로직을 별도 클래스로 분리**하여
코드의 재사용성과 유지보수성을 높일 수 있습니다.

---

# Laravel FormRequest – 실전 활용 가이드

---

## 목차

1. [FormRequest란?](#1-formrequest란)
2. [FormRequest 생성 및 사용법](#2-formrequest-생성-및-사용법)
3. [User/Post/Comment 예제](#3-userpostcomment-예제)
4. [Validation Rule](#4-validation-rule)
5. [팁 & 주의할 점](#5-팁--주의할-점)
6. [결론](#결론)

---

## 1. FormRequest란?

* **FormRequest**는 Laravel의 **HTTP 요청을 객체로 래핑**해
  **입력값 검증**과 **인증 검사** 등 복합한 유효성 검증 로직을 캡슐화하는 클래스입니다.
* 검증 규칙, 에러 메시지, 인증 체크까지 한번에 관리할 수 있습니다.

---

## 2. FormRequest 생성 및 사용법

### 2.1. artisan 명령어로 생성

```bash
php artisan make:request StorePostRequest
```

### 2.2. 예시 코드

```php
// app/Http/Requests/StorePostRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class StorePostRequest extends FormRequest
{
    // 권한 검사
    public function authorize()
    {
       
       // Case 1) 로그인한 사용자만 허용
       // return auth()->check();

       // Case 2) 관리자만 허용
       // return auth()->check() && auth()->user()->is_admin;

       // Case 3) 복잡한 조건도 가능
       // return auth()->user()?->hasPermission('write-post');
       
       // Case 4) 권한 검사 없음
       return true;
    }

    // Validation 규칙
    public function rules()
    {
        return [
            'title'   => 'required|string|max:255',
            'content' => 'required|string|min:10',
            'tags'    => 'array|nullable',
        ];
    }

    // 커스텀 에러 메시지 
    public function messages()
    {
        return [
            'title.required' => '제목은 필수 입력 항목입니다.',
            'content.min'    => '본문은 최소 10자 이상 입력해야 합니다.',
        ];
    }
}
```

더 많은 Validation 규칙은 이전 포스트 [Laravel-05-Validator](/posts/laravel-05-validator/) 를 참고해주세요

### 2.3. 컨트롤러에 FormRequest 적용

```php
use App\Http\Requests\StorePostRequest;

public function store(StorePostRequest $request)
{
    // StorePostRequest의 검증을 통과하면, 이 메소드가 실행
    $post = Post::create($request->validated());
    return redirect()->route('posts.show', $post);
}
```

Laravel 이 해당 라우트/컨트롤러를 호출할 때, 먼저 `StorePostRequest` 객체를 생성하여  
`rules()` 에 정의된 Validation 규칙으로 자동으로 검증을 수행합니다.

만약 입력값이 규칙을 만족/통과하지 못하면, 컨트롤러 메소드는 아예 실행되지 않고
자동으로 이전 페이지로 리다이렉트하면서 에러 메시지를 반환하고,

검증을 모두 통과해야만 컨트롤러의 `store(StorePostRequest $request)` 메소드가 실제로 실행됩니다.


---

## 3. User/Post/Comment 예제

### 3.1. User 회원가입 Request

```bash
php artisan make:request RegisterUserRequest
```

```php
// app/Http/Requests/RegisterUserRequest.php

public function rules()
{
    return [
        'name'     => 'required|string|max:50',
        'email'    => 'required|email|unique:users,email',
        'password' => 'required|min:8|confirmed',
    ];
}
```

### 3.2. Comment 등록 Request

```bash
php artisan make:request StoreCommentRequest
```

```php
public function rules()
{
    return [
        'post_id' => 'required|exists:posts,id',
        'content'    => 'required|string|min:2|max:1000',
    ];
}
```

--- 
## 4. Validation Rule

FormRequest의 강력한 장점 중 하나는 Laravel의 기본 벨리데이션 룰로 검증로직을 구현하기에 부족할 때,  
내 상황에 맞는 Custom Validator를 직접 만들 수 있다는 것입니다.  

### 4.1. Validation Rule 생성

```bash
php artisan make:rule OnlyHangul 
php artisan make:rule PhoneNumber
php artisan make:rule DomainEmail
```

### 4.2. Custom Rule 예시

```php
// app/Rules/OnlyHangul.php
namespace App\Rules;
use Illuminate\Contracts\Validation\Rule;

class OnlyHangul implements Rule
{
    public function passes($attribute, $value)
    {
        // 한글로만 이루어진 문자열이면 true
        return preg_match('/^[가-힣\s]+$/u', $value);
    }

    public function message()
    {
        return ':attribute는 한글만 입력할 수 있습니다.';
    }
}

// app/Rules/PhoneNumber.php
namespace App\Rules;
use Illuminate\Contracts\Validation\Rule;

class PhoneNumber implements Rule
{
    public function passes($attribute, $value)
    {
        // 010-1234-5678 또는 01012345678
        return preg_match('/^01[016789]-?\d{3,4}-?\d{4}$/', $value);
    }

    public function message()
    {
        return ':attribute는 올바른 휴대폰 번호 형식이 아닙니다.';
    }
}

// app/Rules/DomainEmail.php
namespace App\Rules;
use Illuminate\Contracts\Validation\Rule;

class DomainEmail implements Rule
{
    public function passes($attribute, $value)
    {
        // 예시: @my-company.com 이메일만 허용
        return str_ends_with($value, '@my-company.com');
    }

    public function message()
    {
        return '회사 이메일(@my-company.com)만 사용할 수 있습니다.';
    }
}
```

### 4.3. Custom Rule 적용

위에서 생성한 Custom Validation Rule들을 `RegisterUserRequest` 에 적용해보겠습니다.

```php
// app/Http/Requests/RegisterUserRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use App\Rules\PhoneNumber;
use App\Rules\OnlyHangul;
use App\Rules\DomainEmail;

class RegisterUserRequest extends FormRequest
{
    public function authorize()
    {
        return true; 
    }

    public function rules()
    {
        return [
            'name'     => ['required', 'max:50', new OnlyHangul], // 이름은 한글만 허용
            'email'    => ['required', 'email', 'unique:users,email', new DomainEmail], // 회사 이메일만 허용
            'password' => ['required', 'min:8', 'confirmed'],
            'phone'    => ['required', new PhoneNumber], // 휴대폰 번호 형식 검증
        ];
    }

    public function messages()
    {
        return [
            'name.required'      => '이름을 입력하세요.',
            'name.max'           => '이름은 50자 이하로 입력하세요.',
            'email.required'     => '이메일을 입력하세요.',
            'email.email'        => '올바른 이메일 형식이 아닙니다.',
            'email.unique'       => '이미 등록된 이메일입니다.',
            'password.required'  => '비밀번호를 입력하세요.',
            'password.min'       => '비밀번호는 8자 이상이어야 합니다.',
            'password.confirmed' => '비밀번호 확인이 일치하지 않습니다.',
            'phone.required'     => '휴대폰 번호를 입력하세요.',
        ];
    }
}

```

이처럼 기본 Validation에서 제공하지 않는 조건들은 Custom Validation Rule을 통해 구현하여 검증할 수 있습니다.


---

## 5. 팁 & 주의할 점


* **컨트롤러 코드가 매우 깔끔해짐**  
  FormRequest를 쓰면 컨트롤러에서 입력값 검증 코드가 완전히 사라집니다.
  컨트롤러 메소드에서는 **비즈니스 로직**만 남게 되어 코드 가독성과 유지보수성이 크게 향상됩니다.

* **테스트 코드 작성이 쉬워짐**  
  Request 클래스는 독립적으로 인스턴스 생성 및 테스트가 가능하므로 (예: `$request = new StorePostRequest();`),
  다양한 입력값/케이스별 검증 테스트를 컨트롤러와 분리해서 작성할 수 있습니다.

* **Form과 REST API 요청 모두에서 활용 가능**  
  폼(HTML Form) 뿐만 아니라 외부 API 요청에서도 동일하게 사용할 수 있습니다.  
  **JSON 요청일 때는 자동으로 422 응답**(validation 에러, JSON 포맷)이 반환됩니다.

* **인증/권한 체크도 한 번에**
  FormRequest의 `authorize()` 메소드에서, 요청한 사용자가 권한이 있는지를 미리 검사할 수 있습니다.  
  예를 들어, 작성자만 수정/삭제 가능하도록하거나, 관리자인지, 특정 그룹/프로젝트 멤버인지 이런 복잡한 권한 로직도
  라우트/컨트롤러에서 분리해 Request 클래스에 집중시킬 수 있습니다.

* **벨리데이션 실패 시 자동 응답**  
  입력값이 잘못되면 컨트롤러 메소드 진입 전에 각각 
  * 웹 폼(HTML Form)은 이전 페이지로 자동 리다이렉트시키고 에러메시지를 반환합니다
  * API(JSON)는 422 Unprocessable Entity 코드와 상세 에러메시지를 반환합니다

* **rules(), messages(), attributes() 등 다양한 오버라이드 활용**  
  * `rules()`: 입력값별 세부 검증 규칙을 동적으로 리턴 가능
  * `messages()`: 검증 실패 메시지 커스텀, 번역파일과 조합도 가능
  * `attributes()`: 입력 필드명을 한글 등 사용자 친화적으로 바꿔 출력 가능
    실무에서는 다국어/동적 규칙/필드명 커스텀 등 복잡한 요구사항에도 유연하게 대응할 수 있습니다.

---
## 결론

FormRequest는 라라벨에서 **입력값 벨리데이션과 권한 검증을 분리**할 수 있는 강력하고 편리한 핵심 기능입니다.
FormRequest와 Custom Rule을 활용하면 서비스가 커지더라도 컨트롤러는 더 심플해지고, 검증 규칙의 재사용과 유지보수, 테스트도 수월하게 진행할 수 있습니다.

---

### 참고 링크

* [공식 문서 – FormRequest & Validation](https://laravel.com/docs/11.x/validation#form-request-validation)
* [공식 문서 - 커스텀 벨리데이터](https://laravel.com/docs/11.x/validation#custom-validation-rules)

