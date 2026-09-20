# Traitement d'un fichier EAD/XML minimal

Cet exemple présente la transformation d'un fichier **EAD/XML simple** en une série de ressources pouvant ensuite être importées dans Lodex.

L'objectif n'est pas encore de construire un loader EAD générique, mais de comprendre comment une structure hiérarchique EAD peut être **dépliée en ressources indépendantes** avec les outils déjà disponibles dans les loaders Lodex/EZS.

---

## 1. Fichier utilisé

Le fichier utilisé pour ce test est :

```text
NL-TbRAT-115_916_minimal.xml
```

Il contient la description d'une unité archivistique et de dix éléments qui lui sont rattachés.

Sa structure générale est relativement simple :

```text
ead
└── archdesc
    ├── did
    │   ├── unitid
    │   ├── unittitle
    │   └── ...
    │
    └── dsc
        ├── c
        ├── c
        ├── c
        ├── ...
        └── c
```

L'élément `archdesc` constitue ici la description principale.

Les dix éléments `c` contenus dans `dsc` correspondent aux éléments enfants de cette description.

---

## 2. Objectif de la transformation

Le document XML contient une hiérarchie :

```text
archdesc
│
└── dsc
    ├── c
    ├── c
    ├── c
    ├── ...
    └── c
```

Cette structure doit être transformée en une succession de ressources indépendantes :

```text
archdesc
c
c
c
...
c
```

Dans cet exemple, le résultat attendu est donc :

```text
1 archdesc
10 c
---------
11 ressources
```

La relation entre chaque composant `c` et son parent doit cependant être conservée.

Chaque enfant reçoit donc également :

```text
parentIdentifiant
parentTitre
```

ainsi que deux informations ajoutées lors de la transformation :

```text
baliseEad
niveau
```

---

## 3. Lecture du document XML

Le traitement commence par :

```ini

[XMLParse]
separator = /ead/archdesc
```

`XMLParse` permet de transformer le XML en objet exploitable par les instructions suivantes du loader.

Le séparateur :

```ini
separator = /ead/archdesc
```

indique que l'on souhaite récupérer l'élément `archdesc`.

À ce stade, on dispose encore d'un objet hiérarchique contenant notamment :

```text
archdesc
├── did
└── dsc
    └── c
```

Les composants `c` sont donc toujours contenus à l'intérieur de leur parent.

---

## 4. Le texte des éléments XML

Après le passage dans `XMLParse`, le contenu textuel d'un élément XML est accessible avec la propriété `$t`.

Par exemple, pour une structure XML de ce type :

```xml
<unitid>NL-TbRAT-115_916</unitid>
```

la valeur est accessible dans le loader avec :

```ini
get("did.unitid.$t")
```

---

## 5. Transformer la hiérarchie

La transformation principale commence par :

```ini
[exchange]
value = self().thru(parent => \
```

`self()` représente l'objet actuellement traité, ici `archdesc`.  

La fonction `thru` permet de manipuler cet objet avec Lodash.  

On lui donne ici le nom `parent` afin de rendre la suite du traitement plus lisible :

```text
parent = archdesc
```

L'objectif est alors de construire un tableau contenant :

```text
[
  archdesc,
  c,
  c,
  c,
  ...
]
```

---

## 6. Transformer `archdesc` en ressource

La première partie construit la ressource correspondant au parent :

```ini
[ \
    _.set( \
        _.set(_.omit(parent, ['dsc']), 'baliseEad', 'archdesc'), \
        'niveau', 0 \
    ) \
]
```

### Supprimer les descendants

L'instruction :

```javascript
_.omit(parent, ['dsc'])
```

supprime `dsc` de la ressource.

C'est une étape importante.

Sans cette suppression, la ressource `archdesc` conserverait également tous ses composants enfants :

```text
archdesc
└── dsc
    ├── c
    ├── c
    └── ...
```

Lors de l'aplatissement ultérieur des données, toute cette structure serait alors transformée en colonnes.

On souhaite au contraire obtenir une ressource indépendante :

```text
archdesc
├── did
├── ...
├── baliseEad
└── niveau
```

sans ses descendants.

### Identifier le type de ressource

On ajoute ensuite :

```javascript
_.set(..., 'baliseEad', 'archdesc')
```

puis :

```javascript
_.set(..., 'niveau', 0)
```

La ressource obtenue possède donc :

```text
baliseEad = archdesc
niveau    = 0
```

---

## 7. Récupérer les composants `c`

Les enfants sont récupérés avec :

```javascript
_.get(parent, 'dsc.c')
```

Par sécurit" on utilisera :

```javascript
[].concat(_.get(parent, 'dsc.c')).filter(Boolean)
```

Cette écriture permet de toujours travailler avec un tableau.

Selon le contenu du XML, un élément peut en effet être représenté comme un objet unique ou comme un tableau d'objets.

Ainsi :

```javascript
[].concat(...)
```

permet d'obtenir une structure homogène :

```text
[
  c,
  c,
  c,
  ...
]
```

et :

```javascript
.filter(Boolean)
```

élimine une éventuelle valeur vide.

---

## 8. Transformer chaque composant `c`

Chaque enfant est ensuite traité avec :

```javascript
.map(child => ...)
```

On ajoute à chaque composant :

```text
baliseEad = c
niveau    = 1
```

avec :

```javascript
_.set(child, 'baliseEad', 'c')
```

et :

```javascript
_.set(..., 'niveau', 1)
```

On obtient ainsi deux niveaux :

```text
niveau 0 → archdesc
niveau 1 → c
```

---

## 9. Conserver la relation avec le parent

Le dépliage transforme les éléments `c` en ressources indépendantes.

Il faut donc conserver explicitement l'information permettant de retrouver leur parent.

On ajoute :

```javascript
'parentIdentifiant',
_.get(parent, 'did.unitid.$t')
```

et :

```javascript
'parentTitre',
_.get(parent, 'did.unittitle.$t')
```

Si `baliseEad` et `niveau` étaient déterminés manuellement, `parentIdentifiant` et `parentTitre` sont eux créés dynamiquement.

Chaque composant `c` possède alors des informations de ce type :

```text
baliseEad          = c
niveau             = 1
parentIdentifiant  = NL-TbRAT-115_916
parentTitre        = C.J.M. HEUFKE, 1917-1935, Repertoires, 1917
```

La hiérarchie initialement exprimée par l'imbrication XML est ainsi transformée en **relation explicite entre ressources**.

---

## 10. Construire le tableau final

La ressource `archdesc` est combinée avec les ressources `c` grâce à :

```javascript
.concat(...)
```

On construit donc progressivement :

```text
[
  archdesc,
  c1,
  c2,
  c3,
  ...
  c10
]
```

À ce stade, le traitement possède toujours **un seul objet**, qui est en réalité un tableau contenant les onze ressources.

---

## 11. Déplier le tableau avec `ungroup`

La dernière instruction est :

```ini
[ungroup]
```

`ungroup` transforme chaque élément du tableau en une ressource distincte.

On passe donc de :

```text
[
  archdesc,
  c1,
  c2,
  ...
  c10
]
```

à :

```text
archdesc
c1
c2
c3
...
c10
```

Chaque élément devient alors une ressource indépendante dans le flux de traitement.

---

## 12. Résultat obtenu

Le test produit bien :

```text
11 ressources
```

réparties de la manière suivante :

```text
baliseEad    niveau    nombre
--------------------------------
archdesc       0          1
c              1         10
--------------------------------
total                    11
```

La transformation peut être représentée ainsi :

```text
EAD/XML

archdesc
│
└── dsc
    ├── c
    ├── c
    ├── c
    ├── ...
    └── c

        │
        │ XMLParse + transformation
        ▼

JSONL

archdesc
c
c
c
...
c

        │
        ▼

11 ressources indépendantes
```

---

## 13. Conclusions

Ce premier test montre qu'il est possible, avec les mécanismes existants des loaders Lodex/EZS :

- de lire un document EAD/XML gràace à l'instruction EZS `[XMLParse]`
- de récupérer une structure hiérarchique
- d'identifier le parent et ses enfants
- de transformer les différents niveaux en ressources indépendantes
- de conserver explicitement la relation avec le parent
- de produire un datasete préparé pour l'import dans Lodex.

Le traitement reste cependant spécifique à la structure de ce fichier :

```text
archdesc
└── dsc
    └── c
```

Il ne constitue donc pas encore un loader EAD générique.

D'autres fichiers EAD peuvent utiliser des structures différentes, par exemple :

```text
archdesc
└── dsc
    └── c01
        └── c02
            └── c03
```

L'étape suivante du POC consiste donc à tester le même principe sur une structure hiérarchique plus profonde.