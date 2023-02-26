# aBMS

Ein eigenbau voltage monitor aus einfachen Komponenten.
Zusammen mit einem Leistungsschalter kann es ein BMS komplett ersetzen.

Und für alle die sowas nicht bauen wollen kann: man ja durchschmökern wie sowas gebaut ist.
Einzelzellen Messung im mV Bereich ist nicht ganz so einfach.
Dafür gibts spezial chips die alle bekannten BMS einsetzen.
 
Features
- 100% clear to understand by simple open hardware and open source
- no special chips used (only AVR, OpAmps and multiplexer)
- 1-16cells, max. 60V pack voltage. 1 board.
- supply voltage min. 20V, 40mAh power consuption
- single cell monitoring
- 10mV absolute, 1mV differential accuracy (just guessing)
- build from low-cost components
- controlled by an AVR Mega324 @12MHz
- SPI 6-pin programming interface
- 20x4 alphanumeric display
  - min/max cell volt, minCellId, maxCellId, max diff, current, power, packV, SOC%
- shunt current measurement: charge(+) and discharge(-)
  - can be calibrated to any rated shunt
- 5V acoustic beeper output (cell over/under voltage)
- relais output (when cell over voltage occur)
- parameter settings/calibration via rotary shaft encoder
  - menu control (left,right,press)
  - cell calibration on balanced packs
- params are stored permanently in EEPROM
- SD-card logging: id, cell voltages and current via csv file append
  - 32GB SD card works. This is enough for years of constant logging.
- UART logging: id, cell voltages and current
- logging intervals can be configured (eg. every 20th sample)
- 1.6[sec] runtime for all cells and current measurements
- SOC calculation by energy counting and 100% calibration (battery capacity as param)
- 4 LEDs acting as is-alive monitor (binary 4bit counter)
- no MOSFET to break power. This need to be done outside if needed.
- communication to PowMr solar charge controllers possible to stop charge on overvoltage

Aufbau
  Stromversorgung lauft über einen Meanwell DC/DC converter der bei ca. 18V anlauft.
  Der macht stabile 12V.
  Dann ist eine Feinsicherung die gleich abraucht falls was schief geht.
  Aus den 12V werden dann 5V und 6V mit 2 LM2596 DC/DC converters.
  5V für den AVR. 6V für den Differenzverstärker MCP6H84 zur Zellen Messung.
  6V auch für die voltage followers (4x LM324).
  2 16-channel analog Multiplexer schalten eine Zelle zur Messung durch.
  Der AVR lauft mit 12Mhz und steuert alles.
  Es gibt ein 4x20 alphanumeric Display das die live Daten anzeigt.
  Es gibt ein Relais Schaltausgang (potential frei) und beeper Ausgang.
  Weiters einen rotary shaft encoder um parameter zu verstellen.
  Es gibt einen SD card slot zum optionalen loggen der Daten.
  Weiters werden die gleichen Daten über UART optional raus geschrieben.
  Ein Spannungsteiler Netz 1:10 macht die cell voltages handlich zu 0-6V.
  Ein weiterer Differenzverstärker mit Inverter kann den Strom übern Shunt +/- messen.
  Ein SOC[%] wird auch intern gerechnet. Akku Kapazität kann eingestellt werden.

Wie funktioniert die single-cell Spannungs Messung?
- Kurzfassung
  Der ADC im ATMEGA324 misst über einen Differenzverstärker jede Zelle durch.
  Eine Zelle wird mit 2 Multiplexern selected.
  Das Pack wird mit einem 1:10 Spannungsteiler auf handliche 0-6V gebracht.
  Widerstandsungenauigkeiten sind in Software weg kalibiert.

- Längere Version
  Der ADC im ATMEGA324 misst jede Zelle 1000x und macht dann average. Darum die relativ gute
  Genauigkeit da der interne ADC nur 10bit hat und hier mV gemessen werden.
  Ein Differenzverstärker im rail-to-rail MCP6H84 OpAmp verstärkt die Differenzspannung einer Zelle.
  Eine Zelle wird über 2 16-channel analog Multiplexer selektiert.
  Das passiert 16x pro main loop. Es wird also Zelle 0 gemessen, dann Zelle 1 usw. bis Zelle 15.
  Bevor die Zell spannung in den Multiplexer geht ist noch ein voltage follower dazwischen.
  So ähnlich wie beim Instrumentenverstärker. Die voltage follower machen die 4x LM324 OpAmp.
  Das ist ganz wichtig da jeder noch so kleiner Strom die Spannung aus den 200k/20k Spannungsteiler
  einbrechen lassen würde.
  Jede Zelle ist über einen 1:10 Spannungsteiler auf max. 0-6V gebracht.
  Dabei ist wichtig das die Widerstände schon gute grund Genauigkeit haben (0.1%).
  Trotzdem sind die gemessenen Spannungen ungleich bei einem balanced accu pack am start.
  Weil eben die 0.1% Widerstände nicht ganz gleich sind.
  Dazu gibts ein calibration setting in den params (der über den rotary shaft encoder erreichbar ist).
  Dazu schließt man ein balanced pack an (am besten mit dem Heltec 5A active balancer einige Stunden stehen lassen
  dann sind alle Zellen bis auf paar mV gleich). Dann kann man alle Zell Spannungen auf gleichen
  Wert calibrieren. Fertig. Calibrations werden permanent im EEPROM gespeichert.

- Vergleich mit anderem BMS
  Ja hab ich gemacht. Zeigt die gleichen Spannungen an. Die billig china BMS hüpfen oft mehr herum in der Messung unter Last.
  Meist hab ich mit Multimeter vergleichen.
  Ein aBMS hab ich auf LFP (3.45V) kalibiert und dann mit NMC (4.0V) laufen lassen. Spannungen passen.

Was ist noch alles onboard?
- Shunt Current measurement
  Das Ding macht das gleiche wie ein smart shunt. +/- Strom messen.
  Durch einen weiteren LM324 der über ein kleines board mit +/-12V versorgt wird kann der Strom gemessen werden.
  Die symmetrische +/- Versorgungsspannung ist wichtig damit das funktioniert.
  Die Schaltung verstärkt über einen Differenzverstärker das kleine shunt signal 10x.
  Das shunt signal wird wieder über 2 voltage followers geschickt bevor es in den Differenzverstärker kommt.
  Dann wird über einen Inverter die negative Spannung gemacht.
  Da der ADC im ATMEGA324 nur positive Spannungen messen kann wird charge und discharge extra gemessen (PA1+PA2).
  Es wird also 2x ADC gewandelt (wieder 1000x +avg), ist eine davon positiv hat man entweder charge oder discharge state.
  Der offset und scale kann über params eingestellt/kalibiert werden.
  Am AVR sind noch 2 Schottky Dioden mit 1k Widerstand davor verbaut. Das schützt den AVR vor großen negativen Spannungen.
  Das ist die Grundlage der SOC[%] Rechnung.
  Über 2 Parameter (Wh, kWh) kann man die Batt Größe (capacity) einstellen.
  Dann wird die Energie mitgezählt und über Referenzwert params (zB: 3.45V + <5A) auf 100% kalibiert.
  Der angezeige SOC passt so ungefähr was ich so mit beobachtet hatte.
- SD card logging
  Das war mir wichtig onboard zu haben. Um die Vergangenheit Revue passieren zu lassen.
  Eine billige micro SD card reicht aus um Daten von vielen Jahren zu speichern.
  Logging Takt kann eingestellt werden (oder aus deaktiviert). Ein vielfaches von 1.6s.
  ca. 1.6sec braucht ein Durchlauf um alles zu messen.
  Logged werden alle cell voltages, Strom, deviceId.
  Das ermöglicht nachträglich Auswertungen wenn Zellen weglaufen oder sonst was schräges passiert.
  Einfach die SD card rausnehmen, auslesen. Die Daten sind im Klartext in einem .csv gespeichert.
  Es werden lines immer appended, man verliert nichts ausser man löscht das file auf der card. Dann wird ein neues angelegt.
  Es funktioniert mit vielen SD card Herstellern, hab das schon probiert.
  Manchmal hängt das write am start, da hilft rein/raus stecken oder SD power ein/aus. Dann laufts.
- 4 LEDs
  Die zeigen einen 4-bit counter an.
  Das ist das is-alive Signal. Wenn die blinken und counten dann lauft das Ding.
- UART logging
  Das kann verwendet werden um zentral die Daten zu loggen.
  Es wird die gleiche Info wie beim SD logging raus geschrieben.
  9600-8-1-N mit standard parameter. Kann mit jedem xterm angeschaut werden.
  Es kann eingestellt werden ob und wie oft er das macht.
- Relais output
  für die communication mit den PowMrs.
  schließt einfach einen potentialfreien schalter bei cell overvoltage
  max/recover volt kann eingestellt werden.
- beeper output
  beep wenn cell overvoltage haben. Man hört das gut.
  max/recover volt kann eingestellt werden.

Bekannte Probleme/Issues/Erfahrungswerte
- Erfahrungsgemäß kann ich sagen: die Dinger laufen bei mir schon Monate lang durch. Ohne Probleme.
- Die 2 Schottky Dioden 1N5817 und der 1k Widerstand am AVR auf den Pins PA1+PA2 sind nachträglich eingebaut worden.
  Das verhindert große negative Spannung am AVR Pin.
  Weil es kurz große negativen Spannung am AVR gibt wenn der Shunt nachträglich eingesteckt wird.
  Da kann der LM324 durchbrennen beim shunt measurement circuit. Ist mir schon paar mal passiert.
  Aber so soll es save sein.
  Die Schottky Diode lasst max. -0.3V durch. -0.5V vertragt der AVR laut specs an den Pins.
- kein bluetooth oder app
- kein balancer onboard
- keine Temperatur Messung (halte ich für unnötig wenn alles im Keller rennt).
- das Ding hat keine echte communication (nur Relais output). Könnte man aber dazu bauen.
- das Ding kann den Pack nicht trennen (nur extern möglich)
- die Zellen werden asymmetrisch entladen da das Spannungteiler Netz höhere mehr entlädt
  - Abhilfe #1: dauer-active-balancer (Heltec 5A)
  - Abhilfe #2: ein gleiches Spannungsteiler Netz auf (batt+) schalten.

Warum nicht ein fertiges kaufen ? (JK,Ant,JBD,Daly,Seplos,REC,DIY..)
- Ist eh die einfachere Variante Smile ... jedoch:
- kritisches Bauteil: es muss zuverlässig die Ladung stoppen wenn eine Zelle abhaut
- man weiss nicht was drin ist
- man weiss nicht wie die Schaltung dahinter funktioniert
- spezial chips verbaut
- software kann abstürzen (akkudoktor.net/forum/postid/105219/)

Mechanischer Aufbau
  Das ganze Ding ist auf einer standard Lochplatine (2.54mm Rastermaß) aufgebaut.
  Bauteile vorne. Leitungen hinten.
  Die Leitungen sind alle aus Flachbandkabel gemacht.
  Jede Verbindung händisch anlöten.
  Beim Aufbau die LM2596 DC/DC converters gleich auf 5V bzw. 6V einstellen.
  Den MCP6H84 gibts leider nur als SMD. Den muss man in einen DIL14 einbauen. Das ist etwas trickreich.
  Ich hab mal eine partlist angehängt. Ca. 130€ kosten die Einzelteile. Bei größeren Mengen sicher billiger.


Software
  Es ist alles mit CodeVisionAVR V2.05.0 erstellt worden. Language: C
  Das ist eine (alte) Windows Programmierumgebung für AVR Microcontrollers. Mit coolem project wizzard.
  Ich kenne leider nur das, es kann eigentlich alles. Hat libs für SD cards usw.
  Anbei das zip vom kompletten Projekt
  Geflashed hab ich den AVR über SPI Programmer (STK300).
  Es gehört eh alles auf open source avrgcc umgestellt. Naja so laufts mal.

Michael
