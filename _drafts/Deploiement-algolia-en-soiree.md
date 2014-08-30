---
layout: post
title: Petite histoire de la mise en production d'algolia
category: blog
tags: Wizacha Algolia Deploiement Anecdote
author: Guillaume
banner: wizacha-behind-the-scenes.png
---

#Introduction
L'objectif principal de ce [sprint](http://fr.wikipedia.org/wiki/Scrum_%28m%C3%A9thode%29#Le_sprint)
était l'intégration du moteur de recherche [Algolia](https://www.algolia.com/) afin d'avoir
quelque chose de plus sympathique au niveau de resultat de recherche que ce que fournit CsCart de base.

Nous nous sommes donc retrouvé à tester la fusion de la branche algolia avec la branche principal le vendredi après-midi.
Quelques corrections de bugs plus loin, on décide de mettre en prod dans l'après-midi si tout se passe bien puisque le
rollback sur cette fonctionnalité est immédiat et sans perte de données. Quatre heure de l'apres-midi, on finalise deux
trois corrections, on pousse sur bitbucket, on lance les tests.

**PAF** echec. Rapide coup d'oeil au rapport. C'est un bout de code que j'ai donné mais qui a été recopié sans vérifier
et sans relancer le test en local. Zou, on coupe la poire en deux, on prend chaqun la moitié des pompes, on corrige vite
fait, on pousse, on relance les tests. «Question bête, on a donné les droits d'accès à l'application sur la queue ?» ...
«Euh...».

«Bon, ils en sont où les tests ?», «Zut, coincé sur la resolution des dépendances, on a eu un soucis réseau au mauvais
moment, je relance !»

«Déjà 17h15, plus le temps de déployer, plus le temps d'importer toutes les données dans algolia, ça ne va pas être
jouable dans un temps raisonnable. On fera ça lundi». Et chacun retourne à son foyer.

#Le déploiement

##21h : La prise de contact

Je bulle gentiment sur mon pc.

*Notification HipChat* "Benoit : (caruso)"

À ce moment là, sentant arriver la suite, je lance ma plus belle console. Petit coup d'historique, premier tunnel SSH.
Petit coup d'historique, deuxiéme tunnel SSH. Je suis chez moi comme au boulot. Petite capture d'écran.

Benoit : «Bon, tu veux essayer de déployer le bazar? ou sinon on se garde ça pour Lundi». En réponse, la capture d'écran
la réponse : «Je suis chaud comme la braise» (oui, en début de soirée, je ne suis pas inspiré».

Petites vérification sur le déroulement des tests de 17h, test rapide pour voir si on peut partager nos écrans. Tant pis.

C'est parti mon kiki.

##21h15 : le Merge

«heu... il y a des trucs (genre addon) à installer en prod après?», «Il y a la liste sur le tableau du sprint» (oui, il
est purement physique)

##21h25 : LIVE__marketplace - #67 Started by changes from Guillaume Rossignol , Arnaud Benassy , Benoit Viguier (208 file(s) changed)

(note personnelle : j'aime les notifications hipchat de jenkins)

###21h30 : Création de la tâche jenkins pour mettre à jour tout l'index

Pendant le déploiement on s'est dit qu'initier tout l'index, même si on met en queue les appels à algolia, pourrait prendre
du temps. Pour être sur de ne pas taper un timeout sur le back-office, on décide de lancer l'init par ligne de commande.
 Pour ne pas avoir a me connecter directement sur la machine, et parce qu'on risque d'avoir à le refaire dans le futur,
  j'écris rapidement un job jenkins qui fait cela.

##21h31 : DEPLOY_Front_and_API - #81 FAILURE after 2 mn 28 s

**Branle-bas de combat** Rapide coup d'oeil sur les logs, rapide coup d'oeil sur AWS. Une des instances EC2 a été mis
en "instable" par cloudformation. Il n'y a pas de raison pour que ca vienne de nous, je relance la tâche.
(Pour les plus attentif, j'ai volontairement retiré la notification de succés de la premiere tâche et le lancement de celle-là.)

À ce moment, je ne peux m'empêcher de me demander s'il n'existe pas un dieu de l'informatique et si ce n'est pas sa façon
de me dire que «NON, ON NE DÉPLOIE PAS UN VENDREDI À 21H !!!!!» (oui, cinq points d'exclamations)

##21h42 : DEPLOY_Front_and_API - #82 Back to normal after 9 mn 32 s
Je lance l'initialisation d'algolia.

##21h42 Un chouilla après

> Benoit : «attend... le back me dit qu'il n'y a rien dans l'index... et dans le front j'ai des resultats...»

> Moi : «Ah tiens, il y a bien les messages dans la queue»

> B : «Rien ne se met à jour sur algolia»

> M : «mais en fait, les messages ne sont pas traités par les workers»

###21h51 : La révélation
Donc en fait, j'avais bien fait la mise à jour du fichier de config pour le back en précisant, les nouvelles queues, mais
je n'avais pas fait la mise à jour dans la configuration des workers. On avait des résultats en front car on a une
couche d'abstraction sur les recherches qui permet, si algolia n'est pas disponible, ou pas configuré (comme sur les
posts de développement par exemple) d'avoir des resultat via des recherches SQL.

##21h52 : On relance DEPLOY_Front_and_API

###21h53 : Je souhaite une bonne nuit à ma femme

###21h54 : Je m'étale sur le canapé.
Enfin dans une vraie position de travail !

###22h01 : Les premiers messages de la queue sont traités
La tâche de déploiement n'est pas encore finie car elle se déclare finie une fois que les DNS sont à jour. Cependant nos
workers commencent le boulot dès que la machine est lancée.

###22h03 : DEPLOY_Front_and_API - #83 Success after 11 mn

> M : «il va peut etre falloir vider le cache pour que le bloc de recherche soit le bon ?
      comment je peux savoir sur la page quel service je tape ?»
> B : «vu comme c'est rapide maintenant, on tape bien sur algolia»

##22h10 : On se rend compte que les navigateurs râlent lors des recherches
Benoit : «par contre, je pense que le worker genere les template en http (pas en https), du coup les images sont taggées
 en contenu insecure on dirait... j'ai des msg ds firefox»

Les workers ne repondent pas à des requetes http, mais traitent des messages. Du coup, on ne s'est jamais posé la question
si les workers se considerent en http ou https. De plus, CsCart ( de maniére blameless, mais c'est quand même
un peu de sa faute) fait reposer un certain nombre de fonctions centrales (comme la génération des urls) sur des
constantes définie à l'initialisation. Conséquence immédiate, c'est très dur de faire de l'injection de dépendances. En
plus c'est un appel de fonction planqué dans un template.

Bref, on a laissé passer un bug. Pour le coup, il est un petit peu gênant, mais les navigateurs sont heureus dés qu'on
retourne sur les pages produits, donc ça ne mérite pas un rollback.

##22h12 : Un petit tweet pour la forme
**Note pour la prochaine fois** : Envisager un live-tweet

Papotages avec le directeur technique sur les prochaines tâches à traiter, sur les outils, etc.

##22h41 : Erreur 403 de la part d'algolia.
Du coup, je me jette sur SQS pour voir ce que font nos messages. Globalement, ils stagnent. Au moment où je commence à
taper la commande SSH pour me connecter aux workers, benoit confirme ce qu'il craignait : «Too many requests». On a tapé
le quota sur la machine. Algolia, conformement à notre configuration, nous interdit de nous mettre à jour.

Naïvement je m'exclame «Ben on augmente le quota». «Oui, mais du coup, on doit aussi changer la clé, donc est-ce qu'on
change la config en prod à la volée» ?

###22h46 : Les messages sont à nouveau traités

###23h01 (je trouve que le 1 reviens souvent) : Discution docker
Benoit : «Ca va vachement vite un `docker pull node` avec ma grosse connection en fibre optique ; c'est vachement
bien les gros tuyau. Je me demande comment on peut faire avec des débits plus lent» (je ne garantis pas l'exactitude de
la citation).

Moi : «Hmmm, ch'ais pas, il me faut un bon quart d'heure moi»

##23h26 : Messages restant dans la file : 0
La recherche est pertinente, dynamique, rapide.

**Mission accomplie**

#Le bilan
##Les points positifs

* Algolia (le fossé entre la recherche qu'on avait en sql et celle qu'on a maintenant est hallucinant)
* L'architecture de déploiement s'est revelée robuste.
On a un déploiement qui a échoué, on a redéploier plusieurs fois des machines, et cependant, tous les services
sont resté fonctionnels pendant toute la durée du déploiement.
* L'architecture du site s'est revelée robuste.
J'avais fait une erreur sur la configuration de la recherche en front. Le site est quand même resté fonctionnel. Du
coup, on sait que si algolia tombe, ça ne pose pas de probléme majeur.
* La préparation du déploiement : malgré quelques loupé en amont du déploiement, au moment du déploiement, on avait
tout sous la main pour être réactif et analyser les problémes qui se sont présentés.

##Les points négatifs

* On s'y est repris à plusieurs fois pour déployer
* On a fait une modication dans un fichier de config php à la main sur le serveur de prod. Ce qui laisse supposer que
ca devrait être un reglage de l'application qu'on devrait pouvoir modifier dans le back-office
* On a manqué de controle en amont du déploiement

#Les axes d'améliorations
Le déploiement devrait être plus resistant aux echecs, re-essayer en fonction de l'erreur rencontrée.

Il faut mettre de la qualité dans jenkins. Si quelqu'un fait une modification dans un fichier pour remplacer un
`$count = $count + 1 ` par un `$count++` deux autres personnes vont lire la ligne et l'approuver. Par contre, si quelqu'un
supprime la moitié des fichiers de config ou décide de rajouter des etapes au build, personne ne va aller vérifier.

Les déploiements sont longs. Même une fois qu'on a passé les tests, il faut au moins un quart d'heure pour déployer une
pile compléte (front, back, api, workers). Ce n'est pas dramatique vu notre taille actuelle et qu'on fait en sorte
d'être retro-compatible pour que plusieurs versions puissent se chevaucher. Cependant, ca va finir par devenir gênant.

#Conclusion
Je suis très content qu'on ait enfin un moteur de recherche performant. On a tous passés du temps sur l'intégration
d'algolia et ce premier resultat est plus que probant.

N'hésitez pas à signaler et/ou à corriger les coquilles, fautes et erreurs que vous pourriez constater dans cet articles.


...

