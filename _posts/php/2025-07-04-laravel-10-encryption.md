---
title: Laravel 10 - Encryption
description: 라라벨에서 개인정보 처리를 위한 암호화/복호화 방법
author: holymason
categories: [php, laravel, security, encryption]
tags: [php, laravel, 암호화, 개인정보, privacy, encryption, security]
---

이번 포스트에서는 실무에서 사용되는 **UserPrivacy 모델**을 예시로,  
**개인정보 데이터를 안전하게 암호화/복호화**하는 방법을 코드 중심으로 알아보겠습니다

실제 서비스에서는 이름, 전화번호, 주소 등 민감 데이터의 평문 저장이
**법적·기술적 보안 위협**이 되므로,
**암호화 적용은 선택이 아닌 필수**입니다.

---

# Laravel – UserPrivacy 모델 암호화 실전

---

## 목차

1. [왜 개인정보 컬럼 암호화가 필요한가?](#1-왜-개인정보-컬럼-암호화가-필요한가)
2. [라라벨의 암호화 원리와 Crypt 파사드](#2-라라벨의-암호화-원리와-crypt-파사드)
3. [UserPrivacy 모델로 보는 예제](#3-userprivacy-모델로-보는-예제)
4. [팁 & 주의할 점](#4-팁--주의할-점)
5. [결론](#결론)

---

## 1. 왜 개인정보 컬럼 암호화가 필요한가?

2025년 6월, 대한민국 대표 온라인 서점 플랫폼이 대규모 해킹 공격을 받아 서비스가 며칠동안 중단된 사고가 있었습니다.  

만약 서버 내부나 DB에 이름, 전화번호, 주소 등 민감한 개인정보 데이터가 암호화되지 않고 평문으로 저장되어 있었다면,  
더 큰 보안 사고로 이어질 수도 있었지만, 당사의 입장문에 따르면 다행히도 개인정보 유출 정황은 없다고합니다.

해커가 DB에 직접 접근하더라도 APP_KEY 등 복호화 키 없이는 데이터 노출이 원천적으로 불가능했을 것입니다.

이처럼 실무에서 암호화는 선택이 아닌 필수입니다.  
만약 우리 서비스의 서버나 DB가 해킹, 유출되더라도 암호화가 적용되어 있다면 피해 규모와 2차 범죄 위험을 획기적으로 줄일 수 있습니다.

### 참고 기사
* ['시스템 점검 중' 안내하더니.. 뒤늦게 해킹 공지](https://news.sbs.co.kr/news/endPage.do?news_id=N1008132951)
* [서비스 일부 재개됐지만.. 개인정보 유출 불안](https://v.daum.net/v/20250614112509217)

---

## 2. 라라벨의 암호화 원리와 Crypt 파사드

* **대칭키(AES-256-CBC) 암호화** 기본 제공
* `.env`의 `APP_KEY`로 암복호화
* `Crypt` 파사드* 사용
* 단방향 암호화(해시)가 아닌, **양방향(복호화 가능) 암호화**

```php
use Illuminate\Support\Facades\Crypt;

$encrypted = Crypt::encryptString('민감정보');
$decrypted = Crypt::decryptString($encrypted);
```


> **Laravel 파사드(Facade)란?**  
> 
> Laravel 문서 전반에 걸쳐 "facade"를 통해 Laravel의 기능과 상호 작용하는 코드의 예를 볼 수 있습니다.   
> Facade는 애플리케이션의 서비스 컨테이너에서 사용할 수 있는 클래스에 "정적" 인터페이스를 제공합니다.   
> Laravel은 Laravel의 거의 모든 기능에 액세스할 수 있는 여러 Facade와 함께 제공됩니다.  
> 라라벨 파사드는 서비스 컨테이너의 기본 클래스에 대한 "정적 프록시(static proxies)" 역할을 하며, 전통적인 정적 방법보다 더 많은 테스트 가능성과 유연성을 유지하면서 간결하고 표현적인 구문의 이점을 제공합니다. 파사드가 어떻게 작동하는지 완전히 이해하지 못하더라도 괜찮습니다. 그냥 흐름에 따라 라라벨에 대해 계속 배우세요.  
> 모든 라라벨의 파사드는 `Illuminate\Support\Facades` 네임스페이스 안에 정의되어 있습니다.

---

## 3. UserPrivacy 모델로 보는 예제

### 3.1. UserPrivacy 모델 예시

**Eloquent Mutator/Accessor 네이밍 메소드 활용**

```php
// app/Models/UserPrivacy.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Crypt;

class UserPrivacy extends Model
{
    protected $fillable = [
        'user_id', 'phone', 'email', 'address', 'note',
    ];
    
    // Mutator와 Accessor 네이밍 규칙 활용*

    // 전화번호 암호화
    public function setPhoneAttribute($value)
    {
        $this->attributes['phone'] = $value ? Crypt::encryptString($value) : null;
    }
    public function getPhoneAttribute($value)
    {
        return $value ? Crypt::decryptString($value) : null;
    }

    // 이메일 암호화
    public function setEmailAttribute($value)
    {
        $this->attributes['email'] = $value ? Crypt::encryptString($value) : null;
    }
    public function getEmailAttribute($value)
    {
        return $value ? Crypt::decryptString($value) : null;
    }

    // 주소 암호화
    public function setAddressAttribute($value)
    {
        $this->attributes['address'] = $value ? Crypt::encryptString($value) : null;
    }
    public function getAddressAttribute($value)
    {
        return $value ? Crypt::decryptString($value) : null;
    }

    // 메모(비고) 암호화
    public function setNoteAttribute($value)
    {
        $this->attributes['note'] = $value ? Crypt::encryptString($value) : null;
    }
    public function getNoteAttribute($value)
    {
        return $value ? Crypt::decryptString($value) : null;
    }
}
```

Laravel은 **환경설정 파일(.env)** 의 `APP_KEY` 값을 암호화/복호화의 기본 키로 사용합니다.

> **Eloquent Mutator/Accessor 네이밍 규칙이란?**
> 
> Mutator (저장할 때 값 변환): set필드명Attribute($value)  
> Accessor (조회할 때 값 변환): get필드명Attribute($value)  
> 여기서 필드명(컬럼명)은 **카멜케이스(CamelCase)** 로 변환

**Model boot 메소드, 이벤트 활용**
```php

// 암호화/복호화 적용할 필드
public $encryptable_fields = [
    'phone', 'email', 'address', 'note'
];

protected static function boot()
{
    parent::boot();
    
    // 모델이 저장되기 직전 발생하는 이벤트
    // 저장 전 암호화
    static::saving(function ($model) {
        foreach ($model->encryptable as $field) {
            $val = $model->$field;
            if ($val) {
                $model->$field = \Crypt::encryptString($val);
            }
        }
    });

    // 모델이 조회된 직후 발생하는 이벤트
    // 조회 후 복호화
    static::retrieved(function ($model) {
        foreach ($model->encryptable as $field) {
            $val = $model->$field;
            if ($val) {
                $model->$field = \Crypt::decryptString($val);
            }
        }
    });
}

```
**두 가지 방법의 비교**

| 구분           | Mutator/Accessor 방식 | boot() + 이벤트 방식   |
| ------------ | ------------------- | ----------------- |
| **적용 범위**    | 컬럼별 개별 처리           | 여러 컬럼 일괄 처리       |
| **로직 관리**    | 필드별로 분리(코드 명확)      | 한 곳 집중 관리         |
| **코드 중복**    | 많아질 수 있음            | 줄일 수 있음           |
| **관계/Eager** | 완벽 적용               | 제한적(관계형 로딩 시 주의)  |
| **커스텀/조건별**  | 필드별 유연              | 이벤트 함수 내에서 커스텀 용이 |
| **성능**       | 일반적                 | 이벤트 중복 처리시 주의 필요  |
| **대표 사용처**   | 1\~2개 필드, 개별 로직 많음  | 여러 필드 반복적 암호화     |



### 3.2. 저장/조회 예시

```php
// 저장
$privacy = UserPrivacy::create([
    'user_id' => 1,
    'phone'   => '010-1234-5678',
    'email'   => 'user@email.com',
    'address' => '부산광역시 해운대구...',
    'note'    => '특이사항 없음',
]);

// 조회
$privacy = UserPrivacy::where('user_id', 1)->first();
echo $privacy->phone;   // 010-1234-5678 (자동 복호화)
echo $privacy->address; // 부산광역시 해운대구...
```


### 3.3. 마이그레이션 컬럼 타입 주의

```php
// database/migrations/xxxx_xx_xx_create_user_privacies_table.php

$table->unsignedBigInteger('user_id');
$table->text('phone')->nullable();
$table->text('email')->nullable();
$table->text('address')->nullable();
$table->text('note')->nullable();
```

> 암호화 데이터는 평문데이터보다 2~3배 이상 텍스트의 길이가 늘어나므로 **VARCHAR 타입보다 TEXT 타입** 사용을 권장합니다

---

## 4. 팁 & 주의할 점

* **like 검색/정렬/인덱스 불가:**
  암호화 컬럼은 DB 검색 불가하기 때문에, 별도의 검색용/색인 컬럼(`phone_hash` 등)을 활용해야 합니다
* **복호화 실패 예외 처리:**
  APP_KEY 변경/데이터 손상 시 예외가 발생할 수 있기 때문에 try-catch 로 복호화 실패 처리를 권장합니다
* **APP_KEY는 변경 금지/관리에 유의:**
  키를 분실하게 되면 암호화된 모든 데이터를 복구할 수 없게됩니다. 키가 노출되거나 분실되지 않게 키 관리에 유의해야합니다
* **최소화/선별 적용:**
  일반 정보까지 무분별하게 암/복호화 처리를 하면 서버 성능에 문제가 발생할 수 있기 때문에 개인정보 등 민감한 정보만 암호화합니다

---

## 결론

* 웹 서비스의 개인정보 보호와 데이터 보안은 더 이상 선택이 아닌 필수 요건입니다.
  최근 다양한 해킹 사고와 유출 사례에서 보듯, DB에 저장되는 민감 정보(연락처, 주소, 메모 등)를 암호화하지 않고 평문으로 저장하는 것은 곧바로 서비스 신뢰와 기업 책임의 문제로 이어질 수 있습니다.

* Laravel은 Crypt 파사드와 Mutator/Accessor, 혹은 boot() + 이벤트 패턴 등 
강력하면서도 유연한 암호화 도구와 패턴을 제공합니다. 이 기능을 적절히 활용하면 복잡한 구현 없이, 기존의 비즈니스로직에도 간편하게 DB에 저장되는 모든 민감 데이터를 안전하게 보호할 수 있습니다.


### 참고 링크

* [공식 문서 - Laravel Encryption](https://laravel.com/docs/11.x/encryption)
* [공식 문서 - Laravel Facades](https://laravel.com/docs/11.x/facades)
* [Eloquent Mutators & Accessors](https://laravel.com/docs/11.x/eloquent-mutators)

