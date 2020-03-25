---
layout: post
title: Refactoring procesu zapisu formularza
image: /img/vsShortcuts.jpeg
bigimg: /img/vsShortcuts.jpeg
tags: [Refactoring, Modelowanie, C#]
---

TODO

![Alt text](/img/submitApplyNow_old.svg "Stary Proces")

Jak możemy zauwazyć cały proces dzieje sie synchronicznie, jesteśmy bardzo uzależnieni od zewnetrznej infrastruktury - kilenta email, Workflowów(WWF - stary twór Microsoftu, obecnie wspierany tylko w trybie bug-fix), która tez może zacząc niedomagac przy duzym obciazeniu.

Dodatkowo cała logika znajduje sie w kontrolerze(typowy 8 tysięcznik), co nie pomaga w prostej refaktoryzacji.

# Zaproponowane rozwiązanie



![Alt text](/img/submitApplyNow_new.svg "Nowy Proces")

# Podsumowanie
