# Tasmota-basierter Gaszähler-Logger für Volkszähler
Das Project beschreibt die Implementierung eines Geräts zur Übertragung der momentanen Verbrauchswerte sowie des Zählerstandes eines Gaz-Zählers an die [Volkszähler](https://wiki.volkszaehler.org/start)-Middleware

## Prinzip
- Einer der Zahlenräder des Gaszählers beisitz einen Magnet
- Bei voller Umdrehung des Zahlenrades wird der [Reedshalter](https://de.wikipedia.org/wiki/Reedschalter) ein Mal geschlossen
- Beim Schließen wird ein [GPIO](https://www.wemos.cc/en/latest/d1/d1_mini_lite.html#pin) des [ESP8266](https://en.wikipedia.org/wiki/ESP8266)-Chips auf 0 gesetzt
- Die [Tasmota](https://tasmota.github.io/docs/About/)-Software reagiert auf den geänderten Zustand des GPIO und sendet einen Inkrement über das (WiFi)-Netzwerk and die [REST-API](https://wiki.volkszaehler.org/development/api/reference) des [Volkszähler-Servers](https://wiki.volkszaehler.org/software/middleware)
- Zusätzlich inkrementiert Tasmota den eigenen internen Gas-Zähler und schickt seinen Wert ebenfalls and den Server
- Der Volkszähler-Server speichert die übertragenen Werte
- Die Werte können über das [Volkszähler-Frontend](https://wiki.volkszaehler.org/software/frontends/frontend) auf verschiedene Art-und-Weise präsentiert oder ausgewertet werden.

## Hardware

Mein Sender nutzt [Wemos D1](https://www.wemos.cc/en/latest/d1/d1_mini_lite.html) als Platform. Es würde aber mit jedem anderen ESP8266- bzw ESP32-basierten Board funktionieren.

Das Auslesen der momentanen Verbrauchwerte erfolgt über einen [Reedshalter](https://de.wikipedia.org/wiki/Reedschalter). Die Voraussetzung ist, dass der Gaszähler einen magnetischen Signalgeber besitzt.

Ich habe mich für den für meinen Gaszähler originalen [IN-Z61](https://process.honeywell.com/us/en/site/elster-instromet-de/produkte/gasmessung/balgengaszahler/in-z61) entschieden.

### Schaltplan

![Shaltplane](./media/TasmotaGasMeter_CircuitDiagram.png)

#### Erklärung

##### S1 "IN-Z61"
die einzige nicht-optionale Komponente (außer ESP8266) auf diesem Schaltplan. Beim Schließen wird das [D4/GPIO2](https://www.wemos.cc/en/latest/d1/d1_mini_lite.html#pin) Kontakt des Chips auf 0 gesetzt. Der Kontakt D4 besitzt einen internen [Pull-up Resistor](https://en.wikipedia.org/wiki/Pull-up_resistor), somit muss man keinen eigenen einlöten. 

Der Kontakt D4 hat noch eine andere Besonderheit: an ihm ist das Platinen-interne LED angeschlossen. Schliesst man den S1, so leuchtet das LED rein elektronisch (i.e. ohne jegliche Software), was die Suche nach Fehlern erleichtert.

##### C1, C2
Diese beide Kondensatoren sollen die Laufzeit-Stabilität erhöhen. Der kleine filtert hochfrequente Störungen, der Große - die niedrigfrequente Schwankungen.

Die beiden sollen am besten direkt zwischen 3V3 und G(ND) eingelötet werden.

##### Rot-/Grün-LED's
Das grüne LED signalisiert die Verbindung mit dem WiFi (durchgehendes Leuchten).
Das Rote leuchtet kurz auf bei jedem Auslösen des Reedschalters (programmiert).

##### Stromversorgung
Es reicht ein Handelsüblicher USB-Ladegerät mit einem Micro-USB-Anschluss. 