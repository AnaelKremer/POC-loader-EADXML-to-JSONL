# POC — Loader EAD/XML vers JSONL pour Lodex

Ce dépôt présente un **Proof of Concept** permettant de transformer des fichiers **EAD/XML** décrivant des fonds d'archives hiérarchiques en ressources pouvant être importées dans **Lodex**, soit en JSON lines.

## Exemples

Deux structures EAD/XML différentes sont actuellement présentées dans ce POC :

- [Exemple 1 — EAD minimal](./examples/ead-minimal/README.md)  
  Traitement d'un fichier EAD comportant un `archdesc` et un seul niveau de composants `c`.

- [Exemple 2 — EAD à trois niveaux de composants](./examples/ead-3-niveaux/README.md)  
  Traitement d'un fichier EAD comportant trois niveaux de composants `c01`, `c02` et `c03`, avec conservation des relations hiérarchiques.

## Objectif

Un document EAD peut contenir une structure hiérarchique de ce type :  

```text
archdesc
└── dsc
    └── c01
        └── c02
            └── c03
```

L'objectif du POC est de transformer cette arborescence en une succession de ressources indépendantes :

```text
fonds
c01
c02
c03
c03
c02
c03
...
```

tout en conservant des informations permettant d'identifier le niveau de chaque ressource et son parent.

## Principe

Le loader réalise principalement les opérations suivantes :

1. lecture du fichier EAD/XML avec `XMLParse`
2. récupération de la description principale ( le fonds)
3. parcours des composants hiérarchiques
4. transformation de chaque niveau en ressource indépendante
5. conservation d'informations relatives au parent
6. suppression des descendants avant l'aplatissement des objets

Le dépliage de la hiérarchie permet notamment d'éviter que l'aplatissement d'un objet contenant toute sa descendance ne génère un très grand nombre de colonnes.

## Fichiers de test

Le dépôt contient plusieurs fichiers EAD/XML utilisés pour tester le comportement du loader, ils sont issus de [Archives Portal Europe](https://www.archivesportaleurope.net/tools/for-content-providers/standards/ead3-in-archives-portal-europe/)

Ils permettent notamment d'observer que tous les documents EAD ne représentent pas leur hiérarchie de la même manière.

Une structure peut par exemple utiliser :

```text
c01
└── c02
    └── c03
```

alors qu'une autre peut utiliser :

```text
c
└── c
    └── c
```

Cette diversité constitue un point important pour la généralisation du loader.