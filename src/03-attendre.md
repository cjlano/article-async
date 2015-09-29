# Exemple n°3 : Attendre de façon asynchrone

À la fin du précédent exemple, nous avons implémenté un petit mécanisme
*d'annulation* de tâches en cours d'exécution, qui permet d'éviter qu'une tâche
*fille* survive à sa tâche *mère* en s'exécutant plus longtemps qu'elle.
Cependant, il n'est peut-être pas toujours adéquat d'annuler brutalement une
tâche. Par exemple, on peut imaginer que si celle-ci a des ressources à
libérer, l'annuler risque de créer une fuite de mémoire…

Pour cette raison, on veut se donner un moyen *d'attendre* (de façon
asynchrone) qu'une tâche ait fini de s'exécuter. En soi, cela n'est pas bien
difficile, puisqu'il suffit de `yield`-er *tant que* l'événement que
nous attendons ne s'est pas encore produit :

```python
def example():
    print("Tâche 'example'")
    print("Lancement de la tâche 'subtask'")
    sub = yield from ensure_future(subtask())
    print("Retour dans 'example'")
    for _ in range(3):
        print("(example)")
        yield

    print("En attente de la fin de 'subtask'")
    while not sub.is_done() and not sub.is_cancelled():
        yield
    print("Ça y est, je peux quitter")

def subtask():
    print("Tâche 'subtask'")
    for _ in range(5):
        print("(subtask)")
        yield
```

Essayons :

```python
>>> event_loop = Loop()
>>> event_loop.run_until_complete(example())
Tâche 'example'
Lancement de la tâche 'subtask'
Retour dans 'example'
(example)
Tâche 'subtask'
(subtask)
(example)
(subtask)
(example)
(subtask)
En attente de la fin de 'subtask'
(subtask)
(subtask)
<Task 'subtask' [FINISHED] (None)>
Ça y est, je peux quitter
<Task 'example' [FINISHED] (None)>
```

C'est plutôt enfantin, en fait.

Souvenez-vous maintenant de la coroutine `wait()` d'`asyncio` que nous avons
aperçue dans l'exemple n°1 :

```python3
>>> import asyncio
>>> loop = asyncio.get_event_loop()
>>> loop.run_until_complete(asyncio.wait([tic_tac(), spam()]))
Tic
spam
Tac
eggs
bacon
({Task(<tic_tac>)<result='Boum!'>, Task(<spam>)<result='spam'>}, set())
```

Celle-ci permet d'attendre qu'une ou plusieurs tâches (lancées en parallèle)
soient terminées avant de rendre la main. Sa valeur de retour se compose de
deux ensembles (`set()`) :

* Le premier contient les tâches qui se sont terminées normalement ;
* Le second contient les tâches qui ont été annulées ou ont quitté sur une
  erreur.

C'est cette fonction que nous allons implémenter maintenant, parce qu'il serait
vraiment dommage de se passer de sa souplesse d'utilisation !

En fait, le plus difficile dans cette fonction est surtout sa partie
cosmétique. Pour avoir un comportement souple, il faut gérer le cas où les
tâches passées à cette fonction sont des coroutines qui n'ont pas encore été
lancées, ou bien des instances de la classe `Task` que la boucle événementielle
aurait préalablement créées pour programmer leur exécution.

Voilà ce que cela peut donner :

```python
def wait(tasks):
    # On commence par séparer les tâches en cours d'exécution des autres
    running, to_launch, finished, error = set(), set(), set(), set()
    for task in tasks:
        if isinstance(task, Task):
            if task.status == STATUS_FINISHED:
                finished.add(task)
            elif task.status in {STATUS_ERROR, STATUS_CANCELLED}:
                error.add(task)
            else:
                running.add(task)
        else:
            to_launch.add(task)

    # On lance les tâches qui ont besoin d'être lancées
    if to_launch:
        launched = yield CALL_SCHEDULE, to_launch
        running.update(launched)

    # On attend que tout le monde ait fini de s'exécuter
    while running:
        # On itère sur une copie immuable de l'ensemble 'running'
        # de façon à pouvoir modifier celui-ci sans risque dans la boucle
        for task in tuple(running):
            if not (task.is_done() or task.is_cancelled()):
                continue
            running.remove(task)
            if task.status == STATUS_FINISHED:
                finished.add(task)
            else:
                error.add(task)

        # S'il y a encore des tâches inachevées, on suspend la coroutine
        if running:
            yield

    return finished, error
```

Vérifions que nous avons bien un comportement similaire à `asyncio` :

```python
>>> event_loop = Loop()
>>> event_loop.run_until_complete(wait([tic_tac(), spam()]))
Tic
Spam
Tac
Eggs
<Task 'tic_tac' [FINISHED] ('Boom!')>
Bacon
<Task 'spam' [FINISHED] ('SPAM!')>
<Task 'wait' [FINISHED] (({<Task 'tic_tac' [FINISHED] ('Boom!')>, <Task 'spam' [FINISHED] ('SPAM!')>}, set()))>
```

Pas mal ! Et pour attendre une tâche préalablement démarrée ?

```python
>>> def example():
...     print("Tâche 'example'")
...     print("Lancement de la tâche 'subtask'")
...     sub = yield from ensure_future(subtask())
...     print("Retour dans 'example'")
...     for _ in range(3):
...         print("(example)")
...         yield
...
...     print("En attente de la fin de 'subtask'")
...     yield from wait([sub])
...     print("Ça y est, je peux quitter")
...
>>> event_loop.run_until_complete(example())
Tâche 'example'
Lancement de la tâche 'subtask'
Retour dans 'example'
(example)
Tâche 'subtask'
(subtask)
(example)
(subtask)
(example)
(subtask)
En attente de la fin de 'subtask'
(subtask)
(subtask)
<Task 'subtask' [FINISHED] (None)>
Ça y est, je peux quitter
<Task 'example' [FINISHED] (None)>
```

Parfait. Nous venons d'implémenter notre première coroutine *d'attente
asynchrone*. Celle-ci permet d'endormir une tâche pour ne la réveiller que
lorsqu'une ou plusieurs autres tâches auront fini de s'exécuter. S'il y avait
une seule chose à retenir de la programmation asynchrone, c'est bien ce
mécanisme. C'est dans le fait de pouvoir dire, de façon tout à fait explicite :
« je n'ai rien à faire pour le moment, réveillez-moi quand il se sera passé
quelque chose d'intéressant », que réside tout l'intérêt de ce modèle
d'exécution. Et ce genre d'attente est vraiment très fréquent en informatique !

Dans cet article, nous allons nous contenter de *simuler* ces tâches. Pour ce
faire, il suffit de nous doter d'une coroutine d'endormissement que nous
appellerons `async_sleep` :

```python
# Nous utilisons les classes datetime et timedelta de la bibliothèque standard
# La première représente une date (précise), la seconde une durée.
from datetime import datetime, timedelta

def async_sleep(secs):

    # On calcule l'heure à laquelle on doit se réveiller
    wakeup = datetime.now() + timedelta(seconds=secs)

    # On laisse la main tant que l'heure de réveil n'est pas passée
    while datetime.now() < wakeup:
        yield
```

Vérifions rapidement qu'elle fonctionne :

```python
>>> def sleep_test(secs, msg):
...     yield from async_sleep(secs)
...     print(msg)
...
>>> event_loop.run_until_complete(
...     wait([
...         sleep_test(3, 'trois'),
...         sleep_test(1, 'un'),
...         sleep_test(2, 'deux')
...     ])
... )
un
<Task 'sleep_test' [FINISHED] (None)>
deux
<Task 'sleep_test' [FINISHED] (None)>
trois
<Task 'sleep_test' [FINISHED] (None)>
<Task 'wait' [FINISHED] (...)>
```

Bien, nous avons maintenant tout ce qu'il nous faut pour modéliser un système
asynchrone, comme notre employé de *fast food*, par exemple. Pour ramener cet
exemple à des durées plus aisées à vérifier dans un programme, nous prendrons
les temps suivants :

* Préparation d'un soda : 1 seconde,
* Préparation d'un hamburger : 3 secondes,
* Préparation d'une portion de frites : 4 secondes.


```python
def get_soda():
    print("Remplissage du gobelet de soda")
    yield from async_sleep(1)
    print("Le soda est prêt")

def get_fries():
    print("Démarrage de la cuisson des frites")
    yield from async_sleep(4)
    print("Les frites sont prêtes")

def get_burger():
    print("Commande du burger en cuisine")
    yield from async_sleep(3)
    print("Le burger est prêt")
```

Nous n'avons plus qu'à modéliser notre serveur. Commençons par le concevoir de
façon séquentielle :

```python
def serve():
    start = datetime.now()
    yield from get_soda()
    yield from get_burger()
    yield from get_fries()
    print("Client servi en", datetime.now() - start)
```

Lorsqu'il attend "bêtement" que tout soit prêt, le serveur met 8 secondes à
servir un client.

```python
>>> event_loop.run_until_complete(serve())
Remplissage du gobelet de soda
Le soda est prêt
Commande du burger en cuisine
Le burger est prêt
Démarrage de la cuisson des frites
Les frites sont prêtes
Client servi en 0:00:08.000307
<Task 'serve' [FINISHED] (None)>
```

Alors que si nous le modélisons de façon asynchrone…

```python
def async_serve():
    start = datetime.now()
    yield from wait([
        get_soda(),
        get_burger(),
        get_fries()
    ])
    print("Client servi en", datetime.now() - start)
```

… Le client est servi deux fois plus rapidement :

```python
>>> event_loop.run_until_complete(async_serve())
Commande du burger en cuisine
Démarrage de la cuisson des frites
Remplissage du gobelet de soda
Le soda est prêt
<Task 'get_soda' [FINISHED] (None)>
Le burger est prêt
<Task 'get_burger' [FINISHED] (None)>
Les frites sont prêtes
<Task 'get_fries' [FINISHED] (None)>
Client servi en 0:00:04.000214
<Task 'async_serve' [FINISHED] (None)>
```

En modélisant cet exemple du début, nous venons de *construire* l'ensemble des
outils et des primitives nécessaires à la programmation asynchrone. Cela dit,
ce n'est encore que le début du voyage. Dans le prochain exemple, nous allons
nous servir d'`asyncio` pour modéliser ce système d'une façon un peu plus
réaliste, et découvrir que ce serveur de *fast food* représente un véritable
problème d'optimisation d'un serveur asynchrone !
