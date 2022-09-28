# Ура, Compose!

Детект жестов - через `Modifier`, но есть небольшое число Composable, принимающих в себя лямбду-слушателя, например `Button` с его `onClick: () -> Unit`

## Modifier.pointerInput

Гибкий, низкоуровневый, аналог `OnTouchListener`. Передаваемая лямбда запускатся в `PointerInputScope` из которого можно вытащить полезную инфу (размер указателя, таймауты для быстрого/долгого нажатия и т.д) и детектить разнообразные события ввода (через методы, которые начинаются с `detekt`)

### Что интересно (1)

Реализация:

```kotlin
fun Modifier.pointerInput(
    key1: Any?,
    block: suspend PointerInputScope.() -> Unit
): Modifier 
```

Где key1 - объект, при изменении которого при рекомпоузе скоуп обработки нажатий будет перезапущен: текущий отменится, запустится новый

### Что интересно (2)

Функции для детекта жестов - suspend, так что их из одного `pointerInput` вызывать подряд нельзя, т.к. вторая залочится первой. Но можно применять несколько `pointerInput`, они будут работать параллельно

```kotlin
modifier = Modifier
    .pointerInput(Unit) {
        detectTapGestures { // suspend
            println("kek $it")
        }
    }
    .pointerInput(Unit) {
        detectVerticalDragGestures { change, dragAmount -> // suspend
            println("kekek $dragAmount")
        }
    }
```

## Modifier.clickable

Слушает одиночные клики, эквивалент `OnClickListener`. Под капотом работает через
`Modifier.pointerInput { detectTapAndPress() }` (internal-функция с понятным функционалом), добавляя визуальные эффекты и accessibility-штуки. Краткий и понятный

```kotlin
Box(modifier = Modifier.clickable {
    Log.d(TAG, "Box clicked")
})
```

## Modifier.combinedClickable

Старший брат `clickable` и ближаший эквивалент `GestureDetector` из мира классики - может меньше, зато проще

```kotlin
Box(modifier = Modifier.combinedClickable(
    onClick = { Log.d(TAG, "Box clicked") },
    onDoubleClick = { Log.d(TAG, "Box double clicked")},
    onLongClick = { Log.d(TAG, "Box long clicked")}
))
```

### А если братьев заюзать вместе?

Победит последний добавленный в цепочку `Modifier`

```kotlin
Box(modifier = Modifier
    .combinedClickable(
        onClick = { Log.d(TAG, "Click in combinedClickable") },
        onDoubleClick = { 
            Log.d(TAG, "Double click in combinedClickable")
        },
        onLongClick = { 
            Log.d(TAG, "Long click in combinedClickable")
        }
    )
    .clickable { Log.d(TAG, "Click in clickable") } 
    // ^ победит вне зависимости от того, долгий или короткий клик
)
```

> **Однако**
> 
> Если у Composable явный лямбда-параметр для обработки клика (как у `Button`), победит он 

т.к. если смотреть реализацию он добавляется под капотом в самом конце

## Modifier.draggable

Нет аналогов в классике - там традиционно все с этим сложно, сиди сам считай всё и менеджи в `OnTouchListener`.

```kotlin
val state = rememberDraggableState(
    onDelta = { delta -> Log.d(TAG, "Dragged $delta") }
)
Box(modifier = Modifier.draggable(
    state = state,
    orientation = Orientation.Vertical,
    onDragStarted = { Log.d(TAG, "Drag started") },
    onDragStopped = { Log.d(TAG, "Drag ended") }
))
```

Хочешь больше власти и драга в двух направлениях?

```kotlin
Modifier.pointerInput(Unit) {
    detectDragGestures { change, dragAmount ->
        Log.d(TAG, "dragged x: ${dragAmount.x}")
        Log.d(TAG, "dragged y: ${dragAmount.y}")
    }
}
```

## Modifier.scrollable

Двойник `GestureDetector.SimpleOnGestureListener.onScroll()`. 
Работает (и на самом деле вызывает под капотом) как `draggable()`, но не только детектит, 
но и умеет отрисовывать скролл на основе `consumeScrollDelta`

```kotlin
val scrollableState = rememberScrollableState(
    consumeScrollDelta = { delta -> 
        Log.d(TAG, "scrolled $delta")
        0f
    }
)
Box(modifier = Modifier.scrollable(
    state = scrollableState,
    orientation = Orientation.Vertical
))
```

Composable будет сдвинута на разницу `delta` и значения, что мы вернем из `consumeScrollDelta`.

## Modifier.verticalScroll и Modifier.horizontalScroll

Под капотом дергают `scrollable`, но проще, т.к. не нужно возиться с дельтой. 
Могут быть применены одновременно и не будут друг другу мешаться

```kotlin
Box(modifier = Modifier
    .verticalScroll(rememberScrollState())
    .horizontalScroll(rememberScrollState())
)
```

## Modifier.swipeable

Тоже детектит жест "drag", но умеет "допереключать" до одного из состояний, когда пользователь отпускает палец

![](https://miro.medium.com/max/664/1*8XGKE3C48j6A0GPQhR11Xw.gif)

```kotlin
@Composable
fun SwipeableDemo() {
    val width = 300.dp
    val squareSize = 100.dp
    val swipeableState = rememberSwipeableState(States.LEFT)
    val squareSizePx = with(LocalDensity.current) { 
        (width - squareSize).toPx()
    }
    Box(
        modifier = Modifier
            .width(width)
            .swipeable(
                state = swipeableState,
                anchors = mapOf(
                    0f to States.LEFT, 
                    squareSizePx to States.RIGHT
                ),
                thresholds = { _, _ -> FractionalThreshold(0.5f) },
                orientation = Orientation.Horizontal
            )
            .background(Color.LightGray)
    ) {
        Box(
            modifier = Modifier
                .offset { 
                    IntOffset(swipeableState.offset.value.toInt(), 0) 
            }
            .size(squareSize)
            .background(Color.DarkGray)
        )
    }
}
enum class States { LEFT, RIGHT }
```

## Modifier.transformable

Похож на `ScaleGestureDetector` и предоставляет инфу по увеличению (scale), повороту (rotation) и отступу (offset),
но не отвечает за графику - это на разработчике и использовании `rememberTransformableState {}`

![](https://miro.medium.com/max/690/1*oRRFslgtPnWD85tYo7ypBA.gif)

```kotlin
@Composable
fun TransformableDemo() {
    var scale by remember { mutableStateOf(1f) }
    var rotation by remember { mutableStateOf(0f) }
    var offset by remember { mutableStateOf(Offset.Zero) }
    val state = rememberTransformableState { zoomChange, offsetChange, rotationChange ->
        scale *= zoomChange
        rotation += rotationChange
        offset += offsetChange
    }

    Box(
        modifier = Modifier
            .graphicsLayer(
                scaleX = scale,
                scaleY = scale,
                rotationZ = rotation,
                translationX = offset.x,
                translationY = offset.y
            )
            .transformable(state = state)
            .background(Color.Blue)
            .fillMaxSize()
    )
}
```

## Modifier.pointerInteropFilter

Для интеропа между Compose и классикой - чтобы не переписывать уже имеющеюся кастомную обработку тачей

```kotlin
class DemoOnTouchListener : View.OnTouchListener {
    override fun onTouch(v: View, event: MotionEvent): Boolean {
        Log.d(TAG, "Motion event ${event.action}")
        return false
    }
}
// В Composable
val listener = DemoOnTouchListener()
val view = LocalView.current
Box(
    modifier = Modifier
        .pointerInteropFilter { motionEvent ->
            listener.onTouch(view, motionEvent)
        }
)
```