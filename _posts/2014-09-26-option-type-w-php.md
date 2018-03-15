---
layout: post
title: "Typ Option, czyli jak uniknąć Fatal error: Call to member function getTitle() on a non-object..."
author: "psliwa"
tags: [php, programowanie funkcyjne, wzorce projektowe]
---

Poprzedni mój [wpis][2] objaśniał podstawy programowania funkcyjnego, tym razem zajmę się wzorcem projektowym, który wywodzi się z języków funkcyjnych i jest szeroko stosowany w Haskellu, Scali, czy F#. Mowa tutaj o type **Option**. Inne nazwy tego patternu to Optional (java8), czy Maybe (Haskell) - podaję jabyście chcieli coś na ten temat wygooglować.

Nadmiar `ifów` w kodzie nie jest dobry, jest wiele sposobów które pozwalają zredukować ich liczbę. Takimi sposobami są niektóre wzorce takie jak State, czy NullObject. Wzorzec Option Type w php ma ograniczone zastosowanie, ale mimo to w wielu przypadkach jest użyteczny.

### Przykład

Może na początek przykład, który będzie nam towarzyszył do końca artykułu. Jest on z grubsza zaczerpnięty z kodu produkcyjnego, więc jest jak najbardziej praktyczny i nie modeluje zoo.

Na początek zarysuję trochę dziedzinę problemu oraz sam problem. Mamy kanały, które agregują wiadomości, a każdy kanał należy do grupy kanałów. 

	ChannelGroup 1----* Channel 1----* Message

Grupa kanałów m. in. określa, czy w kanałach które do niej należą jest aktywne "zaawansowane targetowanie wiadomości" (cokolwiek to znaczy - dla przykładu nie jest to ważne). Mamy tablicę danych, w których prawdopodobnie jest wartość pod kluczem "channelId", chcemy dodać coś do zapytania z Doctrine2 (np. join lub warunek where) z id grupy, gdy grupa podanego kanału ma włączone zaawansowane targetowanie.

W powyższym opisie występują słowa takie jak: "prawdopodobnie", "gdy", więc kilka `ifów` będzie niezbędnych.

Przykład kodu w stylu imperatywnym:

```php
if(isset($data['channelId'])) {
    $channel = $this->channelRepo->find(data['channelId']);
    
    if($channel !== null) {
        $group = $channel->getGroup();
        
        if($group->hasAdvancedMessageTargeting()) {
            $qb->...;//Dodanie jakiegoś warunku do zapytania
        }
    }
}
```

3 ify, w dodatku zagnieżdżone. Ten kod nie wygrałby konkursu piękności... 

Ale działa.

Czy ten kod da się zapisać nie używając ani jednego wyrażenia warunkowego (ifa lub operatora trójkowego)? Tak, za pomocą typu **Option**.

### Option?

Na wstępie podaję namiary na bardzo dobrą i zdatną do użytku implementację typu `Option` w php: [php-option][6].

Zanim pokażę przykład, najpierw powiem czym jest typ `Option`, bo inaczej pokazany kod mógłby nie być wystarczająco zrozumiały. Option służy do zastąpienia pustych referencji, czyli wartości `null`. Ma 2 stany: 

* gdy zawiera wartość - wtedy nazywamy taki option `Some(x)`, gdzie `x` to wartość opakowywana
* gdy nie zawiera żadnej wartości - nazywamy go wtedy `None`

Wartość można opakować w typ Option za pomocą `Option::fromValue($value)`. Wartość zwrócona przez tą metodę będzie `Some($value)` gdy `$value` nie jest `nullem`, lub w przeciwnym wypadku `None`. Samo z siebie to nie byłoby zbyt użyteczne, ale `Option` ma szereg przydatnych metod. Dzięki nim możemy przekształcać ewentualnie przechowywaną wartość, filtrować ją, podawać wartości domyślne oraz oczywiście pobrać opakowaną wartość. Najważniejsze metody to: `Option map(callable)`, `Option filter(callable)`, `Option flatMap(callable)`, `Option orElse(Option)`, `mixed getOrElse(mixedDefaultValue)`, `mixed get()`, `void forAll(callable)`. 

Poniższa tabelka przedstawia wartości zwrotne z poszczególnych metod w zależności od tego czy Option to `Some(x)`, czy `None`.

<table>
<thead>
<tr><th>metoda</th><th>Some(x)</th><th>None</th></tr>
</thead>
<tbody>   
<tr>
<td><code>map($func)</code></td>
<td><code>new Some($func($x))</code></td>
<td><code>$this</code> - czyli None</td>
</tr>
<tr>
<td><code>flatMap($func)</code></td>
<td><code>$func($x)</code><br/>$func musi zwracać Option!</td>
<td><code>$this</code> - czyli None</td>
</tr>
<tr>
<td><code>filter($predicate)</code></td>
<td><code>$predicate($x)<br/>   ? $this : new None</code></td>
<td><code>$this</code> - czyli None</td>
</tr>
<tr>
<td><code>orElse($option)</code></td>
<td><code>$this</code></td>
<td><code>$option</code></td>
</tr>
<tr>
<td><code>getOrElse($val)</code></td>
<td><code>$x</code> - opakowana wartość</td>
<td><code>$val</code></td>
</tr>
<tr>
<td><code>get()</code></td>
<td><code>$x</code> - opakowana wartość</td>
<td><code>throw new Exception()</code></td>
</tr>
</tbody>
</table>

Metoda `forAll` działa jak `map`, z tą różnicą, że wartość zwrotna jest ignorowana. Można ją użyć, gdy chcemy wykonać kawałek kodu, gdy Option przechowuje jakąś wartość.

Przykładowo jeśli chcemy wyświetlić autora książki, nie wiedząc czy zmienna która reprezentuje książkę jest `nullem`, możemy to zrobić tak:

```php
echo Option::fromValue($book)
    ->map(F::prop('author'))
    ->getOrElse('No book, no author...');
```

Użyłem funkcji `F::prop('author')` (z [rfp][5]), która tworzy funkcję zwracającą atrybut `author` obiektu podanego jako argument. Mógłbym użyć funkcji anonimowej, ale niestaty w php nie są one zwięzłe i unikam ich gdzie tylko mogę. W dalszej części wpisu również będę korzystał z innych funkcji z [rfp][5] (np. `F::pipe`),  zachęcam do  zapoznania sięz [poprzednim wpisem][2], bo pomoże on Wam w jakimś stopniu nabrać "funkcyjnego myślenia" i dzięki temu przykłady z tego wpisu będą bardziej zrozumiałe.

Wracając do tematu, w tym kodzie jest jeden problem, a mianowicie co się stanie gdy książka nie ma autora? Metoda `map`, wg tabelki wyżej, zawsze zwraca `Some`, tak więc zostanie zwrócone `Some(null)` - nie jest to pożądane. Funkcję `map` używa się wtedy, gdy mamy pewność że funkcja mapująca nie zwraca `null`. Jeśli funkcja mapująca może zwrócić `null` używamy funkcji `flatMap`. Ale... Bardziej spostrzegawczy czytalnik zauważył, że wg magicznej tabeli, funkcja przekazana do `flatMap` musi zwrócić `Option`... Mamy 2 wyjścia: 1) atrybut `author` obiektu `$book` będzie miał wartość `Option`, czyli `Option` przesiąknie do naszego publicznego api, lub 2) opakujemy wartość zwrotną funkcji `F::prop('author')` w typ `Option`. Sposób 1) niekoniecznie musi być dobry. Jestem zdania, że typ `Option` w php nie powinien przesiąknąć do publicznego api, gdyż nie wiemy jakiego typu przechowywana jest wartość w `Option`, stracimy (i tak ograniczoną) kontrolę typów na poziomie języka oraz możliwość podpowiadania składni w IDE, gdyż w phpdoc nie ma czegoś takiego jak typy generyczne (np. `Option<Book>`). Pozostaje sposób 2), jak to możemy zrobić? Za pomocą funkcji `F::pipe()` oraz `Option::fromValue`.

Bezpieczna wersja przykładu

```php
echo Option::fromValue($book)
    ->flatMap(
        F::pipe(F::prop('author'), [Option::class,'fromValue'])
    )
    ->getOrElse('No book or no author...');
```

Ok, wróćmy do naszego głównego przykładu.

### Ok, to wracamy do przykładu

Ok, wróciliśmy do przykładu.

Oto kod dodający coś do zapytania `$qb` pod pewnymi warunkami, zapisany z wykorzystaniem typu `Option`.

```php
$option = function($func) {
    return F::pipe($func, [Option::class,'fromValue']);
};

Option::fromArrayValue($data, 'channelId')
        ->flatMap($option([$this->channelRepo,'find']))
        ->map(F::prop('group'))
        ->filter(F::prop('advancedMessageTargeting'))
        ->forAll(function(ChannelGroup $group) use($qb){
          	$qb->...;//dodanie jakiegoś warunku do zapytania
        });
```

Tak jak obiecałem - nie ma ifów, nie ma zagnieżdżonych bloków kodu. Zdefiniowałem funkcję `$option`, aby było czytelniej. Równie dobrze można sobie tą funkcję umieścić w jakiś funkcyjnych utilsach, bo będzie przydatna nie raz.

Wyrażenia warunkowe zostały zastąpione wywołaniami `Option::fromArrayValue`, `flatMap` oraz `filter`. Po każdym wywołaniu tych funkcji, może zostać zwrócone `Some(x)` lub `None`. Jeśli któraś z funkcji zwróci `None`, to reszta wywołań zwraca `None` nie robiąc nic ponadto. Funkcja `map` zawsze zwraca `Some(x)` - w tym przypadku mogłem ją użyć, bo każdy kanał musi należeć do grupy kanałów. Jeśli predykat w funkcji `filter` jest spełniony, to zwracane jest `Some(x)`, w przeciwnym wypadku `None`. Funkcja przekazana do `forAll` zostanie wywołana tylko gdy Option ma wartość, czyli jest `Some(x)`. 

Tak, ten kod też działa.

### Konkluzja

Na co dzień używam trochę prostszych słów na **k**, ale czego się nie robi dla nauki.

Typ `Option` W niektórych statycznie typowanych językach z typami generycznymi (Scala, #F) całkowicie zastąpił wartość null. W php jednak takie podejście, oprócz zalet, ma również poważne wady. Type hinting na poziome języka oraz specyfikacja phpdoc nie przewidziały typów generycznych = brak pomocy IDE przy obsłudze typ `Option`. Są również inne ograniczenia języka, które nie faworyzują tego podejścia względem wartości `null`. Tak więc, typ `Option` nie powinien być używany przy definiowaniu naszego api (nie powinien być zwracany czy przyjmowany jako parametr przez publiczne metody). Jednak nie samym api aplikacja żyje, `Option` doskonale odnajdzie się jako szczegół implementacyjny metod. Dzięki niemu można sprawić, że kod będzie czytelniejszy, bardziej deklaratywny. `Option` doskonale integruje się z innymi technikami funkcyjnymi, które również powinny być szczegółem implementacyjnym - tak więc nie pozostaje nic innego jak zaprząc je razem do tańca.


[1]: https://github.com/igorw/compose
[2]: http://psliwa.org/bo-obiekty-to-za-malo-czyli-o-programowaniu-funkcyjnym-w-php/
[3]: https://github.com/schmittjoh/php-option
[4]: https://github.com/psliwa/rfp
[5]: https://github.com/psliwa/rfp/blob/master/src/Rfp/F.php
[6]: https://github.com/schmittjoh/php-option