# Scroll on View B

`ScrollView.onInterceptTouchEvent()` возвращает `true`, дочерние вью перестают оповещаться о событиях

`ScrollView.onTouchEvent()` возвращает `true`, соответствующий метод парента не вызывается - событие "останавливается"

![img.png](https://miro.medium.com/max/1400/1*2wzhkKPgfQZMqPJs73DUcA.png)
