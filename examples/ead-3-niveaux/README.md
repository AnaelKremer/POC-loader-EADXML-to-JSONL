# Traitement d'un EAD/XML à trois niveaux de composants

Cet exemple présente la transformation d'un fichier **EAD/XML possédant plusieurs niveaux hiérarchiques** en une série de ressources pouvant ensuite être importées dans Lodex.

Il complète l'exemple précédent, dans lequel la structure EAD ne comportait qu'un seul niveau de composants.

Ici, la description archivistique contient trois niveaux de composants :

```text
archdesc
└── dsc
    └── c01
        └── c02
            └── c03
```

L'objectif est de transformer cette arborescence en ressources indépendantes tout en conservant les informations permettant de retrouver leur position dans la hiérarchie.

---

## 1. Fichier utilisé

Le fichier utilisé pour ce test est :

```text
NL-TbRAT-115.xml
```

Sa structure est plus complexe que celle du premier exemple.

Le document contient :

```text
1 archdesc
48 c01
90 c02
1187 c03
```

soit au total :

```text
1326 éléments à transformer en ressources
```

La structure générale peut être représentée ainsi :

```text
archdesc
│
└── dsc
    │
    ├── c01
    │   ├── c02
    │   │   ├── c03
    │   │   ├── c03
    │   │   └── ...
    │   └── ...
    │
    ├── c01
    │   └── ...
    │
    └── ...
```

---

## 2. Objectif de la transformation

Comme dans le premier exemple, Lodex doit pouvoir travailler avec des ressources indépendantes.

Il faut donc passer d'une structure imbriquée :

```text
archdesc
└── c01
    └── c02
        └── c03
```

à une succession de ressources :

```text
archdesc
c01
c02
c03
c03
c02
c03
...
```

La hiérarchie n'est donc plus représentée uniquement par l'imbrication des éléments XML.

Elle doit également être décrite par des informations ajoutées aux ressources :

```text
baliseEad
niveau
parentIdentifiant
parentTitre
```

Les niveaux utilisés dans ce POC sont :

```text
niveau 0 → archdesc
niveau 1 → c01
niveau 2 → c02
niveau 3 → c03
```

---

## 3. Lecture du document XML

Comme dans le premier exemple, le traitement commence par :

```ini
[XMLParse]
separator = /ead/archdesc
```

Le séparateur :

```ini
separator = /ead/archdesc
```

permet de récupérer la description principale et l'ensemble de sa hiérarchie.

À ce stade, on dispose encore d'un seul objet imbriqué :

```text
archdesc
└── dsc
    └── c01
        └── c02
            └── c03
```

Le rôle de la suite du loader est de déplier cette structure.

---

## 4. Construire la liste des ressources

La transformation principale commence par :

```ini
[exchange]
value = self().thru(fonds => \
```

`self()` représente ici l'objet `archdesc` récupéré par `XMLParse`.

On lui donne le nom `fonds` pour faciliter la lecture du traitement.

Le principe consiste à construire progressivement un tableau de cette forme :

```text
[
  archdesc,
  c01,
  c02,
  c03,
  c03,
  c02,
  ...
]
```

Ce tableau sera ensuite déplié afin que chaque élément devienne une ressource indépendante.

---

## 5. Transformer `archdesc` en ressource

On commence par conserver `archdesc` :

```ini
[ \
    _.set( \
        _.set(_.omit(fonds, ['dsc']), 'baliseEad', 'archdesc'), \
        'niveau', 0 \
    ) \
]
```

L'instruction :

```javascript
_.omit(fonds, ['dsc'])
```

supprime `dsc` de la ressource.

Cette suppression est essentielle : `dsc` contient toute la hiérarchie des composants.

Si on conservait cette propriété, la ressource `archdesc` contiendrait encore :

```text
c01
└── c02
    └── c03
```

Lors de l'aplatissement ultérieur des données, tous ces descendants seraient alors également transformés en milier de colonnes.

On ajoute ensuite manuekkement :

```text
baliseEad = archdesc
niveau    = 0
```

La description principale devient ainsi une ressource indépendante, sans ses descendants.

---

## 6. Récupérer les composants `c01`

Les composants de premier niveau sont accessibles dans :

```javascript
_.get(fonds, 'dsc.c01')
```

On utilise :

```javascript
[].concat(_.get(fonds, 'dsc.c01')).filter(Boolean)
```

afin de toujours obtenir un tableau, que le document contienne un ou plusieurs composants.

Chaque `c01` doit ensuite devenir une ressource indépendante.

Pour cela, on supprime ses propres descendants :

```javascript
_.omit(c01, ['c02'])
```

puis on ajoute :

```text
baliseEad = c01
niveau    = 1
```

On obtient donc une ressource `c01` qui ne contient plus ses `c02`.

Le `c01` étant séparé de son `archdesc`, on conserve également les informations permettant d'identifier son parent :

```javascript
'parentIdentifiant',
_.get(fonds, 'did.unitid.$t')
```

et :

```javascript
'parentTitre',
_.get(fonds, 'did.unittitle.$t')
```

Un `c01` contient ainsi des informations de ce type :

```text
baliseEad          = c01
niveau             = 1
parentIdentifiant  = 115
parentTitre        = Notariële archieven Tilburg
```

La relation entre `archdesc` et `c01` reste donc disponible après le dépliage de la structure.

---

## 7. Produire plusieurs ressources à partir d'un même composant

Dans le premier exemple, chaque composant `c` correspondait directement à une seule ressource.

On pouvait donc simplement utiliser `map` :

```javascript
.map(child => ...)
```

La situation est différente dans ce fichier.

Un composant `c01` doit devenir une ressource, mais il contient également des composants `c02` qui doivent eux aussi devenir des ressources indépendantes.

Prenons une structure simplifiée :

```text
c01 A
├── c02 A
└── c02 B

c01 B
└── c02 C
```

Pour chaque `c01`, le traitement construit donc un tableau contenant le `c01` et ses descendants :

```text
c01 A → [c01 A, c02 A, c02 B]

c01 B → [c01 B, c02 C]
```

### Ce que produisait `map`

Un de stests utilisait :

```javascript
.map(c01 => ...)
```

`map` conserverait un résultat pour chaque `c01`.

Comme chacun de ces résultats est lui-même un tableau, on obtennait un **tableau de tableaux** :

```text
[
  [c01 A, c02 A, c02 B],
  [c01 B, c02 C]
]
```

Les ressources n'étaient donc pas toutes placées au même niveau.

### Ce que produit `flatMap`

Dans cet exemple, il faut utiliser :

```javascript
.flatMap(c01 => ...)
```

`flatMap` réalise la transformation puis aplatit d'un niveau les tableaux obtenus.

Le résultat devient directement :

```text
[
  c01 A,
  c02 A,
  c02 B,
  c01 B,
  c02 C
]
```

C'est précisément la structure recherchée avant l'utilisation d' `ungroup`.

`flatMap` est ainsi utilisé lorsque **un élément de départ peut produire plusieurs ressources et que l'on souhaite réunir ces ressources dans un seul tableau**.

## 8. Transformer les composants `c02`

À l'intérieur de chaque `c01`, on récupère :

```javascript
_.get(c01, 'c02')
```

puis :

```javascript
[].concat(_.get(c01, 'c02')).filter(Boolean)
```

Comme pour les `c01`, chaque `c02` doit devenir une ressource indépendante.

On supprime donc ses descendants :

```javascript
_.omit(c02, ['c03'])
```

puis on ajoute :

```text
baliseEad = c02
niveau    = 2
```

---

## 9. Conserver le parent des `c02`

Une fois séparé de son `c01`, le composant `c02` ne possède plus, par son imbrication, l'information permettant de connaître son parent.

Cette relation est donc ajoutée explicitement :

```javascript
'parentIdentifiant',
_.get(c01, 'did.unitid.$t')
```

et :

```javascript
'parentTitre',
_.get(c01, 'did.unittitle.$t')
```

Un `c02` contient ainsi des informations de ce type :

```text
baliseEad          = c02
niveau             = 2
parentIdentifiant  = identifiant du c01
parentTitre        = titre du c01
```

La relation :

```text
c01
└── c02
```

est donc conservée même après la séparation des deux ressources.

---

## 10. Transformer les composants `c03`

Le même principe est appliqué au niveau suivant.

Les `c03` sont récupérés avec :

```javascript
_.get(c02, 'c03')
```

puis normalisés sous forme de tableau avec :

```javascript
[].concat(_.get(c02, 'c03')).filter(Boolean)
```

Chaque `c03` reçoit :

```text
baliseEad = c03
niveau    = 3
```

À ce niveau, il n'y a plus de descendant à supprimer dans le fichier utilisé.

On peut donc directement conserver le composant `c03`.

---

## 11. Conserver le parent des `c03`

Le parent immédiat d'un `c03` est le `c02` dans lequel il se trouvait.

On ajoute donc :

```javascript
'parentIdentifiant',
_.get(c02, 'did.unitid.$t')
```

et :

```javascript
'parentTitre',
_.get(c02, 'did.unittitle.$t')
```

La relation :

```text
c02
└── c03
```

devient ainsi explicite dans les ressources produites.

---

## 12. Construction progressive du tableau

Le traitement construit donc progressivement une liste de ressources.

Pour un document de cette forme :

```text
c01 A
├── c02 A
│   ├── c03 A
│   └── c03 B
└── c02 B
    └── c03 C
```

on obtient :

```text
c01 A
c02 A
c03 A
c03 B
c02 B
c03 C
```

Les descendants ont été séparés de leurs parents, mais les relations nécessaires à la reconstruction de la hiérarchie sont conservées dans les champs ajoutés. Ainsi tous les élélents d'une ressource pourront devenir, dans LODEX, des entités à part entière.

---

## 13. Déplier le tableau avec `ungroup`

À la fin de la transformation, on dispose d'un tableau contenant l'ensemble des ressources :

```text
[
  archdesc,
  c01,
  c02,
  c03,
  c03,
  ...
]
```

La dernière instruction est :

```ini
[ungroup]
```

`ungroup` permet de **séparer les différents éléments du tableau afin de les traiter comme des ressources distinctes**.

On passe donc de :

```text
[
  archdesc,
  c01,
  c02,
  c03,
  ...
]
```

à :

```text
archdesc
c01
c02
c03
...
```

Chaque élément du tableau devient ainsi une ressource indépendante dans le flux de traitement.

---

## 16. Résultat obtenu

Le traitement produit :

```text
1326 ressources
```

réparties ainsi :

```text
baliseEad    niveau    nombre
--------------------------------
archdesc       0           1
c01            1          48
c02            2          90
c03            3        1187
--------------------------------
total                   1326
```

La transformation peut donc être résumée ainsi :

```text
EAD/XML

archdesc
└── 48 c01
    └── 90 c02
        └── 1187 c03

             │
             │ XMLParse + transformation
             ▼

JSONL

1 archdesc
48 c01
90 c02
1187 c03

             │
             ▼

       1326 ressources
```

---

## 17. Conservation de la hiérarchie

Le dépliage du document transforme les différents composants en ressources indépendantes, mais leur position dans la hiérarchie n'est pas perdue.

Chaque ressource, à l'exception de `archdesc`, conserve l'identifiant et le titre de son parent immédiat grâce aux propriétés :

```text
parentIdentifiant
parentTitre
```

On obtient ainsi :

```text
archdesc
└── c01
    parent = archdesc
    │
    └── c02
        parent = c01
        │
        └── c03
            parent = c02
```

Dans le fichier testé, un `c01` contient par exemple :

```text
niveau             = 1
parentIdentifiant  = 115
parentTitre        = Notariële archieven Tilburg
```

Un `c02` conserve de la même manière les informations de son `c01` parent :

```text
niveau             = 2
parentIdentifiant  = identifiant du c01
parentTitre        = titre du c01
```

et un `c03` celles de son `c02` parent :

```text
niveau             = 3
parentIdentifiant  = identifiant du c02
parentTitre        = titre du c02
```

La hiérarchie qui était initialement représentée par l'imbrication des éléments XML est donc transformée en **relations explicites entre les ressources**.

Le document peut ainsi être déplié en 1326 ressources distinctes sans perdre les relations entre ses différents niveaux.

---

## 18. Conclusions

Ce deuxième test montre qu'il est possible, avec les mécanismes existants des loaders Lodex/EZS :

- de lire une structure EAD comportant plusieurs niveaux hiérarchiques
- de parcourir successivement les composants `c01`, `c02` et `c03`, soit tousn les niveaux de hiérarchie connus
- de transformer chaque niveau en ressources indépendantes
- de supprimer les descendants de chaque ressource avant son aplatissement
- de conserver pour chaque composant la relation avec son parent immédiat
- de transformer une arborescence contenant 1326 éléments en 1326 ressources distinctes sans perdre sa structure hiérarchique

> [!WARNING]
> **Le traitement repose actuellement sur une structure connue à l'avance.**
>
> Dans cet exemple, le loader sait explicitement qu'il doit rechercher successivement :
>
> ```text
> dsc.c01
> c01.c02
> c02.c03
> ```
>
> Il connaît donc à l'avance le nom des composants et la profondeur de l'arborescence parce que le XML a été observé en amont.

Le premier exemple utilisait au contraire une autre organisation :

```text
archdesc
└── c
```

Ces deux tests mettent donc en évidence un point important : **la lecture du XML et le dépliage de sa hiérarchie peuvent être réalisés avec les mécanismes existants, mais la structure des fichiers EAD peut varier**.

La généralisation du POC devra par conséquent permettre de **parcourir la hiérarchie sans connaître à l'avance le nombre de niveaux ni la forme utilisée pour représenter les composants**.