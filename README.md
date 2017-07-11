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

### Przygotowanie środowiska dla OpenWhisk'a

W pierwszym kroku, instalujemy `git`, który będzie nam potrzebny do sklonowania repozytorium z Apache OpenWhisk.
```
apt-get update && apt-get install -y git
```

Następnie, klonujemy repozytorium z OpenWhisk'iem do katalogu `openwhisk`.
```
git clone https://github.com/apache/incubator-openwhisk.git openwhisk
```

Po pomyślnym sklonowaniu repozytorium, instalujemy potrzebne narzędzia(m.in. Docker, Ansible, Java8, Scala) za pomocą komendy:
```
./openwhisk/tools/ubuntu-setup/all.sh
```

Podczas wykonywania tego kroku, należy uzbroić się w cierpliwość gdyż może on potrwać od kilku do kilkunastu minut.

### Deployment OpenWhisk'a


#### Konfiguracja data store (CouchDB)

Na potrzeby prezentacji, jako data store wykorzystany została wykorzystana ulotna instancja CouchDB uruchomiona jako kontener dockerowy.
Do wygenerowania konfiguracji dla CouchDB wykorzystany został playbook anisble:
```
cd openwhisk/ansible
ansible-playbook setup.yml
```
W rezultacie, wygenerowany został plik `db_local.ini`, zawierający credentiale do CouchDB.
```
[db_creds]
db_provider=CouchDB
db_username=whisk_admin
db_password=XXXXXXXXX
db_protocol=http
db_host=172.17.0.1
db_port=5984
```

#### Instalacja prerequisites

Do deploymentu potrzebne nam będą jeszcze narzędzia instalowane przy pomocy:
```
cd openwhisk/ansible
ansible-playbook prereq.yml
```

#### Budowanie OpenWhisk'a

Po instalacji potrzebnych narzędzi możemy przystąpić do budowania OpenWhisk'a za pomocą `gradle`:
```
cd openwhisk
./gradlew distDocker
```

Powyższa komenda jest czasochłonna (przy pierwszej pomyślnej próbie zajęło mi to prawie 40 min), więc należy się uzbroić w cierpliwość.
W przypadku maszyny z ilością RAMu mniejszą niż 2 GB nie udawało się pomyślnie ukończyć procesu budowania.


#### Deploy

Po zakończonym procesie budowania, przechodzimy do deploymentu OpenWhisk'a.

W pierwszym kroku deploymentu, postawiony zostanie kontener z CouchDB, przy pomocy komendy:

```
cd openwhisk/ansible
ansible-playbook couchdb.yml
```

Po ukończeniu procesu, przy pomocy komendy `docker ps` możemy potwierdzić, że kontener został uruchomiony poprawnie.
```
docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
6473a644f862        couchdb:1.6         "tini -- /docker-entr"   33 seconds ago      Up 32 seconds       0.0.0.0:5984->5984/tcp   couchdb
```

Następnym krokiem, który musi zostać wykonany zawsze po nowym deploymencie CouchDB jest wykonanie inicjalizacji bazy danych *_subjects.

```
cd openwhisk/ansible
ansible-playbook initdb.yml
```

Poprawnosć wykonania komendy możemy potwierdzić korzystając z API CouchDB:

```
curl localhost:5984/_all_dbs
["_replicator","_users","root_ip-193-187-65-75_subjects"]
```

W odpowiedzi widzimy nowo utworzoną bazę danych `root_ip-193-187-65-75_subjects`.

Następnie, wykonujemy komendę mającą na celu stworzenie/wyczyszczenie baz danych `whisks` i `activations`:
Operację tę należy wykonać tylko podczas nowego deploymentu, w przeciwnym wypadku stracimy uprzednio stworzone akcje (baza `whisks`) i dane na temat aktywacji (baza `activations`). 

```
cd openwhisk/ansible
ansible-playbook wipe.yml
```

Rezultat możemy zaobserwować korzystając z API CouchDB:
```
curl localhost:5984/_all_dbs
["_replicator","_users","root_ip-193-187-65-75_activations","root_ip-193-187-65-75_subjects","root_ip-193-187-65-75_whisks"]
```

W odpowiedzi widzimy nowo utworzone bazy danych `root_ip-193-187-65-75_activations` oraz `root_ip-193-187-65-75_whisks`.

Następnym krokiem będzie deployment API Gateway, które pozwoli na wywoływanie akcji za pomocą HTTP API.
Do tego celu ponownie wykorzystujemy ansible:

```
cd openwhisk/ansible
anisble-playbook apigateway.yml
```

Potwierdzenie pomyślnego deployu gateway'a:

```
docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED              STATUS              PORTS                                                              NAMES
b0390aba1e73        openwhisk/apigateway:0.8.2   "/usr/local/bin/dumb-"   About a minute ago   Up About a minute   80/tcp, 8423/tcp, 0.0.0.0:9000->9000/tcp, 0.0.0.0:9001->8080/tcp   apigateway
28a3797d8bbd        redis:3.2                    "docker-entrypoint.sh"   2 minutes ago        Up 2 minutes        0.0.0.0:6379->6379/tcp                                             redis
6473a644f862        couchdb:1.6                  "tini -- /docker-entr"   14 minutes ago       Up 14 minutes       0.0.0.0:5984->5984/tcp                                             couchdb
```

Jak widać powyżej, uruchomione zostały kontenery z Redisem, wykorzystywanym przez API Gateway oraz kontener z API Gateway.

Przedostatnim krokiem będzie deployment samego 'core' OpenWhisk'a:

```
cd openwhisk/ansible
ansible-playbook openwhisk.yml
```

W celu potwierdzenia pomyślnego deployu, ponownie uzywamy komendy `docker ps`:

```
docker ps
CONTAINER ID        IMAGE                        COMMAND                  CREATED              STATUS              PORTS                                                                                                                                                  NAMES
032687e350d4        nginx:1.11                   "nginx -g 'daemon off"   29 seconds ago       Up 27 seconds       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:8443->8443/tcp                                                                                       nginx
fa20cfb8ef70        whisk/nodejs6action:latest   "/bin/sh -c 'node --e"   About a minute ago   Up About a minute                                                                                                                                                          wsk0_6_warmJsContainer_20170710T192207Z
6f68c77a0d33        whisk/nodejs6action:latest   "/bin/sh -c 'node --e"   About a minute ago   Up About a minute                                                                                                                                                          wsk0_4_warmJsContainer_20170710T192201010Z
1578b1903ddf        whisk/nodejs6action:latest   "/bin/sh -c 'node --e"   About a minute ago   Up About a minute                                                                                                                                                          wsk0_2_warmJsContainer_20170710T192159093Z
9da6c002dcf7        whisk/nodejs6action:latest   "/bin/sh -c 'node --e"   About a minute ago   Up About a minute                                                                                                                                                          wsk0_1_warmJsContainer_20170710T192157221Z
a80a0323b395        whisk/invoker:latest         "/bin/sh -c 'exec /in"   About a minute ago   Up About a minute   0.0.0.0:12001->8080/tcp                                                                                                                                invoker0
b919f46ab2fb        whisk/controller:latest      "/bin/sh -c 'controll"   2 minutes ago        Up 2 minutes        0.0.0.0:10001->8080/tcp                                                                                                                                controller0
a45ede624db9        ches/kafka:0.10.2.1          "/start.sh"              2 minutes ago        Up 2 minutes        7203/tcp, 0.0.0.0:9092->9092/tcp                                                                                                                       kafka
a8d34b033003        zookeeper:3.4                "/docker-entrypoint.s"   4 minutes ago        Up 4 minutes        2888/tcp, 0.0.0.0:2181->2181/tcp, 3888/tcp                                                                                                             zookeeper
55acf30bbc50        gliderlabs/registrator       "/bin/registrator -ip"   4 minutes ago        Up 4 minutes                                                                                                                                                               registrator
cb65c996373e        consul:0.7.0                 "docker-entrypoint.sh"   4 minutes ago        Up 4 minutes        0.0.0.0:8300-8302->8300-8302/tcp, 0.0.0.0:8400->8400/tcp, 0.0.0.0:8301-8302->8301-8302/udp, 0.0.0.0:8500->8500/tcp, 0.0.0.0:8600->8600/udp, 8600/tcp   consul
b0390aba1e73        openwhisk/apigateway:0.8.2   "/usr/local/bin/dumb-"   8 minutes ago        Up 8 minutes        80/tcp, 8423/tcp, 0.0.0.0:9000->9000/tcp, 0.0.0.0:9001->8080/tcp                                                                                       apigateway
28a3797d8bbd        redis:3.2                    "docker-entrypoint.sh"   9 minutes ago        Up 9 minutes        0.0.0.0:6379->6379/tcp                                                                                                                                 redis
6473a644f862        couchdb:1.6                  "tini -- /docker-entr"   21 minutes ago       Up 21 minutes       0.0.0.0:5984->5984/tcp                                                                                                                                 couchdb
```

Jak widzimy powyżej, uruchomiona została znaczna liczba kontenerów, takich jak Nginx, Controller, Invoker, Kafka, Zookeeper, Consul oraz Registrator.
Oprócz tego uruchomione zostały 4 "gorące" kontenery służące jako środowisko uruchomieniowe dla akcji napisanych w Node.js.


Ostatnim krokiem deploymentu jest wykonanie komendy:

```
cd openwhisk/ansible
ansible-playbook postdeploy.yml
```

#### Konfiguracja CLI

Po deploymencie, przystępujemy do konfiguracji CLI, za pomocą którego będziemy używać OpenWhisk'a.
CLI stworzone podczas deploymentu znajduje się pod ścieżką `openwhisk/bin/wsk`.
Do używania CLI, wymagane jest ustawienie dwóch parametrów:
- API host - nazwa lub adres IP
- Authorization Key (nazwa użytkownika i hasło) wykorzystywane do autoryzacji przez API
 
Za pomocą poniższej komendy, ustawiony zostaje API host:

```
./openwhisk/bin/wsk property set --apihost <adres IP>
```

Na potrzeby testu, jako klucz autoryzacyjny, wykorzystany zostanie klucz dla konta `guest`, znajdujący się w katalogu `openwhisk/ansible/files/auth.guest`.
Za pomocą poniższej komendy ustawiony zostaje klucz autoryzacyjny.
```
./openwhisk/bin/wsk property set --auth `cat ./openwhisk/ansible/files/auth.guest`
```

Property przechowywane są przez OpenWhisk'a pod ścieżką `~/.wskprops`, gdzie możemy zweryfikować, że udało nam się je poprawie ustawić.

### Tworzenie oraz wywoływanie akcji w OpenWhisk

Mając skonfigurowane CLI, możemy przystąpić do stworzenia pierwszej akcji w OpenWhisk. 
W tym celu, tworzymy pod ścieżką `~/python-example/action.py` plik z kodem Pythonowym:

```python
def main(args):
    name = args.get("name", "stranger")
    greeting = "Hello from OpenWhisk on e24cloud, " + name + "!"
    print(greeting)
    return {"greeting": greeting}
```

Następnie, przy pomocy CLI tworzymy akcję o nazwie `pyaction`:

```
./openwhisk/bin/wsk -i action create pyaction ./python-example/action.py
```

W celu przetestowania, wywołujemy stworzoną akcję z poziomu CLI:
```
./openwhisk/bin/wsk -i action invoke --result pyaction 
```
Rezultat:
```
{
    "greeting": "Hello from OpenWhisk on e24cloud, stranger!"
}
```

Wywołanie z przekazaniem parametru do funkcji:
```
./openwhisk/bin/wsk -i action invoke --result pyaction --param name Chmuromaniak
```
Rezultat:
```
{
    "greeting": "Hello from OpenWhisk on e24cloud, Chmuromaniak!"
}
```

Jak widzimy, nasza akcja wykonywana jest poprawnie!

Korzystając z komendy `docker ps`, możemy również zaobserwować, że został utworzony kontener dockerowy, służący jako środowisko uruchomieniowe dla naszej akcji.

```
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                                                                                                                                                  NAMES
53255ce5b309        whisk/python2action:latest   "/bin/bash -c 'cd pyt"   3 minutes ago       Up 3 minutes                                                                                                                                                               wsk0_247_guestpyaction001_20170711T153039406Z
```

### Wywoływanie akcji przez API Gateway

Kolejnym punktem będzie dodanie możliwości wykonywania akcji za pomocą API Gateway:

Do tego celu stworzymy nową akcję, tym razem z kodem w Node.js, znajdującym się pod ścieżką `./js-example/action.js`.

```
function main({name:name='Serverless API on e24cloud'}) {
  return {payload: `Hello from ${name}`};
}
```

Przy pomocy CLI, tworzymy akcję o nazwie `jsaction`:

```
./openwhisk/bin/wsk -i action create jsaction ./js-example/action.js --web true
```

Nalezy zwrócić uwagę, że w tym wypadku, akcja została stworzona z flagą `--web true`.

Nastepnie, korzystając z `wsk`, tworzymy API dla naszej akcji:

```
./openwhisk/bin/wsk -i api create /hello /world get jsaction --response-type json
```

W rezultacie powinniśmy ujrzeć URL do naszej akcji:
```
ok: created API /hello/world GET for action /_/jsaction
http://172.17.0.1:9001/api/23bc46b1-71f6-4ed5-8c54-816aa4f8c502/hello/world
```

Aby przetestować czy wywoływanie akcji poprzez API Gateway działa, wykonujemy odpowiedni request:

```
curl http://172.17.0.1:9001/api/23bc46b1-71f6-4ed5-8c54-816aa4f8c502/hello/world
```

Rezultat:
```
{
  "payload": "Hello from Serverless API on e24cloud"
}
```

Jak widzimy w rezultacie, otrzymana odpowiedź zgadza się z oczekiwaną wartością zwróconą przez przygotowaną akcję.

