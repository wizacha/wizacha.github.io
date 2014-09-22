---
layout: post
title: Php perd sa mémoire ?
category: blog
tags: Php Mémoire
author: benoit
banner: php-cow.png
---

L'avantage d'un langage interprété, c'est qu'il n'est pas nécessaire d'allouer ou de désallouer
la mémoire, l'interpréteur le fait tout seul ! Mais il ne le fait pas forcément de la manière
dont on s'y attend, et cela peut avoir quelques effets de bords surprenants.

#Le crash mémoire qui vient de nul part
Rentrons dans le vif du sujet avec du code. Les quelques lignes suivantes ne font rien de bien extraordinaire:

* Création d'un **grand** tableau avec [`array_fill`](http://php.net/manual/fr/function.array-fill.php),
contenant l'entier `42`
* Itération sur le tableau

{% highlight php %}
<?php
$my_array = array_fill(0, 1000000, 42);
foreach($my_array as $value) {}
{% endhighlight %}

On le lance en ligne de commande, et... rien ne se passe ! Enfin si, Php fait plutôt exactement ce qu'on
lui a demandé de faire, tout se passe bien. On introduit donc une *minuscule* différence dans ce bout
de code, pour itérer sur les références des éléments du tableaux (notez bien le `&` ligne 2).

{% highlight php %}
<?php
$my_array = array_fill(0, 1000000, 42);
foreach($my_array as &$value) {}
{% endhighlight %}

Et là, c'est le drame:

~~~
PHP Fatal error:  Allowed memory size of 134217728 bytes exhausted (tried to allocate 32 bytes) in test.php on line 3
~~~

Php nous informe *poliment* qu'il n'y a plus de mémoire disponible[^mem_dispo] [^kill] suite à une instruction en ligne 3.
Et tout ça *juste* à cause d'une référence sur un élément du tableau, au lieu d'une copie.
Le premier réflexe est pourtant de se dire qu'une référence prend moins de place, car elle évite justement de copier la
mémoire, mais cela ne semble pas s'appliquer ici. Bon, pour essayer d'en savoir plus, on va traquer la mémoire
*à l'ancienne*, à coup de [`echo`](http://php.net/manual/fr/function.echo.php)
et de [`memory_get_usage`](http://php.net/manual/fr/function.memory-get-usage.php).

[^mem_dispo]: Si vous avez exécuté ce code chez vous et que ça ne plante pas, c'est que votre machine a plus de RAM
    que la mienne ! Mais ajoutez un zéro ou deux au nombre d'éléments, et vous finirez bien par atteindre votre limite ;)

[^kill]:  Potentiellement, il ne dit rien, mange votre ram, puis votre swap, puis se fait killer sauvagement par le noyau, tout ça à cause d'un `memory_limit = -1` dans la configuration de votre php.
    Sur ma machine, la limite est à 128M.

{% highlight php %}
<?php
$my_array = array_fill(0, 1000000, 42);
$offset   = memory_get_usage();
foreach ($my_array as &$value) {
    echo (memory_get_usage() - $offset) . PHP_EOL;
}
{% endhighlight %}

Ce qui nous donne (à *quelques* lignes près):

~~~
272
352
400
448
496
...
37575480
37575528
37575576

Fatal error: Allowed memory size of 134217728 bytes exhausted (tried to allocate 32 bytes) in test.php on line 5
~~~

Étonnant, mais à chaque tour de boucle Php consomme (au moins[^mem_iter]) 48 octets, juste en itérant sur les éléments du tableau !
Appelez [Rasmus](http://fr.wikipedia.org/wiki/Rasmus_Lerdorf) et son [équipe](http://php.net/credits.php),
on a retrouvé le bug de l'an 2000 !
C'est un *memory leak*, il faut vite arrêter le Php et se mettre à Ruby ou Node,
*ils* nous avaient prévenu, Php est maudit, **il nous anéantira tous !!**

Ou pas...

[^mem_iter]: La première itération consomme plus de 48 octets, car Php a besoin d'allouer
    de la mémoire pour les variables `$offset` et `$value`, qui n'existent pas lors du premier
    appel à `memory_get_usage`. Mais pour être honnête, je ne m'explique pas encore la mémoire
    supplémentaire consommée pour la deuxième itération... Si vous avez la réponse, [je suis
    preneur](https://twitter.com/b_viguier) :).

##Quelques indices

Si vous connaissez un peu la [façon dont sont implémentées les variables en Php](http://php.net/manual/fr/internals2.variables.intro.php)
il se peut que cette valeur de 48 octets vous soit familière: c'est la taille minimale pour une variable Php
sur un système 64bits. Cela semble donc signifier que l'on créé une nouvelle variable à chaque itération.
Se pourrait-il que le tableau ne soit pas *complètement* rempli ?
Modifions juste la façon de générer notre tableau avec une méthode plus *naïve*.

{% highlight php %}
<?php
$my_array = [];
for($i=0; $i<1000000; ++$i) {
    $my_array[] = 42;
}
{% endhighlight %}
Cette boucle n'aura pas le temps de se terminer...

~~~
Fatal error: Allowed memory size of 134217728 bytes exhausted (tried to allocate 32 bytes) in test.php on line 4
~~~

Et oui, Php ne nous laisse pas créer le tableau, il occupe trop de place en mémoire.
Mais alors pourquoi nos précédents `array_fill` ne causaient pas la même erreur ?
Et pourquoi est ce que le crash mémoire est *différé* pour se produire dans la boucle d'après ?

#Le *Copy On Write*

La mémoire est précieuse et il n'est pas toujours évident de bien maîtriser sa consommation sur des gros projets.
C'est l'une des raisons pour laquelle Php se charge d'allouer et de libérer la mémoire silencieusement
sans que le développeur ait à s'en soucier (en général).
Ainsi, pour éviter que les octets ne soient gaspillés inutilement,
[Php utilise le *ref-counting*](http://php.net/manual/fr/features.gc.refcounting-basics.php#example-425)
pour faire du
[*Copy On Write* (*COW* pour les intimes)](http://fr.wikipedia.org/wiki/Copy-On-Write):

> PHP est suffisamment intelligent pour ne pas dupliquer le conteneur lorsque ce n'est pas nécessaire.

Lorsque l'on fait une copie d'une variable vers une autre, la *vraie* copie en mémoire ne sera effective que
lorsque ce sera vraiment nécessaire, c'est
à dire lorsque les contenus des deux variables différeront suite à une écriture.
Il s'agit en fait d'une référence implicite accessible seulement en lecture, et qui provoque une copie de
son contenu lors de la première écriture rencontrée.

Dans notre cas, le *COW* intervient discrètement dans le `array_fill`, puisque l'on construit un tableau
ne contenant qu'une seule valeur Php va s'économiser des opérations de copie en ne gardant que des
références vers une seule et même valeur `42`. On peut le démontrer avec une simple boucle sur la base
du dernier exemple.

{% highlight php %}
<?php
$my_array = [];
$value = 42;
for($i=0; $i<1000000; ++$i) {
    $my_array[] = $value;
}
{% endhighlight %}

Tout se passe bien, aucun crash, tous les éléments du tableau pointent implicitement vers le même
espace mémoire contenant la valeur `42`. Par contre, si je viens faire une écriture (même si c'est la même valeur !)
sur l'un de ces éléments, une copie sera discrètement faite, induisant un coût mémoire.
Dans notre cas l'exemple se veut minimaliste, et nous n'avons même pas eu besoin de faire une écriture.
Le *COW* ne pouvant être fait sur une référence, Php est obligé de faire une distinction nette entre
les différentes valeurs du tableau en faisant une copie de la valeur.

Pour d'autres exemples *amusants* à propos du *COW*, n'hésitez pas à consulter
[*Do not use PHP references*](http://schlueters.de/blog/archives/125-Do-not-use-PHP-references.html).

## Mais alors, c'est *bien* ? Ou *mal* ?

Ce mécanisme est très ingénieux, il permet de copier des valeurs *presque gratuitement* pour peu
qu'on ne les consulte qu'en lecture.
Quand on passe un gros tableau à une fonction, celui ci ne sera pas systématiquement copié, seulement
si nécessaire.

L'inconvénient c'est ce petit effet de bord, ce *décalage* entre le moment ou le programmeur pense
consommer de la mémoire et celui ou Php l'alloue vraiment. Cela peut aussi tromper le développeur
en lui faisant croire qu'il a encore de la marge en mémoire, alors qu'une simple modification
de variable fera grimper le compteur.

**L'important est de connaître ses outils.**


