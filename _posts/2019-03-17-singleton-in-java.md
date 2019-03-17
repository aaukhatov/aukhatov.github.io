---
layout: post
theme: jekyll-theme-cayman
categories: java patterns
title: Singleton Pattern in Java
---
Существует несколько способов создания singletons в Java. Рассмотрим несколько из них, разделив на две группы.
> Шаблон **Singleton** используется для ограничения создания экземпляра класса и **гарантирует** что в JVM **существует только один экземпляр** этого класса.

## Ленивые
Ленивая инициализация - означает, что экземпляр создается по требованию.

### С двойной проверкой (Lazy initialization with double check)
Данный подход в Интеренет ресурсах является самым популярным, только я не знаю почему.
Основная проблема заключается когда два потока входят в первое условие _if_ а экземпляр еще не создан.
Далее блокировка на классе и затем снова проверка экземпляра на _null_, чтобы убедиться окончательно.

```java
public class Singleton {

    private static volatile Singleton INSTANCE = null;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (INSTANCE == null) {
            synchronized (Singleton.class) {
                if (INSTANCE == null) {
                    INSTANCE = new Singleton();
                }
            }
        }
        return INSTANCE;
    }
}
```
### Через внутренний класс (Lazy initialization by using inner class)
Реализация через внутренний класс основана на гарантиях [спецификации языка Java (JLS)](https://docs.oracle.com/javase/specs/jls/se8/html/jls-12.html#jls-12.4.2).
```java
public class Singleton {
    
    private Singleton() {}
    
    private static class SingletonHolder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}
```

## Энергичные
Энергичная инициализация - означает, что экземпляр создается сразу же (обычно это при загрузке класса), еще до требования.

### Через статическое поле (Eager initialization by static member)
Самая простая реализация паттерна Singleton. В этом подходе экземпляр создается при загрузке класса (class loading), потокобезопасность гарантируется спецификацией языка. Единственным минусом является то, что реализация не ленивая.
```java
public class Singleton {
    
    private static final Singleton INSTANCE = new Singleton();
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

### Через перечисление (Eager initialization by using Enum)
Очень элегантный подход, предложанный [Джошуа Блохом](https://en.wikipedia.org/wiki/Joshua_Bloch) в своей книге Effective Java. В данном случае JLS гарантирует единственный экземпляр. Если кто еще не знает, то перечисления в Java могут содержать методы.
```java
public enum Singleton {
    INSTANCE;
    
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
```

## Тонкие места. Сериализация и клонирование.
// TODO
