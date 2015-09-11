---
title: "'N' exemples pour découvrir la programmation asynchrone"
author: "Arnaud Calmettes (nohar)"
date: \today
papersize: a4paper
documentclass: scrartcl
classoption:
    - 11pt
header-includes:
    - \usepackage[left=2cm, right=2cm, bottom=3cm]{geometry}
    - \usepackage[french]{babel}
...

# Ça veut dire quoi, *asynchrone* ?

En un mot comme en cent, un programme qui fonctionne de façon *asychrone*,
c'est un programme qui évite au maximum de passer du temps à *attendre sans
rien faire*, et qui s'arrange pour *s'occuper autant que possible pendant qu'il
attend*. Cette façon d'optimiser le temps d'attente est tout à fait naturelle
pour nous. Par exemple, on peut s'en rendre compte en observant le travail d'un
serveur qui monte votre commande dans un *fast food*.

De façon synchrone :

* Préparer le hamburger:
    * Demander le hamburger en cuisine.
    * Attendre le hamburger (1 minute).
    * Récupérer le hamburger et le poser sur le plateau.
* Préparer les frites:
    * Mettre des frites à chauffer.
    * Attendre que les frites soient cuites (2 minutes).
    * Récupérer des frites et les poser sur le plateau.
* Préparer la boisson:
    * Placer un gobelet dans la machine à soda.
    * Remplir le gobelet (30 secondes).
    * Récupérer le gobelet et le poser sur le plateau.

En gros, si notre employé de *fast food* était synchrone, il mettrait 3 minutes
et 30 secondes pour monter votre commande.

Alors que de façon asynchrone :

* Demander le hamburger en cuisine.
* Mettre les frites à chauffer.
* Placer un gobelet dans la machine à soda et le mettre à remplir.
* Après 30 secondes: Récupérer le gobelet et le poser sur le plateau.
* Après 1 minute: Récupérer le hamburger et le poser sur le plateau.
* Après 2 minutes: Récupérer les frites et les poser sur le plateau.

En travaillant de façon asynchrone, notre employé de *fast food* monte
maintenant votre commande en 2 minutes. Mais ça ne s'arrête pas là !

* **Une commande `A` est confiée à l'employé**
* Demander le burger pour `A` en cuisine
* Mettre les frites à chauffer.
* Placer un gobelet dans la machine à soda pour `A`.
* Après 30 secondes: Récupérer le gobelet de `A` et le poser sur son plateau
* **Une nouvelle commande `B` est prise et confiée à l'employé**
* Demander le burger pour `B` en cuisine
* Placer un gobelet dans la machine à soda pour `B`.
* Après 1 minute: Le burger de `A` est prêt, le poser sur son plateau.
* La boisson de `B` est remplie, la poser sur son plateau.
* Après 1 minute 40: Le burger de `B` est prêt, le poser sur son plateau.
* Après 2 minutes: Les frites sont prêtes, servir `A` et `B`

Toujours en 2 minutes, l'employé asynchrone vient cette fois de servir 2
clients. Si vous vous mettez à la place du client `B` qui aurait dû attendre
que l'employé finisse de monter la commande de `A` avant de s'occuper de la
sienne dans un schéma synchrone, celui-ci a été servi en 1 minute 30 au lieu
d'attendre 6 minutes 30.

Pensez-y la prochaine fois que vous irez manger dans un fast-food, et observez
les serveurs. Leur boulot vous semblera d'un coup beaucoup plus compliqué qu'il
n'y paraît !

En informatique, il existe un type de tâche qui impose aux programmes
d'attendre sans rien faire : ce sont les *entrées/sorties* (ou *IO*). Nous
verrons dans cet article que la programmation asynchrone est une façon
extrêmement puissante d'implémenter des programmes qui réalisent plus d'IO que
de calcul (comme une application de messagerie instantanée, par exemple).

