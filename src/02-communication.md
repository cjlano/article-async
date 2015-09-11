# Exemple n°2 : Communiquer avec la boucle événementielle

Jusqu'ici, vous vous êtes probablement aperçu que nos coroutines ne
`yield`-aient aucune valeur. En effet, le simple fait qu'elles se suspendent
suffit à les faire exécuter en concurrence. Néanmoins, nous avons vu dans un
précédent article que ce mot-clé `yield` était en Python un véritable canal de
communication entre la coroutine et le code qui l'exécute.

Ainsi, nous savons que nous pouvons faire communiquer la boucle avec les tâches
qu'elle exécute. C'est d'ailleurs tout l'intérêt de la ligne suivante dans la
classe `Task` :

```python
class Task:
    def run(self):
        # ...
        return self.coro.send(self.msg)
```

Mais à quoi cela pourrait-il bien servir ? Faisons, si vous le voulez bien un
petit parallèle entre notre boucle événementielle et un système d'exploitation
(OS).

Vous avez peut-être déjà lu ou entendu que l'OS est chargé d'exécuter des
programmes sur l'ordinateur : à chaque instant il sait quel programme est en
train de s'exécuter, il a d'ailleurs le pouvoir de les démarrer et de les
interrompre, exactement comme notre boucle événementielle avec ses tâches.

Parfois (enfin, plutôt souvent, même) un programme `A` a besoin que l'OS lui
rende un service, par exemple pour lancer un autre programme `B`. Dans ce cas,
le programme `A` va réaliser ce que l'on appelle un *appel système*,
c'est-à-dire qu'il va *envoyer un message* à l'OS pour lui demander de bien
vouloir lancer le nouveau programme `B`.

Eh bien nous pourrions tout à fait transposer cette notion d'*appels système* à
notre boucle événementielle. Et nous allons même nous servir de `yield` pour ce
faire.

Mettons nous en situation. Nous voulons que notre coroutine `example` lance
une nouvelle tâche `subtask`.

La façon la plus simple de nous y prendre serait d'appeler simplement la
seconde tâche dans la première, au moyen de `yield from` :

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
    for _ in range(3):
        print("(subtask)")
        yield
```

Voilà ce que cela donnerait :

```python
>>> loop = Loop()
>>> loop.schedule(example())
>>> loop.run_once()
Tâche 'example'
Lancement de la tâche 'subtask'
Tâche 'subtask'
(subtask)
(subtask)
(subtask)
Retour dans 'example'
(example)
(example)
(example)
<Task 'example' [FINISHED] (None)>
```

La tâche `subtask` a été exécutée complètement avant que `example` ne reprenne
la main. C'est pas mal, ça nous montre comment appeler des *sous-coroutines*,
mais comment faire si on avait voulu que la coroutine `subtask` s'exécute en
parallèle de `example` ?

Eh bien pour cela, il faut créer une tâche dédiée à la coroutine `subtask` dans
la boucle événementielle. On peut donc imaginer envoyer un message à la boucle
événementielle avec le mot-clé `yield`. Ce message serait composé de deux
éléments :

* une *action* (l'ordre que l'on envoie à la boucle, ici "exécuter")
* et une *valeur* (ici, la coroutine à exécuter dans une nouvelle tâche)

Notre tâche `example` ressemblerait alors à ceci :

```python
def example():
    print("Tâche 'example'")
    print("Lancement de la tâche 'subtask'")
    yield 'EXEC', subtask()  # Envoi du message
    print("Retour dans 'example'")
    for _ in range(3):
        print("(example)")
        yield
```

Ce message sera retourné à la boucle lorsquelle appellera `task.run()`. Il nous
suffit de rajouter un peu de code à ce niveau pour le lire :

```python
MSG_EXEC = 'EXEC'

class Loop:
    def run_once(self):
        while self._running:
            task = self._running.popleft()
            msg = task.run()

            if task.is_done():
                print(task)
                continue

            self.schedule(task)

            if msg:
                # Si la coroutine a envoyé un message à la boucle
                action, value = msg
                if action == MSG_EXEC:
                    # Exécute une coroutine dans une tâche dédiée
                    self.schedule(value)
```

Et voilà le résultat :

```python
>>> loop = Loop()
>>> loop.schedule(example())
>>> loop.run_once()
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
<Task 'subtask' [FINISHED] (None)>
```

Nous pouvons donc créer notre premier "*message système*" pour masquer les
détails d'implémentation à l'utilisateur:

```python
def launch(task):
    if not isinstance(task, Task):
        task = Task(task)
    yield MSG_EXEC, task
    return task

def example():
    print("Tâche 'example'")
    print("Lancement de la tâche 'subtask'")
    yield from launch(subtask())
    print("Retour dans 'example'")
    for _ in range(3):
        print("(example)")
        yield
```

Nous avons maintenant deux façons d'exécuter une nouvelle coroutine :

* `yield from coroutine()` suspend l'exécution de la tâche en cours jusqu'à ce
  que la coroutine appelée ait terminé son exécution.
* `yield from launch(coroutine())` lance la coroutine dans une tâche séparée
  pour que celle-ci s'exécute *en concurrence* et reprend aussitôt l'exécution de
  la tâche en cours.
