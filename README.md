# Tasmota-basierter Gaszähler-Logger für Volkszähler
Das Project beschreibt die Implementierung eines Geräts zur Übertragung der momentanen Verbrauchswerte sowie des Zählerstandes eines Gaz-Zählers an die [Volkszähler](https://wiki.volkszaehler.org/start)-Middleware

## Prinzip
- Einer der Zahlenräder des Gaszählers beisitz einen Magnet
- Bei einer kompletten Umdrehung des Zahlenrades wird der [Reedshalter](https://de.wikipedia.org/wiki/Reedschalter) durch die Magneteinwirkung ein Mal geschlossen und wieder geöffnet
- Beim Schließen wird ein [GPIO](https://www.wemos.cc/en/latest/d1/d1_mini_lite.html#pin) des [ESP8266](https://en.wikipedia.org/wiki/ESP8266)-Chips auf 0 gesetzt
- Die [Tasmota](https://tasmota.github.io/docs/About/)-Software reagiert auf den geänderten Zustand des GPIO und sendet einen Inkrement über das (WiFi)-Netzwerk and die [REST-API](https://wiki.volkszaehler.org/development/api/reference) des [Volkszähler-Servers](https://wiki.volkszaehler.org/software/middleware)
- Zusätzlich inkrementiert Tasmota den eigenen internen Zähler und schickt seinen Wert ebenfalls and den Server
- Der Volkszähler-Server speichert die übertragenen Werte
- Die Werte können über das [Volkszähler-Frontend](https://wiki.volkszaehler.org/software/frontends/frontend) auf verschiedene Art-und-Weise präsentiert oder ausgewertet werden.

## Hardware

Mein Sender nutzt [Wemos D1](https://www.wemos.cc/en/latest/d1/d1_mini_lite.html) als Platform. Es würde aber mit jedem anderen ESP8266- bzw ESP32-basierten Board funktionieren.

Das Auslesen der momentanen Verbrauchswerte erfolgt über einen [Reedshalter](https://de.wikipedia.org/wiki/Reedschalter). Die Voraussetzung ist, dass der Gaszähler einen magnetischen Signalgeber besitzt.

Ich habe für mich für den originalen [IN-Z61](https://process.honeywell.com/us/en/site/elster-instromet-de/produkte/gasmessung/balgengaszahler/in-z61) Reedschalter entschieden.

## Schaltplan

![Shaltplane](./media/TasmotaGasMeter_CircuitDiagram.png)

### Erklärung zum Schaltplan

#### S1 "IN-Z61"
die einzige nicht-optionale Komponente (außer ESP8266) auf dem Schaltplan. Beim Schließen wird der [D4/GPIO2](https://www.wemos.cc/en/latest/d1/d1_mini_lite.html#pin) Kontakt des Chips auf 0 gesetzt. Der Kontakt D4 besitzt einen internen [Pull-up Resistor](https://en.wikipedia.org/wiki/Pull-up_resistor), somit muss man keinen eigenen einlöten. 

Der Kontakt D4 hat noch eine andere Besonderheit: an ihm ist das Platinen-interne LED angeschlossen. Schliesst man den S1, so leuchtet das LED rein elektronisch (i.e. ohne jegliche Software-Logik), was die Suche nach Fehlern erleichtert.

#### C1, C2
Diese beiden Kondensatoren sollen die Laufzeit-Stabilität erhöhen. Der kleine filtert hochfrequente Störungen, der Große - die niedrigfrequente Schwankungen.

Die beiden sollen am besten direkt zwischen 3V3 und G(ND) eingelötet werden.

#### Rot-/Grün-LED's
Das grüne LED signalisiert die Verbindung mit dem WiFi (durchgehendes Leuchten).
Das Rote leuchtet kurz auf bei jedem Auslösen des Reedschalters (programmiert).

#### Stromversorgung
Es reicht ein Handelsüblicher USB-Ladegerät mit einem Micro-USB-Anschluss.

## Volkszähler-Einstellung

### Volkszähler-Installation
Als Erstes muss die [Volkszähler-Software](https://volkszaehler.org/) selbst [installiert](https://wiki.volkszaehler.org/howto/getstarted) werden. Meine läuft auf einem Raspberry Pi 3.
Theoretisch kann für den Anfang die Demo-Installation auf der [Homepage](https://volkszaehler.org/) verwendet werden.

### Kanäle einrichten
Für das Projekt werden 2 Kanäle benötigt:
1. für momentane Verbrauchwerte
2. für den Zählerstand

Die Konfiguration ist wie auf den Bildern:

![Momentaner Verbrauch](./media/Volkszaehler_Gasverbrauch.png)

![Zählerstand](./media/Volkszaehler_Gas-Zaehlerstand.png)

Der für die Einrichtung der Kanäle benötige Auflösung-Wert ist normalerweise auf dem Gas-Zähler vermerkt. Beispiel:
![Auflösung](./media/GasMeter.jpg)
Der Wert muss für die Eingabe invertiert werden. I.e. 0.01 -> 100

## Tasmota-Einstellungen

### Vorbereitung
Die Tasmota installation is ausführlich im Netz [beschrieben](https://tasmota.github.io/docs/Getting-Started/).
Ich habe die [Web-Installation](https://tasmota.github.io/install/) verwendet.
Vor der Tasmota-Installation müssen die entsprechenden [Windows-Treiber](https://www.wemos.cc/en/latest/ch340_driver.html) für die Übertragung installiert werden.
Nach der erfolgreichen Tasmota-Installation muss diese noch in dem Heim-WiFi-Netzwerk eingeloggt werden. Dafür wird z.B. mit einem Smartphone zunächst direct ins Tasmota-WiFi-Netzwerk eingeloggt. Dadurch wird automatisch dei Konfiguration-Seite angezeigt. Hier konfiguriert man Tasmota für die Verbindung mit dem Heim-WiFi und startet Tamota neu. Anschließend verbindet man sich mit dem Heim-WiFi und ruft die Tasmota-IP-Adresse für weiter Konfigurationen auf.

### Konfiguration der GPIO's
Zuerst wird der Modul-Typ ausgewählt und gespeichert, was zu einem Neustart von Tasmota führt.

![Auswahl des Modul-Types](./media/Tasmota_ModuleType_Generic18.png)

Anschließend werden die GPIO's wie auf dem Bild eingestellt:

![Konfiguration der GPIO's](./media/Tasmota_GPIO_Config.png)
#### D4/Switch/1
Diese GPIO wird elektronisch geschaltet und registriert das Schließen des Reedschalters.
#### D3/Counter/1
Dieser Zähler wird nur programmatisch geschaltet. Tasmota bietet leider keine Möglichkeit eines rein virtuellen Zählers. Es gibt zwar Variablen, deren Werte "überleben" aber den Neustart nicht. Deswegen muss man eine GPIO für einen Zähler "opfern" auch wenn dieser nicht von aussen elektronisch gesteuert wird.
#### D2/Led_i/1 
Diese GPIO wird and das grüne LED angeschlossen und von Tasmota automatisch mit dem Status der WiFi-Verbindung versorgt. Blinken:keine WiFi. Durchgehend leuchten: die WiFi-Verbindung besteht.

#### D1/Relay/1
Diese GPIO wird and das rote LED angeschlossen und programmatisch für kurze Zeit bei jedem Schließen des Reedschalters eingeschaltet.

### Programmierung der Logik
Die Programmierung der Logik erfolgt über die Eingabe der [Tasmota-Commands](https://tasmota.github.io/docs/Commands/) in der Tasmota-Web-Console:

#### Command 1
`Backlog SetOption114 1; SwitchMode 2; SwitchDebounce 50;`

#### Command 2
`Rule1 ON Switch1#State=1 DO Backlog WebSend [192.168.178.77:8080] /data/c8b10730-2e21-11ee-bb5a-3fa7c996303b.json?operation=add; Counter1 +1; Power1 1; Delay 20; Power1 0 ENDON ON Counter1#Data DO WebSend [192.168.178.77:8080] /data/99dc1b00-2e21-11ee-9221-c31d5b1d4b10.json?operation=add&value=%value% ENDON`

Die `Rule1` wird beim Schließen des Reedschalters aktiviert und macht folgendes:
1. Senden eines Verbrauchsignals (d.h. 0.01 m2) and den Verbrauchskanal
2. Inkrementieren des internen Zählerstands
3. Schalten des roten LED für 20 sek
4. Schicken des internen Zählerstands and den Zählerstand-Kanal

Damit die Regel richtig funktioniert müssen folgende Bestandteile durch Ihre Werte ersetzt werden:

|Wert| zu ersetzen durch|
|----|------------------|
|192.168.178.77| die IP-Adresse der Volkszählers in Ihrem Netz|
|c8b10730-2e21-11ee-bb5a-3fa7c996303b| den UUID des von Ihnen erstellten Verbrauchskanals |
|99dc1b00-2e21-11ee-9221-c31d5b1d4b10| den UUID des von Ihnen erstellten Zählerstand-Kanals |

#### Command 3
`Rule1 1`

Aktiviert die `Rule1`

#### Command 4
`Counter1 120843`

Setzt den initialen Wert des internen Zählers auf den realen Wert vom Gaszähler. Dabei soll die ermittelte Auflösung des Zähler-Magnetes berücksichtigt werden. 

In meinem Fall: 
| | |
|---|----|
| Zählerstand | 1208,435 |
| Auflösung | 0,01 |
| Zu setzende Wert des internen Tasmota-Zählers| 120843|

## Aufbau/Gehäuse
Als Gehäuse habe ich einen DLAN-Adapter "ausgeschlachtet" der zwar noch funktionstüchtig aber für mich völlig nutzlos rumlag.

Der Vorteil des DLAN-Adapters für mich war, dass er keine zusätzliche Steckdose belegt.

Für die Stromversorgung habe ich ein integriertes 5v Netzeil besorgt:

![Netzteil](./media/power.png)

Die Schaltung ist auf einer Lochrasterplatine umgesetzt. Die LED's sind so platziert, dass sie durch die bereits vorhandenen LED-Öffnungen des Gehäuses hindurch leuchten.

Aufgrund der engen Platzverhältnisse musste ich auf die sog. Stacked-Implementierung zurückgreifen, bei der die Platinen übereinander platziert sind:

![Stacked Platinen](./media/circuit3.png)

Die finale Zusammensetzung sieht wie folgt aus:

![Assembly](./media/casing5.png)

Die Messing-Abstandshalter der Platinen sind mit dem Heißkleber am Boden des Gehäuses verklebt. Die Platinen selbst können bei Bedarf durch das Abschrauben von den Abstandshalter gelöst werden.

Das Netzeil wird horizontal von 2 Seiten durch die Seitenwände und and der freien Ecke durch den Heißkleber gehalten. Der Kleber haftet aber nur am DLAN-Gehäuse, so dass das Netzteil auch jederzeit entnommen werden kann. Vertikal wird es beim Schließen des Gehäuses durch den Schaumstoff fest gehalten.

![Assembled](./media/casing6.png)