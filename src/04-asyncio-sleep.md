# Exemple n°4 : Modélisons le serveur du *fast food* avec `asyncio`

Maintenant que nous avons compris comment fonctionne sa boucle événementielle,
il est temps de nous mettre en jambes avec `asyncio` en l'utilisant pour
modéliser l'exemple du début de cet article : l'employé de *fast food*.

Voici d'abord à quoi ressemble le programme dans sa version synchrone.

```python
from time import sleep
from datetime import datetime

def get_burger(client):
    print("> Commande du burger pour '{}' en cuisine".format(client))
    sleep(4)
    print("< Le burger de '{}' est prêt".format(client))

def get_fries(client):
    print("> Mettre des frites pour à cuire pour {}".format(client))
    sleep(8)
    print("< Les frites de '{}' sont prêtes".format(client))

def get_soda(client):
    print("> Remplissage du gobelet de soda pour {}".format(client))
    sleep(2)
    print("< Le soda de '{}' est prêt".format(client))

def serve(client):
    print("Préparation de la commande de '{}'".format(client))
    start = datetime.now()
    get_burger(client)
    get_fries(client)
    get_soda(client)
    duration = datetime.now() - start
    print("Commande de '{}' prête en {}".format(client, duration))
```

Résultat : la commande est prête en 14 secondes.

```python
>>> serve('A')
Préparation de la commande de 'A'
> Commande du burger pour 'A' en cuisine
< Le burger de 'A' est prêt
> Mettre des frites pour à cuire pour A
< Les frites de 'A' sont prêtes
> Remplissage du gobelet de soda pour A
< Le soda de 'A' est prêt
Commande de 'A' prête en 0:00:14.015165
```

Il ne nous reste plus qu'à implémenter cet exemple avec `asyncio`. Le premier
jet n'est pas très difficile : il faut juste penser à utiliser la coroutine
`asyncio.sleep()` plutôt que `time.sleep()` pour que l'endormissement de la
tâche ne soit pas bloquant.

```python
import asyncio
from datetime import datetime

@asyncio.coroutine
def get_burger(client):
    print("> Commande du burger pour {} en cuisine".format(client))
    yield from asyncio.sleep(4)
    print("< Le burger de {} est prêt".format(client))

@asyncio.coroutine
def get_fries(client):
    print("> Mettre des frites pour à cuire pour {}".format(client))
    yield from asyncio.sleep(8)
    print("< Les frites de {} sont prêtes".format(client))

@asyncio.coroutine
def get_soda(client):
    print("> Remplissage du gobelet de soda pour {}".format(client))
    yield from asyncio.sleep(2)
    print("< Le soda de {} est prêt".format(client))

@asyncio.coroutine
def serve(client):
    print("Préparation de la commande de {}".format(client))
    start = datetime.now()
    yield from asyncio.wait([
        get_burger(client),
        get_fries(client),
        get_soda(client),
    ])
    duration = datetime.now() - start
    print("Commande de {} prête en {}".format(client, duration))
```

Remarquez que nous avons décoré toutes nos coroutines avec le décorateur
`@asyncio.coroutine` : il ne s'agit que d'une convention pour reconnaître les
fonction synchrones des coroutines asynchrones dans du code utilisant
`asyncio`. En dehors de l'indice visuel, ce décorateur est *sans effet*.

Exécutons-le maintenant :

```python
>>> import asyncio
>>> loop = asyncio.get_event_loop()
>>> loop.run_until_complete(serve('A'))
Préparation de la commande de A
> Commande du burger pour A en cuisine
> Mettre des frites pour à cuire pour A
> Remplissage du gobelet de soda pour A
< Le soda de A est prêt
< Le burger de A est prêt
< Les frites de A sont prêtes
Commande de A prête en 0:00:08.005791
```

Nous sommes passés de 14 secondes à 8 secondes.

Et pour servir deux clients à la fois ?

```python
>>> loop.run_until_complete(asyncio.wait([serve('A'), serve('B')]))
Préparation de la commande de B
Préparation de la commande de A
> Mettre des frites pour à cuire pour B
> Commande du burger pour B en cuisine
> Remplissage du gobelet de soda pour B
> Remplissage du gobelet de soda pour A
> Commande du burger pour A en cuisine
> Mettre des frites pour à cuire pour A
< Le soda de B est prêt
< Le soda de A est prêt
< Le burger de B est prêt
< Le burger de A est prêt
< Les frites de B sont prêtes
< Les frites de A sont prêtes
Commande de B prête en 0:00:08.005967
Commande de A prête en 0:00:08.005940
```

Les deux ont mis tout autant de temps, néanmoins cet affichage ne semble pas
très réaliste.

En effet :

* la machine à soda ne peut faire couler qu'un seul soda à la fois, or ici les deux
  sodas ont été préparés simultanément.
* la cuisine ne comporte que 3 cuisiniers, donc elle peut seulement produire 3
  hamburgers simultanément
* le bac à frites fait cuire en général 5 portions de frites en même temps, or
  ici il a été utilisé deux fois simultanément en parallèle pour produire
  uniquement deux portions de frites.

Comment modéliser ces contraintes ?

La machine à soda est certainement la plus simple. Il est possible de
verrouiller une ressource de manière à ce qu'une seule tâche puisse y accéder à
la fois, en utilisant ce que l'on appelle un **verrou** (`asyncio.Lock`).
Plaçons un verrou sur notre machine à soda :

```python
SODA_LOCK = asyncio.Lock()

@asyncio.coroutine
def get_soda(client):
    # Acquisition du verrou
    with (yield from SODA_LOCK):
        # Une seule tâche à la fois peut exécuter ce bloc
        print("> Remplissage du gobelet de soda pour {}".format(client))
        yield from asyncio.sleep(2)
        print("< Le soda de {} est prêt".format(client))
```

Pour la cuisine, seuls 3 burgers peuvent être fabriqués en même temps. Cela
peut se modéliser en utilisant un **sémaphore** (`asyncio.Semaphore`), qui est
une sorte de "verrou multiple". On l'utilise pour qu'au plus N tâches
puissent exécuter un morceau de code à un instant donné :

```python
BURGER_SEM = asyncio.Semaphore(3)

@asyncio.coroutine
def get_burger(client):
    print("> Commande du burger pour {} en cuisine".format(client))
    with (yield from BURGER_SEM):
        print("* Le burger de {} est en préparation".format(client))
        yield from asyncio.sleep(4)
        print("< Le burger de {} est prêt".format(client))
```

Pour le bac à frites, nous allons prendre plus de risques. Si vous connaissez
la programmation concurrente au moyen de threads, vous réagirez sûrement à la
vue de cet exemple :

```python
FRIES_PARTS = 0
FRIES_LOCK = asyncio.Lock()

@asyncio.coroutine
def get_fries(client):
    global FRIES_PARTS
    with (yield from FRIES_LOCK):
        print(
            "> Récupération d'une portion de frites pour {}".format(client)
        )
        if FRIES_PARTS == 0:
            print("* Mettre des frites à cuire".format(client))
            yield from asyncio.sleep(8)
            FRIES_PARTS = 5
            print("* Les frites sont prêtes".format(client))
        FRIES_PARTS -= 1
        print("< Les frites de {} sont prêtes".format(client))
```

Dans cet exemple, on place un verrou sur le bac à frites pour qu'un seul
serveur puisse y accéder à la fois. Lorsqu'un serveur arrive devant le bac à
frites, soit celui-ci contient encore des portions de frites, auquel cas il en
récupère une et retourne immédiatement, soit le bac est vide, donc le serveur
met des frites à cuire avant de pouvoir en récupérer une portion.

Voyons voir ce que cela donne à l'exécution :

```python
>>> loop.run_until_complete(asyncio.wait([serve('A'), serve('B')]))
Préparation de la commande de B
Préparation de la commande de A
> Remplissage du gobelet de soda pour B
> Commande du burger pour B en cuisine
* Le burger de B est en préparation
> Récupération d'une portion de frites pour B
* Mettre des frites à cuire
> Commande du burger pour A en cuisine
* Le burger de A est en préparation
< Le soda de B est prêt
> Remplissage du gobelet de soda pour A
< Le burger de B est prêt
< Le burger de A est prêt
< Le soda de A est prêt
* Les frites sont prêtes
< Les frites de B sont prêtes
> Récupération d'une portion de frites pour A
< Les frites de A sont prêtes
Commande de B prête en 0:00:08.006742
Commande de A prête en 0:00:08.006859
({Task(<serve>)<result=None>, Task(<serve>)<result=None>}, set())
```

Et voilà. Nos deux tâches prennent le même temps, mais s'arrangent pour ne pas
accéder simultanément à la machine à sodas ni au bac à frites.

Voyons maintenant ce que cela donne si 10 clients passent commande en même
temps :

```python
>>> loop.run_until_complete(
...     asyncio.wait([serve(clt) for clt in 'ABCDEFGHIJ'])
... )
...
# ...
Commande de C prête en 0:00:08.009554
Commande de G prête en 0:00:08.009934
Commande de B prête en 0:00:08.010281
Commande de D prête en 0:00:08.014250
Commande de H prête en 0:00:10.017237
Commande de I prête en 0:00:16.014170
Commande de E prête en 0:00:16.014511
Commande de A prête en 0:00:16.022411
Commande de J prête en 0:00:18.023141
Commande de F prête en 0:00:20.026096
```

On se rend compte que les performances de notre serveur de fast-food se
dégradent plus ou moins, même si on est toujours loin des 140 secondes que le
serveur aurait pris s'il avait traité toutes ces commandes de façon synchrone.

Cela dit, il est plutôt rare que les clients passent leurs commandes tous en
même temps. Une modélisation plus proche de la réalité serait que ces dix
commandes arrivent à deux secondes d'intervalle :

```python
@asyncio.coroutine
def test():
    tasks = []
    for client in 'ABCDEFGHIJ':
        # appel équivalent à notre fonction `launch()`
        task = asyncio.async(serve(client))
        tasks.append(task)
        yield from asyncio.sleep(2)
    yield from asyncio.wait(tasks)
```

Dans ces conditions, les temps d'attente individuels de chaque client sont
assez largement réduits :

```python
>>> loop = asyncio.get_event_loop()
>>> loop = asyncio.run_until_complete(test())
Préparation de la commande de A
Préparation de la commande de B
Préparation de la commande de C
Préparation de la commande de D
Commande de A prête en 0:00:08.004599
Commande de B prête en 0:00:06.002353
Préparation de la commande de E
Commande de C prête en 0:00:04.002907
Préparation de la commande de F
Commande de D prête en 0:00:04.004343
Préparation de la commande de G
Commande de E prête en 0:00:04.004445
Préparation de la commande de H
Préparation de la commande de I
Commande de F prête en 0:00:08.005227
Commande de G prête en 0:00:06.005087
Préparation de la commande de J
Commande de H prête en 0:00:04.003595
Commande de I prête en 0:00:04.003620
Commande de J prête en 0:00:04.004184
```

À raison d'un client toutes les deux secondes, notre serveur est capable de
traiter des commandes avec un temps moyen de 5.2 secondes, le maximum étant 8
secondes et le minimum à 4 secondes.

Et si nous *stressions* un peu notre serveur et qu'on lui passait une commande
par seconde ?

```python
@asyncio.coroutine
def test():
    tasks = []
    for client in 'ABCDEFGHIJ':
        task = asyncio.async(serve(client))
        tasks.append(task)
        yield from asyncio.sleep(1)
    yield from asyncio.wait(tasks)
```

Comme prévu, celui-ci, sous la pression, est un peu moins efficace à cause des
contraintes des différentes machines qu'il utilise, même si l'on reste tout à
fait loin de l'inefficacité pathologique de la version synchrone :

```python
>>> loop = asyncio.get_event_loop()
>>> loop.run_until_complete(test())
Préparation de la commande de A
Préparation de la commande de B
Préparation de la commande de C
Préparation de la commande de D
Préparation de la commande de E
Préparation de la commande de F
Préparation de la commande de G
Préparation de la commande de H
Commande de A prête en 0:00:08.003498
Commande de B prête en 0:00:07.001989
Commande de C prête en 0:00:06.000267
Commande de D prête en 0:00:05.004673
Préparation de la commande de I
Préparation de la commande de J
Commande de E prête en 0:00:06.005059
Commande de F prête en 0:00:11.002432
Commande de G prête en 0:00:10.002475
Commande de H prête en 0:00:09.000938
Commande de I prête en 0:00:10.002911
Commande de J prête en 0:00:11.004703
```

On peut se demander à quel endroit le service est ralenti quand on sert 1
client par seconde :

* S'agit-il de la machine à sodas, qui produit un soda toutes les 2 secondes ?
* S'agit-il du bac à frites, qui peut produire 5 portions en 8 secondes ?
* Ou bien s'agit-il de la cuisine, qui produit un hamburger en 4 secondes, mais
  reste limitée à trois cuisiniers ?

Les réponses à ces questions vous sont laissées en guise d'exercice. Vous
pouvez essayer d'apporter les modifications suivantes au modèle, pour aider le
gérant du restaurant à optimiser son service :

* Mettre en place une seconde machine à sodas aussi rapide que la première.
* Acheter un nouveau bac à frites pouvant produire 6 portions en 7 secondes.
* Embaucher un quatrième cuisinier.

Bon courage !
