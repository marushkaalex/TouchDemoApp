# Скролл на View B. View B переопределяет dispatchTouchEvent() и вызывает parent.disallowTouchEvent(true). View B.onTouchEvent() возвращает true. Скролла не происходит

Метод `onInterceptTouchEvent()` всех родителей View B игнорируется, событие останавливается на `ViewB.onTouchEvent()`. ScrollView не скроллится

![img.png](https://miro.medium.com/max/1400/1*axiXXASOU8mdRRnSqPfG4w.png)
