# Exemple n°3 : Attendre de façon asynchrone

Dans l'exemple n°1, nous avons vu un bref appel à une fonction d'`asyncio` :

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

La fonction intéressante de cet exemple est `asyncio.wait()`. Celle-ci permet
d'attendre qu'une ou plusieurs tâches (lancées en parallèle) soient terminées.
Sa valeur de retour se compose de deux ensembles (`set()`) :

* Le premier contient les tâches qui se sont terminées normalement ;
* Le second contient les tâches qui ne sont pas encore terminées,
  ont été annulées ou ont quitté sur une erreur.

C'est cette fonction que nous allons implémenter maintenant.

En fait, le plus difficile dans cette fonction est surtout sa partie
cosmétique. Pour avoir un comportement souple, il faut gérer le cas où les
tâches passées à cette fonction :

* Sont des coroutines et non des instances de la classe `Task`,
* N'ont pas encore été programmées pour être exécutées par la boucle,
* Sont déjà en train de s'exécuter.

Voilà ce que cela peut donner :

```python
def wait(tasks):
    # On lance les tâches qui ne l'ont pas encore été
    for idx, task in enumerate(tasks):
        # Si c'est une coroutine, on crée la tâche correspondante
        if not isinstance(task, Task):
            task = tasks[idx] = Task(task)
        # Si la tâche n'a pas encore été lancée, on la lance
        if task.status == STATUS_NEW:
            yield from launch(task)

    # On attend que toutes les tâches soient terminées
    while not all(task.is_done() for task in tasks):
        yield

    # On crée les deux ensembles pour le résultat
    finished = set()
    error = set()
    for task in tasks:
        if task.status == STATUS_FINISHED:
            finished.add(task)
        elif task.status == STATUS_ERROR:
            error.add(task)

    return finished, error
```

Nous pouvons vérifier que cette nouvelle fonction marche comme prévu avec les
coroutines suivantes.

```python
>>> def repeat(msg, times):
...    print("lancement de la tâche", repr(msg))
...     for _ in range(times):
...         print(msg)
...         yield
...
>>> def main_task():
...     print("Lancement de 3 tâches en parallèle")
...     returned, error = yield from wait([
...         repeat("spam", 10),
...         repeat("eggs", 3),
...         repeat("bacon", 5),
...     ])
...     print("Retour à la fonction principale: {} OK, {} erreurs".format(
...         len(returned), len(error)
...     ))
...
```

Et le résultat :

```python
>>> loop = Loop()
>>> loop.schedule(main_task())
>>> loop.run_once()
Lancement de 3 tâches en parallèle
lancement de la tâche 'spam'
spam
lancement de la tâche 'eggs'
eggs
spam
lancement de la tâche 'bacon'
bacon
eggs
spam
bacon
eggs
spam
bacon
<Task 'repeat' [FINISHED] (None)>
spam
bacon
spam
bacon
spam
<Task 'repeat' [FINISHED] (None)>
spam
spam
spam
<Task 'repeat' [FINISHED] (None)>
Retour à la fonction principale: 3 OK, 0 erreurs
<Task 'main_task' [FINISHED] (None)>
>>>
```

En guise d'exercice, si cela vous intéresse, vous pouvez :

* Essayer d'expliquer pourquoi les tâches ne démarrent pas toutes en même temps,
* Constater que si vous réimplémentez cet exemple avec `asyncio`, ce problème
  ne se produit pas,
* Modifier la fonction `wait()` et la boucle événementielle pour coller au
  comportement d'`asyncio`.

En ce qui nous concerne, la fonction `wait` que nous venons d'implémenter
nous suffira largement pour la suite.
