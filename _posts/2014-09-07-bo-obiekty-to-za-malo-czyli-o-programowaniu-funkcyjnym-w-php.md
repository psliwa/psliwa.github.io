---
layout: post
title: "Bo obiekty to za mało, czyli o programowaniu funkcyjnym w php"
author: "psliwa"
tags: [php, programowanie funkcyjne]
---

Obiekty\* są fajne, dzięki nim można zrobić miłe w użyciu api.
Obiekty są fajne, bo dają nam narzędzia do reużywalności kodu.
Obiekty są fajne przede wszystkim dlatego, bo dają nam możliwość ukrywania implementacji i hermetyzacji danych.

PHP to przede wszystkim język dwuparadygmatowy, powstał jako język strukturalny. Następnie dodano do niego bardzo kulawe wsparcie do programowania obiektowego (PHP4), które naprostowano w PHP5. Kolejnym krokiem było dodanie trochę mniej kulawego wsparcia dla domknięć (Closure) w PHP 5.3, które częściowo zostało naprostowane w PHP 5.4.

Ale lepszy rydz niż nic. Ten rydz nie jest taki zły, dzięki niemu programowanie funkcyjne w PHP jest łatwiejsze i faktycznie ma sens.

### A o co chodzi?

Nie chcę sypać tutaj definicjami, ale absolutne minimum musi być. W programowaniu funkcyjnym funkcja jest **obywatelem pierwszej kategorii** - to znaczy że można ją przypisać do zmiennej, wykonywać operacje na niej, przekazywać funkcje jako parametr do funkcji itp. Analogicznie do obiektu w OOP - można przypsać obiekt do zmiennej, wywołać jakąś metodę na nim i przekazać ten obiekt jako argument do jakieś metody / funkcji. W php przypisanie do zmiennej i przekazanie jako parametr można zrealizować tak:

```php
//przypisanie do zmiennej
$removeDigits = function($s){
	return preg_replace('/\d/', '', $s);
};

//przekazanie jako argument
$words = array_map($removeDigits, ['śaJ123', '32i1', 'aisogłaM321']);
```

Jest to mały, prosty kawałek kodu zrealizowany w sposób funkcyjny: mamy dane na wejściu w postaci tablicy stringów, a następnie jest wywołana funkcja, która korzystając z innego parametru (funkcji `$removeDigits`), wywołuje operacje na danych wejściowych. Kod ten oczywiście można zrealizować w sposób strukturalny:

```php
$words = array();
foreach(['śaJ123', '32i1', 'aisogłaM321'] as $word) {
	$words[] = preg_replace('/\d/', '', $word);
}
```

Obiektowo również:

```php
class Word {
	private $word;
//... 50 lini później

$firstChars = $stringCleaner->clean([new Word('śaJ123'), new Word('32i1'), new Word('aisogłaM321')]);
```
Jak widać, obiekty nie wszędzie pasują... ;)

### Operacje na funkcjach

W powyższym przykładzie nie pokazałem w jaki sposób można operować na funkcjach - trzeba nadrobić zaległości. Wyobraźmy sobie, że nie dość że chcemy usunąć cyfry, to jeszcze chcemy odwrócić każdy wyraz (za pomocą np. `strrev`).

```php
$removeDigits = function($s){
	//dodane strrev
	return strrev(preg_replace('/\d/', '', $s));
};

$words = array_map($removeDigits, ['śaJ123', '32i1', 'aisogłaM321']);
```

Ok, nazwa `$removeDigits` już nie jest aktualna, nowa nazwa będzie wymagała zastosowanie operatora logicznego `&&` - to jest zapach że funkcja ma więcej niż jedną odpowiedzialność.

Jak przyjrzymy się bliżej i jeśli uważaliśmy na analizie matematycznej, to zauważymy operację zwaną **składaniem funkcji**.

#### Składanie funkcji - compose

```
f ○ g = f(g(x))
```

Argumentami operatora `○` są funkcje, na wyjściu również jest funkcja.

W PHP operator składania funkcji możemy sobie zamodelować jako funkcja `compose`.

```php
function compose($f, $g){
	return function($x) use($f, $g){
    	return $f($g($x));
    };
}
```

Zrefaktoryzowany kod korzystający ze składania:

```php
$removeDigits = function($s){
	return preg_replace('/\d/', '', $s);
};

$words = array_map(compose('strrev', $removeDigits), ['śaJ123', '32i1', 'aisogłaM321']);
```

Na nowo `$removeDigits` faktycznie robi to co mówi że robi ;) Jakby implementacja `compose` nie była tak naiwna, to można by było składać więcej niż 2 funkcje.

Bardzo dobrą implementację funkcji compose dostarcza biblioteka [igorw/compose][6].

#### Curry

Mnie osobiście nie podoba się zapis funkcji `$removeDigits`, dobrze by było, zamiast korzystać z `Closure`, zapisać to bardziej zwięźne. Na szczęście w programowaniu funkcyjnym istnieje operacja **curry**. Nie będę pisał co ona robi, bo przykład powienien powiedzieć to za mnie.

```php
$words = array_map(
	compose('strrev', curry('preg_replace', '/\d/', '')),
    ['śaJ123', '32i1', 'aisogłaM321']
);
```

`curry('preg_replace', '/\d/', '')` tworzy nową funkcje na podstawie `preg_replace` dostarczając 2 pierwsze argumenty dla tej funkcji. Dzięki temu utworzyliśmy funkcję z tylko jednym argumentem.

Prosta definicja funkcji `curry`:

```php
function curry() {
    $args = func_get_args();
    $func = array_shift($args);

    return function() use($func, $args){
        return call_user_func_array(
        	$func, 
            array_merge($args, func_get_args())
        );
    };
}
```

`Curry` jest często nazywane tworzeniem funkcji częściowych (partial function application). Taką też nomenklaturę przyjęła bardzo dobra biblioteka do `curry` w php: [reactphp/partial][7].

Są jeszcze inne operacje na funkcjach, np. `lift` która "podnosi" typy argumentów i wartości zwracanej. W PHP ciężko byłoby napisać generyczną funkcję `lift`, musiałbym zaprząc inny język, tak więc przykład sobie odpuszczę ;)

Ok, możemy przypisać funkcje do zmiennej, przekazać ją jako parametr i wywoływać na niej operacje. Ale gdzie jest mieso? Mięsem są prymitywy funkcyjne.

### Prymitywy funkcyjne

PHP sam w sobie ma trochę prymitywów funkcyjnych: `array_map`, `array_filter`, `array_reduce`, czy `array_sum`. Jednak PHP jest znane z kwiatków w bibliotece standardowej, nie inaczej jest i tym razem. Wystarczy popatrzeć na parametry `array_map` oraz `array_filter`:

```php
array_map($callback, $array1 /*$array1..*/){}
array_filter($array, $callback){}
```

Tak, parametry w tych dwóch funkcjach są odwrotnie... Na szczęście istnieje świetna biblioteka z prymitywami funkcyjnymi: [functional-php][1] - zawiera ona ok. 40 funkcji.

Obliczmy procent kobiet wśród pacjentów za pomocą [functional-php][1]:

```php
use Functional as F;

$patients = array(/*...*/);

list($femalesCount, $malesCount) = F\map(
    F\partition(
        $patients,
        function(Patient $patient) {
            return $patient->sex === 'female';
        }
    ),
    function($patients){
        return count($patients);
    }
);

$percent = $femalesCount / ($femalesCount + $malesCount) * 100;
```

Funkcja `partition` dzieli tablicę na dwie ze względu na predykat. W tym przykładzie pacjenci zostali podzieleni na kobiety i mężczyzn. Problemem w takim podejściu jest to, że w bardziej skomplikowanych przypadkach kod nie jest czytelny, jest wiele zagnieżdżonych funkcji, flow nie jest oczywiste. Każda kolejna operacja wymaga dodania kolejnego zagnieżdżenia funkcji, np. aby obliczyć procent kobiet wśród pacjentów pogrupowanych po grupie krwi, kod by wyglądał mniej więcej tak:

*przykład **e01** - wykorzystywany w dalszej części artykułu*
```php
$result = F\map(
    F\group(
        $patients,
        function(Patient $patient){
            return $patient->bloodType;
        }
    ),
    function(array $patientsByBloodType){
        list($femalesCount, $malesCount) = F\map(
            F\partition(
                $patientsByBloodType,
                function(Patient $patient) {
                    return $patient->sex === 'female';
                }
            ),
            function($patients){
                return count($patients);
            }
        );

        return $femalesCount / 
        	($femalesCount + $malesCount) * 100;
    }
);
```

Problem ten jest podobny do [callback hell][2] w `JavaScripcie`, dla którego rozwiązanie to wzorzec [Promise][3]. `Promise` spłaszcza zagnieżdżone wywołania do postaci wywołań sekwencyjnych. W przypadku programowania funkcyjnego również istnieje rozwiązanie tego problemu - nawet nie jedno.

### Rfp - real functional php

[functional-php][1], mimo iż jest naprawdę dobrą biblioteką, byłaby znacznie lepsza gdyby:

* miała funkcję `compose` i `pipe` - czyli składanie funkcji
* kolekcja byłaby nie pierwszym, a ostatnim argumentem poszczególnych funkcji
* wszystkie funkcje wspierałyby `autocurring`
* istniałoby więcej prymitywów funkcyjnych

> Zanim objaśnię te warunki, to odsyłam do prototypu [rfp][5] - jest  to biblioteka, która właśnie dodaje te rzeczy do [functional-php][1], poszczególne funkcje [rfp][5] są metodami statycznymi klasy `F`.

Pierwszy warunek jest oczywisty, można łatwo go spełnić swoją implementacją `compose` lub korzystając z [gotowej implementacji][6]. Funkcja `pipe` działa jak `compose`, ale odwraca kolejność funkcji, dzięki czemu kod czyta się od góry do dołu i z lewej do prawej strony. Jednym słowem kod używający `pipe` jest łatwiejszy w czytaniu, chyba że pochodzi się z Chin ;)

Drugi i trzeci warunek wydaje się mało zrozumiały, ale spokojnie - już tłumaczę.

Większość funkcji ma sygnaturę tego typu:

```php
function NAZWA($właściweDaneWejściowe, $pomocniczeDane){}
```

`$właściwymiDanymiWejściowymi` są zazwyczaj tablice/kolekcje lub obiekty. Pomocniczymi danymi wejściowymi zaś są inne funkcje lub inna dowolna wartość, która służy głównej funkcji do wywołania odpowiedniej operacji na danych wejściowych. Może to wydawać się trochę zagmatwane, ale spokojnie, już daję mały przykład:

```php
function map($collection, $callback){}
```

Właściwą daną wejściową w tym przypadku to `$collection`, pomocniczą `$callback` - to na `$collection` funkcja `map` operuje wykorzystując `$callback`.

Chodzi o to, aby właściwe dane wejściowe były ostatnim parametrem funkcji. Funkcja map powinna więc wyglądać tak:

```php
function map($callback, $collection){}
```

Dlaczego lepiej jest, aby `$collection` było ostatnim parametrem? Z powodu `autocurry`.

Przejdźmy płynnie do trzeciego warunku, czyli `autocurring` - jak to powinno działać? `Autocurring` polega na tym, że funkcja, jeśli otrzyma mniejszą liczbę parametrów niż oczekuje, to wykona operację `curry` na samej sobie - co to jest `curry` już wcześniej pokazywałem. Jeśli funkcja otrzyma wystarczającą liczbę parametrów, to zamiast `curry`, wykona swoją właściwą operację.

Przykład:
```php
$extractName = F::map(function($obj){
	return $obj['name'];
});
//podano tylko 1 argument, więc funkcja wywołała "curry"
//na samej sobie i utworzy nową funkcję

//$extractName to funkcja przyjmująca kolekcję jako 1 parametr
//i zamieniająca każdy obiekt tej kolekcji na nazwę

$extractName([['name' => 'Antek'], ['name' => 'Józek']]);
//['Antek', 'Józek']

//można również w klasyczny sposób wywołać pełną 
//wersję funkcji map, podając wszystie parametry
F::map(function($obj){ ... }, [ ... ]);
```

Dzięki jednemu z nowych prymitywów funkcyjnych, o których mowa w warunku czwartym, można ten kod skrócić:

```php
	$extractName = F::map(F::prop('name'));
```

`Autocurry` służy nam do budowania sparametryzowanych funkcji, które  nie mają określonych właściwych danych wejściowych. Dlatego  przenieśliśmy `$collection` - aby można było sparametryzować np. funkcję `F::map` funkcją transformującą dane nie przekazując tablicy. Funkcja `F::prop($prop, $object)` również ma odwróconą kolejność parametrów, to `$object` jest naszą właściwą daną, więc jest na końcu.

Ok, fajnie, możemy sobie tworzyć funkcje za pomocą `compose` i `autocurring`, ale co nam to daje? Daje nam to, że możemy tworzyć krótszy, bardziej deklaratywny, pozbawiony szumu składniowego kod, który jest złożony z prostych klocków.

To pierwszy przykład **functional php** zaadaptowany do **rfp**

```php
list($femalesCount, $malesCount) = 
	F::pipe(
        F::partition(
        	F::pipe(F::prop('sex'), F::eq('female'))
        ),
        F::map(F::unary('count'))
    )->apply($patients);

$percent = $femalesCount / ($femalesCount + $malesCount) * 100;
```

`Autocurring` daje nam możliwość tworzenia nowych funkcji w locie oraz przekazywania **funkcji jako wartości** - co nie jest w pełni wspierane w php. Funkcja `F::pipe` daje nam również możliwość tworzenia nowych funkcji z bonusami w postaci spłaszczenia struktury wywołań i zwiększenia czytelności kodu.

```php
//zamiast
F::map(F::unary('count'), F::partition(
	F::pipe(F::prop('sex'), F::eq('female')),
    [ /* dane wejściowe */ ]
));

//mamy

F::pipe(
	F::partition(
        F::pipe(F::prop('sex'), F::eq('female'))
	),
	F::map(F::unary('count'))
)->apply(/* dane wejściowe */);
```

Pokażę ten bardziej skomplikowany przykład z **functional php**, przy którym pisałem o problemach związanych z dużym zagnieżdżeniem funkcji. Przepiszę go na **rfp**.

*przykład e01 zaimplementowany za pomocą rfp*
```php
$percent = function($a, $b){
	return $a / ($a+$b) * 100;
};

$result = F::pipe(
    F::group(F::prop('bloodType')),
	F::map(F::pipe(
        F::partition(
            F::pipe(F::prop('sex'), F::eq('female'))
        ),
        F::map(F::unary('count')), 
        F::apply($percent)
    ))
)->apply($patients);
```

Już nie wygląda tak źle ;) Funkcja `F::apply` działa podobnie jak `call_user_func_array`, czyli przekazuje każdy element tablicy jako kolejny argument funkcji. Funkcja `F::unary` opakowuje funkcję podaną jako argument i tworzy z niej funkcję jednoargumentową. Taki zabieg w tym przypadku jest konieczny, gdyż funkcja `F::map` przekazuje do funkcji mapującej dwa argumenty: wartość oraz indeks elementu. Drugim argumentem funkcji `count` jest `$mode`, nie chcemy aby indeks elementu był przekazywany jako tryb zliczania ;)

Warto zauważyć, że kod napisany za pomocą **rfp** wygląda tak, że najpierw tworzymy funkcją z prymitywów funkcyjnych, a dopiero na końcu wywołujemy tą funkcję dostarczając dane wejściowe. Dzięki temu kod nie jest zanieczyszczony operowaniem na danych wejściowych, a jedynie określa jakie operacje zostaną wykonane. Tym właśnie jest programowanie funkcyjne.

Trzeba pamiętać, że **rfp** to tylko działający prototyp biblioteki, w którym testy i kwestie wydajności zostały pominięte. Służy ona tylko i wyłącznie do tego, aby pokazać kształt i ideę potencjalnej biblioteki, która w lepszy sposób wspiera programowanie funkcyjne. Być może stabilna biblioteka tego rodzaju powstanie. Idea **rfp** została zaczerpnięta z doskonałej biblioteki js [ramda][4].

### FluentTraversable

Inną biblioteką, która wspiera programowanie funkcyjne to [FluentTraversable](8). Biblioteka ta prezentuję trochę inne podejście do programowania funkcyjnego, gdyż dostarcza primitywy w formie [fluent interface](9). Ostatni przykład kodu, który był zaimplementowany najpierw za pomocą functional-php, a następnie rfp:

*przykład e01 zaimplementowany za pomocą FluentTraversable*
```php
    $patients = array(...);

    $info = FluentTraversable::from($patients)
        ->groupBy(get::value('bloodType'))
        ->map(
            FluentComposer::forArray()
                ->partition(is::eq('sex', 'female'))
                ->map(func::fix('count'))
                ->collect(function($elements){
                    list($femalesCount, $malesCount) = $elements;
                    return $femalesCount / 
                    	($femalesCount + $malesCount) * 100;
                })
        )
        ->toMap();
```

FluentTraversable w przeciwieństwie do rfp nie jest w wersji eksperymentalnej i jest gotowa do użytku.

### Podsumowanie

Programowanie funkcyjne całkowicie różni się od programowania obiektowego. Obydwa te paradygmaty mogą i powinny być łączone - OOP do budowania API, programowanie funkcyjne zaś w implementacji metod. Programowanie funkcyjne warto poznać, bo jest to kolejne dobre narzędzie w naszej skrzynce - im więcej takich narzędzi, tym lepiej. Na początku prawdopodobnie będzie ciężko znaleźć zastosowania dla tego podejścia, ale wraz z praktyką, przyjdzie moment w którym będzie to naturalnie. Tak jak wielu strukturalnym programistom php zrozumienie istoty programowania obiektowego zajmuje trochę czasu, tak tutaj może być podobnie. Jakby ktoś miał jeszcze wątpliwosci: tak, w php da się pisać funkcyjnie.

Myślę, że jeszcze jakiś tekst z pogranicza programowania funkcyjnego spłodzę, więc miejcie się na baczności.

Funkcje\*\* są fajne, bo lepiej opisują algorytmy.
Funkcje są fajne, bo są deklaratywne.
Funkcje są fajne przede wszystkim dlatego, bo drastycznie redukują liczbę `ifów` i pętli w kodzie, co sprawia że kod jest bardziej zwięzły i nie ma jawnych rozgałęzień, przez co łatwiej się go czyta.

\* Skrót myślowy od programowania obiektowego

\*\* Skrót myślowy od programowania funkcyjnego

### Zbiór linków

* [functional-php][1]
* [rfp][5]
* [FluentTraversable][8]
* [implementacja funkcji compose w php][6]
* [implementacja curry w php][7]
* [ramda - biblioteka w js, która zainspirowała rfp][4]
* [link do prezentacji o tym, dlaczego underscore.js nie wspiera w pełni programowania funkcyjnego][10]
* [callback hell w js][2]
* [q - implementacja Promise w js][3]
* [fluent interface - wiki][9]


[1]: https://github.com/lstrojny/functional-php
[2]: http://callbackhell.com/
[3]: https://github.com/kriskowal/q
[4]: https://github.com/CrossEye/ramda
[5]: https://github.com/psliwa/rfp
[6]: https://github.com/igorw/compose
[7]: https://github.com/reactphp/partial
[8]: https://github.com/psliwa/fluent-traversable
[9]: http://en.wikipedia.org/wiki/Fluent_interface#PHP
[10]: https://www.youtube.com/watch?v=m3svKOdZijA