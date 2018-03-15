---
layout: post
title: "Przetrwać chaos - ChaosJournal w Akka Persistence"
author: "psliwa"
tags: [scala, testy, akka, chaos, fault tolerant]
---

Przeglądając kod źródłowy akka-persistence, natknąłem się na bardzo ciekawy sposób testowania odporności na błędy [persystencji][1]. W tym teście użyty jest [ChaosJournal][2] - implementacja journala losowo rzucająca wyjątkami przy odczycie lub zapisie danych. "*Właśnie tego szukałem!*" - pomyślałem i zacząłem się bawić nową zabawką.

Nie bez przyczyny zacząłem owe poszukiwania - ostatnio pracowałem nad poprawieniem odporności naszej aplikacji na błędy persystencji - niektóre typy wiadomości nie mogą być gubione, przez co trzeba było wprowadzić  `AtLeastOnceDelivery`, idempotentne reagowanie na zduplikowane wiadomości i inne bajery. Przyszedł czas na przetestowanie szeregu wprowadzonych zmian. Wybór padł na wykorzystanie istniejących już testów E2E z `ChaosJournalem`. Po zwiększeniu `akka.test.timefactor`, przekonfigurowaniu circuit breakerów, backoff supervisorów, przeróżnych timeoutów, przekonfigurowaniu `ChaosJournala`, kilku zmianach w testach E2E i poprawieniu kilku znalezionych błędów, testy zaczęły stabilnie przechodzić.

Trzeba było wprowadzić kilka zmian w testach E2E aby współpracowały one z `ChaosJournalem`, m. in.:

* wiadomości, które mogą być zgubione, muszą być periodycznie powtarzane, a logika aplikacji powinna oczywiście obsługiwać ich duplikaty
* instruując aplikację poprzez wysłanie wiadomości przez api, należy powtarzać wysłanie gdy dostanie się błąd `50x` (błąd rzucany w przypadku błędu persystencji)

Innym pomysłem na testowanie Akkowej aplikacji pod kątem odporności na gubienie wiadomości jest customowa implementacja Mailboxa - gubienie określonego procenta wiadomości. Zaleta jest taka, że tutaj możemy skonfigurować Mailboxa per aktor i przetestować jak aplikacja się zachowuje gdy jeden konkretny aktor gubi wiadomości.

Potężniejszym, bardziej generycznym, ale i trudniejszym do zaimplementowania rozwiązaniem jest użycie np. [docker-compose-testkit][3]. Minusem tego rozwiązania jest to, iż trzeba pisać nowy rodzaj testów specjalnie pod kątem odporności na błędy, a sam sposób pisania takich testów nie przemawia do mnie - wygląda na zbyt złożony. W rozwiązaniu z `ChaosJournalem` / `ChaosMailboxem` nic nie stoi na przeszkodzie, aby wykorzystać już istniejące i działające testy integracyjne lub e2e (lekki lifting może być wskazany) - dzięki temu jest mniej kodu. A jak dobrze wiemy, najlepszy i najczystszy kod to ten, który nie istnieje. Prezentację na temat `docker-compose-testkit` oraz o testowaniu odporności na błędy możecie znaleźć na [githubie][4].

Za niedługo prawdopodobnie zrobię PR do Akka Persistence, który udostępnia `ChaosJournal` jako plugin do persystencji. Jak na razie klasa ta jest tylko w źródłach testowych, można jej użyć stosując znany i lubiany wzorzec projektowy o nazwie "copy & paste". Należy pamiętać, że domyślna implementacja `ChaosJournala` ma globalny storage, więc każda instancja tej klasy współdzieli ten sam stan. Jeśli nie tego oczekujemy, można to zachowanie łatwo zmienić. Innym problemem jest to, iż ta implementacja nie wspiera [Persistence Query][5].

[1]: https://github.com/akka/akka/blob/master/akka-persistence/src/test/scala/akka/persistence/AtLeastOnceDeliveryFailureSpec.scala
[2]: https://github.com/akka/akka/blob/master/akka-persistence/src/test/scala/akka/persistence/journal/chaos/ChaosJournal.scala
[3]: https://github.com/carlpulley/docker-compose-testkit
[4]: https://github.com/carlpulley/docker-compose-testkit/tree/scala-sphere-2017/docs/Scala%20Sphere%202017
[5]: http://doc.akka.io/docs/akka/current/scala/persistence-query.html