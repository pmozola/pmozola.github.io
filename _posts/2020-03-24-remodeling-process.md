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
Formularz który ma do wypelnienia jest skomplikowany, zalezy od wielu ustawień w systemie, podzielony na kilka(kilkanaście) kroków w zalezonosci od ustawień(załozenie konta użytkownika, wybór kierunku, wpisanie dorobku naukowego itd.). Po przejściu kazdego kroku, przyblizającego aplikanta do finalnego wysłania aplikacji, system zapisuje sobie draft takiego podania, dodatkowo sprawdza warunki biznesowe które, jezeli zostaną spełnione moga dodatkowo pozkazać/ukryc niektóre kroki.

# Problem
Głównym problemem, oprócz jakości kodu(jak to zwykle bywa w większych aplikacjach korporacyjnych), jest wydajność rozwiązania.
W tym wpisie skupimy sie już na finnalnym wysłaniu poprawwnie wypełnionej aplikacji kandydata. 
Już szybkie pomiary na maszynach developerskich Requestu do API, pokazały ze request trwa stanowczo za długo - średnio 11sekund, co w moim odczuciu jest wiecznością.
Testy przeprowadzone na srodowiskach(wykonane JMeterem) potwierdziły wyniki otrzymywane na lokalnych instanchach i trwały około 12-13 sekund przy 100 uzytkownikach.
Specyfika domeny biznesowej, jednak sprawia ze jako twórcy aplikacji możemy spodziewać sie duzo wiekszego obciazenia serwerów w trakcie 2-3 dni w ciagu roku(rozpoczecia zapisów). Bardzo prawdopodobnym senariuszem jest to ze aplikacja, zacznie nienadazac za przetwarzabuen reqestów, zaczniemy skalowac ją w szerz, a rachunek za azura/serwerownie wzrosnie nam nieproporcjonalnie do skali problemu biznesowego z ktorym sie mierzymy. Nie mówiac juz o zniechęconych aplikantach którzy nie beda mogli komfortowo przejsc zapisu na studia.

# Jak działa obecnie
Schemat przedstawia działanie systemu podczas przetwarzania zadania zapisu formatki aplikanta

![Alt text](/img/submitApplyNow_old.svg "Stary Proces")

Jak możemy zauwazyć cały proces dzieje sie synchronicznie, jesteśmy bardzo uzależnieni od zewnetrznej infrastruktury - kilenta email, Workflowów(WWF - stary twór Microsoftu, obecnie wspierany tylko w trybie bug-fix), która tez może zacząc niedomagac przy duzym obciazeniu.

Dodatkowo cała logika znajduje sie w kontrolerze(typowy 8 tysięcznik), co nie pomaga w prostej refaktoryzacji.

# Zaproponowane rozwiązanie



![Alt text](/img/submitApplyNow_new.svg "Nowy Proces")

# Podsumowanie
