---
title: Laravel 05 - Validator
description: 라라벨 Validator(유효성 검사) 개념과 사용법, 예제 정리
author: holymason
categories: [php, laravel]
tags: [php, laravel, 라라벨, validator, validation, 유효성검사]
---

이 포스트에서는 Laravel의 Validator(유효성 검사)의 기본 개념과  
`Post`, `User`, `Comment` 모델을 활용한 예제를 중심으로 정리합니다.

# Laravel Validator

Laravel은 강력하고 사용하기 쉬운 **유효성 검사(Validator)** 기능을 제공합니다.  
Validator를 사용하여 다양한 데이터의 유효성을 쉽게 검증할 수 있습니다.

> **유효성 검사(Validation)**  
유효성 검사란 사용자가 입력하거나 서버로 전달되는 데이터가  
서버에서 정한 규칙(ex: 필수값, 데이터 타입, 범위 등)을 만족하는지 사전에 검증하는 것을 말합니다.  
이를 통해 잘못된 데이터가 DB나 서비스 로직에 전달되는 것을 방지할 수 있습니다.

---

## 목차

1. [Validator란?](#1-validator란)
2. [Validator 기본 사용법](#2-validator-기본-사용법)
3. [자주 사용하는 주요 검증 규칙](#3-자주-사용하는-주요-검증-규칙)
4. [Validator 파사드와 $request->validate()의 차이](#4-validator-파사드와-request-validate의-차이)
5. [에러 메시지 처리 및 커스텀 메시지](#5-에러-메시지-처리-및-커스텀-메시지)
6. [FormRequest를 활용한 유효성 검사](#6-formrequest를-활용한-유효성-검사)
7. [커스텀 검증 규칙 만들기](#7-커스텀-검증-규칙-만들기)
8. [결론](#결론)

---

## 1. Validator란?

**Validator**는 Laravel에서 데이터의 유효성을 쉽고 일관성 있게 검증할 수 있게 도와주는 기능입니다.  
Form Data 전달, API 요청 등 다양한 곳에서 활용되며,  
검증에 실패한 데이터에 따라 자동으로 에러 메시지를 반환합니다.

---

## 2. Validator 기본 사용법

### 2.1. 게시글(Post) 저장 검증 예제

```php
use Illuminate\Support\Facades\Validator;

public function store(Request $request)
{
    $validator = Validator::make($request->all(), [
        'title'   => 'required|max:100',
        'content' => 'required|string|min:10',
        'user_id' => 'required|exists:users,id', 
    ]);

    if ($validator->fails()) { // 검증을 실패하면 자동으로 에러메시지 리턴
        return redirect()->back()
            ->withErrors($validator)
            ->withInput();
    }

    // 검증 통과 후 Post 생성
    Post::create($validator->validated());
}
````
* 여러 조건을 `|` 로 나열하여 동시에 설정할 수 있습니다. 
* `'user_id' => 'required|exists:users,id'` 에서 `exists:users,id` 로 조건을 설정하면,  
Laravel Validator가 내부적으로 자동으로 `users` 테이블을 조회해서
`id` 컬럼에 `user_id` 값이 실제 존재하는지 쿼리를 통해 확인하여 검증합니다.

### 2.2. 댓글(Comment) 등록 검증 예제

```php
public function store(Request $request)
{
    $validated = Validator::make($request->all(), [
        'post_id'  => 'required|exists:posts,id', // 해당 게시글이 존재해야 함
        'user_id'  => 'required|exists:users,id',
        'content'  => 'required|string|max:300',
        'parent_id'=> 'nullable|exists:comments,id', // 대댓글의 경우 comments id 도 검증
    ]);

    // Comment 생성
    Comment::create($validated);
}
```

---

## 3. 자주 사용하는 주요 검증 규칙

| 규칙                   | 설명                                         | 예시 코드                                                    |
| -------------------- | ------------------------------------------ |----------------------------------------------------------| 
| required             | 값이 반드시 있어야 함                               | `'title' => 'required'`                                  |        
| string               | 문자열인지 확인                                   | `'name' => 'string'`                                     |        
| integer              | 정수인지 확인                                    | `'age' => 'integer'`                                     |        
| numeric              | 숫자(정수 또는 실수)인지 확인                          | `'price' => 'numeric'`                                   |        
| boolean              | true/false, 0/1 값만 허용                      | `'active' => 'boolean'`                                  |        
| email                | 이메일 형식                                     | `'email' => 'email'`                                     |        
| max\:N               | 최대 길이/값 제한                                 | `'title' => 'max:100'`, `'age' => 'max:99'`              |        
| min\:N               | 최소 길이/값 제한                                 | `'password' => 'min:8'`                                  |        
| between\:A,B         | 값 또는 길이가 A이상 B이하                           | `'age' => 'between:20,60'`, `'title' => 'between:5,100'` |        
| in:값,값               | 지정된 값들 중 하나만 허용                            | `'status' => 'in:pending,approved,rejected'`             |        
| not\_in:값,값          | 지정된 값들 이외의 값만 허용                           | `'role' => 'not_in:admin,superuser'`                     |        
| unique\:table        | DB에서 유일한 값인지                               | `'email' => 'unique:users,email'`                        |        
| exists\:table        | DB에 값이 존재하는지 (외래키 등)                       | `'user_id' => 'exists:users,id'`                         |        
| date                 | 날짜 형식인지 확인                                 | `'published_at' => 'date'`                               |        
| date\_format:형식      | 지정한 날짜/시간 형식과 일치하는지                        | `'start_at' => 'date_format:Y-m-d H:i'`                  |        
| after:날짜필드           | 특정 날짜(또는 필드) 이후여야 함                        | `'end_at' => 'after:start_at'`                           |        
| before:날짜필드          | 특정 날짜(또는 필드) 이전이어야 함                       | `'start_at' => 'before:end_at'`                          |
| confirmed            | 필드와 동일한 값의 \*\_confirmation 필드가 함께 제출되어야 함 | `'password' => 'confirmed'`  → `password_confirmation`도 필요 |
| same:필드명             | 지정한 다른 필드와 값이 동일해야 함                       | `'email' => 'same:email_confirm'`                        |
| different:필드명        | 지정한 다른 필드와 값이 달라야 함                        | `'new_password' => 'different:current_password'`         |
| size\:N              | 정확히 N의 길이/값이어야 함                           | `'phone' => 'size:11'`                                   |
| digits\:N            | 정확히 N자리 숫자여야 함                             | `'phone' => 'digits:11'`                                 |
| digits\_between\:A,B | A\~B자리 숫자여야 함                              | `'zip' => 'digits_between:5,6'`                          |
| array                | 배열인지 확인                                    | `'tags' => 'array'`                                      |
| distinct             | 배열의 각 값이 중복이 없어야 함                         | `'tags.*' => 'distinct'`                                 |
| url                  | 유효한 URL 형식인지 확인                            | `'website' => 'url'`                                     |
| file                 | 파일 업로드인지 확인                                | `'attachment' => 'file'`                                 |
| image                | 이미지 파일인지 확인                                | `'photo' => 'image'`                                     |
| mimes:확장자            | 지정된 확장자 파일만 허용                             | `'attachment' => 'mimes:pdf,docx'`                       |
| nullable             | 값이 없어도(=null) 허용                           | `'middle_name' => 'nullable'`|
| required\_if:필드,값    | 특정 필드 값에 따라 필수값 지정                         | `'reason' => 'required_if:status,rejected'`              |
| sometimes            | 필드가 있을 때만 검증 수행                            | `'nickname' => 'sometimes'`                                | 
| json                 | JSON 문자열인지 확인                              | `'settings' => 'json'`                                   |


> **전체 규칙은 [공식 문서](https://laravel.com/docs/11.x/validation#available-validation-rules) 참고**

또한 아래의 예시처럼, 복잡한 JSON 타입의 데이터도 검증할 수 있습니다.

```json
{
  "data": {
    "name": "Deadline of Weekdays",
    "deadline": [
      { "active": true,  "weekday": 1, "deadline": "17:00:00" },
      { "active": false, "weekday": 2, "deadline": null },
      { "active": false, "weekday": 3, "deadline": null },
      { "active": false, "weekday": 4, "deadline": null },
      { "active": true,  "weekday": 5, "deadline": "14:00:00" },
      { "active": false, "weekday": 6, "deadline": null },
      { "active": false, "weekday": 0, "deadline": null }
    ]
  }
}
```
```php
Validator::make($request->all(), [
    'data.name' => 'required|string|max:255',
    'data.deadline' => 'required|array|size:7',
    'data.deadline.*.active' => 'required|boolean',
    'data.deadline.*.weekday' => 'required|integer|between:0,6', // 0(일) ~ 6(토)
    'data.deadline.*.deadline' => 'nullable|date_format:H:i:s',
]);
```


---

## 4. Validator 파사드와 $request->validate()의 차이

### 4.1. Validator 파사드(Facade)
```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, $rules, $messages = [], $customAttributes = []);
```
* Validator 파사드는 Validator 인스턴스를 직접 생성해서,
세부적인 커스터마이즈와 다양한 활용이 가능합니다.

* 검증 실패시: `$validator->fails()` 로 결과를 수동으로 체크해야 하며, 
실패 후 직접 에러 응답(리다이렉트/JSON 등)을 만들어야 합니다.

```php
if ($validator->fails()) {
    return response()->json([
        'result' => 'fail',
        'errors' => $validator->errors(),
    ], 422);
}
```

### 4.2. $request->validate() (Request 객체의 validate 메서드)

```php
$validated = $request->validate($rules, $messages = [], $customAttributes = []);
```
* Laravel 컨트롤러에서 가장 많이 쓰는 단축 메서드
* 내부적으로는 위의 Validator 파사드를 사용
* 검증 실패시 자동으로 리다이렉트(웹), JSON 응답(API), 에러 메시지 세션 저장까지 처리하여, 별도 에러 체크/응답 코드가 필요 없습니다.
* 검증 통과시 통과된 값만 담긴 배열(`$validated`)을 반환합니다


## 5. 에러 메시지 처리 및 커스텀 메시지

### 5.1. 기본 에러 메시지 예시

`:attribute`, `:max`, `:min`, `:type`, `:format`, `:digits`, `:size`, `:values` 등
플레이스홀더는 실제 필드명/값으로 자동 치환됩니다.

| 규칙           | 기본 메시지(영문)                                                | 예시 출력                                               |
| ------------ | --------------------------------------------------------- | --------------------------------------------------- |
| required     | The \:attribute field is required.                        | The status field is required.                       |
| email        | The \:attribute must be a valid email address.            | The email must be a valid email address.            |
| unique       | The \:attribute has already been taken.                   | The email has already been taken.                   |
| max          | The \:attribute may not be greater than \:max characters. | The name may not be greater than 30 characters.     |
| min          | The \:attribute must be at least \:min characters.        | The password must be at least 8 characters.         |
| in           | The selected \:attribute is invalid.                      | The selected status is invalid.                     |
| not\_in      | The selected \:attribute is invalid.                      | The selected role is invalid.                       |
| exists       | The selected \:attribute is invalid.                      | The selected user\_id is invalid.                   |
| array        | The \:attribute must be an array.                         | The tags must be an array.                          |
| boolean      | The \:attribute field must be true or false.              | The active field must be true or false.             |
| numeric      | The \:attribute must be a number.                         | The age must be a number.                           |
| integer      | The \:attribute must be an integer.                       | The count must be an integer.                       |
| string       | The \:attribute must be a string.                         | The name must be a string.                          |
| type         | The \:attribute must be of type \:type.                   | The file must be of type pdf.                       |
| url          | The \:attribute must be a valid URL.                      | The website must be a valid URL.                    |
| date         | The \:attribute is not a valid date.                      | The published\_at is not a valid date.              |
| date\_format | The \:attribute does not match the format \:format.       | The start\_at does not match the format Y-m-d H\:i. |
| confirmed    | The \:attribute confirmation does not match.              | The password confirmation does not match.           |
| digits       | The \:attribute must be \:digits digits.                  | The phone must be 11 digits.                        |
| size         | The \:attribute must be \:size.                           | The code must be 6.                                 |
| file         | The \:attribute must be a file.                           | The attachment must be a file.                      |
| image        | The \:attribute must be an image.                         | The photo must be an image.                         |
| mimes        | The \:attribute must be a file of type: \:values.         | The attachment must be a file of type: pdf, docx.   |

### 5.2. 커스텀 에러 메시지

`$request` 객체의 `validate()` 메소드의 2번째 파라미터에 커스텀 에러메시지를 설정할 수 있습니다

아래 예시처럼 `:values` 는 `in` 규칙의 허용값 목록으로 자동 치환됩니다.

```php
$request->validate([
    'status' => 'required|in:pending,approved,rejected',
], [
//  'status.in' => '상태는 반드시 pending, approved, rejected 중 하나여야 합니다.',
    'status.in' => '상태는 반드시 :values 중 하나여야 합니다.',
    'status.required' => '상태 값을 입력하세요.',
]);
```

### 5.3. 커스텀 메시지 예시 : User 회원가입

```php
$request->validate([
    'name'     => 'required|string|max:50',
    'email'    => 'required|email|unique:users,email',
    'password' => 'required|string|min:8|confirmed',
], [
    'name.required'      => '이름을 입력하세요.',
    'email.required'     => '이메일을 입력해주세요.',
    'email.unique'       => '이미 사용중인 이메일입니다.',
    'password.confirmed' => '비밀번호 확인이 일치하지 않습니다.',
]);
```

---

## 6. FormRequest를 활용한 유효성 검사

규모가 커질 때는 FormRequest 클래스 사용을 추천합니다.
* FormRequest는 `Illuminate\Foundation\Http\FormRequest` 를 상속한 클래스입니다.
* 컨트롤러에서 유효성 검증 코드를 분리해서 Request 검증 로직을 하나의 클래스에서만 관리할 수 있게하여, 코드 유지보수성을 높입니다.
* 유효성 규칙(rules), 커스텀 메시지(messages), 권한 검사(authorization) 등을 FormRequest 클래스에서 따로 정의합니다.


**StoreCommentRequest 클래스 예시:**
```bash
php artisan make:request StoreCommentRequest
```

```php
public function rules()
{
    return [
        'post_id'  => 'required|exists:posts,id',
        'user_id'  => 'required|exists:users,id',
        'content'  => 'required|string|max:300',
        'parent_id'=> 'nullable|exists:comments,id',
    ];
}

public function messages()
{
    return [
        'post_id.required' => '댓글이 달릴 게시글이 지정되지 않았습니다.',
        'user_id.exists'   => '존재하지 않는 사용자입니다.',
        'content.required' => '댓글 내용을 입력하세요.',
    ];
}
```

**컨트롤러에서 사용:**

```php
public function store(StoreCommentRequest $request)
{
    $data = $request->validated();
    Comment::create($data);
}
```

---

## 7. 커스텀 검증 규칙 만들기

### 7.1 클로저(Closure)로 검증 규칙 추가

```php
$request->validate([
    'content' => [
        'required', 'string', 'max:300',
        function($attribute, $value, $fail) {
            $banned = ['나쁜말', '비속어'];
            foreach ($banned as $word) {
                if (str_contains($value, $word)) {
                    $fail('부적절한 단어가 포함되어 있습니다: ' . $word);
                }
            }
        },
    ],
    'user_id' => 'required|exists:users,id',
    'post_id' => 'required|exists:posts,id',
]);
```

### 7.2 Rule 클래스로 사용자 닉네임 검증

```bash
php artisan make:rule NoSpecialNickname
```

```php
namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class NoSpecialNickname implements Rule
{
    // 검증이 통과하면 true, 실패하면 false 반환
    public function passes($attribute, $value)
    {
        // 영어/숫자/한글만 허용 (특수문자 금지)
        return !preg_match('/[^a-zA-Z0-9가-힣]/u', $value);
    }

    // 검증 실패시 표시할 메시지
    public function message()
    {
        // :attribute는 필드명(nickname 등)으로 자동 치환됨
        return ':attribute에는 특수문자를 사용할 수 없습니다.';
    }
}
```

**사용 예시:**

```php
use App\Rules\NoSpecialNickname;

$request->validate([
    'nickname' => ['required', 'string', 'min:2', 'max:20', new NoSpecialNickname],
]);
```

---

## 결론

Laravel Validator는 서비스에서 가장 많이 사용되는 데이터의 신뢰성과 안정성을 높여주는 필수 기능입니다.

* **입력값 신뢰성 확보**: 잘못된 데이터의 저장/처리를 방지
* **코드 일관성**: 컨트롤러, FormRequest, Rule 등 다양한 방식으로 일관성 있게 관리
* **확장성**: 커스텀 규칙, 인터페이스와 결합해 실전 서비스에 맞게 확장 가능

참고 링크
* [Laravel 공식 문서 - Validation](https://laravel.com/docs/11.x/validation)
