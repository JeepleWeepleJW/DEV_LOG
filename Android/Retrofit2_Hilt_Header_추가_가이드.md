# Retrofit2 + Hilt Header 추가 가이드

## 1. 목적

Retrofit2와 Hilt를 사용하는 Android 프로젝트에서 모든 HTTP 요청에 공통 Header를 추가하는 방법을 정리한다.

이 문서에서는 `User-Agent` Header를 예시로 사용한다.

구성 요소는 다음과 같다.

- Retrofit2
- OkHttp
- OkHttp Interceptor
- Hilt
- Kotlinx Serialization Converter

---

## 2. 전체 구조

공통 Header는 Retrofit API 인터페이스에서 직접 추가하지 않고, OkHttp의 `Interceptor`를 사용해 모든 요청에 자동으로 추가한다.

```text
Retrofit
   ↓
OkHttpClient
   ↓
UserAgentInterceptor
   ↓
HttpLoggingInterceptor
   ↓
Server
```

역할은 다음과 같이 분리한다.

| 구성 요소 | 역할 |
| --- | --- |
| `UserAgentInterceptor` | 모든 HTTP 요청에 `User-Agent` Header 추가 |
| `HttpLoggingInterceptor` | HTTP 요청/응답 로그 출력 |
| `OkHttpClient` | Interceptor 조합 및 HTTP 통신 수행 |
| `Retrofit` | API 인터페이스를 HTTP 요청으로 변환 |
| `NetworkModule` | Hilt를 사용해 네트워크 객체 생성 및 의존성 주입 |

---

## 3. UserAgentInterceptor 구현

`Interceptor`를 구현하여 모든 HTTP 요청에 공통 `User-Agent` Header를 추가한다.

```kotlin
/**
 * HTTP 요청에 User-Agent Header를 추가하는 Interceptor
 */
class UserAgentInterceptor @Inject constructor() : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val userAgent =
            "JeepleWeeple/${BuildConfig.VERSION_NAME} " +
                "(android; build/${BuildConfig.VERSION_CODE})"

        val request = chain.request()
            .newBuilder()
            .header(
                name = "User-Agent",
                value = userAgent,
            )
            .build()

        return chain.proceed(request)
    }
}
```

### 핵심 포인트

`UserAgentInterceptor`는 생성자에 `@Inject`를 사용한다.

```kotlin
class UserAgentInterceptor @Inject constructor() : Interceptor
```

따라서 Hilt가 `UserAgentInterceptor`를 직접 생성할 수 있으며, 별도의 `@Provides` 함수가 필요하지 않다.

또한 Header 추가 시 다음과 같이 `.header()`를 사용한다.

```kotlin
.header("User-Agent", userAgent)
```

`.header()`는 동일한 이름의 기존 Header가 존재하면 해당 값을 교체한다.

`User-Agent`처럼 하나의 값만 유지해야 하는 Header에 적합하다.

---

## 4. Hilt NetworkModule 구성

`UserAgentInterceptor`를 `OkHttpClient`에 등록한다.

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides
    @Singleton
    fun provideJson(): Json = Json {
        ignoreUnknownKeys = true
        explicitNulls = false
        coerceInputValues = true
        encodeDefaults = true
    }

    @Provides
    @Singleton
    fun provideHttpLoggingInterceptor(): HttpLoggingInterceptor =
        HttpLoggingInterceptor { message ->
            if (BuildConfig.DEBUG) {
                Timber.tag("JeepleWeeple").i(message)
            }
        }.apply {
            level = if (BuildConfig.DEBUG) {
                HttpLoggingInterceptor.Level.BODY
            } else {
                HttpLoggingInterceptor.Level.NONE
            }
        }

    @Provides
    @Singleton
    fun provideOkHttpClient(
        userAgentInterceptor: UserAgentInterceptor,
        httpLoggingInterceptor: HttpLoggingInterceptor,
    ): OkHttpClient =
        OkHttpClient.Builder()
            .addInterceptor(userAgentInterceptor)
            .addInterceptor(httpLoggingInterceptor)
            .build()

    @Provides
    @Singleton
    fun provideRetrofitBuilder(
        json: Json,
        okHttpClient: OkHttpClient,
    ): Retrofit.Builder =
        Retrofit.Builder()
            .client(okHttpClient)
            .addConverterFactory(
                json.asConverterFactory(
                    "application/json".toMediaType()
                )
            )

    @Provides
    @Singleton
    fun provideJeepleWeepleNetwork(
        retrofitBuilder: Retrofit.Builder,
    ): JeepleWeepleNetwork =
        JeepleWeepleNetworkImpl(retrofitBuilder)
}
```

---

## 5. Hilt 의존성 주입 흐름

`UserAgentInterceptor`에는 `@Inject constructor()`가 선언되어 있다.

```kotlin
class UserAgentInterceptor @Inject constructor() : Interceptor
```

따라서 다음과 같이 `provideOkHttpClient()`의 파라미터로 선언하면 Hilt가 자동으로 객체를 생성하여 전달한다.

```kotlin
fun provideOkHttpClient(
    userAgentInterceptor: UserAgentInterceptor,
    httpLoggingInterceptor: HttpLoggingInterceptor,
): OkHttpClient
```

의존성 생성 흐름은 다음과 같다.

```text
Hilt
 ├─ UserAgentInterceptor
 ├─ HttpLoggingInterceptor
 │
 └─ OkHttpClient
       ↓
    Retrofit.Builder
       ↓
    JeepleWeepleNetwork
```

---

## 6. Interceptor 등록 순서

권장 순서는 다음과 같다.

```kotlin
OkHttpClient.Builder()
    .addInterceptor(userAgentInterceptor)
    .addInterceptor(httpLoggingInterceptor)
    .build()
```

`UserAgentInterceptor`를 먼저 등록하면 Header가 추가된 요청을 `HttpLoggingInterceptor`가 로깅할 수 있다.

즉 로그에서 다음과 같이 실제 전송될 `User-Agent` 값을 확인하기 쉽다.

```text
User-Agent: JeepleWeeple/1.0.0 (android; build/100)
```

---

## 7. 다른 공통 Header 추가

동일한 방식으로 `Authorization`, 앱 버전, 플랫폼 등의 공통 Header도 추가할 수 있다.

예시:

```kotlin
val request = chain.request()
    .newBuilder()
    .header("User-Agent", userAgent)
    .header("X-App-Version", BuildConfig.VERSION_NAME)
    .header("X-Platform", "android")
    .build()
```

서버에는 다음과 같은 Header가 전달된다.

```http
User-Agent: JeepleWeeple/1.0.0 (android; build/100)
X-App-Version: 1.0.0
X-Platform: android
```

---

## 8. `.header()`와 `.addHeader()` 차이

### `.header()`

기존에 동일한 Header가 있으면 값을 교체한다.

```kotlin
.header("User-Agent", userAgent)
```

다음과 같은 단일 값 Header에 적합하다.

- `User-Agent`
- `Authorization`
- `Content-Type`
- `X-App-Version`

### `.addHeader()`

기존 값을 유지하면서 같은 이름의 Header를 추가한다.

```kotlin
.addHeader("Accept", "application/json")
```

동일한 Header 이름으로 여러 값을 보내야 하는 경우에 사용한다.

공통 앱 Header를 설정할 때는 대부분 `.header()`를 사용한다.

---

## 9. 최종 구조

권장 프로젝트 구조는 다음과 같다.

```text
core/
└── network/
    ├── UserAgentInterceptor.kt
    ├── JeepleWeepleNetwork.kt
    ├── JeepleWeepleNetworkImpl.kt
    └── di/
        └── NetworkModule.kt
```

각 클래스의 책임은 다음과 같다.

```text
UserAgentInterceptor
    └─ HTTP Header 생성 및 추가

NetworkModule
    ├─ Json 생성
    ├─ HttpLoggingInterceptor 생성
    ├─ OkHttpClient 생성
    ├─ Retrofit.Builder 생성
    └─ MykoallaNetwork 생성
```

이 구조를 사용하면 API마다 Header를 반복해서 선언하지 않고 OkHttp 레벨에서 공통 Header를 일괄 관리할 수 있다.

---

## 10. 요약

Retrofit2 + Hilt 환경에서 공통 Header를 추가할 때는 OkHttp `Interceptor`를 사용하는 것이 적합하다.

핵심 구성은 다음과 같다.

```kotlin
class UserAgentInterceptor @Inject constructor() : Interceptor
```

```kotlin
OkHttpClient.Builder()
    .addInterceptor(userAgentInterceptor)
    .addInterceptor(httpLoggingInterceptor)
    .build()
```

이 방식의 장점은 다음과 같다.

- 모든 API 요청에 Header를 자동 적용할 수 있다.
- Retrofit API 인터페이스에 Header 관련 코드가 반복되지 않는다.
- Header 생성 책임을 별도의 Interceptor로 분리할 수 있다.
- Hilt의 Constructor Injection을 그대로 활용할 수 있다.
- LoggingInterceptor를 통해 실제 요청 Header를 쉽게 확인할 수 있다.
