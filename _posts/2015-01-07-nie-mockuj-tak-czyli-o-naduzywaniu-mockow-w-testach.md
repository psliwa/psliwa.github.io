---
layout: post
title: "Nie Mockuj tak! Czyli o (nad)używaniu Mocków w testach..."
author: "psliwa"
tags: [php, testy, phpunit]
---

Na wstępie: fajnie by było, abyś wiedział mniej więcej co to jest Mock, Stub i Fake - nie będę tego jakoś szczególnie objaśniał bo idea tego wpisu jest inna niż wstęp do "zaślepek". [Tutaj][1] możesz poczytać o różnych zaślepkach na przykładzie PHPUnit.

Spis treści, a jak! <a name="toc"></a>

1. [Wstępniak](#wstepniak)
1. [Kiedy Mock?](#kiedy)
    1. [Przykład zaślepiania metody typu Query](#query)
        1. [Zaślepienie metody Query za pomocą Mocka](#query-mock)
        1. [Zaślepienie metody Query za pomocą Stuba](#query-stub)
        1. [Zaślepienie metody Query za pomocą Fake](#query-fake)
        1. [Przemyślenia na temat zaślepiania metod typu Query](#query-summary)
    1. [Przykład zaślepiania metody typu Command](#command)
        1. [Zaślepienie metody Command za pomocą Stuba](#command-stub)
        1. [Zaślepienie metody Command za pomocą Mocka](#command-mock)
        1. [Zaślepienie metody Command za pomocą Fake](#command-fake)
        1. [Przemyślenia na temat zaślepiania metod typu Command](#command-summary)
1. [Końcowe przemyślenia](#summary)
        
### Wstępniak <small>[do góry](#toc)</small><a name="wstepniak"></a>

Dla przypomnienia, zaślepka (z ang. Test Double) to obiekt, który jest przekazywany do obiektu testowanego zamiast obiektu będącego rzeczywistą zależnością w kodzie produkcyjnym. Przykładem może być wstrzyknięcie zaślepki obiektu `ProductRepository` (abstrakcji na persystentną kolekcję produktów) do jakieś usługi, np. `ProductService`.

Aby utworzyć obiekt `ProductService` potrzeba `ProductRepository`, aby utworzyć `ProductRepository` musimy utworzyć `EntityManagera` (zakładam implementację opartą na Doctrine), aby utworzyć `EntityManagera` trzeba utworzyć konfigurację oraz połączenie z bazą danych, aby połaczyć się z bazą danych musimy mieć z czym się połaczyć, więc serwer bazy danych musi być zainstalowany, schema bazy powinno być utworzone itp. Aby utworzyć obiekt konfiguracji, trzeba utworzyć obiekt cache... Na litość Boską, ja tylko chcę przetestować czy `ProductService::createProduct($name, $price)` tworzy obiekt produktu, zapisuje go w repozytorium i triggeruje przy tym odpowiedni event...

Powyższa historyjka pokazuje jedną z przyczyn stosowania zaślepek. Jest ich więcej, np:

* Uproszczenie setUp testu
* Testy w izolacji (tzw. testy jednostkowe), nie chcemy drugi raz testować `DoctrineProductRepository` - ta klasa ma osobne testy!
* Zwiększenie szybkości wykonywania testów - zewnętrzny web service zastępujemy lokalnym Stubem/Fake, repozytorium operujące na bazie zastępujemy implementacją in memory
* Możemy w prosty sposób przetestować sytuacje brzegowe, które ciężko zreprodukować wykorzystując rzeczywisty obiekt
* Podczas pisania implementacji klasy X, która zależy od typu Y, nie mamy jeszcze implementacji Y

Oczywiście są sytuacje, w których nie należy stosować zaślepek. Należy pamiętać, że nie należy zamieniać części obiektu/podsystemu który testujemy. Np. nie powinno się zastępować `EntityManagera` implementacją "in memory" w testach `DoctrineProductRepository`, gdyż testy te mają testować, czy poprawnie szukamy / zapisujemy dane w bazie.

### Kiedy Mock? <small>[do góry](#toc)</small><a name="kiedy"></a>

W idealnym świecie, zgodnie z zasadą **Command-Query separation**, każda metoda powinna być typu **Command** lub **Query**.

* **Command** - zmienia stan obiektu/systemu nie zwracając żadnej wartości - najprostszy przykład to setter ;)
* **Query** - pobranie stanu obiektu/systemu nie zmieniając stanu - najprostszy przykład to getter

W złym guście jest tworzenie metod które zarówno są Command i Query, bo taka funkcja robi de facto dwie rzeczy - zmienia stan i go zwraca. Nie można wywołać 2x taką metodę aby pobrać stan, gdyż za każdym wywołaniem ten stan jest zmieniany.

Związek typu metod z zaślepkami:

1. Mocki **można** stosować do zaślepiania metod typu **Command**, ale **nie powinno się** stosować do zaślepiania metod typu **Query** (są wyjątki)
1. Stuby **można** stosować do zaślepiania metod typu **Query**, ale **nie należy** stosować do zaślepiania metod typu **Command**
1. Fake **można** stosować do zaślepiania metod typu **Query** i **Command**

> **Krótkie wyjaśnienie** aby nie było zamętu: 

> Można utworzyć Stuba za pomocą frameworka do tworzenia Mocków. To brzmi dziwnie, ale jest to jak najbardziej stosowana i poprawna praktyka. [Tutaj][1] są przykłady tworzenia różnych zaślepek za pomocą frameworku do mockowania, który jest wykorzystany w PHPUnit. To czy obiekt jest mockiem, nie jest definiowane przez fakt, że został utworzony przez metodę `getMock` (w PHPUnit), ale przez to, że nałożono na ten obiekt dodatkowe oczekiwania, które są później jawnie bądź niejawnie weryfikowane.

#### Przykład zaślepiania metody typu Query <small>[do góry](#toc)</small><a name="query"></a>

Przykład kodu do którego chcemy napisać test:

```php
//interfejs translatora
interface Translator {
    function trans($id, array $params = [], $locale = null);
}

class UserService {
    private $translator;
    //...
    
    //trywialna metoda którą chcemy przetestować
    function createInvitation(User $user) {
        return new Invitation(
            $user, 
            $this->translator->trans(
                'Hello %user%', 
                ['%user%' => (string) $user]
            ), 
            ...
        );
    }
}

```

Translator ma metodą `trans`, która jest typu **Query** - tak więc nie powinniśmy stosować Mocka, ale dla celów naukowych spróbujmy...

##### Zaślepienie metody Query za pomocą Mocka <small>[do góry](#toc)</small><a name="query-mock"></a>

Uwaga, bardzo **ZŁY** test! Nie rób tego w domu (w pracy też)!
```php
//given

$user = ...;
$translator = $this->getMock('Translator');
$service = new UserService(..., $translator);

$translator->expects($this->once())
    ->method('trans')
    ->with('Hello %user%', ['%user%' => (string) $user ])
    ->willReturn('Hello Peter');
    
//when

$invitation = $service->createInvitation($user);

//then
//... asercje
```

**Co zyskaliśmy**:

* Mamy test, który faktycznie testuje

**Problemy**:

* Część kodu produkcyjnego przesiąkło do testu (argumenty metody `with`)
* Test zna implementację, gdy implementacja się zmieni, trzeba będzie zmienić test - tak więc test jest bardzo kruchy i ciężki w utrzymaniu
* Jest dłuższy od implementacji i niezbyt czytelny
* Gdy będzie więcej metod do zaślepienia, to test będzie jeszcze mniej czytelny - wystąpi "eksplozja oczekiwań"

##### Zaślepienie metody Query za pomocą Stuba <small>[do góry](#toc)</small><a name="query-stub"></a>

Lepsze rozwiązanie od poprzedniego, wg magicznej listy Stuba można wykorzystać do zaślepienia metod typu Query.
```php
//given

$user = ...;
$translator = $this->getMock('Translator');
$service = new UserService(..., $translator);

//usuneliśmy $this->once() i wywołanie "with" - to były oczekiwania,
//a chcemy stworzyć Stuba - $translator mimo że powstał za pomocą
//metody "getMock" nie jest Mockiem
$translator->expects($this->any())
    ->method('trans')
    ->willReturn('Hello Peter');
    
//when

$invitation = $service->createInvitation($user);

//then
//... asercje
```

**Co zyskaliśmy**:

1. Brak kodu produkcyjnego w teście -> test jest mniej kruchy
1. Słabe (ale nie brak) powiązanie kodu testu z implementacją -> test jest jeszcze mniej kruchy

**Problemy**:

1. Test jest zaśmiecony tworzeniem stuba
1. Jakby było więcej metod do zaślepienia, to test byłby jeszcze bardziej zaśmiecony
1. Nie testujemy argumentów przekazywanych do metody `trans` - słabsze pokrycie kodu testami (nie w sensie [Code Coverage][2], który tak naprawdę nie jest dobrą metryką oceny jakości testów, ale w sensie [testów mutacyjnych][3])

##### Zaślepienie metody Query za pomocą Fake <small>[do góry](#toc)</small><a name="query-fake"></a>

```php
//kod Fake
class FakeTranslator implements Translator {
    function trans($id, array $params = [], $locale = null) {
        return strtr($id, $params);
    }
}
//...

//given

$user = ...;
$service = new UserService(..., new FakeTranslator());
    
//when

$invitation = $service->createInvitation($user);

//then
//... asercje
```

**Co zyskaliśmy**:

1. Test jest prosty i skalowalny - mniejszy wpływ liczby zaślepianych metod na długość testu
1. Możliwość testowania argumentów, które są przekazywane do metody `trans`
1. Mozliwość zastosowania `FakeTranslator` w wielu różnych testach oraz w różnych klasach testowych.

**Problemy**:

1. Gdy dojdzie metoda do interfejsu `Translator`, trzeba zaktualizować implementację `FakeTranslator`
1. Gdy **Fake** urośnie i jego logika nie będzie trywialna, należy taką klasę również przetestować. Najlepiej dokładnie tymi samymi testami jak rzeczywista implementacja - o tym, jak to zrobić, pisałem w [jednym z poprzednich wpisów][4].

##### Przemyślenia na temat zaślepiania metod typu Query<a  <small>[do góry](#toc)</small><a name="query-summary"></a>

Przy małej liczbie metod do zaślepienia, można zbudować Stuba za pomocą frameworka do Mockowania. Gdy liczba metod jest większa i w kilku klasach testowych wykorzystywany jest podobny stub, powinno się zastanowić czy wprowadzenie "fałszywej" implementacji nie będzie korzystne. Zastosowanie Mocka w tym przypadku jest bardzo słabiutkie, aczkolwiek istnieją przypadki w których zastosowanie Mocka dla metod typu Query jest wskazane - o tym w końcowych przemyśleniach.

#### Przykład zaślepiania metody typu Command <small>[do góry](#toc)</small><a name="command"></a>

Przykład kodu do którego chcemy napisać test:

```php
interface ProductRepository {
    function save(Product $product);
    function findOneById($id);
}
interface EventDispatcher {
    function dispatch($eventName, $event = null);
}

class ProductService {
    private $repository;
    private $eventDispatcher;
    
    //trywialna metoda, którą chcemy przetestować
    function saveProduct(Product $product) {
        $this->repository->save($product);

        $this->eventDispatcher->dispatch(
            'product.saved', 
            new Event([ 'product' => $product ]
        );
    }
}
```

Metody `ProductRepository::save` i `EventDispatcher::dispatch` są typu **Command** (nic nie zwracają, zmieniają stan), więc powinniśmy zastosować Mocki lub ewentualnie Fake. Nie powinno się stosować Stuba, ale dla cełów naukowych to rozwiązanie idzie na pierwszy ogień...

##### Zaślepienie metody Command za pomocą Stuba <small>[do góry](#toc)</small><a name="command-stub"></a>

Uwaga, okropnie **ZŁY** test!

```php
//given

$product = ...;
$repository = $this->getMock('ProductRepository');
$eventDispatcher = $this->getMock('EventDispatcher');
$service = new ProductService($repository, $eventDispatcher);

$repository->expects($this->any())
    ->method('save');

$eventDispatcher->expects($this->any())
    ->method('dispatch');
    
//when

$service->saveProduct($product);

//then
//??? jak zweryfikować taki test? Nie da się ;)
```

**Co zyskaliśmy**:

* Nic oprócz fałszywego przeświadczenia, że mamy test - ten test nic nie testuje

**Problemy**:

* Zbędny kod, który trzeba utrzymywać
* Fałszywe przeświadczenie, że kod jest przetestowany - [Code Coverage][2] będzie 100% ;)

##### Zaślepienie metody Command za pomocą Mocka <small>[do góry](#toc)</small><a name="command-mock"></a>

```php
//given

$product = ...;
$repository = $this->getMock('ProductRepository');
$eventDispatcher = $this->getMock('EventDispatcher');
$service = new ProductService($repository, $eventDispatcher);

$repository->expects($this->once())
    ->method('save')
    ->with($product);

$eventDispatcher->expects($this->once())
    ->method('dispatch')
    ->with('product.saved', new Event([ 'product' => $product ]);
    
//when

$service->saveProduct($product);

//then
//PHPUnit na końcu odpala weryfikacje mocków
```

**Co zyskaliśmy**:

* Testy, które faktycznie całkiem nieźle testują

**Problemy**:

* Testy są związane z implementacją i są trochę przegadane

##### Zaślepienie metody Command za pomocą Fake <small>[do góry](#toc)</small><a name="command-fake"></a>

```php
//implementacja Fakeów
class FakeProductRepository implements ProductRepository {
    private $products = [];
    function save(Product $product) {
        if($product->getId() === null) {
            $product->setId(rand(0, 9999999));
        }
        $this->products[$product->getId()] = $product;
    }
    
    function findOneById($id) {
        if(!isset($this->products[$id])) throw new Exception();
        return $this->products[$id];
    }
}

class FakeEventDispatcher implements EventDispatcher {
    private $dispatchedEvents = [];
    
    function dispatch($name, $event = null) {
        $this->dispatchedEvents[] = [$name, $event];
    }
    
    function getDispatchedEvents() {
        return $this->dispatchedEvents;
    }
}

//... i testy

//given

$product = ...;
$repository = new FakeProductRepository();
$eventDispatcher = new FakeEventDispatcher();
$service = new ProductService($repository, $eventDispatcher);

//when

$service->saveProduct($product);

//then

$this->assertEquals(
    $product, 
    $repository->findOneById($product->getId()
);
$this->assertEquals(
    [[ 'product.saved', new Event(['product' => $product]) ]], 
    $eventDispatcher->getDispatchedEvents()
);
```

**Co zyskaliśmy**:

* kod testu jest stosunkowo prosty i krótki
* Test działa i faktycznie całkiem nieźle testuje
* Test jest w mniejszym stopniu związany z implementacją `ProductService` - np. nie ma wzmianki o metodzie `ProductRepository::save`

**Problemy**:

* Trzeba było dopisać metodę `FakeEventDispatcher::getDispatchedEvents` aby dało się przetestować, czy event miał miejsce
* Trzeba było napisać obiekty Fake (co z ich testami?)
* Druga asercja (sprawdzanie eventu) jest średio czytelna

##### Przemyślenia na temat zaślepiania metod typu Command <small>[do góry](#toc)</small><a name="command-summary"></a>

W tym konkretnym przypadku, dobrym rozwiązaniem byłoby wykorzystanie `FakeProductRepository` oraz mocka `EventDispatcher`, czyli rozwiązanie hybrydowe. Dzięki temu zniwelowane byłyby problemy związane z zastosowaniem tylko Mocków lub tylko Fakeów. O ile zastosowanie Mocków do zaślepiania metod typu Query ma jeszcze jakikolwiek sens (w niektórych przypadkach), to zastosowanie Stubów do zaślepienia metod typu Command już nie.

### Końcowe przemyślenia <small>[do góry](#toc)</small><a name="summary"></a>

Mocki mają zastosowanie głównie dla zaślepiania metod typu Command. Jednakże nie każda metoda Command powinna być z urzędu zaślepiana Mockiem - niekiedy lepszym rozwiązaniem jest Fake. Kiedy lepszy będzie Fake? Gdy w wielu klasach testowych są wykorzystywane podobne Mocki (Fake pozwala zachować zasadę DRY). Fake pozwala również uprościć kod testów i sprawić, że test będzie w mniejszym stopniu powiązany z implementacją testowanej klasy.

Istnieją sytuacje, w których wskazane jest użycie **Mocka** nawet dla metod typu **Query**, np:

* wywołanie przypadków granicznych, np. rzucenie wyjątku
* konieczność przetestowania wpływu wartości zwrotnych z zaślepionej jednej metody na wywołanie kolejnej zaślepionej metody. Np. testowanie obiektu korzystającego z Cache - gdy metoda `Cache::test` zwróci false, ma zostać wywołana metoda `Cache::save`, ale nie powinna być wywołana metoda `Cache::load` - tutaj dobrym wyborem jest Mock, bo inaczej ciężko to przetestować - mimo iż `test` i `load` są typu Query

Gdy doprowadziło się do sytuacji, w której jeden Mock zwraca drugiego, to jest to zapach złego designu (prawdopodobnie złamanie [prawa Demeter][6]) lub ewentualnie źle dobranych rodzajów zaślepek.

Testy powinny w miarę możliwości traktować obiekt testowany jak czarną skrzynkę: 

1. Dajemy jakieś dane na wejście
1. Odpalamy metodę testowaną
1. Nie wiemy co się dzieje w środku obiektu testowanego (test nie zna implementacji i algorytmu)
1. Sprawdzamy rezultat operacji

Test powinien sprawdzać, czy obiekt zrobił to co chcemy, a nie jak to zrobił. Test powinien pozostać nienaruszony i działający, jeśli zmienimy implementacje metody testowanej, zmienimy algorytm, zrefaktoryzujemy kod itp.

Stosowanie Mocków skutecznie uniemożliwia takie podejście, gdyż samo użycie Mocka wiąże test z implementacją obiektu. Dlatego trzeba w pełni świadomie korzystać z Mocków, bo w przeciwnym wypadku testy będą kruche i każda zmiana w kodzie produkcyjnym będzie ciągnąć za sobą zmianę testów. Może to doprowadzić do kuriozalnej sytuacji, w której nie będziemy refaktoryzować, bo wymagałoby to zmiany w testach. Polecam tego [talka][5] - porusza on m. in. problem "zabetonowania" kodu za pomocą złych testów.

[1]: https://phpunit.de/manual/current/en/test-doubles.html
[2]: http://en.wikipedia.org/wiki/Code_coverage
[3]: http://en.wikipedia.org/wiki/Mutation_testing
[4]: http://psliwa.org/jeden-testcase-dla-wielu-testowanych-klas/
[5]: https://www.youtube.com/watch?v=wbAtJlbRhbQ
[6]: http://pl.wikipedia.org/wiki/Prawo_Demeter