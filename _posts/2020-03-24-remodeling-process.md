---
layout: post
title: Refactoring procesu zapisu formularza
image: /img/vsShortcuts.jpeg
bigimg: /img/vsShortcuts.jpeg
tags: [Refactoring, Modelowanie, C#]
---

Cześc, chciałem podzielic sie z wami jednym z problemów które ostatnio spotkał mnie w pracy

# Opis Procesu Biznesowego
Od pewnego czasu pracuje nad systemem do zarzadzania uczelniami. Jednym z podstawowych procesów jest zapis aplikanta na studia.
Formularz który ma do wypelnienia jest skomplikowany i zalezy od wielu ustawień w systemie, podzielony na kilka(kilkanaście) kroków w zalezonosci od ustawień(załozenie konta użytkownika, wybór kierunku, wpisanie dorobku naukowego itd.). Po przejściu kazdego kroku, przyblizającego aplikanta do finalnego wysłania aplikacji, system zapisuje sobie draft takiego podania, dodatkowo sprawdza warunki biznesowe które, jezeli zostaną spełnione moga dodatkowo pozkazać/ukryc niektóre kroki.
Aplikant po zakończonym procesie aplikacji, ma mozliwosc bezposredniego przejscia na nastepną strone, w której ma mozliwośc podejrzenia statusu aplikacji, a także załadowania dodatkowych dokumentów(plików) zwiazanych ze swoim podaniem. 

# Problem
Głównym problemem, oprócz jakości kodu(jak to zwykle bywa w większych aplikacjach korporacyjnych), jest wydajność rozwiązania.
W tym wpisie skupimy sie już na finnalnym wysłaniu poprawwnie wypełnionej aplikacji kandydata. 
Już szybkie pomiary na maszynach developerskich ządania do API, pokazały ze request trwa stanowczo za długo - średnio 11sekund, co w dobie szybkiego internetu jest wiecznością.
Testy przeprowadzone na srodowiskach(wykonane JMeterem) potwierdziły wyniki otrzymywane na lokalnych instanchach i trwały około 12-13 sekund przy 100 uzytkownikach.
Specyfika domeny biznesowej, jednak sprawia ze jako twórcy aplikacji możemy spodziewać sie duzo wiekszego obciazenia serwerów w trakcie 2-3 dni w ciagu roku(rozpoczecia zapisów). Bardzo prawdopodobnym senariuszem jest to ze aplikacja, zacznie nienadazac za przetwarzabuen reqestów, zaczniemy skalowac ją w szerz, a rachunek za azura/serwerownie wzrosnie nam nieproporcjonalnie do skali problemu biznesowego z ktorym sie mierzymy. Nie mówiac juz o zniechęconych aplikantach którzy nie beda mogli komfortowo przejsc zapisu na studia.

# Jak działa obecnie
Schemat przedstawia działanie systemu podczas przetwarzania zadania zapisu formatki aplikanta

![Alt text](/img/submitApplyNow_old.svg "Stary Proces")

Jak możemy zauwazyć cały proces dzieje sie synchronicznie, jesteśmy bardzo uzależnieni od zewnetrznej infrastruktury - kilenta email, Workflowów(WWF - stary twór Microsoftu, obecnie wspierany tylko w trybie bug-fix), która tez może zacząc niedomagac przy duzym obciazeniu.

Dodatkowo cała logika znajduje sie w kontrolerze(typowy 8 tysięcznik), co nie pomaga w prostej refaktoryzacji.

# Zaproponowane rozwiązanie
Po przejrzeniu mozliwosci, i przedyskutowaniu problemu zdecydowaliśmy sie na rozbicie procesu na 2 etapy. 
Pierwszym krokiem jest zapisanie formulara jako draftu w odpowiednim, nowym statusie. opisujacym nowy proces, oraz wysłanie zadania na kolejke, w celu dalszego przetworzenia i zwrócenie uzytkownikowi informacji, o tym ze jego aplikacja zostala przyjeta, i zostanie za chwile przetworzona.
Nastepnie, wiadomosc o zapisanej aplikacji podejmuje odpowiedni, dzialajacy w tle procesor, dokonczy wszystkie niezbedne akcje potrzebne do poprawnego, z biznesowego punktu widzenia, przetworzenia aplikacji.  

Przemodelowany proces zaprezentowany został na ponizszym schemacie

![Alt text](/img/submitApplyNow_new.svg "Nowy Proces")

# Outbox pattern

Aby uswiadomic sobie potrzebe użycia tego wzorca musimy przeanalizować poniższy pseudo kod:

 ```csharp        
        aplication.changeStatusToBackgroundProcessing();
         using (var dbContex = _databaseContextFactory.GetTransactionalContext())
            {
                dbContex.Add(application);
                dbContex.Commit();
            }

            _backgroundProcessor.Enque(new NewApplicationSubmited(applicaton.Id));
 ```
            
Nie dzieje sie tutaj nic specjalnego - zapisujemy nowa aplikacje z odpowiednim statusem, i wysyłamy wiadomosc do naszego procesora z prosba o dalsze, ale odlozone w czasie, pretwarzanie. Z pozoru nic skomplikowanego, wprowadza jednak spory problem, którego nie jestsm w prosty sposob obsłuzyć.

Załozmy ze ze po zapisaniu naszej aplikacji w bazie danych(dbContext.Commit()), wystapi bląd w naszej aplikacji - aplikacja sie zatrzyma i zostanie zrestartowana, backgroundProcessor nie bedzie dostepny albo bedzie zawierał buga, i nie bedzie w stanie poprawnie skolejkowac naszej aplikacji. 
Co w takim wypadku ? zostaniemy z aplikacja poprawnie zapisaną w bazie danych, jednak nie zostanie ona nigdy do konca poprawnie przetworzona, a aplikant nigdy nie dostanie sie do na swoj wymazony kierunek studiów. Oczywiscie istnieje cień szansy ze sprawny support/admin w logach aplikacji dostrzeze taka sytuacje i recznie, bedzie w stanie(w jakiś sposób) wznowic przetwarzanie aplikacji, ale to bardzo ryzykowne, z punktu prowadzenia biznesu, załozenie.
W powyzszym kodzie nie jestesmy w stanie wprowadzic atomowosci akcji - wykonujemy cała akcje, albo wogóle jej nie wykonujemy 

Odpowiedzia na ten problem jest "Outbox Pattern". Opiera sie on na wprowadzeniu warstwy posredniczacej, ktora zapisze nasze zadanie najpierw do odpowiednio przygotowanej tabeli w bazie danychm a  nastepnie odpowiednio przygotowany proces nasluchujacy na nowe wpisy, pobierze taka wiadomosc i przesle do odpowedniego procesora w celu dalszego przetworzenia wiadomosci.


Nasz kod zmieni sie na taki:

 ```csharp        
        aplication.changeStatusToBackgroundProcessing();
         using (var dbContex = _databaseContextFactory.GetTransactionalContext())
            {
                dbContex.Add(application);
                dbCOntext.Add(new OutboxMessage(new NewApplicationSubmited(applicaton.Id)));
                dbContex.Commit();
            }
 ```

Wszystko( zarowno aplikacja jak i wiadomosc) pojdzie w jednej tranzakcji bazodanowej, zostanie zachowana atomowosc rozwiązania(zapisze sie wszystko, albo nic), i nawet jezeli usluga przetwarzania wiadomosci nie bedzie dostepna przez pewien czas, po jej ponownej uruchomieniu, pobierze zapisane wiadomosci i odpowiednio je przetworzy. 


Polecam link do prezentacji Jimmiego Bogarda
https://www.youtube.com/watch?v=LGG3IIHUG_w

# Obsługa dalszej częsci procesu
Jak zostało wspomniane wczesniej, proces biznesowy zakłada że, aplikant po wypełneniu formularza, moze przejsc do nastepnej podstrony, w celu uzupełnenia dokumentów.
Z racji tego ze częsc aplikacji bedzie przetwarzana asynchronicznie, moze sie zdazyc ze po wejsciu na kolejna podstrone, aplikacja jeszcze nie bedzie przetworzona. 
Rozwiazaniem było sprawdzanie, przed wejsciem na kolejna strone, czy aplikacja została przetworzona, i w razie nie zakonczenia procesu przetwarzania formularza, poinformowania uzytkownika o tym fakcie, i poproszenie zeby odswierzył strone po kilku sekundach. 
Po testach na srodowiskach przypadek nie wystarczajaco szybkiego przetworzenia aplikacji zdazał sie sporadycznie(miedzy innymi dla tego że aplikant po poprawnym złozeniu aplikacji dostaje sporo informacji o dalszym procesie rekrutacji), wiec takie rozwiazanie zostało zaakceptowane.  

# Obsługa wyjątków 

TODOTODO TODO


# Podsumowanie
sam refaktoring kodu, bez podzielenia procesu - czas spadł z 12 sekund na 6(duzy narzut wykonywanych workflowów), po wprowadzeniu przetwarzania asynchronicznego uzytkownik dostawał odpowiedz dotyczaca przyjecia aplikacji( juz nie jej poprawnego przetworzenia, tyko przyjecia do dalszego przetwarzania !) niemal natychmiastowo, co w znacznym stopniu poprawiło odbiór naszej aplikacji u klientów.