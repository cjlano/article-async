# Exemple n°6 : Des applications réseau avec `asyncio`

Le précédent exemple nous a fait toucher du doigt le concept d'*IO
asynchrones*. Il est temps pour nous de nous familiariser avec les objets que
nous propose `asyncio` pour réaliser de telles opérations sur un flux réseau.

Commençons par examiner un exemple que nous détaillerons ensuite. Voici le
client du précédent exemple, réimplémenté avec les outils d'`asyncio` :

```python
@asyncio.coroutine
def echo_client(message):
    reader, writer = yield from asyncio.open_connection('localhost', 1234)

    print("Envoi du message :", message)
    writer.write(message.encode())

    data = yield from reader.read(1024)
    print("Message reçu :", data.decode())

    writer.close()
```

Même sans savoir ce que sont ces objets `reader` et `writer`, la lecture de ce
code est plutôt intuitive :

* La coroutine `ayncio.open_connection()` crée une connexion et retourne deux
  objets :
    * Un "*reader*" que l'on peut utiliser pour recevoir des données,
    * Un "*writer*" dont on se sert pour en envoyer.

Avant d'aller plus loin, lançons le serveur de l'exemple précédent et vérifions
que ce code fonctionne dans la console :

```python
>>> import asyncio
>>> loop = asyncio.get_event_loop()
>>> loop.run_until_complete(echo_client("Coucou!"))
Envoi du message : Coucou!
Message reçu : Coucou!
```

Par de surprise, le code se comporte comme prévu. Attardons-nous un peu sur ces
fameux objets `reader` et `writer` qu'`asyncio` a créés pour nous.
