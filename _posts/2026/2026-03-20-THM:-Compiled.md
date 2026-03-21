---
layout: post
title:	"THM: Compiled"
date:	2026-03-20 20:00:00 +0200 
author: "TheOldOne"
categories:
    - blog
tags:
    - forensics analysis
    - ghidra
    
---


Našou úlohou je získať heslo. Dostaneme súbor bez prípony – Compiled. Prvý krok je vždy zistiť o aký typ súboru ide.  
K tomu použijeme `file`.

<img src="{{ site.baseurl }}/images/posts/2026/THM/Compiled/file.jpg" alt="dns host" style="width:100%; max-width:700px; height:auto; margin-bottom:20px; border-radius:4px;">

#### strings – čitateľné reťazce  
Príkaz `strings` extrahuje všetky čitateľné ASCII reťazce z binárky. Je to najrýchlejší spôsob ako zistiť čo binárka robí bez spustenia.


Čo vidíme? Niekoľko zaujímavých reťazcov:  
**• Password:**              – program žiada heslo od používateľa  
**• DoYouEven%sCTF**         – formátovací reťazec so %s – dopĺňa sa niečo dovnútra  
**• StringsIH a sForNoobH**  – fragmenty... ktoré budú relevantné  
**• Correct! / Try again!**  – program overuje správnosť vstupu   
**• scanf a strcmp**         – použité funkcie z libc  
Používame `scanf` na čítanie vstupu a `strcmp` na porovnanie reťazcov – to je typický vzor overovania hesla v C.  

#### Ghidra – dekompilujeme binárku  
Reťazce nám dali dobrý prehľad, ale heslo vidíme len v kusoch. Otvoríme binárku v Ghidra (NSA open-source dekompilátor) a pozrieme sa na funkciu main.  

Po importe a analýze nájdeme v Symbol Tree funkciu main a zobrazíme jej dekompilovaný kód:  

