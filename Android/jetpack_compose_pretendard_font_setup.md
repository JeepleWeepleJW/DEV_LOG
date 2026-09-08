# Jetpack Compose 앱에 Pretendard 폰트 기본 적용하기

Jetpack Compose + Material Design 3 기반 안드로이드 앱에 [Pretendard](https://github.com/orioncactus/pretendard) 폰트를 "기본 폰트"로 적용한 과정을 정리한다.

## 요약

1. Pretendard의 **굵기별 static OTF 파일**을 `res/font/`에 추가한다.
2. Kotlin `FontFamily`로 굵기별 파일을 `FontWeight`에 매핑한다.
3. Material3 `Typography()`의 **13개 스타일 슬롯 전부**에 해당 `FontFamily`를 지정한다.
4. `MaterialTheme(typography = ...)`에 연결한다.

## 1. Variable Font 대신 static 파일을 고른 이유

Pretendard는 하나의 파일로 여러 굵기를 표현하는 **variable font**도 배포한다. 처음엔 이 방식으로 시도했다.

```kotlin
Font(
    R.font.pretendard_variable,
    FontWeight.Bold,
    variationSettings = FontVariation.Settings(FontVariation.weight(700))
)
```

문제는 `FontVariation.Settings`(variable font의 굵기 축을 지정하는 API)가 **Android API 26(Oreo) 이상에서만 동작**한다.  
`minSdk`가 24~25를 포함한다면, 구형 기기에서는 축 지정이 무시되고 폰트의 기본 인스턴스로만 렌더링되어 `Normal`/`Medium`/`Bold`가 전부 같은 굵기로 보이게 된다.  
그래서 API 버전에 상관없이 동일하게 동작하도록 **굵기별 static OTF 파일**(Regular / Medium / Bold)을 쓰는 방식으로 정리했다.

## 2. 폰트 파일 추가

```
app/src/main/res/font/
├── pretendard_regular.otf
├── pretendard_medium.otf
└── pretendard_bold.otf
```

## 3. FontFamily 정의

각 굵기 파일을 대응하는 `FontWeight`에 매핑한다.

```kotlin
val PretendardFontFamily: FontFamily = FontFamily(
    Font(R.font.pretendard_regular, FontWeight.Normal),
    Font(R.font.pretendard_medium, FontWeight.Medium),
    Font(R.font.pretendard_bold, FontWeight.Bold),
)
```

## 4. Typography 13개 슬롯 전부 채우기

Material3의 `Typography()`는 `displayLarge`부터 `labelSmall`까지 총 13개 스타일 슬롯을 갖는다.  
이 중 일부만 지정하면 지정하지 않은 슬롯은 Material3 기본값(시스템 폰트)으로 남는다.

`Button`, `TopAppBar`, `Chip` 같은 Material3 컴포넌트는 각기 다른 슬롯(`labelLarge`, `titleMedium` 등)을 내부적으로 사용하기 때문에, 앱 전체에서 일관되게 커스텀 폰트가 보이려면 **13개 전부** 채워야 한다.

| 스타일 | FontWeight |
|---|---|
| `displayLarge/Medium/Small` | Normal |
| `headlineLarge/Medium/Small` | Normal |
| `titleLarge` | Normal |
| `titleMedium/Small` | Medium |
| `bodyLarge/Medium/Small` | Normal |
| `labelLarge/Medium/Small` | Medium |

```kotlin
val Typography = Typography(
    displayLarge = TextStyle(
        fontFamily = PretendardFontFamily,
        fontWeight = FontWeight.Normal,
        fontSize = 57.sp,
        lineHeight = 64.sp,
        letterSpacing = (-0.25).sp
    ),
    // displayMedium/Small, headlineLarge/Medium/Small,
    // titleLarge/Medium/Small, bodyLarge/Medium/Small,
    // labelLarge/Medium/Small 도 동일한 패턴으로 정의
    titleMedium = TextStyle(
        fontFamily = PretendardFontFamily,
        fontWeight = FontWeight.Medium,
        fontSize = 16.sp,
        lineHeight = 24.sp,
        letterSpacing = 0.15.sp
    ),
    // ...
)
```

## 5. 테마에 연결

마지막으로 `Typography`를 `MaterialTheme`에 전달한다.

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {
    val colorScheme = /* ... */

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```
