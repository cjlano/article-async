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
* notre coroutine `ensure_future()` devient la **fonction**
  [`asyncio.ensure_future()`](https://docs.python.org/3/library/asyncio-task.html#asyncio.ensure_future).
  Notez toutefois que cette fonction n'existe que depuis Python 3.4.4. Dans les
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
    total = datetime.now() - start_time
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
  qui ne supporte pas les accès concurrents* ;
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
            print("   ** Démarrage de la cuisson des frites")
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

À l'exécution :

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

Nos deux tâches prennent toujours le même temps à s'exécuter, mais
s'arrangent pour ne pas accéder simultanément à la machine à sodas ni au bac à
frites.

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

Cela n'a rien de surprenant. En fait, les performances d'une application
asynchrone ne se mesurent pas en *nombre de tâches traitées simultanément*,
mais plutôt, comme n'importe quel serveur, en *nombre de tâches traitées dans
le temps*. Il est évident que si 10 clients viennent manger dans un fast-food,
il y a relativement peu de chances qu'ils arrivent tous en même temps : ils
vont plutôt passer leur commande à raison d'une par seconde, par exemple.

Par contre, il est très important de noter que c'est bien *le temps d'attente*
individuel de chaque client qui compte pour mesurer les performances (la
qualité) du service. Si un client attend trop longtemps, il ne sera pas
satisfait, peu importe s'il est tout seul dans le restaurant ou que celui-ci
est bondé.

Pour ces raisons, il faut que nous ayons une idée des **objectifs de
performances** de notre serveur, c'est-à-dire que nous fixions, comme but :

* un *temps d'attente maximal* à ne pas dépasser pour servir un client,
* un *volume* de requêtes à tenir par seconde.

Écrivons maintenant une coroutine pour tester les performances
de notre serveur :

```python
# La fonction ensure_future est définie à partir de Python 3.4.4
# Ce bloc la rend accessible pour toutes les versions de Python 3.4
try:
    from asyncio import ensure_future
except ImportError:
    asyncio.ensure_future = asyncio.async

@asyncio.coroutine
def perf_test(nb_requests, period, timeout):
    tasks = []
    # On lance 'nb_requests' commandes à 'period' secondes d'intervalle
    for idx in range(1, nb_requests + 1):
        client_name = "client_{}".format(idx)
        tsk = asyncio.ensure_future(serve(client_name))
        tasks.append(tsk)
        yield from asyncio.sleep(period)

    finished, _ = yield from asyncio.wait(tasks)
    success = set()
    for tsk in finished:
        if tsk.result().seconds < timeout:
            success.add(tsk)

    print("{}/{} clients satisfaits".format(len(success), len(finished)))
```

Cette coroutine va lancer un certain nombre de commandes, régulièrement, et
compter à la fin le nombre de commandes qui ont été honorées dans les temps.

Essayons de lancer 10 commandes à 1 seconde d'intervalle, avec pour
objectif que les clients soient servis en 5 secondes maximum :

```python
>>> loop.run_until_complete(perf_test(10, 1, 5))
# ... sortie filtrée ...
<= client_1 servi en 0:00:04.004044
<= client_2 servi en 0:00:03.002792
<= client_3 servi en 0:00:03.003338
<= client_4 servi en 0:00:03.003653
<= client_5 servi en 0:00:03.003815
<= client_6 servi en 0:00:04.003746
<= client_7 servi en 0:00:03.003412
<= client_8 servi en 0:00:03.002512
<= client_9 servi en 0:00:03.003409
<= client_10 servi en 0:00:03.003622
10/10 clients satisfaits
```

Ce test nous indique que notre serveur tient facilement une charge d'un client
par seconde. Essayons de monter en charge en passant à deux clients par
seconde :

```python
>>> loop.run_until_complete(perf_test(10, 0.5, 5))
# ... sortie filtrée ...
<= client_1 servi en 0:00:04.002629
<= client_2 servi en 0:00:03.502093
<= client_3 servi en 0:00:03.002863
<= client_4 servi en 0:00:04.500168
<= client_5 servi en 0:00:04.500226
<= client_6 servi en 0:00:05.499894
<= client_7 servi en 0:00:05.999704
<= client_8 servi en 0:00:05.998824
<= client_9 servi en 0:00:05.999883
<= client_10 servi en 0:00:07.498776
5/10 clients satisfaits
```

À deux clients par seconde, notre serveur n'offre plus de performances
satisfaisantes pour la moitié des commandes.

Nous pouvons donc poser le problème d'optimisation suivant : le gérant du
restaurant veut devenir capable de servir 2 clients par seconde avec un temps
de traitement inférieur à 5 secondes par commande. Pour cela, il peut :

* Acheter de nouvelles machines à sodas ;
* Embaucher de nouveaux cuisiniers ;
* Remplacer son bac à frites (capable de cuire 5 portions en 4 secondes) par un
  nouveau, qui peut faire cuire 7 portions en 4 secondes.

Évidemment, chacune de ces solutions a un coût, donc il est préférable pour le
gérant de n'apporter que le moins possible de modifications pour tenir son
objectif. Si l'on voulait faire un parallèle avec une application réelle :

* Acheter une seconde machine à sodas coûterait l'occupation à 100% d'un cœur
  de CPU supplémentaire + une augmentation de 100% de la RAM consommée par le
service "soda".
* Embaucher un quatrième cuisinier coûterait un cœur de CPU supplémentaire +
  une augmentation de 33% de la RAM consommée par le service "cuisine".
* Le remplacement du bac à frites augmenterait uniquement de 40% la
  consommation de RAM de ce service…

En guise d'exercice, vous pouvez vous amuser à modifier les contraintes de
notre programme en conséquence pour observer l'impact de vos modifications sur
les performances du serveur : vous vous trouverez alors véritablement dans la
peau d'un architecte système, le salaire et le stress en moins. ;)
