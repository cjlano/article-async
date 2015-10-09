# Exemple n°4 : Modélisons le serveur du *fast food* avec `asyncio`

Maintenant que nous avons fait le tour de toutes les primitives qui permettent
la programmation asynchrone, il est temps pour nous d'étudier un système
asynchrone programmé avec `asyncio`. Afin d'éviter d'alourdir le propos dans
cet exemples, nous nous contenterons de notre exemple *fil rouge* : l'employé
de fast food, en nous promettant d'aborder des applications réseau réelles dans
un article ultérieur.

Commençons par implémenter celui-ci avec `asyncio`.

En réalité, vous n'allez pas tellement être dépaysés puisque le framework
standard reprend plus ou moins la même API que celle que nous avons développé
dans les trois derniers exemples. Les seules différences sont que :

* Toutes les coroutines doivent être décorées par `@asyncio.coroutine`, ce qui
  permet notamment de créer des coroutines qui ne `yield`-ent jamais.
* `wait()` devient
  [`asyncio.wait()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.wait).
* `async_sleep()` devient
  [`asyncio.sleep()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep).
* `ensure_future()` devient
  [`asyncio.ensure_future()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.ensure_future).
  Notez toutefois que cette coroutine n'existe que depuis Python 3.4.4. Dans les
  versions antérieures, celle-ci s'appelle `asyncio.async()`, mais son nom a été
  déprécié au profit de `ensure_future()` afin de libérer le mot-clé `async` pour
  Python 3.5.

```python
import asyncio
from datetime import datetime

@asyncio.coroutine
def get_soda(client):
    print("  > Remplissage du soda pour {}".format(client))
    yield from asyncio.sleep(1)
    print("  < Le soda de {} est prêt".format(client))

@asyncio.coroutine
def get_fries(client):
    print("    > Démarrage de la cuisson des frites pour {}".format(client))
    yield from asyncio.sleep(4)
    print("    < Les frites de {} sont prêtes".format(client))

@asyncio.coroutine
def get_burger(client):
    print("    > Commande du burger en cuisine pour {}".format(client))
    yield from asyncio.sleep(3)
    print("    < Le burger de {} est prêt".format(client))

@asyncio.coroutine
def serve(client):
    print("=> Commande passée par {}".format(client))
    start_time = datetime.now()
    yield from asyncio.wait(
        [
            get_soda(client),
            get_fries(client),
            get_burger(client)
        ]
    )
    print("<= {} servi en {}".format(client, datetime.now() - start_time))
```

Rien de franchement dépaysant.
Pour exécuter ce code, là aussi l'API est sensiblement la même que notre classe
`Loop` :

```python
>>> loop = asyncio.get_event_loop()
>>> loop.run_until_complete(serve("A"))
=> Commande passée par A
    > Remplissage du soda pour A
    > Commande du burger en cuisine pour A
    > Démarrage de la cuisson des frites pour A
    < Le soda de A est prêt
    < Le burger de A est prêt
    < Les frites de A sont prêtes
<= A servi en 0:00:04.003105
```

Pas d'erreur de syntaxe, le code fonctionne. On peut commencer à travailler.

Remarquons dans un premier temps que notre serveur **manque de réalisme**. En
effet, si nous lui demandons de servir deux clients en même temps, voilà ce qui
se produit :

```python
>>> loop.run_until_complete(
...     asyncio.wait([serve("A"), serve("B")])
... )
=> Commande passée par A
=> Commande passée par B
    > Remplissage du soda pour A
    > Commande du burger en cuisine pour A
    > Démarrage de la cuisson des frites pour A
    > Démarrage de la cuisson des frites pour B
    > Remplissage du soda pour B
    > Commande du burger en cuisine pour B
    < Le soda de A est prêt
    < Le soda de B est prêt
    < Le burger de A est prêt
    < Le burger de B est prêt
    < Les frites de A sont prêtes
    < Les frites de B sont prêtes
<= A servi en 0:00:04.002609
<= B servi en 0:00:04.002792
```

Les deux commandes ont été servies simultanément, de la même façon. La
préparation des trois ingrédients s'est chevauchée, comme s'il était possible
de faire couler une infinité de sodas, de cuire une infinité de
frites *à la demande* pour les clients, et de préparer une infinité de
hamburgers en parallèle.

En bref : **notre modélisation manque de contraintes**.

Pour améliorer ce programme, nous allons modéliser les contraintes suivantes :

* La machine à sodas ne peut faire couler **qu'un seul soda à la fois**. Dans
  une application réelle, cela reviendrait à *requêter un service synchrone
qui ne supporte pas l'accès parallèle* ;
* Il n'y a que 3 cuisiniers dans le restaurant, donc **on ne peut pas préparer
  plus de trois hamburgers en même temps**. Dans la réalité, cela revient à
*requêter un service synchrone dont trois instances tournent en parallèle* ;
* Le bac à frites s'utilise en faisant cuire 5 portions de frites d'un coup,
  pour servir ensuite 5 clients instantanément. Dans la réalité, cela revient,
  à peu de choses près, à *simuler un service synchrone qui fonctionne avec un
  cache*.

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
        print("    > Remplissage du soda pour {}".format(client))
        yield from asyncio.sleep(1)
        print("    < Le soda de {} est prêt".format(client))
```

Le `with (yield from SODA_LOCK)` signifie que lorsque le serveur arrive à la
machine à soda pour y déposer un gobelet :

* soit la machine est libre (déverrouillée), auquel cas il peut la verrouiller
  pour l'utiliser immédiatement,
* soit celle-ci est déjà en train de fonctionner, auquel cas il attend que le
  soda en cours de préparation soit prêt avant de verrouiller la machine à son
  tour.

Passons à la cuisine. Seuls 3 burgers peuvent être fabriqués en même temps. Cela
peut se modéliser en utilisant un **sémaphore** (`asyncio.Semaphore`), qui est
une sorte de "verrou multiple". On l'utilise pour qu'au plus N tâches
puissent exécuter un morceau de code à un instant donné.

```python
BURGER_SEM = asyncio.Semaphore(3)

@asyncio.coroutine
def get_burger(client):
    print("    > Commande du burger en cuisine pour {}".format(client))
    with (yield from BURGER_SEM):
        yield from asyncio.sleep(3)
        print("    < Le burger de {} est prêt".format(client))
```

Le `with (yield from BURGER_SEM)` veut dire que lorsqu'une commande est passée
en cuisine :

* soit il y a un cuisinier libre, et celui-ci commence immédiatement à
  préparer le hamburger,
* soit tous les cuisiniers sont occupés, auquel cas on attend qu'il y en ait un
  qui se libère pour s'occuper de notre hamburger.

Passons enfin au bac à frites. Cette fois, `asyncio` ne nous fournira pas
d'objet magique, donc il va nous falloir réfléchir un peu plus. Il faut que
l'on puisse l'utiliser *une fois* pour faire les frites des 5 prochaines
commandes. Dans ce cas, un compteur semble une bonne idée :

* Chaque fois que l'on prend une portion de frites, on décrémente le compteur ;
* S'il n'y a plus de frites dans le bac, il faut en refaire.

Mais attention, si les frites sont déjà en cours de préparation, il est inutile de
lancer une nouvelle fournée !

Voici comment on pourrait s'y prendre :

```python
FRIES_COUNTER = 0
FRIES_LOCK = asyncio.Lock()

@asyncio.coroutine
def get_fries(client):
    global FRIES_COUNTER
    with (yield from FRIES_LOCK):
        print("    > Récupération des frites pour {}".format(client))
        if FRIES_COUNTER == 0:
            print("  ** Démarrage de la cuisson des frites")
            yield from asyncio.sleep(4)
            FRIES_COUNTER = 5
            print("   ** Les frites sont cuites")
        FRIES_COUNTER -= 1
        print("    < Les frites de {} sont prêtes".format(client))
```

Dans cet exemple, on place un verrou sur le bac à frites pour qu'un seul
serveur puisse y accéder à la fois. Lorsqu'un serveur arrive devant le bac à
frites, soit celui-ci contient encore des portions de frites, auquel cas il en
récupère une et retourne immédiatement, soit le bac est vide, donc le serveur
met des frites à cuire avant de pouvoir en récupérer une portion.

Voyons voir ce que cela donne à l'exécution :

```python
>>> loop.run_until_complete(asyncio.wait([serve('A'), serve('B')]))
=> Commande passée par B
=> Commande passée par A
    > Remplissage du soda pour B
    > Récupération des frites pour B
  ** Démarrage de la cuisson des frites
    > Commande du burger en cuisine pour B
    > Commande du burger en cuisine pour A
    < Le soda de B est prêt
    > Remplissage du soda pour A
    < Le soda de A est prêt
    < Le burger de B est prêt
    < Le burger de A est prêt
   ** Les frites sont cuites
    < Les frites de B sont prêtes
    > Récupération des frites pour A
    < Les frites de A sont prêtes
<= B servi en 0:00:04.003111
<= A servi en 0:00:04.003093
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
# ... sortie filtrée ...
<= C servi en 0:00:04.004512
<= D servi en 0:00:04.004378
<= E servi en 0:00:04.004262
<= F servi en 0:00:06.008072
<= A servi en 0:00:06.008074
<= G servi en 0:00:08.006399
<= H servi en 0:00:09.009187
<= B servi en 0:00:09.009118
<= I servi en 0:00:09.015023
<= J servi en 0:00:12.011539
```

On se rend compte que les performances de notre serveur de fast-food se
dégradent : certains clients attendent jusqu'à trois fois plus longtemps que
les autres.



------------------
**/!\\ Le texte est à refondre à partir d'ici**




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
