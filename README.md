# Apache OpenWhisk w chmurze e24cloud.com

Artykuł ten powstał w ramach konkursu "KONKURS Z CHMURĄ - Pokaż jak używasz chmury" zorganizowanego przez chmurowisko.com.


Konkurs: https://chmurowisko.pl/konkurs-z-chmura/

## Czym jest Apache OpenWhisk?

Rozwijany w ramach projektu "Incubator" Apache Software Foundation, OpenWhisk jest open-source'ową platformą Function-as-a-Service(FaaS), pozwalającą na uruchamianie kodu (akcji) w odpowiedzi na eventy, zwane również triggerami.

## Jak działa Apache OpenWhisk ?

Główną ideą OpenWhisk'a jest wykonywanie odpowiednich akcji (actions), w odpowiedzi na zdarzenia (eventy).

![diagram OpenWhisk](https://github.com/apache/incubator-openwhisk/blob/master/docs/images/OpenWhisk.png)
źródło: https://github.com/apache/incubator-openwhisk/blob/master/docs/images/OpenWhisk.png

Akcje są to bezstanowe funkcje, napisane w języku takim jak Python, Swift, Javascript, mogą być również dowolnym programem spakowanym w kontener Docker'owy.
Do zdarzeń w wyniku których wywoływane są akcje należeć mogą proste requesty HTTP, zmiany w bazie danych, upload pliku graficznego czy też pojawienie się nowego commitu w repozytorium.

Zdarzenia te trafiają do odpowiednich triggerów, które następnie na podstawie reguł wywołują odpowiednie akcje. 

()

