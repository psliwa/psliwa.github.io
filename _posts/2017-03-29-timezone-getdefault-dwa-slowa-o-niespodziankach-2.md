---
layout: post
title: "TimeZone.getDefault() - dwa słowa o niespodziankach"
author: "psliwa"
tags: [php, scala, java]
---

Ostatnio przygotowywałem pull requesta dla biblioteki [typesafe/config][1] - prosty ficzer. Okazało się, że po moich zmianach build na windowsach przestał przechodzić. Po czasochłonnej inwestygacji, debugowaniu i namierzaniu problemu, okazało się, że winowajcą jest tytułowy `TimeZone.getDefault()`. Gdy to odkryłem, uświadomiłem sobie dlaczego tak bardzo doceniłem niemodyfikowalne obiekty, brak side effectów i czyste funkcje.

W czym był problem? Pliki `HOCON` (format plików konfiguracyjnych wspierany przez tą bibliotekę) mogą includować inne pliki konfiguracyjne. Do odczytywania takich plików używana jest klasa `URLConnection`. W implementacji tej biblioteki wywoływana jest funkcja `URLConnection.getContentType()` aby sprawdzić jakiego typu plik jest wczytywany. Metoda ta pobiera nagłówki połączenia, w którym jest data ostatniej modyfikacji. Jeśli jest data, to trzeba ją sformatować. Klasa `SimpleDateFormat` używa domyślnej strefy czasowej, chyba że powiemy jej inaczej. `TimeZone.getDefault()`, wbrew sugestywnej nazwie i zasadzie najmniejszego zaskoczenia, ma side effect, a jak! Jak to, dlaczego? Bo może! Wywołuje bowiem `System.setProperty("user.timezone", <strefa czasowa>)` jeśli takie property nie jest jeszcze ustawione. Pech w tym, że typesafe/config wczytuje java propertisy robiąc w testach asercje na tychże wartościach. W zależności czy `TimeZone.getDefault()` było wcześniej wywołane i czy `user.timezone` było jawnie ustawione, testy mogły failować lub też nie (co też miało miejsce).

[1]: https://github.com/typesafehub/config