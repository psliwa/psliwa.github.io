---
layout: post
title: "Jeden TestCase dla wielu testowanych klas"
author: "psliwa"
tags: [php, testy, phpunit]
---

Na początku nakreślę problem. Mamy klasę `Product` oraz interfejs `ProductRepository`.

```php
interface ProductRepository {
    function save(Product $product);
    function delete(Product $product);
    function findOne($id);
    function findAll();
    //...
}
```

`ProductRepository` może mieć kilka implementacji, np.:

* `DoctrineProductRepository` - zapisywanie obiektów do bazy danych z wykorzystaniem `EntityManagera` z Doctrine, nasza "rzeczywista" implementacja używana przez aplikację
* `InMemoryProductRepository` - trzymanie obiektów w pamięci, klasa wykorzystywana np. do testów funkcjonalnych kontrolerów lub serwisów - nie chcemy angażować bazy danych aby testy były szybsze, prostsze do konfiguracji itp.

Obie implementacje muszą działać tak samo, tak więc powinny mieć taki sam zestaw testów.

### Rozwiązanie problemu

Jednym z rozwiązań jest napisanie abstrakcyjnej klasy testu, testowany obiekt oraz tworzenie danych testowych są w formie metod abstrakcyjnych.

```php
abstract class ProductRepositoryTest extends PHPUnit_Framework_TestCase {
    protected $repository;
    
    protected function setUp() {
        $this->repository = $this->createSUT();
    }
    
    //tworzenie obiektu testowanego
    abstract protected function createSUT();
    
    //umieszczanie Produktu w źródle danych    
    abstract protected function insertProduct(Product $product);
    
    /**
     * @test
     */
    public function givenExistingProduct_whenItIsDeleted_thenProductCouldNotBeFound() {    
        //given        
        $product = new Product(...);
        $this->insertProduct($product);
        
        //when        
        $this->repository->delete($product);
        $actualProduct = $this->repository->findOne($product->getId());
        
        //then        
        $this->assertNull($actualProduct);
    }
    
    //...
}
```

W konkretnych podklasach nadpisujemy metody `createSUT` i `insertProduct`.

Jest jednak problem... Klasa testu operującego na rzeczywistej bazie danych niekoniecznie rozszerza bezpośrednio `PHPUnit_Framework_TestCase`. Klasa bazowa dla testów na bazie dostarcza tworzenie połączenia, obtacza testy w transakcję która na końcu jest rollbackowana itp. Tak więc występuje potrzeba wielodziedziczenia, `DoctrineProductRepositoryTest` powinien rozszerzać zarówno `ProductRepositoryTest` oraz `DoctrineTestCase`. Nie jest to możliwe, chyba że zamienimy `ProductRepositoryTest` w trait.

```php
/**
 * @mixin \PHPUnit_Framework_TestCase
 */
trait ProductRepositoryTest {
    //to samo co w klasie ProductRepositoryTest
    //w poprzednim listingu
}
```

Adnotacja `@mixin` jest interpretowana przez PhpStorm tak, jakby `ProductRepositoryTest` "rozszerzał" `PHPUnit_Framework_TestCase`, dzięki czemu poprawnie działa podpowiadanie metod (np. asercji).

Możemy zatem wykorzystać trait do stworzenia testu dla `DoctrineProductRepository`.

```php
class DoctrineProductRepositoryTest extends DoctrineTestCase {
    use ProductRepositoryTest;
    
    protected function createSUT() {
        return new DoctrineProductRepository($this->em);
    }
    
    protected function insertProduct(Product $product) {
        $this->em->persist($product);
        $this->em->flush();
    }
}

```

Podobnie by wyglądał `InMemoryProductRepositoryTest`. Gdy dopisujemy jakiś test do `ProductRepositoryTest`, automatycznie będzie on wykonywany dla dwóch różnych implementacji `ProductRepository`. Minusem tego rozwiązania może być zaburzenie pętli TDD. Pisząc jeden test tak naprawdę mamy w rezultacie dwa czerwone testy - czyli w następnym kroku musimy zaimplementować dwie funkcjonalności dla dwóch całkowicie różnych klas, aby na nowo mieć przed oczami tylko zieleń. Pętla "Red -> Green -> Refactor" zamienia się w "Red x2 -> Green, Red -> Green x 2 -> Refactor x 2". Można temu przeciwdziałać wstawiając test najpierw do `DoctrineProductRepositoryTest`, a po zaimplementowaniu funkcjonalności i refaktoryzacji, przenieść go do `ProductRepositoryTest`.

Mając na początku interfejs `ProductRepository` i pisząc pierwszą implementację (np. `DoctrineProductRepository`), nie powinniśmy zaczynać od razu od tworzenia cechy/klasy abstrakcyjnej `ProductRepositoryTest`, gdyż tak naprawdę nie wiemy czy ona będzie potrzebna. Metody testowe należy umieszczać bezpośrednio w `DoctrineProductRepositoryTest`, a gdy będziemy mieć drugą implementację `ProductRepository`, powinniśmy dokonać refaktoryzacji wydzielenia cechy (trait) bądź klasy.

Bonusem tego rozwiązania jest to, że `ProductRepositoryTest` zawiera tylko czyste testy bez szczegółów tworzenia SUT i środowiska testowego, dzięki temu zyskujemy na czytelności.

### (Wielo)dziedziczenie, a kompozycja

Nie jestem zwolennikiem dziedziczenia jako mechanizmu składającego funkcjonalności, gdyż więcej i czyściej można zrobić stosując kompozycję obiektów. W tym przypadku, czysto teoretycznie, również można rozwiązać ten problem kompozycją.

```php
class ProductRepositoryTest extends PHPUnit_Framework_TestCase {
    private $repository;
    private $productFactory;
    
    public function __construct(
        ProductRepository $productRepository, 
        ProductFactory $productFactory
    ) {
        //...
    }
    
    //testy
}

interface ProductFactory {
    function insertProduct(Product $product);
}
```

Mamy klasę testu, którą możemy parametryzować konkretnymi implementacjami `ProductRepository` oraz `ProductFactory` (tworzy dane testowe). Problem otaczania testów transakcją można rozwiązać za pomocą listenera, a całość sklejać za pomocą kontenera wstrzykiwania zależności. Jednak PHPUnit takiego rozwiązania nie obsługuje natywnie, trzeba by było mocno grzebać i napisać swoje rozszerzenia. Czy warto? Wg mnie nie. Przewaga kompozycji nad (wielo)dziedziczeniem jest bezdyskusyjna, ale w przypadku testów nie ma większych różnic. 

Kompozycję stosujemy m. in. dlatego aby mieć mniejsze, łatwiej testowalne, spójne, z jedną odpowiedzialnością klocki, z których budujemy bardziej złożone struktury wykorzystując składanie obiektów, a nie dziedziczenie (większa elastyczność, brak eksplozji kombinatorycznej bytów itp.). Testów się nie testuje (w sensie pisania testów do testów, można "testować" np. [testami mutacyjnymi][1]), nie ma potrzeby aby mieć możliwość "składania" testów w runtime, może być to osiągnięte dziedziczeniem. Fabryki danych testowych faktycznie mogą być wydzielone do osobnych, spójnych klas (aby można było je używać w wielu klasach testowych), ale tworzenie obiektów fabryk może być zahardcodowane np. w metodzie `setUp`.

### Podsumowanie

Pisząc testy zaślepiamy nieporęczny obiekt Stubem/Fakem - na początku z zaślepioną tylko jedną metodą która jest nam akurat potrzebna. Często bywa tak, że po jakimś czasie ten Stub/Fake staje się kompletną implementacją, gdyż wraz powstawaniem kolejnych testów, każda lub większość z metod Stuba została poprawnie zaimplementowania. Wtedy należy się zastanowić, czy nie należy pokryć tego Stuba testami, gdyż od poprawności jego działania zależy, czy testy które z niego korzystają są poprawne. Test bezgranicznie ufa Stubowi, my nie powinniśmy - więc jeśli zawiera on jakąś logikę, powinno się go również testować. Jak? Np. technikami które przedstawiono wyżej.

Jeden TestCase dla wielu testowanych klas nie ogranicza się tylko do przypadku stubów testowych, czy zaślepianiu baz danych. Ten przypadek można uogólnić do sytuacji, gdzie mamy kilka implementacji tego samego interfejsu.

[1]: http://pl.wikipedia.org/wiki/Testowanie_mutacyjne