---
layout: post
title: Joel Test
category: blog
tags: Wizacha Team
author: benoit
banner: dev-team.png
---

Nous venons juste de publier une
[annonce pour un développeur backend](https://wizacha.com/wizacha-recrute.html),
mais il est toujours un peu frustrant de devoir résumer l'état d'esprit d'une équipe en
seulement quelques lignes, un peu de la même manière qu'un candidat qui doit se résumer
sur une feuille A4. Voici donc l'occasion de présenter un peu plus notre environnement
de travail à l'aide du fameux *Test de Joel*.

#Le test de Joel

Sur son blog, [Joel Spolsky](http://www.joelonsoftware.com/AboutMe.html)
 écrit un [petit test en 12 questions](http://www.joelonsoftware.com/articles/fog0000000043.html)
pour essayer de mesurer rapidement la qualité
d'une équipe pour le développement. Bien que ne payant pas de mine, ce test est suffisamment simple et pertinent
pour servir de référence, de point de comparaison. Par exemple, lorsque vous souhaitez déposer une annonce
sur [stackoverflow](http://stackoverflow.com/), vous êtes priés de [renseigner ces 12 questions](http://careers.stackoverflow.com/jobs/post)
pour que les candidats puissent savoir
*où* ils postulent[^stackoverflow]. Je trouve l'idée très bonne, elle permet de montrer en toute transparence le niveau d'une
équipe, et vers quoi on se dirige. Voici donc notre résultat à ce test, au moment ou j'écris ces lignes.

[^stackoverflow]: Il se trouve que le créateur de ce test est aussi l'un des fondateurs du site... ça aide un peu!

###Do you use source control?

Oui, oui, et **oui**! Même si on travaille seul, il est indispensable d'utiliser un outils de contrôle de version,
quel qu'il soit. Cela permet de répondre à des questions simples comme:

* Qu'est ce que j'ai modifié aujourd'hui?
* Il y a un bug en production/chez un client, sur quelle version du code cela se produit?
* Comment mettre mon travail de côté pour partir sur une autre tâche urgente?
* ...

Et dès qu'on est deux à travailler sur le même code, cela permet d'éviter les copiés/collés hasardeux, et
autre méthodes comme le [CPOLD](http://roland.entierement.nu/blog/2008/01/22/cpold-la-poudre-verte-du-suivi-de-versions.html).
Avec subversion, on pouvait (parfois) avoir la flemme d'installer un serveur centralisé, mais avec
[git](http://git-scm.com/) et [mercurial](http://mercurial.selenic.com/),
avec [Github](https://github.com/) et [Bitbucket](https://bitbucket.org/), il n'y a vraiment pas à hésiter!

Nous utilisons actuellement Mercurial via Bitbucket, mais nous avons quand même quelques projets sous Git (ne serait-ce que ce blog su Github).
L'explication de ces choix fera peut-être l'objet d'un futur article.

###Can you make a build in one step?

*Yes, we can!* Bien que le *build* d'un logiciel ne soit pas le même que pour une application web,
il y a toujours des choses à automatiser avant de livrer du code. Si ce n'est pas fait exactement
de la même manière à chaque fois, il y aura une erreur, c'est obligé. Et en général ça arrivera
un vendredi soir avant de partir en vacances... On utilise massivement [Jenkins](http://jenkins-ci.org/)
sur un petit serveur en local en tant que serveur d'intégration continue,
et aussi pour automatiser le déploiement. Certains collègues pointilleux
([pas de noms!](/about.html#guillaume)) feraient remarquer qu'il faut un peu plus de 1 clic pour déployer,
certes. On pourrait rendre ça encore plus *direct*, mais pour le moment ça répond bien à notre manière de
travailler.

###Do you make daily builds?

Une question pas facilement transposable à une application Web. Mais nous essayons le plus possible
de respecter les principes du [déploiement continu](http://en.wikipedia.org/wiki/Continuous_delivery),
donc il n'est pas rare que l'on déploie une nouvelle version par jour, ou même plusieurs. Par contre,
lors de notre processus de déploiement nous avons la possibilité de faire un *rollback* pour revenir à
la version précédente si l'on détecte des erreurs.

###Do you have a bug database?

> Mais pour quoi faire? Nous n'avons pas de bugs!

*(rires jaunes)*

C'est indispensable, mais dans notre cas elle se limite à un mur tapissé de post-it (*true story*).
On en a essayé quelques outils, mais nous étions alors encore une (très) petite équipe et nous avions
du mal à nous adapter aux outils. On a donc préféré utiliser quelque chose de très simple à utiliser
et qui pouvait s'adapter à nous, évoluer rapidement lorsque l'on change de *process*. La tableau
*physique* nous a semblé le plus naturel. Aujourd'hui, on commence à regarder du côté des *vrais* outils,
pour avoir un meilleur suivi. Bon, disons qu'on compte un demi point pour cette question[^IKnow].

[^IKnow]: Oui, je sais, on n'a pas le droit normalement, mais bon... :)

###Do you fix bugs before writing new code?

Oui, surtout s'ils empêchent le site de tourner! Heureusement, la plupart sont detectés avant d'être
remontés en production, grâce aux [tests unitaires](http://docs.atoum.org/fr/),
aux revues de code ou aux tests *à la main*.
Ensuite il est vrai que tous les bugs n'ont pas la même criticité et que parfois certains finissent
sur notre tableau de post-it, parce qu'ils sont rares ou avec un trop faible impact par rapport
aux autres tâches en cours.

###Do you have an up-to-date schedule?

Nous nous appuyons beaucoup sur les méthodes Agiles pour travailler, et notre mur de post-it
nous permet d'avoir une vue globale de *qui travaille sur quoi*, et de *qu'est ce qu'il reste à faire*.
Mais dans notre contexte de *startup dynamique*, il nous arrive parfois d'interrompre un sprint[^IKnow]
pour attaquer d'autres tâches qui deviennent subitement beaucoup plus importantes. Nous essayons
de trouver le bon équilibre entre la théorie et la pratique. Comptons un demi point pour celle là aussi[^IKnow].

###Do you have a spec?

Pour chaque nouvelle fonctionnalité, nous avons un descriptif détaillé de la part de l'équipe *fonctionnelle*.
Cela se fait en général devant un tableau blanc, avec beaucoup de dessins et de discussions. Et comme
on ne pense jamais à tout du premier coup, le reste se fait souvent avec quelques allers-retours
supplémentaires entre ceux qui ont les *idées* et ceux qui savent comment les réaliser.

###Do programmers have quiet working conditions?

Oui, et c'est très agréable!
Tout d'abord notre matériel est neuf, acheté à l'arrivée de chaque développeur, et correspondant au mieux à ces
habitudes (souris, trackball, disposition du clavier...). Bien qu'on soit tous dans le même bureau, les développeurs
ont leur propre espace pour ne pas déranger les autres, ni être trop dérangés. Certains aiment bien écouter
de la musique au casque en travaillant, mais on reste toujours joignable via notre système de *tchat* (Hipchat, et Google).
Nos locaux sont au sein [d'une pépinière d'entreprise](http://rives-numeriques.fr/),
ce qui nous permet de partager des espaces communs agréables et tous neufs: cuisine, salles de réunions, babyfoot...
Non, franchement, on ne se plaint pas :)

###Do you use the best tools money can buy?

Dans une startup, les dépenses ne peuvent pas se faire à la légère, mais lorsqu'on c'est pour nous faire
gagner du temps ou en qualité on hésite rarement. Pour l'IDE, nous avons des licences
[PhpStorm](http://www.jetbrains.com/phpstorm/). Pour le site internet, nous faisons appel à plusieurs services
tiers payants pour économiser en temps, et gagner en qualité (emails, moteur de recherche...).


###Do you have testers?

Non, nous n'avons pas de testeurs à temps plein. L'équipe fonctionnelle valide chaque livraison d'une
nouvelle fonctionnalité pour vérifier que ça colle avec la demande, qu'il n'y a pas de bugs de dernière minute.

###Do new candidates write code during their interview?

Oui, mais surtout on essaye de parler pour mieux se connaître et savoir si on travaille de la même façon.
Cela sera plus détaillé dans un prochain article sur notre processus de recrutement.

###Do you do hallway usability testing?

Encore un demi point en vue. Grâce aux revues de code, chaque développeur peut faire des remarques
sur le nouveau code, et notre équipe fonctionnelle est souvent sollicitée pour valider
et définir les interfaces utilisateurs, mais on sait qu'on peut encore s'améliorer sur ce point.

#Le total

C'est l'heure des comptes! Les pessimistes nous donnerons 7/12, les optimistes 8,5/12.
L'article original dit que la plupart des société de logiciel tournent autour de 2 ou 3...
j'espère vraiment que ça a évolué en 10 ans. Quelque soit la note obtenue, on peut toujours
s'améliorer, et c'est ce que nous essayons de faire progressivement, en accord avec
toutes les *à côtés* qui peuvent survenir dans la vie d'une startup.