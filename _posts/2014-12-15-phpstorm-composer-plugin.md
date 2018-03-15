---
layout: post
title: "PhpStorm? composer? plugin?"
author: "psliwa"
tags: [php, open source, ide]
---

Napisałem prosty plugin do PhpStorma (w wersji od 8.0.2 wzwyż, więc nie obijajcie się z aktualizacją). Jest on zatytuowany [**PHP Composer AutoCompletion**][1], jak sama nazwa wskazuje, dodaje on podpowiadanie składni do pliku `composer.json`. Działa podpowiadanie struktury pliku oraz nazw paczek wraz z wersjami (tylko z packagist.org).

Issue na bugtrakerze PhpStorma z pomysłem tego ficzera wisi od ponad 2 lat...

Plugin w obecnej wersji ma minimalny zbiór funkcjonalności. Z czasem będą dodawane kolejne rzeczy, takie jak ulepszony autocomplete wersji paczek (wildcary, zakresy itp.), walidacja schematu i podkreślanie błędów, inspekcje, wyświetlanie obecnie zainstalowanej wersji (z composer.lock), wykrywanie dodanych paczek i ich instalowanie, optymalizacja całości.

Kod źródłowy można znaleźć na [githubie][2], jak się podoba, to zainstalowanie i [ocenienie][1] będzie mile widziane.

Plugin w akcji:

![Podgląd][3]

[1]: https://plugins.jetbrains.com/plugin/7631
[2]: https://github.com/psliwa/idea-composer-plugin
[3]: http://plugins.jetbrains.com/files/7631/screenshot_14847.png