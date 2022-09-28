# Слушатели

1. Зачем? Ну не каждый же раз создавать кастомную вью.
2. Слушатели `обычно` вызываются до соответствующих им функций. Вернул `true` => функция не вызвалась

Кусок исходников `View.dispatchTouchEvent()`

```kotlin
if (li.mOnTouchListener != null 
  && li.mOnTouchListener.onTouch(this, event)) {
    result = true;
}
if (!result && onTouchEvent(event)) {
  result = true;
}
```

## OnTouchListener

Соответствует `View.onTouchEvent()`

Максимально общий слушатель. Получает вью и событие.

```kotlin
myView.setOnTouchListener { view, event ->
  Log.d(TAG, "$event")
  true
}
```

## OnClickListener

Соответствует `View.performClick()`

Если нужна обработка только кликов. Не принимает параметров и ничего не возвращает => проще.

Вместо `performClick()` можно вызвать только слушателей через `callOnClick()` (не будет проделано доп действий, например acessability event)

Если установлены и `OnTouchListener` и `OnClickListener`, первым вызовется тач. Вернет `true` - клик не вызовется

## OnGenericMotionListener

Соответствует `View.onGenericMotionEvent()`

События включат в себя:

* Нажатия джойстика
* Хаверы курсора мыши
* Прикосновения к трекпаду
* Прокрутка колеса мыши
* И др.

Прикосновения в generic-события не включаются. Приверы экшнов: `ACTION_HOVER_MOVE`, `ACTION_HOVER_ENTER`, `ACTION_HOVER_EXIT`, `ACTION_SCROLL`

Кусок исходников View

```kotlin
public final boolean dispatchPointerEvent(MotionEvent event) {
  if (event.isTouchEvent()) {
    return dispatchTouchEvent(event);
  } else {
    return dispatchGenericMotionEvent(event);
  }
}
```

## OnContextClickListener

Соответствует `View.performContextClick()`, который вызывается в `dispatchGenericMotionEvent()`

Вызывается такими событиями как использование стилуса или нажатие правой кнопки мыши

## GestureDetector

Все слушатели выше объявлены в классе View, а он - сам по себе.

В конструкторе принимает `OnGestureListener`, в котором:

- onDown() // вызывается в начале каждого нажатия, лучше переопределить и вернуть true чтоб показать заинтересованность в обработке жеста
- onShowPress() // юзер нажал, но еще не убрал палец. Отличное место чтобы показать визульный фидбек нажатию
- onSingleTapUp() // юзер убрал палец и это было короткое нажатие
- onScroll()
- onLongPress()
- onFling()

Также есть `SimpleOnGestureListener`. Имплементит `OnDoubleTapListener`  и `OnContextClickListener`, так что в нем дополнительно:

- onSingleTapConfirmed()
- onDoubleTap()
- onDoubleTapEvent()
- onContextClick()

```kotlin
val gestureDetector = GestureDetector(
  context, 
  object : GestureDetector.SimpleOnGestureListener() {
    
    override fun onDoubleTap() {
      Log.d(TAG, "double tap")
      return true
    }
  }
)
// Можно использовать в setOnTouchListener()
setOnTouchListener { _, event ->
  gestureDetector.onTouchEvent(event)
}
// Или в onTouchEvent() в кастомной view
override fun onTouchEvent() {
  if (gestureDetector.onTouchEvent()) {
    return true
  }
  return super.onTouchEvent()
}
```

## ScaleGestureDetector

Похож на `GestureDetector`. Берет слушатель `OnScaleGestureListener` в котором:

- onScale()
- onScaleBegin()
- onScaleEnd()

```kotlin
val scaleGestureDetector = ScaleGestureDetector(
  context,
  object : ScaleGestureDetector.SimpleOnScaleGestureListener() {
    
    override fun onScale(detector: ScaleGestureDetector): Boolean {
      Log.d(TAG, "scale ${detector.scaleFactor}")
      return true
    }
  }
)

setOnTouchListener { _, event ->
  scaleGestureDetector.onTouchEvent(event)
}
```