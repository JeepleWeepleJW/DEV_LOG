# Jetpack Compose에서 디바이스 폰트 스케일에 영향받지 않는 고정 크기 텍스트 만들기

## 정리
Jetpack Compose에서 텍스트 크기를 지정할 때는 보통 `sp` 단위를 쓴다.  
`sp`는 사용자가 시스템 설정에서 글자 크기(폰트 스케일)를 키우거나 줄이면 그 비율만큼 같이 커지고 작아진다.  
저시력 사용자를 위한 접근성 기능이기 때문에 일반적인 텍스트라면 이게 맞는 동작이다.

다만 내부 정책 혹은 텍스트 크기를 유지해야할경우가 있다.
이때 사용하기위한 dpToSp 함수를 작성했다.

## 코드

```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.TextUnit

@Composable
fun dpToSp(dp: Dp): TextUnit = with(LocalDensity.current) { dp.toSp() }
```

사용 예시:

```kotlin
Text(
    text = "12",
    fontSize = dpToSp(10.dp) // 아이콘 크기에 맞춘 고정 배지 숫자
)
```

## 동작 원리

Compose의 `Density.toSp()`는 내부적으로 다음과 같이 계산된다.

```
sp 값 = dp.value / fontScale
```

렌더링 시점에 이 `sp` 값에는 다시 `fontScale`이 곱해지므로, 최종적으로 화면에 그려지는 크기는

```
dp.value / fontScale * fontScale = dp.value
```

가 되어 `fontScale`이 얼마든 상관없이 항상 원래 `dp` 값과 같은 시각적 크기가 나온다.
