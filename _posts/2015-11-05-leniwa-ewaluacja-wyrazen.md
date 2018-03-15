---
layout: post
title: "Bądź leniwy, a nie chciwy - słów kilka o leniwej ewaluacji"
author: "psliwa"
tags: [php, programowanie funkcyjne, scala]
---

Ostatnio można było odnieść wrażenie, że tak jak tytuł mówi, jestem leniwy (ostatni wpis na początku roku), ale nic bardziej mylnego. Już zabieram się za temat leniwej ewaluacji. Wyjdę od Scali, a później przejdę do PHP.

## Leniwa ewaluacja na przykładzie Scali

Są dwa sposoby ewaluacji wyrażeń: chciwe (strict) oraz leniwe (lazy). Scala jest językiem, który domyślnie wyrażenia ewaluuje chciwie (tak jak większość języków), ale gdy tego chcemy, możemy oznaczyć aby dane wyrażenie było leniwe.

```scala
lazy val a = 1 + 2
lazy val b = a + 3
val c = a + b // obliczenie wartości a i b odbywa się dopiero tutaj
              // a nie w momencie inicjalizacji zmiennej
```

Listy i większość innych kolekcji w scali są chciwe, jedną z leniwych struktur jest Stream - jest to leniwa wersja listy. Przykład strumienia będącego nieskończonym ciągiem liczb naturalnych:

```scala
val naturals: Stream[Int] = 0 #:: naturals.map(_ + 1)
//res0: Stream[Int] = Stream(0, ?)
```

Powyższy zapis jest skróconym zapisem:

```scala
val naturals: Stream[Int] = Stream.cons(0, naturals.map(_ + 1))
```
Poniżej dwa akapity wyjaśnień, bo mimo iż zapis ten jest prosty, naturalny i dosyć matematyczny, mózg może się przed nim lekko opierać.

Co to jest rekurencyjna struktura danych? Wyjaśnię to na przykładzie. [Matrioszka][1] to klasyczna rosyjska zabawka składająca się z `X` lalek. `X - 1` lalek jest otwieranych i może zawierać w sobie kolejną (mniejszą) lalkę. Ostatnia lalka nie jest otwierana i nie można już do niej nic wsadzić. Tak więc pierwsza lalka zawiera drugą, która zawiera trzecią i tak do ostatniej, która nie zawiera już kolejnej lalki. Każda z lalek ma ten sam "interfejs", wygląda podobnie. Ostatnia lalka różni się od reszty tym, że nie zawiera kolejnej. Podobnie jest ze strukturami rekurencyjnymi. W przypadku strumieni, "lalkę" mogąca zawierać inną "lalkę" nazywamy `Cons`, zaś ostatnią pustą "lalkę" `Empty`. `Cons` jest węzłem, który ma wartość i referencję do pozostałej części strumienia (ogona). `Empty` to pusty strumień. Konstruktor `Cons` wygląda następująco: `Cons(value, Stream)` - ma referencję do przechowywanej wartości oraz dalszej części strumienia. Dalsza część strumienia może być `Cons` lub `Empty` (otwierana lub ostatnia pusta "lalka"). Przykładowy strumień złożony z 2 elementów: `Stream.cons(0, Stream.cons(1, Stream.Empty))` lub składnia skrócona: `0 #:: 1`. Jako referencji do całego strumienia używamy referencję do jego pierwszego elementu - tak samo jak w przypadku matrioszki, pierwsza lalka jest lalką samą w sobie, nie jest potrzebne dodatkowe opakowanie tak jak w przypadku klasycznej listy powiązanej.

Wracając do naszego ciągu liczb naturalnych (ignorujemy to że może się przekręcić po dojściu do max int). `0` to pierwszy element strumienia, `naturals.map(_ + 1)` to strumień będący dalszą częścią głównego strumienia (drugim argumentem konstruktora `Cons`). Bardzo ważne jest to, że `naturals.map(_ + 1)` jest **ewaluowane leniwie**, czyli wartość drugiego elementu zostanie wyliczona tylko gdy będziemy chcieli go odczytać. Zapis może na początku lekko kołować, ale wystarczy sobie uświadomić, że jest to rekurencyjna definicja równoznaczna z rekurencyjnym wywołaniem funkcji:

```scala
def createNaturals(from: Int): Stream[Int] = from #:: createNaturals(from+1)
val naturals = createNaturals(0)
```

Jeśli dwa poprzednie akapity nie są jasne, przeczytaj je jeszcze raz (rekurencyjnie, aż do spełnienia warunku wyjścia) ;)

Wyliczenie wartości następuje tylko gdy chcemy tą wartość odczytać:

```scala
naturals(5) //pobranie 5-tego elementu ciągu
            //następuje ewaluacja ciągu do 5-tego 
            //elementu włącznie, 6-ty element nie jest ewaluowany
//res1: Int = 5
```

Ok, teraz trochę bardziej złożony przykład - definicja ciągu fibonacciego:

```scala
val fib: Stream[Int] = 0 #:: 1 #:: fib.tail.zip(fib).map{ case(a, b) => a + b }
//res2: Stream[Int] = Stream(0, ?)
```

Fragment `fib.tail.zip(fib)` tworzy nowy strumień, którego elementy to pary odpowiadających sobie elementów ze strumieni `fib` oraz `fib.tail`. `fib.tail` to strumień `fib` "przesunięty" o jedną pozycję do przodu, dzięki czemu otrzymujemy sumę dwóch kolejnych wyrazów ciągu.

Można się pobawić ciągiem fibonacciego:

```scala
fib(10)
//res3: Int = 55
fib.take(5).toList //pobranie 5 pierwszych wyrazów ciągu 
                   //i skonwertowanie na listę
//res4: List[Int] = List(0, 1, 1, 2, 3)
```

### Zalety strumieni i innych leniwych struktur

1. **Liczba operacji**
Do wyliczenia ostatecznego wyniku zostanie wykonana **konieczna** liczba operacji, **ale nie więcej** ;) Można operować na dużych lub nieskończonych strumieniach, a na końcu ograniczyć wynik do jakiegoś zakresu elementów lub pobrać jeden konkretny element. Podczas obliczeń nie trzeba się przejmować rozmiarem strumienia. 

1. **Złożoność pamięciowa**
Wykonując kilka operacji jedna po drugiej, nie powstają strumienie z danymi przejściowymi. Przykładowo wywołanie `Stream(1, 2, 3, 4, 5).map(_+1).filter(_ % 2 == 0).toList` powoduje, że najpierw operacje map i filter są wywołane na pierwszym elemencie, następnie na drugim itp. Tak więc **następuje jedna iteracja** po elementach strumienia. W przypadku list jest inaczej, wyniki przejściowe są wyliczane. Dla wyrażenia `List(1, 2, 3, 4, 5).map(_+1).filter(_ % 2 == 0)` najpierw na wszystkich elementach jest wywoływana operacja `map`, a następnie na wszystkich elementach nowej listy jest wywołana operacja filter. Wydawać by się mogło, że dzięki temu łączenie w łańcuch wywołań na strumieniu zawsze będzie mieć złożoność obliczeniową O(n) - niestety w praktyce tak nie jest. Teoretycznie tak powinno być, ale strumienie są zaimplementowane za pomocą `lazy val`, które nie są szczególnie wydajne, gdyż wykorzystują pod maską blok synchronizowany. Przez to **wydajność operacji łączonych na strumieniu jest mniej przewidywalna niż w przypadku listy**, jednakże nadal złożoność **pamięciowa jest wyraźnie lepsza** i cechuje się faktycznie na poziomie **O(n)**. Są także inne leniwe struktury w Scali, np. widoki, czy iteratory, które już nie mają tego problemu. Aby utworzyć leniwy widok dla listy, wystarczy wywołać na niej funkcję `view`.

1. **Bezpieczna rekurencja**
Jak wiadomo rekurencja może powodować **przepełnienie stosu (stack overflow)**. W językach funkcyjnych rekurencja jest czymś powszednim i nie można sobie pozwolić na stack overflow np. przy operowaniu na liście o długości 2000 elementów. Bardzo wiele kombinatorów jest zaimplementowanych za pomocą rekurencji, co normalnie by powodowało stack overflow przy np. wywołaniu operacji `map` na liście z 2000 elementów. Jednakże istnieje optymalizacja kompilatora, która sprawia, że wywołania rekurencyjne nie odkładają się na stosie, jeśli mamy do czynienia z tzw. **tail recursion**. Nie będę się na ten temat więcej rozwodził - zainteresowanym polecam google. Niektóre algorytmy ciężko zapisać za pomocą tail recursion. W przypadku leniwych strumieni rekurencyjna funkcja nie zaimplementowana za pomocą tail recursion, która operuje na tym strumieniu, nie spowoduje stack overflow. **Rekurencyjne funkcje operujące na leniwych kolekcjach są bezpieczne** i nie trzeba ich zapisywać za pomocą tail recursion. W PHP, o którym za chwilę, tak czy siak nie ma optymalizacji tail recursion, a leniwe kolekcje nie uchronią nas przed stack overflow.

## Leniwy PHP

Zanim przejdziesz dalej, spróbuj zaimplementować analogiczny ciąg liczb naturalnych i ciąg fibonacciego w php.

. *przerwa aby uchronić przed spojlerem*

.

.

.

W PHP 5.5 został wprowadzony nowy ficzer: **generatory**. Generator to nic innego jak iterator stworzony za pomocą wygodnej składni i z leniwie wyliczanym kolejnym elementem.

```php
function naturals() {
    $i = 0;
    while(true) yield $i++;
}

$naturals = naturals();//nieskończony ciąg liczb 
                       //naturalnych (może się przekręcić ;))
```

Słowo kluczowe `yield` generuje jeden element kolekcji, drugi zostanie wygenerowany tylko wtedy gdy jawnie uzyska się do niego dostęp. W PHP 7 generatory zostały ulepszone i generator może korzystać z innego generatora w celu dostarczenia kolejnych wartości. Służy do tego składnia `yield from $generator` (patrz poniższa definicja funkcji `zip`).

Od razu zdefiniuję kilka leniwych kombinatorów, które będą przydatne.

```php
function map(callable $f, $coll) {
    foreach($coll as $element) {
        yield $f($element);
    }
}

//pobiera $count pierwszych elementów
function take($count, $coll) {
    $i = 0;
    foreach($coll as $element) {
        if($i++ >= $count) break;
        yield $element;
    }
}

//porzuca $count pierwszych elementów
function drop($count, $coll) {
    $i = 0;
    foreach($coll as $element) {
        if($i++ >= $count) yield $element;
    }
}

//pobranie $n-tego elementu
function get($n, $coll) {
    $i = 0;
    foreach($coll as $element) {
        if($i++ === $n) return $element;
    }
    
    return null;
}

//Poniższa implementacja działa tylko w PHP 7+.
//Standardowa funkcja zip, z dwóch iteratorów o długości "n"
//tworzy jeden, którego "n-ty" element to wynik wywołania $func z 
//"n-tymi" elementami dwóch iteratorów.
function zip($func, \Iterator $as, \Iterator $bs) {
    if($as->valid() && $bs->valid()) {
        yield $func($as->current(), $bs->current());
        $as->next(); $bs->next();
        yield from zip($func, $as, $bs);
    }
}

//tworzy funkcję dodającą $a do argumentu
function addX($a) {
    return function($b) use($a) {
        return $a + $b;
    };
}

//funkcja dodająca dwie liczby
function add($a, $b) {
    return $a + $b;
}
```

Ciąg liczb naturalnych już zaimplementowaliśmy, teraz pora na ciąg fibonacciego (wersja iteracyjna dla PHP >=5.5 i <7).

```php
function fib() {
    $a = 0; $b = 1;
    yield $a;
    yield $b;

    while(true) {
        $c = $a + $b;
        yield $c;
        $a = $b;
        $b = $c;
    }
}

$fib = take(20, drop(10, map(addX(2), fib())));
//szaleństwo, mapuję po nieskończonym ciągu,
//później porzucam 10 elementów i wybieram 20 kolejnych,
//czy to przejdzie i zadziała? Tak.
```

Wersja rekurencyjna (PHP 7+) - ten sam algorytm co w scali. Wykorzystanie rekurencyjnych generatorów:

```php
function fib() {
    yield 1;
    yield 2;
    yield from zip('add', fib(), drop(1, fib()));
}
```

### Zalety generatorów (PHP), a zalety strumieni (Scala)

1. **Liczba operacji**
Generatory w php także wykonają minimalną, konieczną pracę.

1. **Złożoność pamięciowa**
Generatory w php zachowują się podobnie przy łączeniu operacji co strumienie w scali. Czas wykonania jest zbliżony do operacji nie generujących generatory. Z czego to wynika? Nie wiem, być może również ze szczegółów implementacyjnych jak w przypadku strumieni w scali. Złożoność pamięciowa jest podobna jak w strumieniach, nie są generowane przejściowe wyniki.

1. **Bezpieczna rekurencja**
Niestety generatory nie chronią przed stack overflow przy rekurencji.

### Inne zalety generatorów

1. Pozwalają wygodnie i elegancko iterować po dużej strukturze danych, mając w pamięci tylko jeden jej element. Np. iterowanie po liniach pobranych z dużego pliku.
1. Tworzenie generatorów wymaga minimum boilerplatu. Generator można wykorzystywać do tworzenia iteratorów za pomocą czytelnej i przyjaznej składni.
1. Kod klienta nie musi wiedzieć, że korzysta z generatora. Struktura danych, przekazana do przykładowej funkcji `map`, może być tablicą bądź dowolnym `Traversable`, a na wyjściu otrzymamy generator.
1. W PHP 7 generatory mogą korzystać z innych generatorów, dzięki czemu można stworzyć generator rekurencyjny. Pozwala to na krótszy i bardziej elegancki kod.

[1]: https://pl.wikipedia.org/wiki/Matrioszka