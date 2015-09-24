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
tâche prête à être exécutée, mais n'exécute pas les instructions qu'elle
contient.

[^corogen]: En toute rigueur il s'agit d'un *générateur*, mais comme nous avons
pu l'observer dans un [précédent
article](https://zestedesavoir.com/articles/232/la-puissance-cachee-des-coroutines/),
les générateurs de Python sont implémentés comme de véritables coroutines.

```python
>>> task = tic_tac()
>>> task
<generator object tic_tac at 0x7fe157023280>
```

En termes de vocabulaire, on dira que notre fonction `tic_tac` est une
*fonction coroutine*, c'est-à-dire une fonction qui **construit une
coroutine**. La coroutine est contenue ici dans la variable `task`.

Nous pouvons maintenant exécuter son code jusqu'au prochain `yield`, en nous
servant de la fonction standard `next()` :

```python
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
sorcier. Dès lors, on peut imaginer créer une petite boucle pour exécuter cette
coroutine jusqu'à épuisement :

```python
>>> task = tic_tac()
>>> while True:
...     try:
...         next(task)
...     except StopIteration as stop:
...         print("valeur de retour:", repr(stop.value))
...         break
...
Tic
Tac
valeur de retour: 'Boum!'
```

Afin de nous affranchir de la sémantique des itérateurs de Python, créons une
classe `Task` qui nous permettra de manipuler nos coroutines plus aisément :

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
        self.return_value = None  # Valeur de retour de la coroutine
        self.error_value = None  # Exception levée par la coroutine

    # Exécute la tâche jusqu'à la prochaine pause
    def run(self):
        try:
            # On passe la tâche à l'état RUNNING et on l'exécute jusqu'à
            # la prochaine suspension de la coroutine.
            self.status = STATUS_RUNNING
            next(self.coro)
        except StopIteration as err:
            # Si la coroutine se termine, la tâche passe à l'état FINISHED
            # et on récupère sa valeur de retour.
            self.status = STATUS_FINISHED
            self.return_value = err.value
        except Exception as err:
            # Si une autre exception est levée durant l'exécution de la
            # coroutine, la tâche passe à l'état ERROR, et on récupère
            # l'exception pour laisser l'utilisateur la traiter.
            self.status = STATUS_ERROR
            self.error_value = err

    def is_done(self):
        return self.status in {STATUS_FINISHED, STATUS_ERROR}

    def __repr__(self):
        result = ''
        if self.is_done():
            result = " ({!r})".format(self.return_value or self.error_value)

        return "<Task '{}' [{}]{}>".format(self.name, self.status, result)

```

Son fonctionnement est plutôt simple. Réimplémentons notre boucle en nous
servant de cette classe :

```python
>>> task = Task(tic_tac())
>>> task
<Task 'tic_tac' [NEW]>
>>> while not task.is_done():
...     task.run()
...     print(task)
...
Tic
<Task 'tic_tac' [RUNNING]>
Tac
<Task 'tic_tac' [RUNNING]>
<Task 'tic_tac' [FINISHED] ('Boom!')>
>>> task.return_value
'Boom!'
```

Bien. Nous avons une classe qui nous permet de manipuler des tâches en cours
d'exécution, ces tâches étant implémentées sous la forme de coroutines. Il ne
nous reste plus qu'à trouver un moyen d'exécuter plusieurs coroutines de façon
**concurrente**, c'est-à-dire en parallèle les unes des autres. En effet, tout
l'intérêt de la programmation asynchrone est d'être capable d'occuper le
programme pendant qu'une tâche donnée est en attente d'un événement.

Pour cela, il suffit de construire une *file d'attente* de tâches à exécuter.
En Python, l'objet le plus pratique pour modéliser une file d'attente est la
classe standard `collections.deque` (*double-ended queue*). Cette classe
possède les mêmes méthodes que les listes, auxquelles viennent s'ajouter :

* `appendleft()` pour ajouter un élément au tout début de la liste,
* `popleft()` pour retirer (et retourner) le premier élément de la liste.

Ainsi, il suffit ajouter les éléments à une extrémité de la file (`append()`),
et consommer ceux de l'autre extrémité (`popleft()`). On pourrait arguer qu'il
est possible d'ajouter des éléments n'importe où dans une liste avec la méthode
`insert()`, mais la classe `deque` est vraiment *faite pour* créer des files et
des piles : ses opérations aux extrémités sont bien plus efficaces que la
méthode `insert()`.

Essayons d'exécuter en parallèle deux instances de notre coroutine `tic_tac` :

```python
>>> from collections import deque
>>> running_tasks = deque()
>>> running_tasks.append(Task(tic_tac()))
>>> running_tasks.append(Task(tic_tac()))
>>> while running_tasks:
...     # On récupère une tâche en attente et on l'exécute
...     task = running_tasks.popleft()
...     task.run()
...     if task.is_done():
...         # Si la tâche est terminée, on l'affiche
...         print(task)
...     else:
...         # La tâche n'est pas finie, on la replace au bout
...         # de la file d'attente
...         running_tasks.append(task)
...
Tic
Tic
Tac
Tac
<Task 'tic_tac' [FINISHED] ('Boom!')>
<Task 'tic_tac' [FINISHED] ('Boom!')>
```

Voilà qui est intéressant : la sortie des deux coroutines est entremêlée !
Cela signifie que les deux tâches ont été exécutées simultanément, de façon
**concurrente**.

Nous avons tout ce qu'il nous faut pour modéliser une boucle événementielle,
c'est-à-dire une boucle qui s'occupe de programmer l'exécution et le réveil
des tâches dont elle a la charge. Implémentons celle-ci dans la classe `Loop`
suivante :

```python
from collections import deque

class Loop:
    def __init__(self):
        self._running = deque()

    def _loop(self):
        task = self._running.popleft()
        task.run()
        if task.is_done():
            print(task)
            return
        self.schedule(task)

    def run_until_empty(self):
        while self._running:
            self._loop()

    def schedule(self, task):
        if not isinstance(task, Task):
            task = Task(task)
        self._running.append(task)
        return task
```

Vérifions :

```python
>>> def spam():
...     print("Spam")
...     yield
...     print("Eggs")
...     yield
...     print("Bacon")
...     yield
...     return "SPAM!"
...
>>> event_loop = Loop()
>>> event_loop.schedule(tic_tac())
>>> event_loop.schedule(spam())
>>> event_loop.run_until_empty()
Tic
Spam
Tac
Eggs
<Task 'tic_tac' [FINISHED] ('Boom!')>
Bacon
<Task 'spam' [FINISHED] ('SPAM!')>
```

Tout fonctionne parfaitement. Dotons tout de même notre classe `Loop` d'une
dernière méthode pour exécuter la boucle jusqu'à épuisement d'une coroutine en
particulier :

```python
class Loop:
    # ...
    def run_until_complete(self, task):
        task = self.schedule(task)
        while not task.is_done():
            self._loop()
```

Testons-la :

```python
>>> event_loop = Loop()
>>> event_loop.run_until_complete(tic_tac())
Tic
Tac
<Task 'tic_tac' [FINISHED] ('Boom!')>
```

Pas de surprise.

Toute la programmation asynchrone repose sur ce genre de boucle qui sert en
fait d'*ordonnanceur* aux tâches en cours d'exécution. Pour vous en convaincre,
regardez ce bout de code qui utilise `asyncio` :

```python
>>> import asyncio
>>> loop = asyncio.get_event_loop()
>>> loop.run_until_complete(tic_tac())
Tic
Tac
'Boom!'
>>> loop.run_until_complete(asyncio.wait([tic_tac(), spam()]))
Spam
Tic
Eggs
Tac
Bacon
({Task(<tic_tac>)<result='Boom!'>, Task(<spam>)<result='SPAM!'>}, set())
```

Drôlement familier, n'est-ce pas ? Ne bloquez pas sur la fonction
`asyncio.wait` : il s'agit simplement d'une coroutine qui sert à lancer
plusieurs tâches en parallèle et attendre que celles-ci se terminent avant de
retourner. Nous la reprogrammerons nous-mêmes très bientôt. ;)

