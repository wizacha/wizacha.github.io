---
layout: post
title: Petite histoire de la mise en production d'Algolia
category: blog
tags: Wizacha Algolia Deploiement Anecdote
author: guillaume
banner: algolia-night.png
---

C'est bien connu, on ne [déploit jamais un vendredi!](http://www.estcequonmetenprodaujourdhui.info/)
C'est vrai, imaginez que ça se passe mal, ou qu'il y ai un ennui de dernière minute.
Ceci étant dit...

#Préambule
##Notre stack technique
Pour les besoins de cet article, voici une description partielle de notre stack. Il y aura certainement un article plus complet un jour pour décrire toute notre architecture.

* [Jenkins](http://jenkins-ci.org/) : Pour lancer les tests, les déploiements, etc.
* [Atoum](https://github.com/atoum/atoum) : Pour les tests unitaires
* [BBQ](https://github.com/eventio/bbq) : Pour la mise en queue d'actions
* [AWS](http://aws.amazon.com/) : Comme solution de cloud, et notamment : 
 * CloudFormation : Permet de déployer un ensemble de machines et de services
 * SQS : Système de file de message (on a écrit un driver pour BBQ)

##Notre architecture web
Notre marketplace tourne sous un [CsCart](http://www.cs-cart.com) beaucoup modifié.
A chaque déploiement, nous lançons, via un template CloudFormation et un ensemble de scripts bash : 

* Un groupe de machines pour le back marchands
* Un groupe de machines pour le front clients {{site.wizacha}}
* Un groupe de machines pour l'API
* Un groupe de machines *workers* pour la gestion des messages en file
* La mise en place de la doc de l'API en statique sur [S3](http://docs.aws.amazon.com/gettingstarted/latest/swh/website-hosting-intro.html)

#Introduction
L'objectif principal de ce [sprint](http://fr.wikipedia.org/wiki/Scrum_%28m%C3%A9thode%29#Le_sprint)
était l'intégration d'un moteur de recherche pour palier le manque de glamour de la recherche de CsCart.

##Les choix techniques
Nous avons envisagé un temps de gérer une base de données NoSQL pour la gestion des recherches. Finalement
suite à une discussion avec la maîtrise d'ouvrage, tout le monde était d'accord pour utiliser une solution 
«toute prête» pour nous faire gagner de la qualité, et du temps (et donc de l'argent).

Nous avons retenu [Algolia](https://www.algolia.com/).

###Coté back
La mise a jour des données dans algolia sera faite de manière asynchrone grâce aux files de messages.
Ça permet de garantir que les données seront toujours à peu près justes même si on a une panne réseau temporaire.
Ça permet aussi de ne pas se poser la question de ce qu'il faut faire lorsqu'un marchand veut mettre à jour un produit et qu'Algolia n'est pas disponible. Le message restera juste un peu plus longtemps dans la file. (**ALERTE SPOILER** ça va se révéler une excellente décision)

De plus, CsCart ne vérifiant pas si un produit est modifié lors d'une mise à jour avant de le déclarer mis à jour,
nous avons mis en place un systéme qui permet de savoir si un produit a réellement été modifié ou non.
Tout ça dans le but de ne pas renvoyer des données à algolia si on n'en a pas besoin. Plusieurs dizaines de milliers de produits (à l'heure où j'écris ces lignes) étant mis à jour via CSV chaque nuit, il serait
rapidement couteux de tout mettre à jour systématiquement dans algolia.

###Coté site web
Algolia garantissant des temps de réponse rapides, nous avons décidé d'implémenter la recherche entièrement
coté javascript. La documentation d'algolia est bien faite et vient avec des exemples. De toutes façons que 
la réquete à algolia soit faite par le navigateur ou par nos serveurs, ça reste toujours une requete vers
algolia. Autant décharger au maximum notre infrastructure, surtout lorsque ce n'est pas plus cher.

#Le déploiement

##Ce qui était prévu en ce bel après midi de vendredi

Nous devions donc tester la fusion de la branche algolia avec la branche principale le vendredi après-midi, lancer les tests unitaires, déployer trois heures avant de finir la journée, s'assurer que tout marche
et rentrer chez nous, remplis du sentiment du devoir accompli.

##Ce qui s'est réellement passé

> Tiens, dans ce cas là le produit n'est pas mis à jour !

> Tiens, ici CsCart fait une mise à jour directement dans la base de données, du coup, notre hook n'est pas utilisé !

> Tiens, les tests unitaires ne marchent plus ! 

> Quelqu'un a donné les droits sur la file SQS pour la prod ? 

Finalement, la branche prête, il est 17 heures. On est vendredi, nous sommes raisonnables, le déploiement est reporté.

###21h : La prise de contact

Je bulle gentiment sur mon pc.

> *Notification HipChat* "Benoit : (caruso)"

À ce moment là, sentant arriver la suite, je lance ma plus belle console. Petit coup d'historique, premier tunnel SSH.
Petit coup d'historique, deuxième tunnel SSH. Je suis chez moi comme au boulot.

> Benoit : «Bon, tu veux essayer de déployer le bazar? ou sinon on se garde ça pour Lundi».

> Guillaume (moi) : «Je suis chaud comme la braise» (oui, en début de soirée, je ne suis pas forcement inspiré.)

Petites vérification sur le déroulement des tests de 17h, test rapide pour voir si on peut partager nos écrans. Echec. Tant pis.

C'est parti mon kiki.

###21h15 : le Merge

> «heu... il y a des trucs (genre addon) à installer en prod après?»

> «Il y a la liste sur le tableau du sprint... au boulot»

(oui, il est purement physique)

###21h25 : `LIVE__marketplace - #67 Started by changes from Guillaume Rossignol , Arnaud Benassy , Benoit Viguier (208 file(s) changed)`

(note personnelle : j'aime les notifications [hipchat](https://www.hipchat.com/) de jenkins)

###21h30 : Création de la tâche jenkins pour mettre à jour tout l'index

Pendant le déploiement on s'est dit qu'initialiser tout l'index, même si on met en queue les appels à algolia, pourrait prendre
du temps. Pour être sûr de ne pas taper un timeout sur le back-office, on décide de lancer l'init par ligne de commande.

Pour ne pas avoir à me connecter directement sur la machine, et parce qu'on risque d'avoir à le refaire dans le futur, j'écris rapidement un job jenkins qui fait cela.

###21h31 : `DEPLOY_Front_and_API - #81 FAILURE after 2 mn 28 s`

**Branle-bas de combat** Rapide coup d'oeil sur les logs, rapide coup d'oeil sur AWS. Une des instances EC2 a été mise
en *instable* par CloudFormation. Il n'y a pas de raison pour que ça vienne de nous, je relance la tâche.
(Pour les plus attentifs, j'ai volontairement retiré la notification de succès de la première tâche et le lancement de celle-là.)

À ce moment, je ne peux m'empêcher de me demander s'il n'existe pas un dieu de l'informatique et si ce n'est pas sa façon
de me dire que «NON, ON NE DÉPLOIE PAS UN VENDREDI À 21H !!!!!» (oui, cinq points d'exclamations)

###21h42 : `DEPLOY_Front_and_API - #82 Back to normal after 9 mn 32 s`
Je lance l'initialisation d'algolia.

###21h42 Un chouilla après

> B : «Attend, c'est pas cohérent... le back me dit qu'il n'y a rien dans l'index... et dans le front j'ai des résultats...»

> G : «Ah tiens, il y a bien les messages dans la queue»

> B : «Rien ne se met à jour sur algolia»

> G : «mais en fait, les messages ne sont pas traités par les workers»

###21h51 : La révélation
Donc en fait, j'avais bien fait la mise à jour du fichier de config pour le back en précisant, les nouvelles queues, mais
je n'avais pas fait la mise à jour dans la configuration des workers. On avait des résultats en front car on a une
couche d'abstraction sur les recherches qui permet, si algolia n'est pas disponible, ou pas configuré (comme sur les
posts de développement par exemple) d'avoir des resultat via des recherches SQL.

###21h52 : On relance `DEPLOY_Front_and_API`

###21h54 : Je m'étale sur le canapé.
Enfin dans une vraie position de travail !

###22h01 : Les premiers messages de la queue sont traités
La tâche de déploiement n'est pas encore finie car elle se déclare finie une fois que les DNS sont à jour. Cependant nos
workers commencent le boulot dès que la machine est lancée.

###22h03 : `DEPLOY_Front_and_API - #83 Success after 11 mn`

> G : «il va peut etre falloir vider le cache pour que le bloc de recherche soit le bon ?
      comment je peux savoir sur la page quel service je tape ?»

> B : «vu comme c'est rapide maintenant, on tape bien sur algolia»

###22h10 : On se rend compte que les navigateurs râlent lors des recherches (perte du cadenas https)
> B : «par contre, je pense que le worker génère les templates en http (pas en https), du coup les images sont taggées en contenu insecure on dirait... j'ai des msg dans firefox»

**Explication** :
Pour que l'affichage de la requete puisse être traité exclusivement entre le navigateur et algolia, il faut 
bien à un moment stocker chez algolia le bout d'HTML qui sera rendu dans la page de recherche.
On s'est rendu compte que les url des images sont en `http://`  au lieu de `https://` Du coup, les navigateurs
font remarquer à juste titre qu'on a un contenu non sécurisé sur une page qu'on voudrait sécurisée. Du coup, 
adieu joli cadenas vert.

Les workers ne répondent pas à des requêtes http, mais traitent des messages. Du coup, on ne s'est jamais posé la question
si les workers se considèrent en http ou https. De plus, CsCart ( de manière blameless, mais c'est quand même
un peu de sa faute) fait reposer un certain nombre de fonctions centrales (comme la génération des urls) sur des
constantes définies à l'initialisation. Conséquence immédiate, c'est très dur de faire de l'injection de dépendances. En
plus c'est un appel de fonction planqué dans un template.

Bref, on a laissé passer un bug. Pour le coup, il est un petit peu gênant, mais les navigateurs sont heureux dès qu'on
retourne sur les pages produits, donc ça ne mérite pas un rollback.

###22h12 : Un petit [tweet](https://twitter.com/Rgousi/statuses/505449895445397507) pour la forme
**Note pour la prochaine fois** : Envisager un live-tweet

Papotage avec le directeur technique sur les prochaines tâches à traiter, sur les outils, etc.

###22h41 : Erreur 403 de la part d'algolia.
Du coup, je me jette sur SQS pour voir ce que font nos messages. Globalement, ils stagnent. Au moment où je commence à
taper la commande SSH pour me connecter aux workers, Benoit confirme ce qu'il craignait : `«Too many requests»`. On a tapé
le quota sur la machine. Algolia, conformément à notre configuration, nous interdit de nous mettre à jour.

> G: «Ben on augmente le quota».

> B: «Oui, mais du coup, on doit aussi changer la clé, donc est-ce qu'on
change la config en prod à la volée» ?

###22h46 : Les messages sont à nouveau traités

###22h50 : Tout se déroule normalement

###22h55 : Tout se déroule normalement, quelque chose doit être en train de se préparer.

###23h01 : Moment de détente, discution [docker](http://docker.io)
> B : «Ca va vachement vite un `docker pull node` avec ma grosse connection en fibre optique ; c'est vachement
>bien les gros tuyaux. Je me demande comment on peut faire avec des débits plus lents»

 **Note:** je ne garantie pas l'exactitude cette citation...

> G : «Hmmm, ch'ais pas, il me faut un bon quart d'heure moi»

###23h26 : Message restant dans la file : 0
La recherche est pertinente, dynamique, rapide.

**Mission accomplie**

#Le bilan
##Les points positifs

* Algolia (le fossé entre la recherche qu'on avait en sql et celle qu'on a maintenant est hallucinant)
* L'architecture de déploiement s'est revelée robuste.
On a un déploiement qui a échoué, on a redéployé plusieurs fois des machines, et cependant, tous les services
sont restés fonctionnels pendant toute la durée du déploiement.
* L'architecture du site s'est révélée robuste.
J'avais fait une erreur sur la configuration de la recherche en front. Le site est quand même resté fonctionnel. Du
coup, on sait que si algolia tombe, ça ne pose pas de probléme majeur.
* La préparation du déploiement : malgré quelques loupés en amont du déploiement, au moment du déploiement, on avait
tout sous la main pour être réactifs et analyser les problèmes qui se sont présentés.

##Les points négatifs

* On s'y est repris à plusieurs fois pour déployer
* On a fait une modication dans un fichier de config php à la main sur le serveur de prod. Ce qui laisse supposer que
ca devrait être un reglage de l'application qu'on devrait pouvoir modifier dans le back-office
* On a manqué de controle en amont du déploiement

##Les axes d'améliorations
Le déploiement devrait être plus resistant aux échecs, re-essayer en fonction de l'erreur rencontrée.

Il faut mettre de la qualité dans jenkins. Si quelqu'un fait une modification dans un fichier pour remplacer un
`$count = $count + 1 ` par un `$count++` deux autres personnes vont lire la ligne et l'approuver. Par contre, si quelqu'un
supprime la moitié des fichiers de config ou décide de rajouter des étapes au build, personne ne va aller vérifier.

Les déploiements sont longs. Même une fois qu'on a passé les tests, il faut au moins un quart d'heure pour déployer une
pile complète (front, back, api, workers). Ce n'est pas dramatique vu notre taille actuelle et parce qu'on fait en sorte
d'être retro-compatible pour que plusieurs versions puissent se chevaucher. Cependant, ça va finir par devenir gênant.

##Conclusion
Je suis très content qu'on ait enfin un moteur de recherche performant. On a tous passé du temps sur l'intégration
d'algolia et ce premier résultat est plus que probant.

*Have fun with {{site.wizacha}}*

