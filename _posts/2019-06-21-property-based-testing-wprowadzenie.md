---
layout: post
title: "Property based testing - wprowadzenie"
author: "psliwa"
tags: [php, scala, java, testy, programowanie funkcyjne, property based testing]
---

Ostatnio prowadziłem krótki warsztat na temat property based testing na
przykładzie [scalachecka][1]. W tym wpisie postaram się przedstawić ideę
jaka stoi za tym sposobem testowania. Kod źródłowy ze wspomnianego
warsztatu jest dostępny na [githubie][5]. Jest tam zalążek projektu,
który się kompiluje, a na branchu `solutions` są rozwiązania do ćwiczeń.

Klasyczne podejście do testowania to testy w postaci "Given / When /
Then". Sekcja "Given" polega na zdefiniowaniu stanu początkowego
środowiska wokół tego co testujemy i utworzenie samej jednostki do
testowania. Następnie w "When" wywołujemy metodę / funkcję którą chcemy
przetestować przekazując określone dane wejściowe i w "Then" sprawdzamy
czy zadziało się to, czego oczekujemy. Dane wejściowe są wpisane na
sztywno - w teście podajemy przykładowe dane. Stąd też potoczna nazwa
"example based testing", gdyż w tych testach wypisujemy przykładowe
scenariusze z konkretnymi danymi. Czasami dwa scenariusze różnią się od
siebie tylko danymi testowymi, prowadzi to do stosowania
parametryzowanych testów (w [junit][2], w [phpunit][3] itp.). Co by się
stało, jeśli wszystkie testy były z definicji parametryzowane, a w
dodatku parametry byłyby wygenerowane przez silnik testowy? Doszlibyśmy
do "property based testing". Wtedy nie mielibyśmy przykładowych
konkretnych scenariuszy, tylko "właściwość" naszego kodu.

Przykład testu example based:

```scala
test("map list values") {
  List(1, 2, 3).map(_.toString) shouldBe List("1", "2", "3")
}
```

W jaki sposób zapisać taki test używając podejście property based?

```scala
test("map list values") {
  // forAll to funkcja, która mówi, że "dla każdego zestawu argumentów podana właściwość 
  // zachodzi". Można przyjąć, że jest ona równoznaczna z kwantyfikatorem "dla każdego" 
  // w matematyce, tak więc test zapisany w taki sposób jest w pewnym sensie twierdzeniem 
  // matematycznym.
  forAll { (as: List[Int], f: Int => String) =>
    as.map(f) ... // problem, bo rzuca się w oczy asercja typu as.map(f) shouldBe as.map(f), 
                  // która nie ma sensu i nic nie sprawdza
  }
}
```

Jest to probematyczne, gdyż musimy zmienić lekko sposób myślenia o
teście. Musimy się zastanowić jakie właściwości ma operacja `map`,
zamiast podawać już konkretne, skomplikowane scenariusze, które mogą
powodować, że tak na prawdę powtórzymy w teście kod produkcyjny lub kod
testowy. Np. możemy zauważyć, że dla każdej listy wywołanie
`as.map(identity)` powinno dać tą samą listę (`identity` to funkcja
jednoargumentowa zwracająca argument bez modyfikacji). Zapiszmy to:

```scala
test("map list values using identity function should preserve original list") {
  forAll { as: List[Int] =>
    as.map(identity) shouldBe as
  }
}
```

Mamy pierwszą właściwość operacji `map`. Możemy się dalej zastananawiać,
jakie właściwości ma operacja `map`. Inną właściwością może być to, że
jeśli mamy wartość `a` i utworzymy z niej listę i wywołamy `map(f)`, to
dostaniemy dokładnie `List(f(a))`. Zapiszmy więc to:

```scala
test("map over single value") {
  forAll { (a: Int, f: Int => String) =>
    List(a).map(f) shouldBe List(f(a))
  }
}
```

Ten test w sumie wygląda podobnie jak pierwszy scenariusz "example
based". Możemy jeszcze pogłówkować dalej i spróbować wymyśleć jakąś inną
właściwość `map`. Co jeśli mamy dwie funkcje: `f` i `g`? Czy `map`
wspiera komponowanie funkcji? Sprawdźmy to:

```scala
test("map is composable") {
  forAll { (as: List[Int], f: Int => String, g: String => Int) =>
    as.map(a => g(f(a))) shouldBe as.map(f).map(g)
  }
}
```

Wychodzi na to, że tak. Tak więc zdefiniowaliśmy 3 właściwości operacji
`map`. Tak na prawdę `map` ma [2 podstawowe prawa (właściwości)][4] - w
naszym przypadku pierwsza i trzecia właściwość. Druga właściwość wynika
z trzeciej. Zadanie domowe, aby uzasadnić w jaki sposób ;)

Jak widzimy, nasze testy mają całkowicie inną postać niż w przypadku
"example based". Ważną rzeczą jest nie uogólnianie testów "example
based" tylko spojrzenie na funkcję z dalszej perspektywy: jakie operacja
faktycznie ma właściwości. Innym ważnym czynnikiem jest, aby właściwości
były możliwe jak naprostsze, gdyż z natury mogą one być abstrakcyjne.
Jeśli dodatkowo będą skomplikowane, to taki kod nie będzie czytelny i za
pół roku czytając taką skomplikowaną właściwość będziesz się drapał po
głowie myśląc "co autor miał na myśli". Zaletą takiego podejścia jest
testowanie wielu corner casów za darmo, gdyż tym zajmuje się silnik do
generowania danych.

Skąd się biorą wartości na których operujemy w teście? Są one
wygenerowane przez silnik [scalacheck][1]. Dba on o to, aby wygenerować
wartości specjalne (np. pusta lista, lista jedno elementowa, zero, min
int, max int itp.) i wartości nie specjalne. Za każdym uruchomieniem
zestaw wartości może być różny, gdyż są one dobierane w sposób pseudo 
losowy. Przykład z listą bazuje na podstawowych typach, więc scalacheck
jest w stanie wygenerować nam odpowiednie wartości: listy, funkcje, czy
pojedyncze wartości. Co jeśli mamy własne typy, które byśmy chcieli
przetestować? Musimy napisać własne generatory, o których napiszę w
kolejnym wpisie za jakiś czas :)

Chciałbym też zwrócić uwagę, że "property based testing" to nie święty
graal testowania. Takiego podejścia nie używa się wszędzie, tylko w
uzasadnionych przypadkach. Osobiście w większości przypadków stosuję
klasyczne podejście. Jednak gdy widzę, że property based testing by
przyniósł wartość przy testowaniu konkretnej funkcji, to używam go mając
również jeden / dwa scenariusze "example based".

[1]: https://www.scalacheck.org/
[2]: https://github.com/junit-team/junit4/wiki/parameterized-tests
[3]: https://phpunit.readthedocs.io/en/8.0/writing-tests-for-phpunit.html#data-providers
[4]: https://wiki.haskell.org/Functor#Description
[5]: https://github.com/psliwa/scalacheck-workshop
