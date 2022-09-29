# События в иерархии Composable

## MotionEvent -> Modifier

И `ComponentActivity.setContent()` и `ComposeView.setContent()` под капотом используют `AndroidComposeView`.

`AndroidComposeView.dispatchTouchEvent()` переопределяет соответствующий метод `View` - здесь и происходит конверсия `MotionEvent` -> `PointerInputEvent`

Т.о. путь события выглядит так:

`Activity.dispatchTouchEvent()` -> `SomeViewGroup.dispatchTouchEvent()` -> `ComposeView.dispatchTouchEvent()` -> `AndroidComposeView.dispatchTouchEvent()` -> `Мир компоуза ✨`

После на стороне Compose:

- `PointerInputEvent` передается параметром в `PointerInputEventProcessor.process()`
- Вычисляется разница между текущим и предыдущим событием, чтобы вычислить его тип (н/р down, move)
- "hit-test" начиная от рутовой ноды до дочерних

## Проход по иерархии

Более интуитивно, чем в классике: три прохода разного типа из энама `PointerEventPass`

- `PointerEventPass.Initial` - вниз по дереву от родителей к дочерним, позволяя родителям обработать какие-либо аспекты до детей. Пример: "scroller" может заблочить кнопки внутри себя от нажатия при начале скролла
- `PointerEventPass.Main` - вверх от детей к родителям. Основной проход, большинство обработок жестов делается здесь. Здесь кнопка может "отреагировать" на нажатие до того, как это сделает ее контейнер
- `PointerEventPass.Final` - снова от родителей к детям, чтобы те "узнали" какие аспекты события были "поглощены" во время `Main pass`. Например здесь кнопка понимает, что ей больше не нужно реагировать на то, что юзер убрал палец от экрана, т.к. родительский scroller уже законсьюмил движен 

Для всех обработок событий, назначенных в `Modifier`, обработка идет в `PointerEventPass.Main`

У Composable стоит соответствующий `Modifier`? Она консьюмит событие. Нет? Тогда оно идет вверх по дереву в поисках того, кто его законсьюмит

Если две Composable накладываются друг на друга, то выигрывает та, которая была добавлена позже и, соответственно, была отрисована "выше"

```kotlin
Box(modifier = Modifier.align(Alignment.Center) {
    Box(
        modifier = Modifier
            .size(200.dp)
            .background(Color.LightGray)
            .clickable { Log.d(TAG, "Light gray box clicked") }
    )
    Box(
        modifier = Modifier
            .size(100.dp)
            .background(Color.DarkGray)
            .clickable { Log.d(TAG, "Dark gray box clicked") }
    )
})
```

Если будет клик в месте пересечения боксов - dark gray побеждает

Это дополнительно можно контролировать с помощью `Modifier.zIndex()` - 
дочерний элемент у которого zIndex выше будет отрисован позже, и доступ к жесту будет иметь раньше

## Ограничения

Сейчас нет способа детектить жесты без их "потребления" из коробки. 
То есть просто вставить логирование - уже проблема. Однако есть воркэраунды со StackOverFlow, по факту повторяющие реализации текущих детектов, но с некоторыми отличиями (можно посмотреть больше [здесь](https://proandroiddev.com/android-touch-system-part-5-how-gestures-work-in-jetpack-compose-ef7e74703b6a))

```kotlin
suspend fun PointerInputScope.detectTapUnconsumed(
    onTap: ((Offset) -> Unit)
) {
    val pressScope = PressGestureScopeImpl(this)
    forEachGesture {
        coroutineScope {
            pressScope.reset()
            awaitPointerEventScope {
                awaitFirstDown(requireUnconsumed = false).also {
                    it.consumeDownChange() 
                }
                val up = waitForUpOrCancellationInitial()
                if (up == null) {
                    pressScope.cancel()
                } else {
                    pressScope.release()
                    onTap(up.position)
                }
            }
        }
    }
}
suspend fun AwaitPointerEventScope.waitForUpOrCancellationInitial(): PointerInputChange? {
    while (true) {
        val event = awaitPointerEvent(PointerEventPass.Initial)
        if (event.changes.fastAll { it.changedToUp() }) {
            return event.changes[0]
        }
        if (event.changes.fastAny { 
            it.consumed.downChange || 
                it.isOutOfBounds(size,extendedTouchPadding)
            }
        ) {
            return null
        }
        // Check for cancel by position consumption. 
        // We can look on the Final pass of the existing 
        // pointer event because it comes after the Main 
        // pass we checked above.
        val consumeCheck = awaitPointerEvent(PointerEventPass.Final)
        if (consumeCheck.changes.fastAny { 
                it.positionChangeConsumed() 
            }
        ) {
            return null
        }
    }
}
```