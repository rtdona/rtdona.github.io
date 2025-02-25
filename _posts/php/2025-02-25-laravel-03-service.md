---
title: Laravel 02 - Service Container, Service Provider
description: Service Container, Service Provider
author: holymason
categories: [php]
tags: [php, laravel, 라라벨, mvc]
---

이 포스트에서는 Laravel의 기능 중 에 대해 간단하게 소개합니다.

# Laravel 서비스 컨테이너와 서비스 프로바이더 활용법

## 1. 서비스 컨테이너란?

Laravel의 **서비스 컨테이너(Service Container)** 는 의존성 주입(Dependency Injection, DI)을 관리하는 강력한 IoC(Inversion of Control) 컨테이너입니다.

### 1.1 서비스 컨테이너의 역할
- **객체 관리**: 클래스의 인스턴스를 자동으로 생성하고 관리
- **의존성 주입(DI)**: 클래스가 필요로 하는 의존성을 자동으로 주입
- **싱글톤 관리**: 특정 클래스의 인스턴스를 전역적으로 공유 가능

## 2. 서비스 컨테이너 사용법

### 2.1 바인딩(Binding) 개념
Laravel의 `AppServiceProvider` 또는 `routes/web.php`에서 서비스 컨테이너를 통해 클래스를 바인딩할 수 있습니다.

#### 2.1.1 기본 바인딩
```php
use App\Services\PaymentGateway;
use Illuminate\Support\Facades\App;

// 바인딩 예제 (서비스 프로바이더 또는 부트스트랩 파일에서 등록 가능)
App::bind('PaymentGateway', function () {
    return new PaymentGateway();
});

// 바인딩된 서비스 사용
$payment = App::make('PaymentGateway');
```

### 2.2 싱글톤 바인딩
하나의 인스턴스를 여러 곳에서 재사용하고 싶을 때 `singleton` 메서드를 사용합니다.

```php
App::singleton('Logger', function () {
    return new \App\Services\LoggerService();
});
```

이제 `Logger`의 동일한 인스턴스가 애플리케이션 전체에서 공유됩니다.

### 2.3 인터페이스 바인딩
인터페이스를 사용하여 결합도를 낮출 수 있습니다.

#### 2.3.1 인터페이스와 구현 클래스 정의
```php
namespace App\Contracts;
interface PaymentGatewayInterface {
    public function processPayment($amount);
}
```

```php
namespace App\Services;
use App\Contracts\PaymentGatewayInterface;

class StripePaymentGateway implements PaymentGatewayInterface {
    public function processPayment($amount) {
        return "Processing payment of $amount via Stripe";
    }
}
```

#### 2.3.2 인터페이스와 구현 클래스 바인딩
```php
use Illuminate\Support\Facades\App;
use App\Contracts\PaymentGatewayInterface;
use App\Services\StripePaymentGateway;

App::bind(PaymentGatewayInterface::class, StripePaymentGateway::class);
```

#### 2.3.3 컨트롤러에서 의존성 주입 사용
```php
use App\Contracts\PaymentGatewayInterface;

class PaymentController {
    protected $paymentGateway;

    public function __construct(PaymentGatewayInterface $paymentGateway) {
        $this->paymentGateway = $paymentGateway;
    }

    public function pay() {
        return $this->paymentGateway->processPayment(100);
    }
}
```

## 3. 서비스 프로바이더란?

서비스 프로바이더(Service Provider)는 **서비스 컨테이너를 통해 바인딩된 클래스를 등록하는 역할**을 합니다. Laravel의 모든 서비스는 서비스 프로바이더를 통해 등록됩니다.

### 3.1 서비스 프로바이더 생성
```bash
php artisan make:provider PaymentServiceProvider
```

이 명령어를 실행하면 `app/Providers/PaymentServiceProvider.php` 파일이 생성됩니다.

### 3.2 서비스 프로바이더 설정
```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Contracts\PaymentGatewayInterface;
use App\Services\StripePaymentGateway;

class PaymentServiceProvider extends ServiceProvider {
    public function register() {
        $this->app->bind(PaymentGatewayInterface::class, StripePaymentGateway::class);
    }

    public function boot() {
        // 필요한 초기화 작업 수행
    }
}
```

### 3.3 서비스 프로바이더 등록
`config/app.php`의 `providers` 배열에 추가해야 합니다.
```php
'providers' => [
    ...
    App\Providers\PaymentServiceProvider::class,
];
```

이제 `PaymentGatewayInterface`를 의존성으로 요청하면 `StripePaymentGateway`가 자동으로 주입됩니다.

## 4. 서비스 컨테이너 & 서비스 프로바이더 활용

### 4.1 컨트롤러에서 사용하기
```php
use App\Contracts\PaymentGatewayInterface;

class CheckoutController {
    protected $paymentGateway;

    public function __construct(PaymentGatewayInterface $paymentGateway) {
        $this->paymentGateway = $paymentGateway;
    }

    public function checkout() {
        return $this->paymentGateway->processPayment(200);
    }
}
```

### 4.2 Artisan 명령어에서 사용하기
```php
use Illuminate\Console\Command;
use App\Contracts\PaymentGatewayInterface;

class ProcessPaymentCommand extends Command {
    protected $signature = 'payment:process';
    protected $description = 'Process a payment';

    public function __construct(protected PaymentGatewayInterface $paymentGateway) {
        parent::__construct();
    }

    public function handle() {
        $this->info($this->paymentGateway->processPayment(500));
    }
}
```

## 5. 정리 및 결론
- **서비스 컨테이너**는 클래스의 인스턴스를 생성하고, 의존성을 주입하는 역할을 한다.
- **서비스 프로바이더**는 서비스 컨테이너를 통해 클래스를 등록하고 관리하는 역할을 한다.
- **인터페이스 바인딩**을 통해 결합도를 낮추고 확장성을 높일 수 있다.
- **의존성 주입(DI)**을 활용하면 테스트가 용이하고 유지보수가 쉬운 코드 작성이 가능하다.

이제 Laravel의 서비스 컨테이너와 서비스 프로바이더를 활용하여 **더 유연하고 확장성 있는 애플리케이션**을 개발해보세요! 🚀

