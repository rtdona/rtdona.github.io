---
title: JavaScript로 키워드 필터링하기
description: JavaScript로 텍스트를 필터링하여 *로 변환해보기 
author: holymason
categories: [javascript]
tags: [javascript, jquery, filtering, masking, 필터링, 마스킹]
---

개발하다보면 이름이나 연락처 같은 개인정보 데이터를 웹, 앱에 보여줘야할 때가 생기는데,
개인정보는 노출되면 안되는 민감한 정보들이기 때문에, 이 데이터들은 그대로 보여주기보다는 필터링, 마스킹하여 보여줘야합니다.

# JavaScript로 DOM 텍스트 검증 및 변환

이 포스트에서는 **JavaScript**를 사용하여 DOM에 있는 텍스트를 검증하고,  
특정 키워드 배열에 포함된 텍스트만 변환하는 방법을 다룹니다.  
예를 들어 "코딩"이라는 단어는 "코*"로, "프로그래밍"은 "프***밍"으로 변환하는 작업을 해보겠습니다.

## HTML 구조

먼저, 간단한 HTML 페이지를 작성하여 텍스트를 검증하고 변환할 요소들을 정의합니다.

```html
  <p id="text1">코딩을 배우기 시작했습니다.</p>
  <p id="text2">코딩과 프로그래밍은 뭐가 다른건가요</p>
  <p id="text3">저는 프로그램을 만드는 프로그래머 입니다</p>
  <script src="app.js"></script>
```

## javascript (app.ks)

그리고 javascript에서 필터링을 처리해봅니다.

```javascript
// app.js

// 텍스트에 포함된 키워드 배열
const keywords = ['코딩', '프로그래밍'];

// containsAny 함수: 텍스트가 배열에 포함된 키워드 중 하나라도 포함되는지 검사
function containsAny(str, arr) {
  return arr.some(subStr => str.includes(subStr));
}

// 텍스트 변환 함수
function maskText(text) {
  // 배열에 포함된 키워드가 있는지 검사하고, 있으면 변환
  if (containsAny(text, keywords)) {
    keywords.forEach(keyword => {
      const regex = new RegExp(keyword, 'g'); // 키워드의 정규식
      
      // 첫글자, 끝글자를 제외한 문자는 *로 변환하여 리턴
      return text.replace(regex, match => {
        if(match.length < 1){
          return "";
        } else if (match.length == 1) {
          return "*";
        } else if (match.length == 2) {
          return match[0] + "*";
        } else {
          return match[0] + "*".repeat(match.length - 2) + match[match.length - 1];
        }

      });
    });
  }
  return text;
}

// DOM에서 텍스트 가져와 변환하기
document.querySelectorAll('p').forEach((element) => {
  let originalText = element.textContent;
  let maskedText = maskText(originalText);
  element.textContent = maskedText;
});


```
필터링을 적용하면...  
페이지에 표시되는 내용은 변환된 텍스트로 바뀌게 됩니다. 키워드가 포함된 부분만 *로 변환되며, 다른 텍스트는 그대로 유지됩니다.

```html
  <p id="text1">코*을 배우기 시작했습니다.</p>
  <p id="text2">코*과 프***밍은 뭐가 다른건가요</p>
  <p id="text3">저는 프로그램을 만드는 프로그래머 입니다</p>
```

## 결론
이번 예제에서는 JavaScript를 사용하여 DOM의 텍스트에 포함된 특정 키워드를  
`RegExp`와 `some()` 메서드를 활용하여 필터링하고 마스킹까지 해보았습니다

정규식은 몇년을 봐도 어색한 녀석이라 다음에 깊게 다루는걸로...


### 참고
arr.some(subStr => str.includes(subStr));
: Array.prototype.some()은 배열의 요소 중에서 하나라도 주어진 조건을 만족하는지 검사하는 메서드입니다. 이 메서드는 조건을 만족하는 요소가 하나라도 있으면 true를 반환하고, 그렇지 않으면 false를 반환합니다.

const regex = new RegExp(keyword, 'g'); // 키워드의 정규식
: RegExp는 JavaScript에서 정규식을 처리하기 위한 객체입니다. 정규식(Regular Expressions)은 문자열 내에서 특정 패턴을 찾고, 대체하거나 검증하는 데 사용됩니다. new RegExp(pattern, flags) 형태로 사용되며, pattern은 정규식 패턴을, flags는 정규식에 대한 설정 옵션을 지정합니다.


