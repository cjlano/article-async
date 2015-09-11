# Exemple n°1 : Une boucle événementielle, c'est essentiel

La notion fondamentale autour de laquelle `asyncio` a été construite est celle
de *coroutine*.

Une coroutine est une tâche qui peut décider de se suspendre elle-même au moyen
du mot-clé `yield`, et attendre jusqu'à ce que le code qui la contrôle décide
de lui rendre la main en *itérant* dessus.

On peut imaginer, par exemple, écrire la fonction suivante :

```python
def tic_tac():
    print("Tic")
    yield
    print("Tac")
    yield
    return "Boum!"
```

Cette fonction, puisqu'elle utilise le mot-clé `yield`, définit une
*coroutine*[^corogen]. Si on l'invoque, la fonction `tic_tac` retourne une
tâche prête à être exécutée. Pour faire avancer la tâche jusqu'au prochain
`yield`, il suffit d'itérer dessus au moyen de la builtin `next()` :

[^corogen]: En toute rigueur il s'agit d'un *générateur*, mais comme nous
avons pu l'observer dans un précédent article, les générateurs de Python sont
implémentés comme de véritables coroutines.

```python
>>> task = tic_tac()
>>> next(task)
Tic
>>> next(task)
Tac
>>> next(task)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration: Boum!
```

Lorsque la tâche est terminée, une exception `StopIteration` est levée.
Celle-ci contient la valeur de retour de la coroutine. Jusqu'ici, rien de bien
sorcier. Dès lors, on peut imaginer créer une petite boucle événementielle pour
exécuter cette coroutine. Il suffit en fait de la considérer comme une file de
tâches à exécuter jusqu'à épuisement.


```python
>>> from collections import deque
>>> events_loop = deque()
```

Dotons-nous de fonctions utilitaires pour ajouter des tâches à la boucle
événementielle et pour exécuter cette dernière :

* `schedule(loop, task)` : programmer l'exécution d'une tâche ;
* `run_until_empty(loop)`: exécuter la boucle événementielle jusqu'à épuisement ;
* `run_once(loop, task)`: raccourcis pour exécuter une coroutine.

```python
def schedule(loop, task):
    loop.append(task)

def run_until_empty(loop):
    while loop:
        task = loop.popleft()
        try:
            # On fait avancer la coroutine jusqu'au prochain "yield"
            next(task)
            # Si celle-ci n'a pas fini son travail,
            # on la programme pour qu'elle le reprenne plus tard.
            schedule(loop, task)
        except StopIteration as res:
            # La coroutine a terminé son exécution.
            # On affiche sa valeur de retour.
            print(
                "Task {!r} returned {!r}".format(task.__name__, res.value)
            )

def run_once(loop, task):
    schedule(loop, task)
    run_until_empty(loop)
```

Nous pouvons maintenant nous servir de la boucle événementielle pour exécuter
la tâche `tic_tac`:

```python
>>> run_once(events_loop, tic_tac())
Tic
Tac
Task 'tic_tac' returned 'Boum!'
```

Tout fonctionne comme prévu. Et si nous donnions deux coroutines différentes à
exécuter à boucle événementielle ?

```python
>>> def spam():
...     print("spam")
...     yield
...     print("eggs")
...     yield
...     print("bacon")
...     yield
...     return 'spam'
...
>>> schedule(events_loop, tic_tac())
>>> schedule(events_loop, spam())
>>> run_until_empty(events_loop)
Tic
spam
Tac
eggs
Task 'tic_tac' returned 'Boum!'
bacon
Task 'spam' returned 'spam'
```

Voilà qui est intéressant : la sortie des deux coroutines est entremêlée !
Cela signifie que les deux tâches ont été exécutées simultanément, de façon
**concurrente**.

Ce genre de boucle événementielle existe bien évidemment dans Python, c'est
même exactement **le centre nerveux** du module `asyncio`. Regardez :

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

Dans tous les exemples qui suivent, nous allons nous attarder sur les
différentes fonctionnalités qui font d'`asyncio` un puissant *framework* de
programmation asynchrone. Nous reprogrammerons certaines de ces fonctionnalités
pour mieux les comprendre au début, puis nous glisserons progressivement vers
les objets de plus haut niveau que nous propose ce *framework* pour nous
concentrer sur des applications **pratiques** de la vie réelle.

Commençons par modéliser un peu mieux notre système. Jusqu'ici, il se
compose uniquement de deux types d'objets : une **boucle** (`Loop`) qui exécute
des **tâches** (`Task`) de façon concurrente.

Définissons notre classe `Task` en premier :

```python
STATUS_NEW = 'NEW'
STATUS_RUNNING = 'RUNNING'
STATUS_FINISHED = 'FINISHED'
STATUS_ERROR = 'ERROR'

class Task:
    def __init__(self, coro):
        self.coro = coro  # Coroutine à exécuter
        self.name = coro.__name__
        self.status = STATUS_NEW  # Statut de la tâche
        self.msg = None  # Message à envoyer à la tâche
        self.return_value = None  # Valeur de retour de la coroutine
        self.error_value = None  # Exception levée par la coroutine

    # Exécute la tâche jusqu'à la prochaine pause
    def run(self):
        try:
            self.status = STATUS_RUNNING
            return self.coro.send(self.msg)
        except StopIteration as err:
            self.status = STATUS_FINISHED
            self.return_value = err.value
        except Exception as err:
            self.status = STATUS_ERROR
            self.error_value = err

        def is_done(self):
            return self.status in {STATUS_FINISHED, STATUS_ERROR}

        def __repr__(self):
            return "<Task '{name}' [{status}] ({res!r})>".format(
                name=self.name,
                status=self.status,
                res=(self.return_value or self.error_value)
            )
```

Cette classe a une utilisation très simple. On lui passe une coroutine, et on
se contente de l'appeler via sa méthode `run()` :

```python
>>> task = Task(tic_tac())
>>> task
<Task 'tic_tac' [NEW] (None)>
>>> while not task.is_done():
...     task.run()
...     print(task)
...
Tic
<Task 'tic_tac' [RUNNING] (None)>
Tac
<Task 'tic_tac' [RUNNING] (None)>
<Task 'tic_tac' [FINISHED] ('Boum!')>
```

Reste la boucle événementielle `Loop` :

```python
from collections import deque

class Loop:
    def __init__(self):
        self._running = deque()

    def run_once(self):
        while self._running:
            task = self._running.popleft()
            task.run()

            if task.is_done():
                print(task)
                continue

            self.schedule(task)

    def schedule(self, task):
        if not isinstance(task, Task):
            task = Task(task)
        self._running.append(task)
```

Rien que nous n'ayons pas déjà vu :

```python
>>> loop = Loop()
>>> loop.schedule(tic_tac())
>>> loop.schedule(spam())
>>> loop.run_once()
Tic
spam
Tac
eggs
<Task 'tic_tac' [FINISHED] ('Boum!')>
bacon
<Task 'spam' [FINISHED] ('spam')>
```

Voilà, nous pouvons commencer à nous amuser avec nos coroutines. :)
