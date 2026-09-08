# Compose: VectorDrawable 이미지 크기와 터치 영역 확장

## 이미지 크기와 터치 영역을 분리하고 싶을 때

터치 영역(클릭 가능 범위)은 크게, 실제 보이는 이미지는 작게 유지하고 싶은 경우(접근성/탭 편의성 목적으로 흔한 패턴) 세 가지 방법이 있다.

### 방법 1: Box로 감싸기

```kotlin
Box(
    modifier = Modifier.size(58.dp),
    contentAlignment = Alignment.Center
) {
    Image(
        painter = painterResource(id = R.drawable.ic_closed),
        modifier = Modifier.size(24.dp),
        contentDescription = "닫기 버튼",
    )
}
```

`IconButton` 내부도 이 방식을 쓴다.

### 방법 2: ContentScale.None

```kotlin
Image(
    painter = painterResource(id = R.drawable.ic_closed),
    modifier = Modifier.size(58.dp),
    contentScale = ContentScale.None,
    contentDescription = "닫기 버튼",
)
```

painter의 intrinsic 크기 그대로(스케일 없이) 박스 중앙에 그린다.  
단 XML의 `android:width/height`가 실제로 원하는 크기(예: 24dp)일 때만 유효하다  
— intrinsic 크기에 의존하므로 원하는 표시 크기가 XML 크기와 다르면 쓸 수 없다.

### 방법 3: Box 없이 Modifier 체이닝 (XML 크기와 무관하게 임의 크기 지정 가능)

```kotlin
Image(
    painter = painterResource(id = R.drawable.ic_closed),
    modifier = Modifier
        .size(width = 56.dp, height = 58.dp)   // 터치 영역
        .clickable {
            Timber.i("이미지 클릭")
        }
        .wrapContentSize()                     
        .size(24.dp),                          // 실제 표시 이미지 크기
    contentDescription = "닫기 버튼",
    contentScale = ContentScale.None
)
```

**동작 원리**

1. `.size(width, height)`가 최종 레이아웃 크기(=클릭 히트 영역)를 고정한다.`clickable`이 이보다 앞(바깥)에 있으므로 터치 반응 영역도 이 크기 전체가 된다.
2. `.wrapContentSize()` — 시그니처는 `wrapContentSize(align: Alignment = Alignment.Center, unbounded: Boolean = false)`. 기본값이 이미 `Alignment.Center`이므로 명시할 필요 없음. `unbounded`도 **안쪽 크기가 바깥 크기보다 작을 때는** 기본값(`false`)으로 충분 — 어차피 clamp될 일이 없기 때문. (안쪽 크기가 바깥보다 커지는 경우에만 `unbounded = true`가 필요: 이땐 `false`면 부모 크기로 잘리고, `true`면 부모 영역을 넘어서 그대로 그려진다.)
3. `.size(24.dp)` — Image의 실제 측정 크기를 강제 지정. XML의 intrinsic 크기와 무관하게 원하는 임의 크기로 표시 가능.

→ Box 없이 Image 하나의 modifier 체인만으로 "터치 영역은 크게, 표시 이미지는 XML 크기와 무관한 임의 크기로" 구현 가능

**참고**: 접근성 최소 터치 영역(48dp) 기준만 맞추면 되는 단순한 경우엔 Material의 `Modifier.minimumInteractiveComponentSize()`도 있지만 48dp 고정입니다.
