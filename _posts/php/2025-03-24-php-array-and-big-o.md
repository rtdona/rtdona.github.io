---
title: Php 배열과 시간복잡도 (Big O표기법)
description: Php 배열 예제로 알고리즘 알아보기
author: holymason
categories: [php, algorithm]
tags: [php, algorithm, big-o, array-flip]
---

배송 지역 및 지도 관련 API를 테스트하던 중,  
특정 권역에 대한 우편번호 데이터를 배열 형태로 제공받아, 이 배열 내에서 로직에 필요한 특정 우편번호를 검색하여 사용할 필요가 있었습니다.  
처음에는 간단한 코드 `in_array($myPostCode,$postCodeArray)` 를 사용해 배열 검색을 처리하려고 했지만, 더 효율적인 방법을 찾기 위해 고민하게 되었습니다.   
이에 대한 내용을 포스트로 정리해보았습니다.

## 목차

1. [알고리즘과 복잡도](#1-알고리즘과-복잡도)
2. [배열 검색](#2-배열-검색)
3. [결론](#결론)


## 1. 알고리즘과 복잡도

### 1.1. 알고리즘 Algorithm

* **알고리즘**은 특정 문제를 해결하기 위한 명확하게 정의된 일련의 단계적 절차입니다. 
* 정확성: 문제에 대한 올바른 해결책을 제공해야 합니다.
* 효율성: 시간과 공간(메모리) 자원을 최소화해야 합니다.
* 결정성: 동일한 입력에 대해 항상 동일한 결과를 반환해야 합니다.
* 유한성: 유한한 시간 내에 종료되어야 합니다.

## 1.2. 시간 복잡도(Time Complexity)와 공간 복잡도(Space Complexity)
* 복잡도란, 알고리즘의 성능과 효율성을 나타내는 척도입니다.
* **시간 복잡도**는 알고리즘이 실행되는 데 필요한 시간(Time)을 정량화한 것으로, 입력 크기에 따른 연산 횟수의 증가율을 측정합니다.
* **공간 복잡도**란 알고리즘이 실행되는 데 얼마나 많은 공간(Space, Memory)이 필요한지를 나타냅니다.

## 1.3. 빅오표기법 (Big-O Notation)
* Big O 표기법은 입력 크기(n)에 따라 알고리즘의 실행 시간이 어떻게 증가하는지를 표현하는 방식입니다.
* 알고리즘의 **최악의 경우(Worst Case)** 를 기준으로 성능을 측정합니다.

| Big O      | 의미        | 설명                                    |
|------------|------------|---------------------------------------|
| **O(1)**  | 상수 시간  | 입력 크기와 관계없이 항상 일정한 실행 시간을 가집니다        |
| **O(log n)** | 로그 시간 | 데이터 크기를 절반씩 줄여가며 탐색합니다 (예: 이진 탐색)     |
| **O(n)**  | 선형 시간  | 입력 크기에 비례하여 실행 시간이 증가합니다              |
| **O(n log n)** | 로그 선형 시간 | 정렬 알고리즘 (예: 병합 정렬)                    |
| **O(n²)** | 제곱 시간 | 두 개의 중첩 반복문을 사용 (예: for in for, 버블 정렬) |
| **O(2ⁿ)** | 지수 시간 | 입력 크기가 증가할수록 실행 시간 급격히 증가 (예: 피보나치) |

여기까지 알고리즘에 대해서 간단히 알아보았고, 더 깊은 내용은 다른 포스트에서 다뤄보겠습니다

## 2. 배열 검색

php 배열 검색 예제를 통해 각각의 알고리즘의 성능을 확인해보겠습니다.

### 2.1. 테스트 준비

```php
// 우편번호 생성 : 랜덤 문자열
function randomString($length = 5) {
    return substr(str_shuffle("0123456789"), 0, $length);
}

// 50,000개의 우편번호를 가진 배열 생성
$arraySize = 50000;
$zipcodes = [];
for ($i = 0; $i < $arraySize; $i++) {
    $zipcodes[] = randomString();
}
```

### 2.2. in_array() 함수
> `in_array(mixed $needle, array $haystack, bool $strict = false): bool`  
> Searches for `$needle` in `$haystack` using loose comparison unless strict is set.  
> 번역 : strict가 설정되어 있지 않으면 느슨한 비교를 사용하여 `$haystack`(배열)에서 `$needle`(찾을 값)을 검색합니다.  
> 출처 : https://www.php.net/manual/en/function.in-array.php

* in_array() 함수는 내부적으로 선형 탐색(Linear Search, O(n)) 알고리즘을 사용합니다.  
* 따라서, 배열의 크기가 커질수록 성능이 크게 저하될 수 있습니다.

```php
// in_array() 검색 예제 : O(n)
$start_in_array = microtime(true);
$needle = $zipcodes[array_rand($zipcodes)]; // 검색할 값 (배열 중 하나 선택)
$result = in_array($needle, $zipcodes);
$end_in_array = microtime(true);
$time_in_array = ($end_in_array - $start_in_array) * 1000; // 실행시간 ms(밀리초)
echo "in_array 실행 시간: {$time_in_array} ms";
```

`in_array()` 실행결과

```
in_array 실행 시간: 0.47993659973145 ms 
```

### 2.3. array_flip() 함수
> `array_flip(array $array): array`  
> `array_flip()` returns an array in flip order, i.e. keys from array become values and values from array become keys.
> 번역 : `array_flip()`은 뒤집힌 순서로 배열을 반환합니다. 즉, 배열의 키가 값이 되고, 배열의 값이 키가 됩니다.  
> 출처 : https://www.php.net/manual/en/function.array-flip.php

* array_flip() 함수는 내부적으로 배열을 처음부터 끝까지 선형 탐색(Linear Search, O(n)) 알고리즘을 사용합니다.
* 따라서, 배열의 크기가 커질수록 성능이 크게 저하될 수 있습니다.

```php
// array_flip() + isset() 사용 
$start_array_flip = microtime(true);
$flipped_zipcodes = array_flip($zipcodes);  // O(n)
$needle = $zipcodes[array_rand($zipcodes)];
$result = isset($flipped_zipcodes[$needle]); // O(1)
$end_array_flip = microtime(true);
$time_array_flip = ($end_array_flip - $start_array_flip) * 1000;
echo "array_flip 실행 시간: {$time_array_flip} ms";
```

`array_flip()` 실행결과

```
array_flip 실행 시간: 2.7070045471191 ms
```

두 함수의 실행 시간을 볼 때, `in_array()` 보다 `array_flip()` + `isset()` 의 실행시간이 더 긴 것을 확인할 수 있는데, 이는 `isset()` 함수를 실행할 때 O(1) 의 시간만큼 더 소요되기 때문입니다.

### 2.4. 반복 검색

* 앞서 `in_array()` 와 `array_flip()` + `isset()` 각각의 실행 시간을 확인해보았는데, 배열에서 반복 검색할 때는 어떤지 확인해보겠습니다.

```php
$count = 1000; // 각각의 배열($zipcodes, $flipped_zipcodes)을 검색할 횟수

// 1. in_array() 
$start_in_array = microtime(true);
for($j=0; $j < $count; $j++){
   $needle = $zipcodes[array_rand($zipcodes)]; // 검색할 값 (배열 중 하나 선택)
   $result = in_array($needle, $zipcodes);
}
$end_in_array = microtime(true);
$time_in_array = ($end_in_array - $start_in_array) * 1000;
echo "in_array 실행 시간: {$time_in_array} ms";

// 2. array_flip() + isset() 
$start_array_flip = microtime(true);
$flipped_zipcodes = array_flip($zipcodes);  // O(n)
for($k=0; $k < $count; $k++) {
   $needle = $zipcodes[array_rand($zipcodes)];
   $result = isset($flipped_zipcodes[$needle]); // O(1)
}
$end_array_flip = microtime(true);
$time_array_flip = ($end_array_flip - $start_array_flip) * 1000;
echo "array_flip 실행 시간: {$time_array_flip} ms";
```

반복 실행 결과

```
in_array 실행 시간: 1695.3239440918 ms
array_flip 실행 시간: 3.2780170440674 ms
```

반복 실행으로 확인해본 결과, `in_array()`는 O(n) 선형 탐색을 반복해서 실행하는 반면,  
`array_flip()`을 사용하면 O(n) 으로 1회 변환 후 반복해서 실행하는 로직은 O(1) 로 검색할 수 있어 반복 검색 시 훨씬 빠른 결과를 나타냅니다.

## 결론 
우편번호 데이터 **배열의 크기**와 그리고 **배열 검색 횟수**에 따라 가장 효율적인 성능을 낼 수 있는 함수를 선택하여 사용하면 됩니다.
* `array_flip()` 방식은 검색 횟수가 많을수록 `isset()`의 O(1) 속도 덕분에 훨씬 더 빠르게 동작합니다.
* `in_array()`는 배열 크기가 커지고, 검색 횟수가 많아질수록 선형 탐색 O(n) 비용이 누적되며 점점 느려집니다.
* 따라서, 자주 검색해야 하는 경우에는 `array_flip()` 을 활용하여 `isset()`을 사용하는 것이 훨씬 효율적입니다.
