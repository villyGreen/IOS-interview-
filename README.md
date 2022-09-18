# IOS-interview
It is useful guide for future IOS interview 

Hi, in this file i collected many useful information, then can be halped for your technical interview

## Overview:
- Value Semantics
- Memmory management
- Collections
- Multithreading
- Dispatching


# Swift Value Semantics
  В языке Swift имеется два вида значений:
  1) Ссылочный тип (классы, clousers)
  2) Тип значения (структуры, перечисления)
  
Ссылочный тип - тип, который хранит в себе указатель на обьект (адресс этого обьекта на каком-то участке памяти), хранится на куче (динамическая память), при этом сам указатель хранится в стеке 
 
Тип значения - хранится на стеке (статическая память). Есть два сценария когда value type не будет хранится на стеке:
a) Когда внутри value type есть reference.
b) когда type конформит какой-нибудь проток.
с) когда захватывается в блоке и мутируется.
Если размер value type слишком большой, то происходит inline optimization и он переходи на кучу, а в стеке остается только указатель на него
### Стек и Куча

Стек используется для статического выделения памяти, а куча — для динамического выделения памяти, которые хранятся в оперативной памяти компьютера.

Переменные, размещенные в стеке, хранятся непосредственно в памяти, и доступ к этой памяти очень быстрый, а ее выделение определяется при компиляции программы. Когда функция или метод вызывает другую функцию, которая, в свою очередь, вызывает другую функцию и т. д., выполнение всех этих функций приостанавливается до тех пор, пока самая последняя функция не вернет свое значение. Стек всегда резервируется в порядке LIFO, самый последний зарезервированный блок всегда является следующим освобождаемым блоком. Это упрощает отслеживание стека. Освобождение блока из стека — это не что иное, как корректировка одного указателя.

![image](https://i.stack.imgur.com/gKoxc.png)

# Memory Management
Обьект находятся в памяти до тех пор, пока на него ссылается хоть одна сильная ссылка, для избежания retain-cycle (обьекты ссылаются друг на друга, и не могут быть освобожденны из памяти) применяются weak и unowned ссылка

## Memmory Managenemt до Swift 4
До 4 версии swift, счетчик сильных слабых и сильных ссылок, хранился прямо в обьекте вместе со свойствами
Для примера возьмем следующею ситуацию:

1) На обьект указывает сильная и слабая ссылка
2) Сильная ссылка была уничтожена, данные обьекта уничтожаются, но память так и не была освобождена, обьект становится zombie (NSZombie) на который ссылается слабая ссылка
3) Как только будет обращение по этой ссылке, runtime проверит zombiie этот обьект или нет, если да - то уменьшит счетчик ссылок

Какие минусы:
 1) Неоптимизированное использование памяти, такие обьекты могут долго хранится в памяти
 2) Не потоко-безопасно:
  import Foundation

```
import Foundation
class Target {}
class WeakHolder {
   weak var weak: Target?
}
for i in 0..<1000000 {
   print(i)
   let holder = WeakHolder()
   holder.weak = Target()
   dispatch_async(dispatch_get_global_queue(0, 0), {
       let _ = holder.weak
   })
   dispatch_async(dispatch_get_global_queue(0, 0), {
       let _ = holder.weak
   })
}
```

Данный кусок кода может получить ошибку в **Runtime**. Суть именно в том механизме, который был рассмотрен ранее. Два потока могут одновременно обратиться к объекту по слабой ссылке. Перед тем, как получить объект, они проверяют, является ли проверяемый объект «зомби». И если оба потока получат ответ true, они отнимут счётчик и постараются освободить память. Один из них сделает это, а второй просто вызовет краш, так как попытается освободить уже освобожденный участок памяти.
Такая реализация не очень хороша и с этим нужно что-то делать.

## Memmory Managenemt сейчас
 В актуальной версии swift используется side tables. Side table - это область в памяти, которая хранит в себе информацию об колличестве ссылок на обьект.
 Как только мы начинаем ссылаться на объект слабо `weak reference` - то создается боковая таблица, и теперь объект вместо сильного счетчика ссылок хранит ссылку на боковую таблицу.

Сама боковая таблица также имеет ссылку на объект. Еще боковая таблица может создаваться, когда `происходит переполнение счетчика`, и он уже не помещается в поле (счетчики ссылок будут маленькими на 32-битных машинах).

С таким механизмом слабые ссылки ссылаются не напрямую на объект, а на боковую таблицу, которая указывает на объект. Это решает две предыдущие проблемы:
  - Экономие памяти: объект удаляется из памяти, если на него больше нет сильных ссылок.
  - Это позволяет безопасно обнулять слабые ссылки, поскольку слабая ссылка теперь не указывает напрямую на объект и не является предметом `race condition.`

Вот так **side table** выглядит:

```class HeapObjectSideTableEntry {
  std::atomic<HeapObject*> object;
  SideTableRefCounts refCounts;
}
```
  - В самом первом поле хранится указатель на объект, которому принадлежит эта **side table**. 
  - Следом лежит его счетчик ссылок, представленный структурой типа **SideTableRefCounts**. Внутри нее хранится битовое поле со счетчиками ссылок и флагами.

Счетчик ссылок хранится внутри структуры **HeapObject**. **HeapObject** – это внутреннее представление объекта в рантайме. То есть каждый экземпляр класса в рантайме это экземпляр структуры с типом **HeapObject**.

В алгоритме работы счетчика ссылок определено пять состояний, в которых объект находится на всем пути от создания до удаления из памяти. 
Можно провести параллель с жизненным циклом **UIViewController**. 
Он создается, отображает визуальные элементы, реагирует на вызовы от операционной системы и в конце деаллоцируется. 
Состояния объекта перечислены ниже:

1. `Live` – объект создан и находится в памяти.
2. `Deiniting` – объект находится в процессе деинициализации, то есть у него вызван метод `deinit`.
3. `Deinited` – объект полность деинициализирован.
4. `Freed` – выделенная память под объект освобождена, но `side table` еще существует.
5. `Dead` – память занятая side table освобождается.

![image](https://user-images.githubusercontent.com/47610132/176542135-43d31345-7d44-4197-bafa-96b37afc7e69.png)

  - `Live` его счетчики инициализируются со значениями strong — 1, unowned — 0, weak — 0 (weak появляется только с боковой таблицей). 
На данный момент нет боковой таблицы. Операции с **unowned** переменными работают нормально. Когда `strong reference count` достигает нуля, вызывается deinit(), и объект переходит в следующее состояние (deiniting).
  - `Deiniting` - на данном этапе операции со `strong` ссылками не действуют. При чтении через **unowned** ссылку будет срабатывать **assertion failure**. 
Но новые **unowned** ссылки еще могут добавляться. Если есть боковая таблица, то **weak** операции будут возвращать nil. Далее из этого состояния уже можно перейти в два других.
  - Если нет боковой таблицы, т.e нет `weak ссылок` и нет `unowned ссылок`, то объект переходит в `Dead` состояние и сразу удаляется из памяти.
  - Если у нас есть `unowned или weak ссылки`, объект переходит в состояние `Deinited`. 
В этом состоянии функция deinit() завершена, сохранение и чтение сильных или слабых ссылок невозможно. 
Как и сохранение новых `unowned ссылок`. При попытке чтения `unowned ссылки` вызывается assertion failure. Из этого состояния также возможно два исхода.
  - В случае наличия `weak ссылок`, а значит и боковой таблицы, осуществляется переход в состояние `Freed`. 
В `Freed` состоянии объект уже полностью освобожден и не занимает места в памяти, но его боковая таблица остается жива.
  - После того как счетчик слабых ссылок достигает нуля, боковая таблица также удаляется и освобождает память, и осуществляется переход в финальное состояние — `Dead`. 
В этом состоянии от объекта ничего не осталось, кроме указателя на него. Указатель на `HeapObject` освобождается из кучи, не оставляя следов объекта в памяти.

# Difference between setNeedsLayout and layoutIfNeeded
  У UIView есть методы, которые отвечают за изменения размеров и констрейнтов, один из методов - это LayoutSubviews()
 Для более понятного обьяснения, необходимо вспомнить life cycle view controller а именно методы:
 
ViewWillLayoutSubviews:
По умолчанию ничего не делает. Когда границы представления изменяются, представление корректирует положение своих подпредставлений. Контроллер представления может переопределить этот метод, чтобы внести изменения до того, как представление разместит свои подпредставления.

viewDidLayoutSubviews:
Этот метод вызывается после того, как viewController приспосабливается к своему подпредставлению после изменения его границы. Добавьте сюда код, если вы хотите внести изменения в подпредставления после того, как они были установлены.

Для примера создадим вью и применим к нему corner radius, все это будет происходить во viewDidLoad,  сожалению мы не увидим желаемый результат, т.к наше вью не обработало изменение размеров, для решения этой проблеммы мы могли весь этот код положить в метод viewDidLayoutSubviews - получим желаемый рузультат. 
Иногда нам нужно вызвать этот метод самостоятельно, для этого нам на помощь приходят setNeedLayoutSubview и layoutIfNeeded.
UIView.layoutIfNeeded() - Происходит новый перерасчет размеро и ограничений для всех subviews.
Пример: Есть вью, которая привязана к правому, левому, верхему краю констрейнтами и имеет констрейнт на ширину 200, мы хотим в блоке анимации умешьшать её высоту, для этого перед блоком анимации желательно вызвать view.layoutIfNeeded (рекомендация от Apple, для завершения всех предыдущих расчетов), и самом блоке анимации опять вызвать этот метод, как итог - мы получим плавно изменяющуюся высоту view 

<img src="https://i.stack.imgur.com/UB2AF.gif"/>

В чем сообственно разница между setNeedLayoutSubviews и layoutIfNeeded ?

Когда приложение iOS запускается, UIApplication в iOS запускает основной цикл выполнения для приложения, которое выполняется в основном потоке. Основной цикл выполнения обрабатывает события (например, касания пользователя) и обрабатывает обновления интерфейсов на основе представлений. По мере возникновения событий, таких как прикосновение, обновление местоположения, движение и управление мультимедиа, цикл выполнения находит подходящий обработчик для событий, вызывает соответствующие методы, которые вызывают другие методы и т. д. В какой-то момент времени все события будут обработаны, и управление вернется в цикл выполнения. Обозначим эту точку, где управление возвращается в цикл выполнения, как цикл обновления.

Пока события обрабатываются и запрашиваются некоторые изменения в представлении, эти изменения не обновляются немедленно. Вместо этого система ждет завершения существующего процесса и наступления следующего цикла перерисовки. Существует периодический интервал между обработкой события и обработкой обновления макета пользовательского интерфейса.

### setNeedsLayout

Метод setNeedsLayout для UIView сообщает системе, что вы хотите, чтобы она разметила и перерисовала это представление и все его подпредставления, когда придет время для цикла обновления. Это асинхронное действие, потому что метод завершается и возвращается немедленно, но макет и перерисовка фактически происходят только через какое-то время, и вы не знаете, когда будет этот цикл обновления.

### LayoutIfNeeded

Напротив, метод layoutIfNeeded являетсясинхронныйвызов, который сообщает системе, что вы хотите макет и перерисовку представления и его подвидов, и вы хотите, чтобы это было сделано немедленно, не дожидаясь цикла обновления. Когда вызов этого метода завершен, макет уже настроен и отрисован на основе всех изменений, которые были отмечены до вызова метода.
