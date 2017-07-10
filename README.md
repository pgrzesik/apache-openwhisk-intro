# Apache OpenWhisk w chmurze e24cloud.com

Artykuł ten powstał w ramach konkursu "KONKURS Z CHMURĄ - Pokaż jak używasz chmury" zorganizowanego przez chmurowisko.com.

Konkurs: https://chmurowisko.pl/konkurs-z-chmura/

W ramach projektu chciałbym przedstawić czym jest Apache OpenWhisk, jak działa, a także pokazać, że również na e24cloud możemy budować architekturę Serverless/FaaS.

## Czym jest Apache OpenWhisk?

Rozwijany w ramach projektu "Incubator" Apache Software Foundation, OpenWhisk jest open-source'ową platformą Function-as-a-Service(FaaS), pozwalającą na uruchamianie kodu (akcji) w odpowiedzi na eventy, zwane również triggerami.

## Jak działa Apache OpenWhisk ?

Główną ideą OpenWhisk'a jest wykonywanie odpowiednich akcji (actions), w odpowiedzi na zdarzenia (eventy).

![diagram OpenWhisk](https://github.com/apache/incubator-openwhisk/blob/master/docs/images/OpenWhisk.png)
źródło: https://github.com/apache/incubator-openwhisk/blob/master/docs/images/OpenWhisk.png

Akcje są to bezstanowe funkcje, napisane w języku takim jak Python, Swift, Javascript, mogą być również dowolnym programem spakowanym w kontener Docker'owy.
Do zdarzeń w wyniku których wywoływane są akcje należeć mogą proste requesty HTTP, zmiany w bazie danych, upload pliku graficznego czy też pojawienie się nowego commitu w repozytorium.

Zdarzenia te trafiają do odpowiednich triggerów, które następnie na podstawie reguł wywołują odpowiednie akcje. 

![flow openwhisk](https://github.com/pgrzesik/apache-openwhisk-intro/blob/master/img/flow_openwhisk.png)


## Elementy architektury Apache OpenWhisk

OpenWhisk do działania wykorzystuje technologie takie jak Nginx, CouchDB, Kafka oraz Docker, przestawione poniżej.

![elementy openwhisk](https://github.com/pgrzesik/apache-openwhisk-intro/blob/master/img/elementy_openwhisk.png)

### Nginx

API oferowane przez OpenWhisk'a jest w pełni oparte o HTTP, w związku z tym Nginx pełni rolę reverse proxy dla całego API, a także odpowiada za terminację SSL.
Z racji tego, że jest on w pełni bezstanowy, jest łatwo skalowalnym elementem systemu.

### Controller

Po przejściu przez Nginx'a, request trafia do kontrolera, będącego centralną częścią systemu. Jest on odpowiedzialny za autoryzację oraz autentykację (w tym celu łączy się z CouchDB) oraz dalszą obsługę requestów trafiajacych do API.
Controller został zaimplementowany w Scali.

### CouchDB

CouchDB przechowuje pełne dane na temat stanu systemu. Przechowywane w nim są zarówno credentiale, metadane, a także definicje akcji, triggerów oraz reguł.

### Consul

Consul wykorzystywany jest do service discovery w ramach OpenWhisk'a. Jest on odpytywany przez kontroler w celu ustalenia, które z Invoker'ów są dostępne do wykonania akcji.

W połączeniu z Consul'em, wykorzystywany jest Registrator, którego zadaniem jest monitorowanie kontenerów Dockerowych oraz zapis tych danych w Consulu. 

### Kafka

Komunikacja między Invokerem a Controllerem odbywa się w całości przez Kafkę, która buforuje wiadomości wysyłane przez Controller. Kiedy wiadomość zostaje dostarczona, przez Contoller zwrócony zostaje ActivationId, który może następnie zostać wykorzystany do sprawdzenia rezultatu konkretnego wywołania.

Do zarządzania oraz utrzymywania Kafki wykorzystywany jest Apache Zookeeper.

### Invoker

Podobnie jak Controller, Invoker został zaimplementowany w Scali. Jego głównym zadaniem jest wykonywanie akcji wywoływanych przez Controller, do czego wykorzystuje kontenery Dockerowe.
Dla każdego wywołania, Invoker przygotowuje kontener w którym akcja zostanie wykonana, wstrzykuje do niego kod akcji, który następnie jest wykonywany z podanymi parametrami. Po zwróceniu rezultatu i zapisaniu go w CouchDB, kontener zostaje zniszczony.

Możliwe jest również utrzymywanie "gorących" kontenerów, które pozwalają zaoszczędzić czas przy wielokrotnym wywoływaniu tych samych akcji.


## OpenWhisk na e24cloud.com

### Przygotowanie serwera

Do przetestowania OpenWhisk'a w akcji, wykorzystałem serwer na e24cloud o następujących parametrach:

- Rdzenie: 1
- Pamięć RAM: 2GB (Minimalna ilość RAMu na jakiej udało mi się pomyślnie uruchomić wszystkie komponenty OpenWhisk'a)
- Dysk: 40GB HDD
- OS: Ubuntu 14.04 (nic nie stoi na przeszkodzie by użyc innego systemu operacyjnego)

Jako że był to mój pierwszy serwer na e24cloud, musiałem również wygenerować parę kluczy ssh.

Po kilku sekundach od utworzenia serwera otrzymałem mailowo informację o tym, że mój serwer jest już gotowy do użytku, także nie pozostało nic innego niż zalogować się na niego i przystąpić do instalacji OpenWhisk'a.

### Instalacja OpenWhisk

TODO 