---
layout: post
title: "Side effecty, a Mock"
author: "psliwa"
tags: [testy, scala, programowanie funkcyjne, php, java]
---

Jakiś czas temu rozpisywałem się o [mockach][1]. Teraz pora na krótką opowiastkę o tym samym, ale z innej perspektywy.

Przez ostatni rok piszę w scali, nie napisałem w niej ani jednego mocka (nie licząc `TestProbe` z Akki). Niedawno też wróciłem popisać sobie w javie, praktycznie od razu gdy miałem zamiar pisać test jednostkowy, chciałem użyć mocka. Dlaczego? Odpowiedź jest prosta i po części znajduje się w przytoczonym już [wpisie][1]. Scala to funkcyjno-obiektowy język, duży nacisk jest kładziony na **niemodyfikowalność** i **brak efektów ubocznych**, język do tego zachęca. Jeśli nie ma side effectów (wywołań typu **command**), to po co korzystać z mocków? Nie mają racji bytu. Java nie zachęca tak do niemodyfikowalności i braków side effectów, tak więc w niektórych przypadkach mock jest wskazany.

Wspomniałem, że jedyne mocki, które napisałem w scali, to te testujące aktorów z Akki - to jest naturalne. Akka jest oparta na side effectach, każda wysłana wiadomość do dowolnego aktora jest efektem ubocznym.

Nauka z tego jest taka, że zbiór rozwiązań danych problemów z języka X nie przekłada się na zbiór rozwiązań tych problemów w języku Y. Dlaczego? Bo w języku Y te problemy mogą w ogóle nie istnieć lub mogą istnieć inne narzędzia do ich rozwiązania. Przykładowo wzorce obiektowe w programowaniu funkcyjnym oczywiście nie mają zastosowania, do rozwiązania tych problemów stosuje się funkcje wyższego rzędu (np. zamiast strategii), składanie funkcji (np. zamiast dekoratora), czy innych funkcyjnych konstrukcji językowych (np. pattern matching zamiast visitora).

[1]: http://psliwa.org/nie-mockuj-tak-czyli-o-naduzywaniu-mockow-w-testach/