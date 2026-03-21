---
layout: post
title:	"Tech: Meteostanica - MeteoGeM_v1"
date:	2025-08-31 10:00:00 +0200 
author: "0x00mg"
categories:
    - blog
tags:
    - elektro
    - esp8266
    - MQTT
   
image: /images/posts/2025/meteo/meteo1.jpg
---

<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async
  src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js">
</script>


**.......článok je ešte v procese.......**

### Ako som postavil vlastnú batériovú meteostanicu s ESP8266 a MQTT

Už dlho ma lákala myšlienka mať prehľad o tom aká je teplota a vlhkosť nielen v byte ale aj vonku. Chcel som si to ale riešiť po svojom žiadne hotové IoT hračky ale vlastný projekt ktorý si sám zladím a prispôsobím.  
A tak sa zrodil nápad na dvojicu meteostaníc: jednu do bytu a druhú von s tým že obidve budú napájané z batérie a čo najviac úsporné.

#### Nápad a požiadavky

**Moje požiadavky boli:**  
1. Meranie teploty, vlhkosti a tlaku každých 15 minút.  
2. Odosielanie dát na NAS cez MQTT aby som ich mohol ďalej spracovávať.  
3. Zariadenia musia byť úplne autonómne a napájané z Li-Ion batérií.  
4. Medzi meraniami sa má celé zariadenie vypnúť aby batérie vydržali čo najdlhšie. (žiadny deep sleep mode)
5. Sledovanie stavu batérie

> Znie to jednoducho ale ako vždy... diabol je v detailoch.  

#### Výber komponentov

**Po chvíľke premýšľania som skončil pri týchto súčiastkach:**  
**ESP8266** – lacný WiFi mikrokontrolér.  
**BME280** – senzor na teplotu, vlhkosť a tlak. Malý, presný a komunikuje cez I²C.  
**TPL5110** – „super power timer“ od Texas Instruments ktorý dokáže úplne odpojiť napájanie a prebudiť zariadenie v zadanom intervale.  
**Pololu U1V11F3** – DC/DC menič na stabilné 3,3V pre ESP. Regulátor zvyšuje napätie v rozsahu od 0,5 do 5,5 V na konštantnú hodnotu 3,3 V.  
**2× Li-Ion 18650** – zdroj energie.

<img src="{{ site.baseurl }}/images/posts/2025/meteo/meteo2.jpg" alt="Broadboard" style="width:100%; max-width:400px; height:auto; margin-bottom:20px; border-radius:4px;">
<img src="{{ site.baseurl }}/images/posts/2025/meteo/meteo5.jpg" alt="Broadboard" style="width:100%; max-width:400px; height:auto; margin-bottom:20px; border-radius:4px;">

**(....kompletný zoznam súčiastok a schéma budú doplnené.......)**

#### Softvérová časť

Samozrejme s pomocou AI som napísal jednoduchý kód v Arduine ([code MeteoGeM_v1](https://github.com/0x00mg/C-language/tree/main/MeteoGeM)). Funguje to takto:  
1. Po zapnutí sa ESP pripojí na WiFi.  
2. Inicializuje BME280 ak ho nenájde reštartuje sa.  
3. Získa aktuálny čas z NTP a prevedie ho na lokálny (vrátane letného/zimného času).  
4. Pripojí sa na MQTT broker (bežiaci na NAS).  
5. Zmeria teplotu, vlhkosť, tlak a napätie batérie.  
6. Všetko odošle na príslušné MQTT topicy.  
7. Vysiela DONE signál pre TPL5110 a čaká kým sa zariadenie úplne vypne.  
8. Ak sa čokoľvek pokazí (WiFi, MQTT, senzor), ESP sa jednoducho reštartuje a skúsi to znova.  

**TPL5110** – malý čip, veľká záchrana batérie  
Najväčšia výzva bola nastaviť TPL5110. Tento malý čip sa správa ako inteligentný „vypínač“ – drží ESP úplne vypnuté a prebúdza ho len v nastavených intervaloch. Nakonfigurovať ho na 15 minút bola celkom sranda.
Po tom čo ESP odošle dáta musí ešte odoslať DONE signál. Tým dá TPL5110 najavo že môže zase odpojiť napájanie a čakať na ďalšie prebudenie.

> Vďaka tomu mám istotu že ESP medzi meraniami nežerie batériu v deep sleep režime ale je naozaj úplne odpojené.

#### Testovanie a výsledky

Keď som to prvýkrát spustil, bol to krásny moment... nič nefungovalo. :)  
ESP sa prebudilo, pripojilo na WiFi, poslalo dáta a... ostalo zapnuté....  
**(Popis problému v procese....)**

Na NAS-e som si nastavil MQTT server a vizualizáciu dát. Teraz môžem sledovať históriu teplôt a vlhkosti a porovnávať byt vs. vonkajšie prostredie.  
**(Popis nastavenia a testovanie v procese....)**

<img src="{{ site.baseurl }}/images/posts/2025/meteo/meteo4.jpg" alt="esp mqtt" style="width:100%; max-width:400px; height:auto; margin-bottom:20px; border-radius:4px;">  

<img src="{{ site.baseurl }}/images/posts/2025/meteo/meteo3.jpg" alt="test mqtt" style="width:100%; max-width:400px; height:auto; margin-bottom:20px; border-radius:4px;">

#### Testovanie a vyhodnotenie spotreby meteostanice

Po dokončení hardvéru bolo potrebné otestovať spotrebu a odhadnúť ako dlho dokáže meteostanica fungovať na batérie.  
Cieľ bol jasný – zistiť či 2× Li-Ion 18650 (2200 mAh, paralelne = 4400 mAh) vydržia aspoň niekoľko mesiacov pri intervale odosielania dát každých 15 minút.

#### Spotreba jednotlivých komponentov

| Komponent       | Stav      |   Prúd(mA) &nbsp;&nbsp;| Poznámka                                     |
|-----------------|-----------|------------:|----------------------------------------------|
| WeMos D1 Mini   &nbsp;&nbsp;| aktívny   &nbsp;&nbsp;&nbsp;&nbsp;|        ~150 &nbsp;&nbsp;&nbsp;| WiFi + MQTT, cca 5 s                         |
| BME280          &nbsp;&nbsp;| meranie   &nbsp;&nbsp;&nbsp;&nbsp;|        ~0.2 &nbsp;&nbsp;&nbsp;| zanedbateľné oproti WeMos                    |
| TPL5110         &nbsp;&nbsp;| standby   &nbsp;&nbsp;&nbsp;&nbsp;|    0.000035 &nbsp;&nbsp;&nbsp;| len počas vypnutia, zanedbateľné             |
| Pololu U1V11F3  &nbsp;&nbsp;| prevádzka &nbsp;&nbsp;&nbsp;&nbsp;|           – &nbsp;&nbsp;&nbsp;| efektivita cca 85–90 % (zohľadnené vo výpočtoch) |  

   &nbsp;
**Poznámka:**  
Počas „spánku“ je WeMos fyzicky odpojený od napájania pomocou TPL5110 takže spotreba v neaktívnej fáze je prakticky nulová.  
Hlavnú časť dennej spotreby teda tvorí krátka aktívna fáza každých 15 minút.

#### Výpočet spotreby a odhad výdrže batérie

**Predpoklady a hodnoty použité vo výpočtoch**  &nbsp;

- **Prúd počas aktívnej fázy pri 3.3 V:** WeMos + BME280  

$$
I_{\text{active}} = 150.2\ \mathrm{mA}
$$

- **Dĺžka aktívnej fázy:**  

$$
t_{\text{active}} = 5\ \mathrm{s}
$$

- **Interval prebudenia:** 15 min  

$$
n_{\text{cyklov}} = 96\ \mathrm{cyklov/deň}
$$

- **DC/DC účinnosť:**  

$$
\eta_{\text{DC}} = 0.85 \ (\text{85 \%})
$$

- **Batéria:** 2×18650 2200 mAh paralelne  

$$
Q_{\text{bat}} = 4400\ \mathrm{mAh}
$$

- **Použiteľná frakcia kapacity (derating):**  

$$
f_{\text{usable}} = 0.90 \ (\text{90 \%})
$$

- **Samovybíjanie:** približne 2 % mesačne  

$$
r_{\text{sd,day}} \approx \frac{0.02}{30} \approx 0.00067\ \text{(zlomok/deň)}
$$

#### Výpočet

**1) Spotreba počas jedného cyklu na 3.3 V strane**  

$$
t_{\text{active}} = \frac{5}{3600}\ \mathrm{h} \approx 0.001389\ \mathrm{h}
$$

$$
Q_{\text{cyklus,load}} = I_{\text{active}} \cdot t_{\text{active}} 
\approx 150.2 \cdot 0.001389 \approx 0.208\ \mathrm{mAh}
$$

**2) Prepočet na batériovú stranu (zohľadnenie DC/DC účinnosti)**  

$$
Q_{\text{cyklus,batt}} = \frac{Q_{\text{cyklus,load}}}{\eta_{\text{DC}}} 
\approx \frac{0.208}{0.85} \approx 0.245\ \mathrm{mAh}
$$

**3) Denná spotreba bez samovybíjania**  

$$
Q_{\text{den}} = n_{\text{cyklov}} \cdot Q_{\text{cyklus,batt}} 
= 96 \cdot 0.245 \approx 23.52\ \mathrm{mAh/deň}
$$

**4) Pripočítanie samovybíjania batérie (denný príspevok)**  

$$
Q_{\text{sd/day}} = Q_{\text{bat}} \cdot r_{\text{sd,day}} 
= 4400 \cdot 0.00067 \approx 2.95\ \mathrm{mAh/deň}
$$

**Celková denná spotreba:**  

$$
Q_{\text{den,eff}} = Q_{\text{den}} + Q_{\text{sd/day}} \approx 23.52 + 2.95 \approx 26.47\ \mathrm{mAh/deň}
$$

**5) Použiteľná kapacita batérií (derating)**  

$$
Q_{\text{bat,usable}} = Q_{\text{bat}} \cdot f_{\text{usable}} = 4400 \cdot 0.9 = 3960\ \mathrm{mAh}
$$

**6) Odhad výdrže**  

$$
t_{\text{vydrz}} = \frac{Q_{\text{bat,usable}}}{Q_{\text{den,eff}}} 
= \frac{3960}{26.47} \approx 149.6\ \mathrm{dní} \approx 5.0\ \mathrm{mesiacov}
$$


Poznámky:  
Ak by DC/DC účinnosť bola lepšia (napr. 90 %), alebo samovybíjanie menšie, výdrž by rástla.  
V reálnom prostredí môže teplota výrazne ovplyvniť kapacitu batérie (nižšie teploty = nižšia použiteľná kapacita).  
Ak by sa aktívny čas predĺžil (napr. dlhšie WiFi pripojenie), Q_cycle sa zvýši proporcionálne.

<img src="{{ site.baseurl }}/images/posts/2025/meteo/meteo1.jpg" alt="Broadboard" style="width:100%; max-width:600px; height:auto; margin-bottom:20px; border-radius:4px;">

#### Čo ďalej?

Plánujem:

Pridať webové rozhranie alebo integráciu do Home Assistanta.  
Vytlačiť elegantnú krabičku.  
Otestovat spotrebu v reálnych podmienkach.


#### Záver
Tento projekt mi ukázal že aj relatívne jednoduchá myšlienka (merať teplotu každých 15 minút) sa dá spraviť elegantne a efektívne ak si človek dá pozor na spotrebu energie.
ESP8266, BME280 a TPL5110 sa ukázali ako ideálna kombinácia pre lacnú a úspornú meteostanicu ktorá dokáže fungovať na batériu veľmi dlho.
