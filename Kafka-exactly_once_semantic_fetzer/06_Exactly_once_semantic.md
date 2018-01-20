# Kafka – Exactly Once Semantic

Im Folgenden Kapitel wird näher über die Bedeutung der Semantik "Exactly Once" eingegangen und wie dies mit Apache Kafka umgesetzt werden kann. Bei Messaging Systemen können verschiedene Herausforderungen auftreten, welches es schwierig macht genau eine Nachricht über das Netzwerk zu schicken. Entweder kann ein Broker ausfallen, der Schreibvorgang der Nachricht auf der Partition fehlschlagen, die Bestätigung der Nachricht durch einen Netzwerkfehler nicht ankommen etc. Durch diese Problemstellungen kann man folgende Semantiken festhalten.

Wenn der Broker alle ACKs geschickt hat, ist die Nachricht genau einmal angekommen, der Optimalfall also. Wenn der Producer in der vorgegebenen Zeit keine ACK sendet (in timeout läuft) oder einen Fehler bekommt, könnte der Producer die Nachricht nochmals schicken mit der Annahme, dass Kafka die erste Nachricht nicht verarbeitet hat. Wenn die erste Nachricht aber korrekt geschrieben worden ist, aber nur die Bestätigungsbenachrichtigung (ACK) der erfolgreichen Verarbeitung durch einen Netzwerkfehler fehlgeschlagen ist, wird die Nachricht mehrfach verarbeitet. Dies bedeutet eine Garantie von mindestens einem korrekten Schreibvorgang. Es kann aber auch doppelte Arbeit mit duplizierten Nachrichten oder unvollständige Verarbeitung bedeuten.
## At most once semantics
Bei dieser Semantic wird davon ausgegangen, dass die Nachricht korrekt verarbeitet wurde, wenn keine Fehlermeldung zurückkommt. Hierbei wird es bevorzugt Duplikate zu vermeiden anstatt bei Fehlendem ACK die Nachricht nochmals zu senden. In den meisten Fällen werden die Nachrichten korrekt zugestellt und eine Fehlende Nachricht wird in Kauf genommen.
## Exactly once semantics
Dies ist das Idealbild der Messaging Verarbeitung. Es werden alle Nachrichten genau einmal vom Consumer empfangen auch wenn der Producer diese öfters sendet. Um dies zu erreichen benötigt man eine Zusammenarbeit von Producer und Consumer sowie von dem Message-Verarbeitungssystem. Wie diese Semantic umgesetzt werden kann und welche Herausforderungen hierbei zu beachten sind wird im Folgenden beschrieben.
### Broker Fehler

### RPC Call von Producer zu Broker
Der Producer kann sich erst sicher sein, dass der Broker die Nachricht erhalten hat, wenn dieser auch das ACK bekommt. Der Producer weiß aber nicht, ob nur das ACK fehlgeschlagen ist und somit die Nachricht korrekt geschrieben wurde, oder aber ob der Schreibvorgang selbst fehlerhaft ist. Der Producer ist also gezwungen die Nachricht nochmals abzuschicken, da er nicht weiß ob die Nachricht auch korrekt geschrieben worden ist. Je nach Semantik kann der Consumer die Nachricht also doppelt bekommen und somit befindet sich die Nachricht zweimal im Partition-Log.
### Consumer Fehler

## Kafkas Lösungen des Exacly Once Problems

Bedeutet, kann beliebig oft wiederholt werden mit immer dem gleichen Ergebnis. Producer können ihre Nachrichten beliebig oft senden, Sie werden in die Partition immer nur einmal geschrieben. Dies wird erreicht, in dem mit der Nachricht eine Sequenznummer mitgeschickt wird, anhand derer der Broker Duplikate identifizieren und vermeiden kann. Nun können im Fehlerfall oder bei fehlendem ACK die Nachrichten nochmals gesendet werden, ohne das diese zweimal in die Partition geschrieben werden.
### Transaktionen
Durch atomare Transaktionen können nun mehrerer Producer in unterschiedlichen Partitionen beliefert werden. Entweder sind alle Nachrichten sichtbar oder keine. Dies erlaub nun den Versand des Consumer Offsets zusammen mit den Daten in einer Transaktion. Sozusagen als eine Art Batchverarbeitung. 

Alle Nachrichten werden gelesen, ohne auf Comsumer-Seite auf den Transaktionscontext zu achten. Transaktionsmarker werden nicht beachtet.
#### read_committed

### Stream Processing
Um nun Idempotence und Transaktional in Kafka zu aktivieren, muss in der Konfiguration der Eintrag: “processing.guarantee=exactly_once” gemacht werden. 

Kafka wurde designed um einen hohen Schreib.- und Lesedurchsatz der Nachrichten zu haben und dies mit geringer Ausfallzeit. Laut dem Hersteller Benchmark haben Transaktionen mit 1KB großen Nachrichten, welche im Abstand von 100 ms abgeschickt werden, nur einen Overhead von 3-5%. Transaktionale Batchverarbeitung mit großen Datenmengen weisen einen viel besseren Performance-Durchsatz auf, wie vergleichsweise nicht Transaktionale Übertragung.  Performance Verbesserung durch das letzte Release wurden vor allem durch ein effizienteres Nachrichtenformat erreicht. Durch Hinzufügen von Metadaten im Header mit Delta-Encoding auf Nachrichten ebene. (Grahsl, 2017)