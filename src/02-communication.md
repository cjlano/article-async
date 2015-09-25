# Exemple n°2 : Exécuter des tâches concurrentes

Dans le précédent exemple, nous avons déclenché l'exécution concurrente de deux
coroutines en les programmant explicitement dans la boucle :

```python
>>> event_loop = Loop()
>>> event_loop.schedule(Task(tic_tac()))
>>> event_loop.schedule(Task(spam()))
>>> event_loop.run_until_empty()
# les deux coroutines s'exécutent en parallèle.
```

Ce petit morceau de code nous a permis de prouver qu'avec une boucle
événementielle et des coroutines, on avait tout ce qu'il nous fallait pour
définir un modèle d'exécution concurrent.

Cependant, dans la pratique, on ne devrait pas avoir à manipuler directement la
boucle pour réaliser quelque chose d'aussi simple que le lancement d'une tâche
parallèle. Si l'on se réfère aux autres modes de concurrence :

* On peut lancer un *thread* et attendre la fin de l'exécution de celui-ci
  depuis n'importe quel *thread* en cours d'exécution.
* On peut créer, attendre ou arrêter un processus fils depuis n'importe quel
  processus en cours d'exécution.

Qu'il s'agisse de *threads* ou de processus, ces actions ne requièrent à aucun
moment d'interagir directement avec l'ordonnanceur de votre système
d'exploitation (OS).[^scheduler]

[^scheduler]: D'ailleurs, le noyau de l'OS est justement là pour vous empêcher
de toucher vous-même à l'ordonnanceur !

Par extension, **on devrait pouvoir lancer, attendre la fin de l'exécution, ou
annuler une « sous-coroutine » depuis n'importe quelle coroutine en cours
d'exécution**, sans avoir à appeler la méthode `schedule()` ni même toucher
directement à la boucle événementielle.

Commençons par chercher le moyen de faire appel à une coroutine depuis une
tâche en cours d'exécution. Rappelons d'abord que la syntaxe `yield from`
introduite dans le langage depuis Python 3.3 nous permet déjà de passer la main
à une autre coroutine. Par exemple, dans le code suivant, la coroutine
`example` utilise cette syntaxe pour laisser temporairement la main à la
coroutine `subtask` :


```python
def example():
    print("Tâche 'example'")
    print("Lancement de la tâche 'subtask'")
    yield from subtask()
    print("Retour dans 'example'")
    for _ in range(3):
        print("(example)")
        yield

def subtask():
    print("Tâche 'subtask'")
    for _ in range(2):
        print("(subtask)")
        yield
```

Vérifions :

```python
>>> event_loop = Loop()
>>> event_loop.run_until_complete(example())
Tâche 'example'
Lancement de la tâche 'subtask'
Tâche 'subtask'
(subtask)
(subtask)
Retour dans 'example'
(example)
(example)
(example)
<Task 'example' [FINISHED] (None)>
```

Ainsi, Python nous fournit déjà nativement un élément de syntaxe pour *lancer
une tâche de façon séquentielle* à l'intérieur d'une coroutine. Mais ce n'est
pas tout à fait ce que nous voulons : notre but, ici, est de réussir à lancer
la coroutine `subtask` de façon qu'elle s'exécute *en parallèle* de la
coroutine `example`, et non d'attendre que `subtask` soit terminée pour
reprendre l'exécution de `example`.

Pour ce faire, nous allons tirer parti d'une particularité des coroutines que
nous avons découverte
[dans cet article](https://zestedesavoir.com/articles/232/la-puissance-cachee-des-coroutines/) :
il est possible d'échanger des messages avec une coroutine en cours
d'exécution.

Pour bien comprendre ce qui va suivre, continuons, si vous le voulez bien,
notre parallèle avec le fonctionnement d'un système d'exploitation. Lorsqu'un
processus[^process] `A` a besoin d'exécuter un programme `B` dans un processus
concurrent, celui-ci réalise ce que l'on appelle un *appel système*,
c'est-à-dire qu'il *envoie un message* à l'OS pour lui demander de bien vouloir
lancer le nouveau programme `B`.[^syscall] Dans la pratique, cet *appel
système* ressemble à s'y méprendre à n'importe quel appel de fonction.
L'idée-clé, c'est que le processus peut *communiquer* avec le noyau du système
d'exploitation en échangeant des *messages* avec lui.

[^process]: Un *processus* peut être vu comme **un programme en cours
d'exécution**.

[^syscall]: Ce comportement n'est pas exclusif aux processus : de la même
façon, un *thread* `A` envoie au noyau un appel système lorsqu'il veut lancer
un nouveau *thread* `B`.

Nous allons maintenant transposer cette notion à notre cas : nous voulons
réussir à faire communiquer la boucle événementielle avec les coroutines
qu'elle exécute grâce à des *messages*.

Rappelons rapidement que l'on peut envoyer une donnée à une coroutine au moment
où celle-ci se suspend, en utilisant la méthode `send` des coroutines :

```python
>>> def receiver():
...     while True:
...         data = yield
...         print('received:', repr(data))
...
>>> coro = receiver()
>>> next(coro)  # La coroutine doit être démarrée pour recevoir des données
>>> coro.send("spam")
received: 'spam'
>>> coro.send("eggs")
received: 'eggs'
>>> coro.send("bacon")
received: 'bacon'
```

À l'opposé, on peut également envoyer des données depuis une coroutine en
passant un argument au mot-clé `yield` :

```python
>>> def sender():
...     while True:
...        yield "spam"
...        yield "eggs"
...        yield "bacon"
...
>>> coro = sender()
>>> coro.send(None)  # Équivalent à next(coro)
'spam'
>>> coro.send(None)
'eggs'
>>> coro.send(None)
'bacon'
```

On peut donc parfaitement imaginer simuler un appel système :

```python
>>> def requester():
...     data = yield 'REQUÊTE'
...     print('data:', repr(data))
...     return
...
>>> coro = requester()
>>> coro.send(None)  # On lance la coroutine
'REQUÊTE'
>>> # La coroutine vient de nous envoyer un message.
... # On lui répond.
...
>>> coro.send('RÉPONSE')
data: 'RÉPONSE'
```

Comme vous le constatez, la syntaxe de Python rend assez intuitif le fait que
notre coroutine peut envoyer des requêtes à son environnement d'exécution, et
se suspendre jusqu'à ce qu'on lui envoie le résultat de cette requête.

Pour que ces messages soient transmis par notre classe `Task` du premier
exemple, nous devons l'accomoder un petit peu :

```python
class Task:
    def __init__(self):
        # ...
        self.msg = None

    def run(self):
        try:
            self.status = STATUS_RUNNING
            return self.coro.send(self.msg)
        # ...
```

Ainsi, on peut envoyer un message à une tâche simplement en affectant son
attribut `task.msg`, et la méthode `task.run()` retourne maintenant tous les
messages que la coroutine pourrait `yield`-er.

Reste à déterminer la forme des messages grâce auxquels la coroutine peut faire
appel à sa boucle d'exécution. De manière similaire aux appels système, on peut
par exemple décider que ces messages seront un tuple sous la forme suivante :

```python
(TYPE_APPEL, arguments)
```

Par exemple, pour demander à la boucle événementielle de lancer une ou
plusieurs coroutines en parallèle, le message serait :

```python
('SCHEDULE', [task1, task2, ...])
```

Modifions la méthode `_loop()` de notre boucle pour qu'elle réagisse à ces
messages :

```python
CALL_SCHEDULE = 'SCHEDULE'

class Loop:
    def __init__(self):
        self._running = deque()

    def _loop(self):
        task = self._running.popleft()
        msg = task.run()  # Réception du message

        if task.is_done():
            print(task)
        else:
            self.schedule(task)

        if not msg:
            return

        msg_type, args = msg
        if msg_type == CALL_SCHEDULE:
            # Si le message est un ordre d'exécuter des tâches parallèles,
            # on programme ces tâches et on les retourne à la tâche
            # appelante.
            task.msg = tuple(self.schedule(subtask) for subtask in args)
        else:
            raise RuntimeError("Message inconnu : {}".format(msg_type))
```

On peut maintenant abstraire ce message derrière une coroutine que nous
appellerons `ensure_future()`, pour respecter la même nomenclature
qu'`asyncio` :

```python
def ensure_future(task):
    # L'appel à CALL_SCHEDULE retourne un tuple à un seul élément dans ce cas
    (task,) = yield CALL_SCHEDULE, [task]
    return task
```

Nous disposons désormais de deux façons d'exécuter une sous-tâche depuis
une coroutine :

* `yield from subtask()` : lance l'exécution d'une coroutine de façon
  *séquentielle*, c'est-à-dire en lui laissant la main jusqu'à ce que celle-ci
  se soit terminée.

* `yield from ensure_future(subtask())` : lance l'exécution d'une coroutine de
  *en parallèle*.

Ainsi, si nous modifions notre coroutine `example()` définie plus haut en
conséquence, nous pouvons vérifier que nos tâches sont bien exécutées de façon
concurrente :

```python
>>> def example():
...     print("Tâche 'example'")
...     print("Lancement de la tâche 'subtask'")
...     yield from ensure_future(subtask())
...     print("Retour dans 'example'")
...     for _ in range(3):
...         print("(example)")
...         yield
...
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
<Task 'subtask' [FINISHED] (None)>
<Task 'example' [FINISHED] (None)>
```

Magique, n'est-ce pas ?

Par contre, une fois que notre coroutine est lancée, nous n'avons pas tout à
fait le contrôle de son exécution. Par exemple, si nous rendions la tâche
`subtask` plus longue qu'`example`, celle-ci lui « survivrait » :

```python
>>> def subtask():
...     print("Tâche 'subtask'")
...     for _ in range(5):
...         print("(subtask)")
...         yield
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
<Task 'example' [FINISHED] (None)>
>>> # L'exécution s'arrête avec la fin de la coroutine 'example'
... # Vidons ce qu'il reste dans la boucle événementielle :
...
>>> event_loop.run_until_empty()
(subtask)
(subtask)
<Task 'subtask' [FINISHED] (None)>

```

**Que faire si nous ne voulons pas qu'une coroutine quitte avant une
sous-tâche qu'elle aurait lancée en parallèle ?**

Nous avons deux solutions. La première, dont nous nous contenterons dans cet
exemple, serait de pouvoir *annuler* une tâche en cours d'exécution. Il nous
suffit pour cela de créer un nouvel état dans notre classe `Task` :

```python
STATUS_CANCELLED = "CANCELLED"

class Task:

    # ...

    def cancel(self):
        if self.is_done():
            # Inutile d'annuler une tâche déjà terminée
            return
        self.status = STATUS_CANCELLED

    def is_cancelled(self):
        return self.status == STATUS_CANCELLED
```

Rajoutons un test dans la boucle événementielle pour déprogrammer les tâches
annulées :

```python
class Loop:

    # ...

    def _loop(self):
        task = self._running.popleft()

        if task.is_cancelled():
            # Si la tâche a été annulée,
            # on ne l'exécute pas et on "l'oublie".
            print(task)
            return

        # ... le reste de la méthode est identique
```

Il ne nous reste plus qu'une petite coroutine utilitaire à écrire pour annuler
une tâche en cours d'exécution :

```python
def cancel(task):
    # On annule la tâche
    task.cancel()
    # On laisse la main à la boucle événementielle pour qu'elle ait l'occasion
    # de prendre en compte l'annulation
    yield

def example():
    print("Tâche 'example'")
    print("Lancement de la tâche 'subtask'")
    sub = yield from ensure_future(subtask())
    print("Retour dans 'example'")
    for _ in range(3):
        print("(example)")
        yield
    yield from cancel(sub)
```

Vérifions :

```python
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
<Task 'subtask' [CANCELLED]>
<Task 'example' [FINISHED] (None)>
```

Notre mécanisme d'annulation fonctionne comme prévu. Cela dit, on peut aussi
imaginer tout simplement vouloir *attendre* de façon asynchrone que la
sous-tâche ait terminé son exécution avant de quitter proprement…

Mais ça, je vous le garde pour le prochain exemple. ;)
