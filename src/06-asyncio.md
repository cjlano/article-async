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
    * Un "*writer*" dont on se sert pour en envoyer,
* On se sert du `writer` pour envoyer un message au serveur (attention : la méthode
  `write()` est une fonction et non une coroutine ; pas de `yield from`),
* On appelle ensuite, cette fois de façon asynchrone, la méthode `reader.read()`
  pour récupérer des données (on s'attend à moins de 1024 octets),
* On ferme le `writer` pour clore la connexion.

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
fameux objets qu'`asyncio` a créés pour nous. Il s'agit en fait d'instances de
[`asyncio.StreamReader`](https://docs.python.org/3/library/asyncio-stream.html#streamreader)
et
[`asyncio.StreamWriter`](https://docs.python.org/3/library/asyncio-stream.html#streamwriter).

Il s'agit de l'interface haut niveau d'asyncio pour manipuler une
connexion de type "*Stream*" (typiquement, TCP). La même interface existe
également pour les sockets Unix. Ces objets représentent un les deux sens d'un
même flux réseau. Ainsi, vous avez grosso-modo 4 méthodes à retenir :

* La *coroutine* `StreamReader.read(size)` sert à recevoir un bloc de `size`
  octets de données maximum, de façon asynchrone ;
* La *coroutine* `StreamReader.readline()` sert à recevoir une "ligne" de
  données, terminée par le caractère ASCII spécial *LINE_FEED* (`'\n'`) ;
* La *fonction* `StreamWriter.write(data)` sert à envoyer des données (sous la
  forme d'un objet `bytes`) sur le flux. En fait, les données seront placées en
  attente dans un tampon (*buffer*) qui ne sera vidé qu'au prochain `yield`…
* Ce qui nous amène à la *coroutine* `StreamWriter.drain()`, qui sert à vider
  explicitement le tampon du *writer* pour envoyer les données sur le flux.

C'est plutôt simple, non ? Ces objets reposent en fait sur une autre API, un
petit peu plus bas niveau, d'`asyncio`, qui modélise des *protocoles* que l'on
utilise par-dessus un *transport* (TCP, UDP…). Nous ne parlerons pas de cette
API dans cet article, mais si vous êtes curieux, [sa
documentation](https://docs.python.org/3/library/asyncio-protocol.html) est
plutôt claire et détaillée.
