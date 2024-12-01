---
title: Speculation rules API - 사전 렌더링으로 페이지 전환 속도를 개선하기
description: Speculation rules API에 대해 알아보고 사전 렌더링을 통해 페이지 전환 속도를 개선하는 방법에 대해 알아봅니다.
date: 2024-12-01 17:30:00 +0900
categories: [Web, Performance]
tags: [speculation rules api, 속도개선, 성능개선]
toc: true
toc_sticky: true
toc_label: 목차
pin: true
math: true
mermaid: true
# image:
#   path: /commons/devices-mockup.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices.
---

> 이 글은 24년 Google I/O 에서 Barry Pollard 가 발표한 **From fast loading to instant loading** 이라는 제목의 발표를 참고하여 작성했습니다.

<!-- start post -->

## 들어가는 글

---

현 시점 대형 웹 서비스들은 기본적으로 수많은 페이지로 구성되어 있습니다. 그리고 사용자들은 서비스를 이용할 때 한 페이지에서 오랫동안 머물지 않고, 많은 페이지를 이동하면서 서비스를 소비합니다. 이러한 환경에서 페이지 이동시 페이지의 로드 속도는 서비스 사용성의 핵심 지표이며, 이 속도가 느릴 때 사용자들은 답답함을 느끼게 됩니다.

구글은 수 년전, 웹 서비스의 사용성 측정 지표로 LCP(Largest Contentful Paint) 를 제안했습니다. LCP는 말그대로 페이지의 가장 큰 콘텐츠가 화면에 그려지기 까지의 시간을 의미하며, 지표상 2.5초 이내이면 Normal한 것으로 간주됩니다. 하지만 2.5초가 충분할까요? 페이지를 이동했을 때 콘텐츠가 그려지기까지 평균 2.5초가 걸린다면 사용자는 답답함을 느낄겁니다. 사실 2.5초라는 길어보일 수 있는 기준이 세워진 데에는 기술적인 달성가능성도 고려되었기 때문이라고 합니다. 페이지의 로드 속도에는 네트워크, 서버 성능, 클라이언트 디바이스의 성능, 페이지를 구성하는 코드 등 여러가지 복합적인 요소가 얽혀있기 때문에 마법처럼 로드 속도를 제로로 만드는 것은 불가능에 가깝습니다.

React.js로 대표되는 SPA(Single Page Application) 에서는 페이지 전환 시 로드 속도와 사용자 경험을 개선하기 위해 소위 Client side routing 이라는 기능을 제공합니다. react-router 와 같은 라이브러리가 대표적이죠. Client side routing을 이용하면, 페이지 전환 시 클라이언트에서 서버에 페이지, 즉, HTML을 요청하지 않습니다. 대신 Javascript 를 이용해서 페이지가 바뀌었을 때 필요한 리소스만 받아오고, 필요한 부분만 업데이트하죠. 이는 마치 사용자가 서비스를 이용할 때, 페이지를 이동하더라도 마치 하나의 웹페이지를 탐색하는 것과 같은 사용자 경험을 제공합니다. 따라서 SPA 내에서의 페이지 전환은 거의 Instant Loading에 가깝습니다.

하지만, 엔터프라이즈 서비스에서는 서비스를 통째로 SPA로 만들기는 어렵습니다. 오히려 대형 서비스를 하나의 SPA로 만든다면 영향도 분리가 되지 않아 위험할 수 있습니다. 그래서 여러 엔터프라이즈 서비스는 여러 SPA로 이루어진 MPA(Multi Page Application) 형태로 서비스하고 있습니다. 문제는 분리된 서비스간 링크 이동 시 발생합니다. 이 경우 전통적인 링크 이동 방식(`a` 태그 혹은 `window.location.href` 와 같은) 을 사용하게 되고, 여러분이 알고계신 것과 같은 과정이 수행됩니다. 이 과정에서 사용자나 서비스 인프라의 네트워크 상태, 서버의 성능 상태, 사용자 디바이스의 성능 상태 등등 여러가지 변수에 의해 사용자는 느린 페이지 로드를 경험할 수 있고, 이런 경우 사용자는 답답함을 느끼고 서비스가 느리다고 불평하게 됩니다.

## 웹 페이지 로드 속도를 개선하는 방법

---

웹페이지 로드 속도를 개선하는 방법에는 두 가지 방법이 있습니다.

### 1. Load faster

말 그대로 로드 속도를 개선하는 방법입니다. 서버 성능 개선, 프론트엔드 코드의 성능 개선, 네트워크와 인프라 개선 등등이 포함될 수 있으며, 우리가 계속 하고 있는 것들이죠. 하지만 이 방법에는 당연히 한계가 있고, 조금의 시간을 줄이기 위해 많은 리소스가 필요하기도 합니다.

### 2. Load in advance

다음 페이지의 리소스를 미리 로드하는 방법입니다. 다음 페이지의 리소스를 미리 로드하면, 사용자가 링크를 누르기 전에 미리 리소스를 로드하기 때문에 위에서 말한 변수가 어느 정도 있는 상태라도, 미리 받아놓은 리소스로 빠르게 페이지를 전환시키는 것이 가능합니다.

## 페이지 로드 속도를 개선하기 위한 시도들

---

1. `rel=preload`

   “현재 페이지”에서 사용되는 하위 정적 리소스(JS, CSS, Image 등)를 좀 더 빠르게 로드하기 위해 사용됩니다.

2. `rel=prefetch`

   다음 페이지에서 사용할 리소스를 미리 fetch 하여 캐시에 저장합니다. 하지만, 다음 페이지의 다양한 정적 리소스를 하나하나 추가해주어야 하며, HTML을 미리 fetch할 때는 잘 동작하지 않는 경우가 많았습니다.

3. `rel=prerender`

   Chrome 외에 광범위하게 지원되지 않았으며, 그다지 표현력이 좋은 API도 아니었습니다. 현재 지원 중단 되었습니다.

## Speculation rules API

---

Speculation rules API 는 구글에서 제안한 JSON 기반 API입니다. Spculation rules API의 기본적인 예시는 아래와 같습니다.

```html
<script type="speculationrules">
  {
    "prerender": [
      {
        "urls": ["/next-page", "/next-page-2"]
      }
    ]
  }
</script>
```

또는 [URL Pattern API](https://developer.mozilla.org/en-US/docs/Web/API/URL_Pattern_API) 기반의 href selector 나 css selector 를 이용하여, 매칭되는 링크의 페이지만 사전 렌더링할 수 있고 필터링도 할 수 있습니다.

```html
<script type="speculationrules">
  {
    "prerender": [
      {
        "where": {
          "and": [
            { "href_matches": "/*" },
            { "not": { "href_matches": "/wp-admin" } },
            { "not": { "href_matches": "/*\\?*(^|&)add-to-cart=*" } },
            { "not": { "selector_matches": ".do-not-prerender" } },
            { "not": { "selector_matches": "[rel~=nofollow]" } }
          ]
        }
      }
    ]
  }
</script>
```

또는, javascript 로 동적으로 추가하는 방법도 있습니다.

```jsx
if (
  HTMLScriptElement.supports &&
  HTMLScriptElement.supports("speculationrules")
) {
  const specScript = document.createElement("script");
  specScript.type = "speculationrules";
  const specRules = {
    prerender: [
      {
        urls: ["/next.html"]
      }
    ]
  };
  specScript.textContent = JSON.stringify(specRules);
  document.body.append(specScript);
}
```

위 코드에 포함되어 있지만, 아래와 같이 Speculation rules API 지원 여부를 체크할 수 있습니다.

```jsx
if (
  HTMLScriptElement.supports &&
  HTMLScriptElement.supports("speculationrules")
) {
  console.log("Your browser supports the Speculation Rules API.");
}
```

## 개발자 도구 지원

---

Speculations rules API는 크롬 개발자도구에서 지원하고 있습니다.

크롬 개발자도구를 열어서 애플리케이션 탭을 확인해보면, 백그라운드 서비스 - 추측 로드에서 Speculations rules API로 현재 설정되어있는 규칙들과, 상태 등을 확인할 수 있습니다.

![chrome-devtool-speculation-rules-api](/assets/img/speculation-developertools1.png)

![chrome-devtool-speculation-rules-api](/assets/img/speculation-developertools2.png)

## Speculation rules API 실습 해보기

---

사용법을 알았으니, 실제로 사용을 해보고 효과를 확인해보겠습니다. 이미지와 컨텐츠가 많은 무신사 서비스를 이용해서 테스트 해보겠습니다.

![musinsa-home](/assets/img/speculation-musinsa-main.png)

무신사의 홈 화면 입니다. 무신사의 서비스를 이용하는 유저들이 홈화면에 들어온 후, **하단 네비게이션의 좋아요 탭으로 이동하는 비율이 높다**고 가정해보겠습니다.

![musinsa-like](/assets/img/speculation-navs.png)

좋아요 탭은 `<a>` 태그로 전통적인 링크 이동을 하고 있어, 이동 시 서버로부터 새로 리소스를 받아온다는 사실을 알 수 있습니다.

그럼 Speculation rules API를 사용하지 않은 상태에서 좋아요 탭으로 이동해보고 성능을 확인해보겠습니다.

좀 더 드라마틱한 효과를 보기 위해 개발자도구에서 **네트워크 속도를 “빠른 4G”**로 변경하고 테스트를 진행해보겠습니다.

![musinsa-like-before](/assets/img/speculation-test1.gif)
![musinsa-like-before-lcp](/assets/img/speculation-test1-performance.png)

측정된 LCP는 **1.34초** 입니다.

그럼 이제 Speculation rules API를 사용해서 좋아요 페이지를 prerender 해보겠습니다.

![musinsa-snippet](/assets/img/speculation-snippet.png)

크롬 개발자 도구에서 위와 같이 코드 스니펫을 활용하여, `/like/goods` 경로를 prerender 하도록 세팅하겠습니다.

![musinsa-devtools-application](/assets/img/speculation-devtool-application.png)
위와 같이 크롬 개발자도구 애플리케이션 탭의 추측 로드에서 `/like/goods` 가 Prerender 된 것을 확인할 수 있습니다. 다시 좋아요 탭을 클릭해보겠습니다.

![musinsa-like-after](/assets/img/speculation-test2.gif)
![musinsa-like-after-lcp](/assets/img/speculation-test2-performance.png)

빨라진 속도가 체감 되시나요? 사전 렌더링이 수행되었기 때문에, SPA 웹서비스에서 클라이언트 사이드 렌더링으로 이동된 것과 비슷하게 느껴지는 것 같습니다.

측정 항목을 확인해보면 LCP가 0.15초로 약 88% 정도나 감소한 것을 알 수 있습니다.

## 과잉 추측을 방지하기

---

위에서 살펴본대로, Speculations rules API를 활용하면, 멀티 페이지 웹 어플리케이션에서도 사전 렌더링을 통해, 페이지 전환의 속도를 다른 차원으로 끌어올릴 수 있다는 것을 알게 되었습니다.

그럼 페이지의 모든 링크에 Speculations rules API를 적용하는 것이 좋을까요?

예상 하시겠지만, 그렇지 않습니다. 사용자가 링크를 눌러서 페이지를 이동하기 전에, 미리 예측하여 사전 로드와 사전 렌더링을 수행하는 것은 곧 예측이 실패하여 사용자가 그 링크를 누르지 않는다면 불필요한 대역폭, 메모리, CPU 리소스를 사용하게 된다는 것을 의미합니다. 사용자 입장에서는 불필요한 다운로드가 발생하게 될 것이고, 서비스 제공자 입장에서는 더 많은 요청량이 발생하기 때문에 인프라 비용이 증가할 수 있는 등의 위험성이 있습니다. 따라서 데이터를 기반으로 클릭율이 높은 링크의 페이지를 사전렌더링 하도록 설정하는 것이 좋습니다.

### eagerness 설정

Speculations Rules API 에는 과잉 추측으로 리소스를 낭비하는 것을 방지하기 위한 `eagerness` 설정이 있습니다. 이 설정값에 따라서 추측할 시점과 추측을 실행할 시점을 선택할 수 있습니다.

```jsx
<script type="speculationrules">
{
  "prerender": [{
    "source": "document",
    "where": {
      "href_matches": "/*"
    },
    "eagerness": "moderate"
  }]
}
</script>
```

아래는 `eagerness` 설정 값과 그에 대한 설명입니다.

- **`immediate`:** 가능한 한 빨리, 즉 추측 규칙이 관찰되는 대로 추측할 때 사용합니다.
- **`eager`:** 현재 `immediate` 설정과 동일하게 작동하지만, 향후에는 `immediate`와 `moderate`의 중간 정도로 작동할 예정입니다.
- **`moderate`:** 링크에 마우스를 200밀리초 동안 가져가면 (또는 그보다 빠른 경우 [`pointerdown`](https://developer.mozilla.org/docs/Web/API/Element/pointerdown_event) 이벤트에서 그리고 `hover` 이벤트가 없는 모바일에서) 추측을 실행합니다.
- **`conservative`:** 포인터 다운 또는 터치 다운 시 추측합니다.

## prefetch

---

prefetch 는 사전 렌더링은 수행하지 않고, 페이지를 미리 가져오기만 합니다. 바로 prerender를 사용하기에는 부담되는 경우, 먼저 prefetch를 적용해보고 점진적으로 prerender를 적용해볼 수 있습니다.

```jsx
<script type="speculationrules">
{
  "prefetch": [
    {
      "urls": ["/next-page", "/next-page-2"]
    }
  ]
}
</script>
```

## 제한사항

---

아쉽게도 Speculations Rules API는 아직 브라우저의 지원 범위가 넓지 않습니다. 크롬 브라우저는 109 버전부터 지원하고 있지만, 파이어폭스나 사파리에서는 아직 지원하고 있지 않습니다. 웹뷰도 안드로이드 웹뷰에서만 지원하고 있습니다.

### 🤔 모든 브라우저에서 지원 안되면 어차피 못쓰는 거 아닌가요?

아직 브라우저 지원 범위가 넓진 않지만, 그렇다고 쓸 수 없는 건 아닙니다. 만약 Speculations Rules API가 Javascript 문법으로 호출해야 했다면, 다른 브라우저에서는 에러가 발생할 수 있기 때문에 번거롭게도 개발자가 별도의 처리를 해주어야 했을겁니다. 하지만 크롬 팀에서는 이와 같은 문제를 인지하고 있었고, Speculations Rules API를 위에서 보신대로 JSON 형태로 구현했습니다. **이 API를 지원하지 않는 브라우저에서는 인식할 수 없는 스크립트로 유형으로 인지하여 무시됩니다.** 개발자는 룰을 추가하는 것 외에 다른 어떤 조치를 할 필요가 없습니다. 또한 별도의 처리가 필요한 경우, 브라우저가 Speculations Rules API를 지원하는지 확인할 수 있는 아래와 같은 API를 제공합니다.

```jsx
if (
  HTMLScriptElement.supports &&
  HTMLScriptElement.supports("speculationrules")
) {
  console.log("Your browser supports the Speculation Rules API.");
}
```

## 마치며

---

웹 개발자, 특히 프론트엔드 개발자에게 웹 페이지의 로드와 렌더링 속도를 높여 사용자에게 좋은 사용성을 제공하는 것은 어떤 서비스를 만들던지 항상 중요한 목표로 여겨집니다. 이를 위해 코드 상에서의 처리 속도를 높이고, 리렌더링을 줄이고, 캐시를 활용하고, CDN을 사용하고, 다운로드 받는 리소스의 용량을 줄이는 등 복잡한 문제들을 해결하고 있습니다.

Speculations Rules API는 아예 페이지로 접속하기 전, 예측을 통해 해당 페이지를 사전 렌더링하는 형태로 문제에 대한 접근을 했다는 점, 그리고 그것을 비교적 간단한 사용 방법으로 구현해냈다는 점에서 인상적이었습니다. 이 API의 브라우저 지원 범위가 하루 빨리 확장 되었으면 좋겠네요.

## 참고 문서

---

- [즉각적인 페이지 탐색을 위해 Chrome에서 페이지 사전 렌더링](https://developer.chrome.com/docs/web-platform/prerender-pages?hl=ko#add_speculation_rules_dynamically_through_javascript)
- [Speculation Rules API 개선](https://developer.chrome.com/blog/speculation-rules-improvements?hl=ko)
