---
layout: post
lang: fr
title: OUPS! un bug dans mon backlog!
tags:
- scrum
- kanban
- backlog
---


Au cours de son excellente conférence [l'anatomie d'une mission agile](http://www.areyouagile.com/2011/11/anatomie-dune-mission-agile/) durant laquelle il nous a exposé ses retours d'expériences sur le déploiement de l'agilité au sein d'une société d'édition logiciel, [Pablo Pernot](http://www.areyouagile.com/) a évoqué son échec sur la mise en place d'un KanBan pour la gestion des défauts.

Il est en effet fréquent de constater la mise en place d'un workflow de traitements des bugs dissocié de la gestion centrale du projet. Dans son cas, la gestion du projet était outillée par SCRUM qui est axé autour d'un backlog de produit.

## Rappelons rapidement ce qu'est un backlog de produit:

Un backlog est tout simplement une liste récursive (arborescence) des éléments qui devront composer le produit: fonctionnalités, tests, cas d'utilisations, tâches... plus généralement, c'est une vision à l'instant t de ce que sera le produit dans son état final (et englobe donc ce qu'est le produit dans son état actuel, le cas échéant, il faut se poser des questions...).

Au cours des phases de production, les éléments consituants le backlog sont traités individuellement par l'équipe en respectant un workflow précis garant de la qualité.

Exemple de ce que pourrait être le workflow de traitement d'un élément du backlog:

* priorisation
* plannification
* développement
* test
* dépôt dans la release suivante

## Qu'est-ce qu'un bug?

Un bug est un défaut (visuel, fonctionnel, sécuritaire...) livré sur une release du produit. Généralement il est livré involontairement par l'équipe de production et remonté par les utilisateurs.

Malheureusement, il fait lui aussi partie intégrante du produit.

La correction d'un bug, au même titre que le développement d'une nouvelle fonctionnalité, consomme des ressources. Il est donc tout naturel de penser à en planifier son traitement et à définir un workflow de correction.

Attention: Un défaut sur une fonctionnalité en cours de développement n'est pas un bug

Exemple de ce que pourrait être le workflow de traitement d'un bug:

* priorisation
* plannification
* écriture de tests fonctionels et/ou unitaires reproduisant le bug
* correction
* mise en production
* dépôt dans la release suivante

Concrètement, un bug a toutes les caractéristiques nécessaires à son intégration dans le backlog de produit:

* il qualifie le produit
* il est planifiable
* il nécessite la définition d'un workflow de traitement
* il peut être rattaché à une fonctionnalité (ou autre élément du backlog)

Il est donc tout à fait cohérent de fusioner la gestion des développements de nouvelles fonctionnalités à celle de la correction des bugs et donc: d'intégrer les bugs au backlog de produit et de mutualiser l'outillage.

Un petit exemple de ce que pourrait être un backlog incluant un bug:


    * #feature accounts
    ** #feature registration
    *** #story as an anonymous user I can register new account
    **** #task create registration form
    ***** #defect doesn't prevent CSRF
    ** #feature authentication
    ...
