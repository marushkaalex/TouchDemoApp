# Скролл на View B. View B переопределяет dispatchTouchEvent() и вызывает parent.disallowTouchEvent(true). View B.onTouchEvent() возвращает false. ScrollView скроллится

`onInterceptTouchEvent()` всех родителей View B игнорируются, но так как ViewB.onTouchEvent() не обрабатывают события, оно прокидывается до родительских onTouchEvent(). ScrollView.onTouchEvent() возвращает true, говоря, что событие обработано. Происходит скролл и событие останавливается

![img.png](https://miro.medium.com/max/1400/1*pXZ3DglS_xbH8tnwxQMr4Q.png)
