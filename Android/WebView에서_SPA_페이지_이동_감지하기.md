# Android WebView에서 SPA 페이지 이동 감지하기

## 개요

Android 앱에서 WebView로 SPA(Single Page Application) 기반 웹페이지를 표시하는 경우

웹페이지 내부에서 화면이 전환되더라도 다음 WebView 콜백이 호출되지 않을 수 있습니다.

```kotlin
shouldOverrideUrlLoading()
onPageFinished()
```

반면 SPA 내부에서 URL 또는 History가 변경되는 경우에는 다음 콜백을 통해 변경 사항을 감지할 수 있습니다.

```kotlin
doUpdateVisitedHistory()
```

이는 SPA의 페이지 전환 방식이 일반적인 웹페이지의 navigation 방식과 다르기 때문입니다.

Android WebView 입장에서는 새로운 페이지를 다시 로드한 것이 아니라, 기존 페이지 상태에서 URL과 화면만 변경된 것으로 처리될 수 있습니다.

---

## 1. 일반적인 웹 페이지 이동

일반적인 웹페이지는 다른 페이지로 이동할 때 새로운 HTML document를 요청합니다.

예를 들어 다음과 같은 흐름입니다.

```text
https://www.naver.com
        ↓
사용자가 메뉴 클릭
        ↓
https://www.naver.com/profile 요청
        ↓
새 HTML document 로드
        ↓
WebView navigation 발생
```

이 경우 Android WebView에서는 일반적으로 다음과 같은 콜백이 호출됩니다.

```text
shouldOverrideUrlLoading()
        ↓
onPageStarted()
        ↓
onPageFinished()
```

WebView가 새로운 URL로 navigation을 수행하고 새로운 document를 로드하기 때문에 Android 앱에서는 `WebViewClient`의 관련 콜백을 통해 페이지 이동 상태를 확인할 수 있습니다.

---

## 2. SPA에서 `shouldOverrideUrlLoading()`이 호출되지 않는 이유

SPA는 페이지를 이동할 때 새로운 HTML document를 다시 요청하지 않는 경우가 많습니다.

React, Vue, Next.js, Nuxt 등의 SPA 라우터는 일반적으로 JavaScript의 History API를 사용해 URL과 화면 상태를 변경합니다.

예를 들어:

```javascript
history.pushState({}, "", "/ko/profile")
```

또는:

```javascript
history.replaceState({}, "", "/ko/profile")
```

와 같은 방식입니다.

실제 동작은 다음과 비슷합니다.

```text
https://www.naver.com/
        ↓
사용자가 메뉴 클릭
        ↓
JavaScript Router가 페이지 이동 처리
        ↓
history.pushState(...)
        ↓
URL을 /profile 로 변경
        ↓
기존 document에서 화면만 다시 렌더링
```

이 경우 WebView가 새로운 URL을 직접 로드하는 navigation 과정은 발생하지 않습니다.

따라서 Android의 다음 콜백이 호출되지 않을 수 있습니다.

```kotlin
shouldOverrideUrlLoading()
```

`shouldOverrideUrlLoading()`은 WebView에서 URL navigation이 발생할 때 해당 URL을 WebView에서 계속 처리할지

Android 앱에서 별도로 처리할지 판단하기 위한 콜백입니다.

SPA 내부 라우팅은 JavaScript에서 URL과 화면 상태를 직접 변경하기 때문에 WebView 자체의 URL loading 과정이 발생하지 않을 수 있습니다.

결과적으로 Android 앱에서는 `shouldOverrideUrlLoading()`을 통해 SPA 내부 페이지 이동을 감지하지 못할 수 있습니다.

---

## 3. SPA에서 `onPageFinished()`가 호출되지 않는 이유

`onPageFinished()`는 WebView의 main frame에서 페이지 로딩이 완료되었을 때 호출되는 콜백입니다.

일반적인 페이지 이동에서는 다음과 같은 흐름으로 동작합니다.

```text
새 HTML document 요청
        ↓
document 로딩
        ↓
onPageFinished()
```

하지만 SPA에서는 새로운 HTML document를 다시 로드하지 않고 기존 document를 계속 사용합니다.

```text
기존 document 유지
        ↓
JavaScript Router 실행
        ↓
DOM 변경
        ↓
화면만 다시 렌더링
```

Android WebView 입장에서는 기존 페이지의 document가 이미 로딩 완료된 상태입니다.

SPA 내부에서 화면이 변경되더라도 새로운 document를 로드한 것이 아니기 때문에:

```kotlin
onPageFinished()
```

가 다시 호출되지 않을 수 있습니다.

정리하면 SPA 내부 페이지 이동은 WebView 관점에서 다음과 같이 볼 수 있습니다.

```text
SPA 페이지 이동

새 document 로드         X
WebView navigation     X
URL 변경                O
DOM / 화면 변경          O
```

따라서 WebView 콜백은 다음과 같이 동작할 수 있습니다.

| WebView Callback             | SPA 내부 페이지 이동 |
| ---------------------------- | ---------- |
| `shouldOverrideUrlLoading()` | 호출되지 않을 수 있음 |
| `onPageStarted()`            | 호출되지 않을 수 있음 |
| `onPageFinished()`           | 호출되지 않을 수 있음 |
| `doUpdateVisitedHistory()`   | 호출         |

---

## 4. `doUpdateVisitedHistory()`란

`doUpdateVisitedHistory()`는 WebView의 방문 기록(History)이 변경되었을 때 호출되는 `WebViewClient` 콜백입니다.

Android에서는 다음과 같이 사용할 수 있습니다.

```kotlin
override fun doUpdateVisitedHistory(
    view: WebView,
    url: String?,
    isReload: Boolean
) {
    super.doUpdateVisitedHistory(view, url, isReload)
}
```

SPA에서는 새로운 document를 로드하지 않더라도 JavaScript Router가 다음과 같은 History API를 사용할 수 있습니다.

```javascript
history.pushState(...)
history.replaceState(...)
```

이 경우 WebView의 document는 그대로 유지되지만 URL 또는 History 정보는 변경됩니다.

따라서 Android WebView에서는:

```kotlin
doUpdateVisitedHistory()
```

를 통해 해당 변경을 감지할 수 있습니다.

Android 개발자 관점에서 각 콜백의 차이는 다음과 같습니다.

```text
shouldOverrideUrlLoading()
    → WebView navigation 발생 여부 확인

onPageFinished()
    → document 로딩 완료 여부 확인

doUpdateVisitedHistory()
    → WebView URL / History 변경 여부 확인
```

SPA에서는 새로운 페이지를 로드하지 않더라도 URL 또는 History가 변경될 수 있기 때문에 `doUpdateVisitedHistory()`를 사용할 수 있습니다.

---

## 5. SPA 페이지 이동 감지는 `doUpdateVisitedHistory()`로 대체 가능

Android 앱에서 SPA 내부 페이지 이동이나 URL 변경을 확인해야 하는 경우:

```kotlin
shouldOverrideUrlLoading()
```

또는:

```kotlin
onPageFinished()
```

만으로는 페이지 이동을 감지하지 못할 수 있습니다.

이 경우 `doUpdateVisitedHistory()`를 사용할 수 있습니다.

```kotlin
webView.webViewClient = object : WebViewClient() {

    override fun shouldOverrideUrlLoading(
        view: WebView,
        request: WebResourceRequest
    ): Boolean {
        Log.d(
            "WebView",
            "shouldOverrideUrlLoading: ${request.url}"
        )

        return false
    }

    override fun onPageFinished(
        view: WebView,
        url: String
    ) {
        super.onPageFinished(view, url)

        Log.d(
            "WebView",
            "onPageFinished: $url"
        )
    }

    override fun doUpdateVisitedHistory(
        view: WebView,
        url: String?,
        isReload: Boolean
    ) {
        super.doUpdateVisitedHistory(view, url, isReload)

        Log.d(
            "WebView",
            "doUpdateVisitedHistory: $url"
        )
    }
}
```

예를 들어 SPA 내부에서 URL이 다음과 같이 변경된다고 가정합니다.

```text
https://www.naver.com/
        ↓
https://www.naver.com/profile
```

새로운 document를 로드하지 않는 SPA 내부 navigation이라면 Android에서는 다음과 같이 동작할 수 있습니다.

```text
shouldOverrideUrlLoading()   X
onPageFinished()             X
doUpdateVisitedHistory()     O
```

따라서 Android 앱에서 SPA 내부 페이지 이동 또는 URL 변경 여부를 확인해야 한다면:

```kotlin
doUpdateVisitedHistory()
```

를 사용할 수 있습니다.

특히 기존에 `onPageFinished()`에서 URL 변경에 따른 후처리를 수행하고 있었다면

SPA에서는 해당 로직을 `doUpdateVisitedHistory()`에서 처리하는 방식으로 대체할 수 있습니다.

---

## 6. 쿠키 저장과 함께 사용하는 경우

Android WebView에서 쿠키를 사용하고 있고, SPA 내부 페이지 이동 시점에 쿠키를 persistent storage에 반영해야 하는 경우 다음과 같이 처리할 수 있습니다.

```kotlin
override fun doUpdateVisitedHistory(
    view: WebView,
    url: String?,
    isReload: Boolean
) {
    super.doUpdateVisitedHistory(view, url, isReload)

    CookieManager.getInstance().flush()
}
```

기존에 다음과 같이 `onPageFinished()`에서 쿠키를 저장하고 있었다고 가정합니다.

```kotlin
override fun onPageFinished(
    view: WebView,
    url: String
) {
    super.onPageFinished(view, url)

    CookieManager.getInstance().flush()
}
```

일반적인 페이지 이동에서는 `onPageFinished()`가 호출되기 때문에 문제가 없습니다.

하지만 SPA 내부 페이지 이동에서는 새로운 document를 로드하지 않으므로 `onPageFinished()`가 호출되지 않을 수 있습니다.

그 결과:

```kotlin
CookieManager.getInstance().flush()
```

도 실행되지 않습니다.

이 로직을 `doUpdateVisitedHistory()`로 옮기면 SPA 내부에서 URL 또는 History가 변경되는 시점에도 쿠키 저장 로직을 실행할 수 있습니다.

```text
SPA 내부 페이지 이동
        ↓
URL / History 변경
        ↓
doUpdateVisitedHistory()
        ↓
CookieManager.flush()
```

따라서 SPA 기반 WebView에서 페이지 이동 시점에 필요한 후처리가 있다면 `doUpdateVisitedHistory()`를 활용할 수 있습니다.

---

## 정리

Android WebView에서 SPA 기반 웹페이지를 사용하는 경우, SPA 내부 페이지 이동은 일반적인 웹페이지 navigation과 다르게 동작합니다.

SPA에서는 대부분 새로운 HTML document를 다시 로드하지 않고 JavaScript Router를 통해 기존 document의 URL과 화면 상태만 변경합니다.

따라서 Android WebView에서는:

```kotlin
shouldOverrideUrlLoading()
```

이 WebView navigation 자체가 발생하지 않아 호출되지 않을 수 있습니다.

또한:

```kotlin
onPageFinished()
```

도 새로운 document 로딩이 없기 때문에 호출되지 않을 수 있습니다.

반면:

```kotlin
doUpdateVisitedHistory()
```

는 WebView의 URL 또는 History 변경을 감지할 수 있으므로 SPA 내부 페이지 이동에서도 호출될 수 있습니다.

Android 개발자 관점에서 각 콜백의 역할을 정리하면 다음과 같습니다.

```text
shouldOverrideUrlLoading()
    → WebView navigation 처리 여부 확인

onPageFinished()
    → WebView document 로딩 완료 확인

doUpdateVisitedHistory()
    → WebView URL / History 변경 확인
```

따라서 Android 앱에서 SPA 내부의 URL 변경이나 페이지 이동을 감지해야 하는 경우 `shouldOverrideUrlLoading()`이나 `onPageFinished()`만 사용하는 것보다:

```kotlin
doUpdateVisitedHistory()
```

를 활용하는 방식이 더 적합할 수 있습니다.

특히 SPA 페이지 이동 시 URL 변경에 따른 후처리, 쿠키 저장, 상태 동기화 등의 작업이 필요하다면 `doUpdateVisitedHistory()`를 대체 콜백으로 사용할 수 있습니다.
