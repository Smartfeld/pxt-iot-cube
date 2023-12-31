# Blöcke

Auf dieser Seite werden einzelne Blöcke der Erweiterung beschrieben.
 - [LoRa Netzwerk](#lora_netzwerk)
   - [Uplink](#lora_uplink)
   - [Downlink](#lora_downlink)
 - [Cayenne LPP](#cayenne_lpp)
 - [GPIO](#gpio)
 - [weitere](#weitere)

------------------------------------------------
## LoRa Netzwerk
Die LoRa Kommunikation ist das Herzstück der Erweiterung. Im Fokus steht die Steuerung des [RAK3172 LoRa Moduls](https://docs.rakwireless.com/Product-Categories/WisDuo/RAK3172-Module/Overview/). 
### LoRa Konfiguration
Um eine Verbindung zum LoRa Netzwerk herzustellen, muss zuerst das LoRa Modul mit den notwendigen Parametern konfiguriert werden. Der Block dazu befindet sich in **IoT Cube -> Konfiguration -> OTAA Setup**. Mit diesem Block werden DevEUI, AppKey etc. auf das Modul geschrieben und über einen Stromzyklus hinweg gespeichert. Dementsprechend braucht es den Block nur bei der ersten Konfiguration des Zugangspunkt. 
```blocks
IoTCube.OTAA_Setup(
"11111111111",
"2222222222",
"3333333333",
eBands.EU868,
"A"
)
``` 
Der Block schreibt die einzelnen Parameter nach einander auf das LoRa Modul und führt zum abschluss einen Neustart aus.

### LoRa Verbindungsaufbau
Wenn die Konfiguration erfolgreich gespeichert wurde, kann ein Verbindungsaufbau gestartet werden. Der Verbindungsversuch kann mit den Parametern konfiguriert werden. So kann die Anzahl Neuversuche und Wartezeit zwischen Versuchen eingestellt werden. Der Parameter "Automatisch" ermöglicht den automatischen Verbingungsaufbau beim Start des Moduls. 
```blocks
IoTCube.LoRa_Join(eBool.enable,eBool.enable,10,8)
```

### LoRa Uplink
Bei einer Verbindung mit dem LoRa Netzwerk, können Daten übermittelt werden. Im Normalfall wird ein Buffer mit den Daten vorbereitet und anschliessend mit dem Block *SendBuffer* gesendet. Der F-Port ist Standardmässig auf 1 gesetzt und kann durch das Aufklappen des Blocks geändert werden. Die LoRa spezifikation besagt, dass eine Nachricht-länge von 51 Bytes nicht überschritten werden soll. Daten werden dem Buffer nur hinzugefügt, wenn der Platz ausreicht. Überflüssige Daten werden ignoriert.
```blocks
IoTCube.SendBuffer(IoTCube.getCayenne())
IoTCube.SendBuffer(IoTCube.getCayenne(), 5)
```

### LoRa Downlink
Die Erweiterung prüft im Hintergrund, ob ein Downlink empfangen worden ist. Bei einem Class-A Gerät ist dies nach einem Uplink möglich, sprich ein Downlink kann nur kurz nach einem Uplink empfangen werden. 
Wird ein Downlink Event ausgelöst, dann wird der Ablauf innerhalb dieses Blocks ausgeführt. Die Nachricht ist bereits in den Variablen aufbereitet nach dem Cayenne LPP Standard. *Channel* entspricht dem value-Index und *value* ist der zugewiesene Wert.
```blocks
IoTCube.DownlinkEvent(function(channel: number, value: number) {
    
})
```
Ein Downlink kann Beispielsweise über die TTN-Konsole oder per MQTT geplant werden.

### LoRa Modulsteuerung
Für die spezifische Steuerung des LoRa Moduls sind eigene Blöcke implementiert. So kann Beispielsweise ein Reset oder Sleep ausgeführt werden. Um die volle Funktionalität des Moduls anzubieten, können Paramater gezielt gesetzt werden.

| Name  | Block                       | Beschreibung                                 |
|-------|-----------------------------|----------------------------------------------|
| Reset | IoTCube.resetModule(false)  | LoRa reset (Software oder Hardware)          |
| Sleep | IoTCube.sleep(500)          | Aktiviere Low-Power-Modus für bestimmte Zeit |
| Set parameter | IoTCube.setParameter(eRUI3_PARAM.CLASS, "B") |                               |
| Get parameter | IoTCube.getParameter(eRUI3_PARAM.RSSI) | Auslesen der Register im LoRa Modul |


### Reset
Das LoRa Modul kann mit dem Reset Befehl neu gestartet werden. Dies wird Beispielsweise nach der LoRa Konfiguration ausgeführt. Der Reset kann als UART-Kommando ausgelöst werden. Sollte allerdings die serielle Verbindung probleme machen, oder wie beim Schlafmodus nicht verfügbar sein, so kann auf Hardware Ebene ein Neustart erzwungen werden. Dabei bleiben die Einstellungen erhalten. Lediglich die LoRa verbindung bleibt nicht erhalten und muss bei Bedarf wieder neu aufgebaut werden. Mit dem automatischen Verbindungsaufbau aktiv, wird dies vom Modul selber erledigt.
```blocks
input.onButtonPressed(Button.A, function() {
    IoTCube.resetModule()
})

input.onButtonPressed(Button.B, function() {
    IoTCube.resetModule(true)
})
```
Im Beispiel "Button A" wird ein Reset auf Software Ebene durchgeführt.
Im Beispiel "Button B" wird ein Reset auf Hardware Ebene durchgeführt.

### Sleep Modus
Der Sleep Modus dient zum Stromsparen bei Akkubetrieb. Beim Ausführen wird eine Nachricht an das LoRa Modul geschikt und ab dann ist das Modul in einem Low-power Modus. Der Modus hält solange an, wie die beim Aufruf angegeben Zeit (in Milisekunden). Während dem Schlafmodus besteht keine Kommunikation zwischen micro:bit und LoRa Modul. Der Schlafmodus wird nach ablauf der Zeit verlassen oder kann durch einen Hard-Reset erzwungen werden. Nach dem Hard-Reset muss die Verbindung neu aufgebaut werden, während beim normalen Aufwachen die Verbindung schneller wieder bereit ist.
```blocks
IoTCube.sleep(60000)  

input.onButtonPressed(Button.A, function() {
    IoTCube.resetModule(true)
})
```

---------------------------------------------

## GPIO
Der IoT-Würfel besitzt eine Benutzer-LED sowie zwei Open-Collector Ausgänge. Für die Ansteuerung dieser Komponenten befinden sich die Blöcke in **IoT Cube** in der Gruppe **Pins**. 
```blocks
IoTCube.togglePin(MCP_Pins.USR_LED)
IoTCube.setPin(MCP_Pins.OC1, true)
IoTCube.setPin(MCP_Pins.OC2, false)

if (IoTCube.getStatus(eSTATUS_MASK.JOINED)) {
    IoTCube.addDigitalOutput(IoTCube.getPin(MCP_Pins.OC1), 1)
    IoTCube.SendBuffer(IoTCube.getCayenne(), 1)
}
```
------------------------------------------------
## Cayenne LPP
Das Cayenne Low Power Payload Format dient der übertragung von Daten in komprimierter Form. Die Idee ist, dass Werte mit einem Index versehen werden, um deren Inhalt deuten zu können. Der Index besteht aus einem Byte (also 256 möglichkeiten) und ist in der Tabelle vermerkt. Im The-Things-Network kann ein Parser für das Cayenne Format aktiviert werden. Dieser liest die Binär-Daten und stellt sie im JSON Format zur verfügung.

Beispielsweise wird ein Temperaturwert von 23.9°C mit Cayenne index 12 übermittelt. Im TTN sieht der decodierte Payload wie folgt aus:
``` json
{ "temperature_12": 23.9 }
```
Der Datentyp (Temperatur) wird mit einem String beschrieben und der Index (12) angehängt. Die Daten befinden sich im Value der JSON Nachricht.

| Type              | Index | Data Size   | Data Resolution per bit           |
|-------------------|-------|-------------|-----------------------------------|
| Digital Input     | 0     |  1          | 1                                 |
| Digital Output    | 1     |  1          | 1                                 |
| Analog Input      | 2     |  2          | 0.01 (von -327.68 bis 327.67)     |
| Analog Output     | 3     |  2          | 0.01 (von -327.68 bis 327.67)     |
| Illuminance Sensor| 101   |  2          | 1                                 |
| Presence Sensor   | 102   |  1          | 1                                 |
| Temperature Sensor| 103   |  2          | 0.1°C Signed MSB                  |
| Humidity Sensor   | 104   |  1          | 0.5 % Unsigned                    |
| Accelerometer     | 113   |  6          | 0.001 G Signed MSB per axis       |
| Barometer         | 115   |  2          | 0.1 hPa Unsigned MSB              |
| Gyrometer         | 134   |  6          | 0.01 °/s Signed MSB per axis      |
| GPS Location      | 136   |  9          | Latitude : 0.0001 ° Signed MSB    |
| GPS Location      | 136   |  9          | Longitude : 0.0001 ° Signed MSB   |
| GPS Location      | 136   |  9          | Altitude : 0.01 meter Signed MSB  |

Die Dokumentation zu Cayenne ist bei [docs.mydevices.com](https://docs.mydevices.com/docs/lorawan/cayenne-lpp) verfügbar.

### Cayenne Buffer
Für das übermitteln von Daten nach dem Cayenne Standard, stellt die Erweiterung einen Buffer zur verfügung. Der Buffer kann gefüllt werden und schlussendlich als LoRa Uplink verschikt werden. Der Buffer bietet den Vorteil, dass meherere Messwerte (z.B. Temperatur und Luftfeuchtigkeit) gemeinsam in einer Nachricht verschikt werden können. Dadurch wird die Übermittlung effizienter. Der Buffer ist auf 51 Bytes limitiert.

Für jeden Unterstützten Datentyp gibt es einen eigenen Block. Bei all diesen Blöcken wird zu beginn geprüft, ob im Buffer noch Platz ist. Sollte der Platz nicht reichen, werden die entsprechenden Daten nicht angefügt.

### Skalierung
Der Skalierungsblock passt den Wert and den Cayenne Bereich an. Für den micro:bit sind die analogen Werte im Bereich von *0* bis *1023*.
```blocks
IoTCube.addAnalogInput(IoTCube.scaleToCayenne(795), 1)
```

### Analog Input
Ein analoger Eingang kann Werte von *-327.68* bis *+327.67* annehmen. Dies ist Aufgrund des Cayenne Standards.
```blocks
IoTCube.addAnalogInput(26.53, 1)
IoTCube.addAnalogInput(pins.analogReadPin(AnalogPin.P0), 1)
```

Für den Micro:bit sind die analogen Werte im Bereich von *0* bis *1023*. Die Skalierung von micro:bit zu cayenne kann im Block aktiviert werden. Die Option ist hinter dem *+* versteckt und Normalerweise deaktiviert.
```blocks
IoTCube.addAnalogInput(26.53, 1, True)
IoTCube.addAnalogInput(pins.analogReadPin(AnalogPin.P0), 1, true)
```

### Helligkeit
```blocks
IoTCube.addIlluminance(pins.analogReadPin(AnalogPin.P1), 1)
```


### Beschleunigungssensor
Beschleunigungsdaten von Sensor auf dem micro:bit können direkt dem Block übergeben werden.
```blocks
IoTCube.addAccelerometer(input.acceleration(Dimension.X), input.acceleration(Dimension.Y), input.acceleration(Dimension.Z), 1)
```

### GPS Position
Der GPS Block übermittelt eine Position über LoRa. Dies kann von einem GPS Modul geschehen oder eine manuelle Eingabe beim erstellen des Programms. Die GPS Position wird als solche vom The-Things-Network erkannt und dort in einer Karte angezeigt.
```blocks
IoTCube.addGPS(47.0748, 12.6852, 3798, 1)
```

### Downlink mit Cayenne
Ein Downlink im Cayenne Format ist im Prinzip wie bei den Analog Werten aufgebaut. Es können 256 unterscheidbare Kanäle verwendet werden. Die Kanäle sind jeweils im Format `value_XY` aufgebaut, wobei XY eine Zahl von 0 bis 255 ist.

```json
{ "value_10": 1 }
```

------------------------------------------------

## Weitere
Gewisse Funktionen sind Komplexer aber haben dafür mehr Potential. Diese Kategorie ist für Fortgeschrittene empfehlenswert.

### Event
Es gibt mehrere Events, welche vom LoRa Modul gemeldet werden (z.B. "JOINED", Verbunden). Diese Events kommen in der Regel für das System unerwartet. Der Micro:bit empfängt die Events und speichert das Auftreten. Beim Abfragen eines Event-Typs, wird Wahr oder Falsch zurückgegeben. Nach dem Lesen des Events wird es zurückgesetzt (cleared).
Im folgenden Beispiel wird geprüft, ob kürzlich das Event "Verbunden" eingetroffen ist. Ist dies der Fall, spielt der Buzzer auf dem micro:bit einen Ton ab:
```blocks
if(IoTCube.checkEvent(eRAK_EVT.JOINED)){
    music.playTone(Note.C, music.beat(BeatFraction.Whole))
}
```
Die Events werden Teilweise von der Erweiterung selbst verwendet (RX_1/2 für Downlink).

| Typ     | Bit          | Referenz                 | Beschreibung                             |
|---------|--------------|--------------------------|------------------------------------------|
| Event   | 9 (0x0100)   | JOIN_FAILED              | Verbindungsversuch fehlgeschlagen        |
| Event   | 10 (0x0200)  | JOINED                   | Verbindungsversuch erfolgreich           |
| Event   | 11 (0x0400)  | SEND_CONFIRMED_FAILED    | Sendebestätigung fehlgeschlagen          |
| Event   | 12 (0x0800)  | SEND_CONFIRMED_OK        | Sendebestätigung erfolgreich             |
| Event   | 13 (0x1000)  | RX_1 (used for downlink) | Empfangen im RX-Window 1                 | 
| Event   | 14 (0x2000)  | RX_2 (used for downlink) | Empfangen im RX-Window 2                 | 


### RAK AT-Commands
Das LoRa Modul wird über Befehle per UART gesteuert.

[RAK3172 AT-Commands](https://docs.rakwireless.com/Product-Categories/WisDuo/RAK3172-Module/AT-Command-Manual/)

```blocks
IoTCube.writeATCommand("DEVEUI", "1122334455667788")
```



<script src="https://makecode.com/gh-pages-embed.js"></script><script>makeCodeRender("{{ site.makecode.home_url }}", "{{ site.github.owner_name }}/{{ site.github.repository_name }}");</script>

