Warning: truncated output (original token count: 39438)
Total output lines: 1980

# Architecture

Ce document décrit l'architecture de Pianola, une application de piano roll avec rendu audio intégré.

Il fixe le vocabulaire courant, les responsabilités des couches et leurs dépendances. Les scénarios détaillés sont regroupés dans [etudes-de-cas.md](etudes-de-cas.md).

## Navigation

- [Vue d'ensemble](#vue-densemble)
- [Périmètre fonctionnel](#périmètre-fonctionnel)
- [Domaine](#domaine)
- [Application](#application)
- [Présentation](#présentation)
- [Infrastructure](#infrastructure)
- [Dépendances architecturales](#dépendances-architecturales)
- [Questions ouvertes](#questions-ouvertes)
- [Arborescence cible](#arborescence-cible)

## Vue d'ensemble

Pianola sépare quatre responsabilités :

| Couche | Responsabilité | Exemples |
| --- | --- | --- |
| Domaine | Représenter la composition, garantir ses invariants et expliciter ses échecs attendus. | `Project`, `Score`, `Note`, `Result`, temps musical et pitch |
| Application | Orchestrer l'édition et la lecture à partir du domaine. | état de l'éditeur, cas d'usage, ports |
| Infrastructure | Réaliser les capacités techniques demandées par l'application. | Web Audio, catalogue concret, persistance |
| Présentation | Afficher la grille et traduire les gestes utilisateur. | clips, piano roll, inspecteur |

```mermaid
flowchart LR
    Presentation["Présentation"] --> Application["Application"]
    Application --> Domain["Domaine"]
    Infrastructure["Infrastructure"] --> Application
    Infrastructure --> Domain
```

## Périmètre fonctionnel

Pianola est destiné à l'écriture et au processus initial de composition, pas à la production audio.

Le premier périmètre comprend :

- une grille bidimensionnelle de clips ;
- un axe horizontal représentant un temps global continu ;
- des pistes instrumentales ordonnées sur l’axe vertical, chacune associée à un instrument ;
- un placement horizontal et une référence de piste persistants pour chaque `Clip` ;
- des `Score` constituant des contenus musicaux indépendants, éditables et partageables, même sans clip ;
- des `Clip` référençant obligatoirement un `Score` par `scoreId` : modifier ce contenu modifie tous les clips correspondants ;
- la duplication de clips par référence ou avec création de scores indépendants ;
- un tempo unique appartenant au projet ;
- des scores sans instrument propre, dont les notes sont jouées avec l’instrument de la piste de chaque clip ;
- l'absence de chevauchement entre notes de même hauteur dans un score, avec résolution explicite `SLICE` ou `MERGE` ;
- des chronologies locales de métrique et d’harmonie pour chaque score ;
- la lecture simultanée de tous les clips dont les intervalles globaux se chevauchent, y compris sur une même piste ;
- l'édition du contenu d’un score dans un piano roll ;
- la lecture du projet depuis sa tête globale ou depuis un tick global explicite ;
- la lecture isolée du score édité depuis sa tête locale ou depuis un tick local explicite ;
- la préécoute soutenue d’une hauteur avec l’instrument de la piste d’écoute du piano roll ;
- la préécoute brève et simultanée des hauteurs uniques d’une sélection de notes ;
- un catalogue d'instruments échantillonnés intégrés et non éditables, rendus par `smplr` ;
- l’annulation et le rétablissement des éditions validées ;
- l’enregistrement et la réouverture d’un fichier de projet versionné.

Il ne comprend pas :

- une chronologie de tempo : une seule valeur s'applique au projet entier ;
- les contrôles globaux d'audibilité par instrument ;
- les automations et événements de contrôle ;
- la création, l'import ou l'édition d'instruments par l'utilisateur ;
- un état audio transitoire sauvegardé dans le projet ;
- les suggestions automatiques de remplacement harmonique ;
- le redimensionnement automatique lors d’un changement de métrique.

Les `Note` sont le seul contenu sonore des scores. La métrique et l’harmonie sont des données structurelles locales, et non des automations.

---

## Domaine

Le domaine représente les intentions musicales indépendamment de React, Zustand, du stockage et du moteur Web Audio.

### Modèle de composition

Le `Project` possède des `Track` ordonnées, des `Score` constituant les contenus musicaux éditables et des `Clip` placés dans la composition. Chaque `Clip` référence exactement un score par `scoreId: ScoreId` et une piste par `trackId: TrackId`. Les trois collections appartiennent directement au projet ; les clips ne sont pas dupliqués dans les pistes.

Un score existe indépendamment de son utilisation dans la composition : il peut être créé et édité avant tout placement, puis référencé par zéro, un ou plusieurs clips. Le partage est uniquement une conséquence de l’égalité des `scoreId`. Aucun indicateur de partage, propriétaire privilégié, type de source hybride ou contenu musical embarqué dans un clip n’est nécessaire. Deux scores de contenu identique mais d’identités différentes restent indépendants.

```mermaid
flowchart TD
    Project --> Tempo
    Project --> Score["Score · contenu musical indépendant"]
    Project --> OccA["Clip · placement A"]
    Project --> OccB["Clip · placement B"]
    OccA --> Score
    OccB --> Score
    Project --> Track["Track · piste instrumentale"]
    OccA --> Track
    OccB --> Track
    Track --> Instrument
    Score --> Content["Notes · métrique · Harmony"]
```

La coordonnée horizontale d’un clip est son instant de départ global. Sa position verticale est dérivée du rang de sa piste dans `Project.tracks`. La durée du bloc est dérivée de la durée locale du score référencé et du `repeatCount` du clip.

Les pistes possèdent une identité stable, un nom et un instrument. Leur ordre est persistant ; les réordonner ne change ni les références des clips ni leur son. Déplacer un clip vers une autre piste change son instrument effectif si les deux pistes utilisent des instruments différents. Deux clips peuvent se chevaucher sur une même piste comme sur des pistes différentes : ils sont lus simultanément, sans priorité ni contrainte de collision entre leurs notes. Chaque clip garde son contexte audio indépendant ; une piste n’introduit pas de bus de mixage partagé.

### Vue des concepts

| Concept | Nature | Rôle principal |
| --- | --- | --- |
| `Project` | Entity et racine d'agrégat | Posséder le document musical, le tempo, les pistes ordonnées, les scores et les clips qui les référencent |
| `Track` | Entity interne | Porter l’identité, le nom et l’instrument d’une piste |
| `Score` | Entity interne | Porter un contenu musical indépendant, éditable et partageable, sans instrument propre |
| `Clip` | Entity interne | Référencer un score et une piste, porter son début global et ses répétitions |
| `Note` | Entity interne | Représenter une note locale ; son instrument est résolu depuis la piste lors de la lecture |
| `Instrument` | Entity de référence | Décrire publiquement un instrument intégré |
| `Tempo` | Value Object | Définir la vitesse unique du projet |
| `MeterChange` | Entity interne | Placer une métrique sur la chronologie locale d’un score |
| `HarmonyChange` | Entity interne | Placer un accord ou une gamme sur la chronologie harmonique locale d’un score |

### Result et validation du domaine

Le domaine représente toute validation attendue par un `Result<T, E>` explicite :

```ts
type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

type ValidationError<
  Code extends string,
  Details extends object
> = {
  kind: "VALIDATION_ERROR";
  code: Code;
  details: Details;
};
```

Les branches sont discriminées par `ok`. Des helpers purs `ok(value)` et `err(error)` peuvent construire les deux variantes ; `map` et `flatMap` peuvent les composer sans extraire prématurément une valeur.

Chaque module déclare près de ses invariants ses propres codes et détails, puis compose les unions nécessaires. Il n'existe pas d'erreur générique contenant seulement un texte. Exemples :

```ts
type TempoValidationError = ValidationError<
  "TEMPO_OUT_OF_RANGE" | "TEMPO_PRECISION_EXCEEDED",
  { received: number; min: number; max: number; decimals: number }
>;

type NoteOverlap = {
  manipulatedNoteId: NoteId;
  overlappingNoteIds: readonly NoteId[];
};

type NoteOverlapError = ValidationError<
  "NOTE_OVERLAP",
  { scoreId: ScoreId; overlaps: readonly NoteOverlap[] }
>;
```

Les `code` sont stables et indépendants de la langue. `details` contient uniquement les données structurées nécessaires pour comprendre et traiter l'échec ; le domaine ne produit aucun message destiné à l'utilisateur.

Les constructeurs capables de créer un état invalide restent privés. Les factories de Value Objects et d'entités, la reconstitution d'un agrégat et les opérations qui peuvent violer un invariant retournent un `Result` :

```ts
Tempo.create(bpm: number): Result<Tempo, TempoValidationError>;
Score.create(input: CreateScoreInput): Result<Score, ScoreValidationError>;
Clip.create(input: CreateClipInput): Result<Clip, ClipValidationError>;
Project.create(input: CreateProjectInput): Result<Project, ProjectValidationError>;
moveClips(project: Project, command: MoveClipsCommand): Result<Project, ProjectEditError>;
editNote(score: Score, command: EditNoteCommand): Result<Score, ScoreValidationError>;
```

Une branche `ok: false` ne modifie jamais l'objet d'origine et ne publie aucun état partiel. Dans le premier périmètre, une opération retourne la première erreur selon un ordre de validation déterministe ; l'accumulation de plusieurs erreurs pourra être ajoutée sans changer la forme de `Result`.

Les violations prévisibles d'une règle métier ne lèvent pas d'exception. Les exceptions restent réservées aux défauts de programmation et aux défaillances techniques inattendues ; elles ne sont pas converties artificiellement en `ValidationError`.

### Bornes numériques et valeurs élémentaires

Le premier périmètre fixe les limites suivantes :

```ts
const MAX_TRACK_COUNT = 128;
const MAX_TICK = 2_147_483_647;
const MAX_REPEAT_COUNT = 65_535;
```

| Valeur | Domaine valide |
| --- | --- |
| Nombre de pistes | entier de `0` à `MAX_TRACK_COUNT = 128` inclus |
| `Pitch.midiNumber` | entier de `0` à `127` inclus |
| `Velocity` | entier de `1` à `127` inclus |
| `Meter.beatsPerMeasure` | entier de `1` à `32` inclus |
| `Meter.beatUnit` | `1`, `2`, `4`, `8`, `16`, `32` ou `64` |
| `RootNote.letter` | `A`, `B`, `C`, `D`, `E`, `F` ou `G` |
| `RootNote.accidental` | `FLAT`, `NATURAL` ou `SHARP` |
| `Tick` | entier de `0` à `MAX_TICK` inclus |
| `Duration.ticks` | entier de `1` à `MAX_TICK` inclus |
| `repeatCount` | entier de `1` à `MAX_REPEAT_COUNT` inclus |

`Velocity = 0` n’est pas une vélocité persistante valide : Pianola représente le relâchement par une commande `NOTE_OFF` explicite. Les doubles altérations ne font pas partie du catalogue initial ; leur ajout futur étendra `Accidental` sans modifier la représentation de `RootNote`.

Toute factory valide également les résultats composés, pas seulement leurs opérandes. Les calculs temporels suivants doivent notamment rester inférieurs ou égaux à `MAX_TICK` :

```text
timeRange.start + timeRange.duration
clip.start + score.duration * clip.repeatCount
ticksPerMeasure = beatsPerMeasure * (3840 / beatUnit)
```

Une valeur reçue hors de ces bornes produit une `ValidationError` typée. Aucun arrondi, clamp ou débordement silencieux n’est effectué par le domaine. Les opérations de présentation peuvent proposer une valeur corrigée, mais doivent la soumettre explicitement comme une nouvelle intention.

### Project

`Project` représente le document musical complet ouvert dans l'application.

Attributs possibles :

- `id` ;
- `name` ;
- `tempo` ;
- `tracks` : collection ordonnée de pistes ;
- `scores` ;
- `clips` ;
- `createdAt` ;
- `updatedAt`.

Responsabilités et invariants :

- servir de racine de sauvegarde ;
- posséder directement les pistes, tous les scores et tous les clips ;
- posséder exactement un `Tempo` ;
- garantir l’unicité des `TrackId`, des `ScoreId` et des `ClipId` dans leurs collections respectives ;
- limiter la collection à `MAX_TRACK_COUNT = 128` pistes ;
- garantir que chaque `Clip.scoreId` référence un score existant et chaque `trackId` une piste existante ;
- permettre la création, le renommage, le réordonnancement et le changement d’instrument des pistes ;
- interdire la suppression d’une piste encore référencée par un clip ;
- permettre la création et l'édition des scores ainsi que l’ajout, le déplacement, la duplication et la suppression des clips qui les référencent ;
- interdire la suppression d’un score encore référencé par un clip ;
- accepter un projet sans piste ni clip, des pistes vides et des scores sans clip.

Supprimer le dernier clip d’un score ne supprime jamais implicitement sa source. Le score reste disponible pour être replacé et ne disparaît que par une commande manuelle de suppression, valide uniquement lorsqu’aucun clip ne le référence.

Une création de clip avec un nouveau contenu insère le score et le clip dans une même transaction. Le projet ne publie jamais un clip sans score ni une référence en attente de résolution. Les erreurs de référence appartiennent à `ProjectValidationError` et sont propagées par `ProjectEditError` :

| Code | Détails | Condition |
| --- | --- | --- |
| `SCORE_NOT_FOUND` | `scoreId`, et `clipId` si la référence vient d’un clip | Identité de score absente de `Project.scores` |
| `SCORE_IN_USE` | `scoreId`, `clipIds` | Suppression d’un score encore référencé dans le résultat de la commande |
| `DUPLICATE_SCORE_ID` | `scoreId` | Plusieurs scores portent la même identité dans la collection |

Un `scoreId` absent, nul ou mal formé relève de `ClipValidationError` (`INVALID_SCORE_ID`) avant la résolution de référence. Les mêmes validations s’appliquent à une création interactive, une duplication et une reconstitution depuis un fichier. Une commande collective peut retirer les clips puis leur score explicitement ; seul un résultat complet valide est publié.

Le tempo ne possède ni position, ni changement programmé. Il s'applique uniformément à toute la timeline et convertit les ticks globaux ou locaux en secondes.

La durée structurelle du projet est dérivée de la fin globale la plus tardive parmi ses clips. Elle vaut zéro lorsque le projet ne contient aucun clip.

### Track

`Track` représente une piste instrumentale appartenant au projet.

Attributs :

- `id: TrackId` ;
- `name` ;
- `instrumentId: InstrumentId`.

Une piste référence exactement un instrument. Son rang est donné uniquement par l’ordre de `Project.tracks` : aucun indice de placement ni champ d’ordre redondant n’est sauvegardé dans la piste ou dans les clips. Plusieurs pistes peuvent porter le même nom ou utiliser le même instrument ; leur identité les distingue.

La piste détermine l’instrument de tous les clips qui la référencent, sans posséder leur contenu musical. Un même score peut ainsi être joué au piano sur une piste et au vibraphone sur une autre. Modifier l’instrument d’une piste ne modifie aucun score et n’affecte pas les clips placés ailleurs.

Une piste vide est valide et peut servir de piste d’écoute au piano roll. La suppression d’une piste non vide retourne `TRACK_IN_USE`, sans suppression en cascade ni déplacement implicite. Une commande collective peut déplacer ou supprimer explicitement les clips qui la référencent puis supprimer la piste, sous réserve que le résultat complet reste valide.

La création au-delà de la limite retourne `TRACK_LIMIT_EXCEEDED` ; une référence absente retourne `TRACK_NOT_FOUND`. Ces erreurs structurées appartiennent aux validations du projet. Le réordonnancement doit conserver exactement les identités existantes, sans doublon ni omission. Les contrôles de volume, mute, solo, effets et routage ne font pas partie de ce premier périmètre.

### Score

`Score` représente un contenu musical indépendant, éditable dans le piano roll et partageable par plusieurs clips. Il décrit les notes et leur organisation locale sans imposer de placement, de répétition ou d’instrument de lecture.

Attributs possibles :

- `id: ScoreId` ;
- `name` ;
- `duration` ;
- `notes` ;
- `meterChanges` ;
- `harmonyChanges`.

Responsabilités et invariants :

- définir sa durée canonique locale en ticks ;
- contenir des notes positionnées relativement à son début local, sans association instrumentale persistante ;
- empêcher le chevauchement temporel de deux notes de même hauteur ;
- contenir et ordonner ses deux chronologies locales : métrique et harmonie ;
- fournir la métrique et l’harmonie actives à une position locale ;
- posséder un `HarmonyChange` initial obligatoire au tick `0`, dont la valeur peut être un accord ou une gamme ; la création utilise `SCALE · C CHROMATIC` par défaut ;
- garantir la cohérence locale de ses notes et changements.

Un `Score` ne possède ni début global, ni piste, ni instrument, ni nombre de répétitions. Modifier son contenu ou sa durée modifie la source commune observée et jouée par tous les clips qui le référencent.

Ses identités locales (`NoteId`, `MeterChangeId`, `HarmonyChangeId`) sont uniques dans leurs collections respectives. Une référence de contenu se résout toujours dans un `ScoreId` explicite. Une duplication indépendante crée un nouveau `ScoreId` et de nouvelles identités pour toutes ses notes et tous ses changements, y compris les changements initiaux ; elle conserve leurs valeurs musicales et leur ordre. Elle ne conserve aucun lien de synchronisation avec l’original. Les Value Objects immuables peuvent être réutilisés en mémoire sans partager l’identité des entités.

### Clip

`Clip` représente l’application concrète d’un `Score` dans la composition. Il prend la forme d’un bloc persistant dans la grille globale.

Attributs possibles :

- `id` ;
- `scoreId: ScoreId` ;
- `start` ;
- `trackId` ;
- `repeatCount`.

`start` est un `Tick` interprété depuis le début du projet. `trackId` est un `TrackId` stable référençant une piste existante. Un clip référence exactement un `Score` existant et ne duplique jamais son contenu local. Son instrument est obtenu en recherchant dans `Project.tracks` la piste dont l’identité vaut `trackId`, puis en lisant son `instrumentId`.

`repeatCount` vaut `1` par défaut. Il accepte un entier de `1` à `MAX_REPEAT_COUNT = 65_535` et indique le nombre total de lectures contiguës du score référencé. Chaque répétition recommence au tick local `0`.

La fin globale structurelle est calculée ainsi :

```text
clipEnd = clip.start
              + score.duration * clip.repeatCount
```

Déplacer un clip modifie son `start` ou son `trackId`. Le déplacement horizontal change son instant de lecture ; le déplacement vers une autre piste change son instrument effectif si les deux pistes utilisent des instruments différents. Le score source et les autres clips restent inchangés. Le redimensionner depuis la grille globale ne modifie jamais la durée du score partagé : l’opération ajoute ou retire uniquement des répétitions complètes en modifiant son `repeatCount`.

Le bord droit conserve `start` et détermine le nouveau nombre de répétitions depuis sa position quantifiée :

```text
repeatCount = max(
  1,
  round((quantizedRightEdge - clip.start) / score.duration)
)
```

Le bord gauche conserve l’ancienne fin globale et modifie atomiquement `start` et `repeatCount` :

```text
previousEnd = clip.start + score.duration * clip.repeatCount
repeatCount = max(
  1,
  round((previousEnd - quantizedLeftEdge) / score.duration)
)
start = previousEnd - score.duration * repeatCount
```

Dans les deux cas, le bord effectivement retenu s’aimante à une frontière de répétition complète. Chaque répétition recommence au tick local `0` ; aucune durée partielle ni aucun décalage de phase propre au clip n’est introduit. L’opération reste soumise aux bornes globales ; un chevauchement avec un autre clip, quelle que soit sa piste, est valide.

Dupliquer un bloc par référence crée un nouveau `ClipId` qui conserve le même `scoreId` et, par défaut, le même `trackId`. C’est le comportement de duplication par défaut. Une duplication indépendante crée également une copie du score avec un nouveau `ScoreId`, puis fait référencer cette copie par le nouveau clip. Le clip d’origine conserve son score. Dans les deux cas, le placement de la copie est explicite et validé ; aucun déplacement des autres clips n’est implicite.

Rendre un clip existant indépendant applique la même copie de contenu, puis remplace uniquement son `scoreId`, en conservant son `ClipId`, son `trackId`, son `start` et son `repeatCount`. La création du score et le changement de référence sont atomiques. Le score d’origine reste disponible même s’il perd son dernier clip. Cette opération ne crée aucun nouveau type de clip : chaque bloc référence toujours exactement un score.

Les intervalles des clips sont semi-ouverts. Leur recouvrement avec une durée strictement positive exprime une lecture simultanée, y compris sur une même piste ; des bornes contiguës ne constituent pas un recouvrement. La piste détermine l’instrument, sans modifier le calcul des intervalles temporels.

### Partage et duplication

| Intention | Résultat persistant | Effet des éditions ultérieures |
| --- | --- | --- |
| Créer un score sans clip | Un nouveau `Score` dans `Project.scores` | Contenu éditable avant tout placement |
| Placer un score existant | Un nouveau `Clip` avec son `scoreId` | Tous les clips de ce score reflètent les modifications |
| Dupliquer un clip par référence | Un nouveau `ClipId`, même `scoreId` | Contenu commun |
| Dupliquer un clip indépendamment | Un nouveau `ClipId` et un nouveau score | Original et copie évoluent séparément |
| Rendre un clip existant indépendant | Même `ClipId`, nouveau score référencé | Seul ce clip utilise la copie à sa création |
| Dupliquer un score seul | Un nouveau score, aucun clip ajouté | La copie peut être éditée puis placée séparément |

Pour une sélection de clips, la duplication par référence conserve les relations existantes. La duplication indépendante crée un score distinct pour chaque clip copié, même si les originaux partageaient un score. Pour réutiliser ensuite une même copie dans plusieurs clips, il suffit de placer ce nouveau score ou de dupliquer l’un de ses clips par référence. Aucun score n’est fusionné automatiquement sur la base de son contenu ou de son nom.

### Note

`Note` représente une note placée dans un score. Elle ne porte aucun instrument : celui-ci est résolu depuis la piste du clip ou la piste d’écoute du piano roll.

Attributs possibles :

- `id` ;
- `pitch` ;
- `range` ;
- `velocity`.

Une note possède une identité afin de conserver sa continuité lorsqu'elle est déplacée, redimensionnée, transposée ou modifiée. Une copie ou une duplication reçoit une nouvelle identité.

Invariants :

- la position locale de début est positive ou nulle ;
- la durée est strictement positive ;
- la note se termine au plus tard à la fin locale du score ;
- `Pitch.midiNumber` reste entre `0` et `127`, et `Velocity` entre `1` et `127` ;
- deux notes de même `Pitch` ne se chevauchent jamais avec une durée strictement positive dans un même score ;
- son rôle dans l’accord ou la gamme active est dérivé et n’est pas sauvegardé ;
- une note extérieure à l’accord ou à la gamme active reste valide.

Les intervalles sont semi-ouverts : deux notes de même hauteur peuvent être contiguës lorsque la fin de l'une est égale au début de l'autre. Des notes de hauteurs différentes peuvent se chevaucher ; cet invariant préserve donc la polyphonie et les accords.

Une création, un déplacement, un redimensionnement ou une transposition qui produirait un chevauchement de même hauteur n'est jamais appliqué implicitement. Le domaine signale la collision à l'application, qui doit obtenir de la présentation un mode de résolution `SLICE` ou `MERGE` avant de soumettre une nouvelle tentative explicite.

Modifier le tempo du projet ou la métrique locale ne déplace pas la note : sa position et sa durée restent exprimées dans les ticks canoniques du score.

Lorsqu’une note traverse un `HarmonyChange`, son `TimeRange` est analysé par portions délimitées par ces changements ainsi que par le début et la fin de la note. Chaque portion produit son rôle dans l’accord ou la gamme active. Cette segmentation est une vue dérivée : elle ne découpe ni ne modifie la `Note` persistante.

Les événements instantanés `NoteOn` et `NoteOff` ne sont pas des objets persistants du domaine. Ils sont produits par le service de lecture.

`Velocity` ne possède actuellement aucun usage indépendant de `Note`, mais sa valeur et ses invariants sont isolés dans `domain/models/composition/Velocity.ts` afin que `Note.ts` reste centré sur l'entité.

### Résolution des chevauchements de notes

```ts
type NoteOverlapResolution = "SLICE" | "MERGE";
```

L'invariant d'absence de chevauchement appartient au modèle `Score`. Sa détection et sa résolution sont cependant des algorithmes purs isolés dans `domain/operations/composition/NoteOverlap.ts`. Les transformations de score les utilisent pour valider la collection complète sans alourdir `Score.ts`. Le cas d’usage obtient le choix utilisateur auprès de la présentation et transmet ce mode au domaine.

Une commande collective fournit ses `manipulatedNoteIds` dans un ordre stable. Cet ordre définit la priorité de résolution sans introduire de `primaryNoteId` supplémentaire.

`SLICE` donne priorité à la première note manipulée, puis à chacune des suivantes dans l’ordre de la commande. Chaque note conserve son identité, son intervalle et sa vélocité tant qu’elle n’est pas découpée par une note manipulée plus prioritaire. Les notes non manipulées sont moins prioritaires que toutes les notes manipulées. Toute note moins prioritaire de même hauteur est remplacée par la différence entre son intervalle et celui de la note prioritaire :

- une partie entièrement couverte est supprimée ;
- un chevauchement sur un bord raccourcit la note moins prioritaire ;
- une note moins prioritaire qui contient entièrement la note prioritaire est scindée en deux fragments ; le fragment gauche conserve son `NoteId` et sa vélocité, tandis que le fragment droit reçoit un nouveau `NoteId` avec la même vélocité.

`MERGE` calcule séparément l'union de chaque groupe transitif de notes de même hauteur en collision. La note résultante conserve le `NoteId`, le `Pitch` et la `Velocity` de la première note manipulée du groupe selon l’ordre de la commande ; les autres notes du groupe sont supprimées. Deux notes seulement contiguës ne sont ni en collision ni fusionnées automatiquement.

Une détection collective retourne une seule `NoteOverlapError` dont `details.scoreId` désigne le score concerné et `details.overlaps` contient toutes ses collisions, dans l’ordre stable des notes manipulées. Chaque entrée associe une note manipulée à tous ses `overlappingNoteIds`, qu’ils désignent des notes manipulées ou non manipulées. Une commande touchant plusieurs scores les valide dans un ordre déterministe et retourne la première erreur ; aucune collision n’est recherchée entre des notes de scores différents ou entre leurs clips.

Après résolution, le `Score` valide de nouveau l'ensemble de ses notes. Il retourne `ok(score)` lorsque le résultat satisfait tous les invariants, ou une erreur typée sans modifier le score d'origine. `SLICE` comme `MERGE` forme une seule transformation atomique sur l’ensemble de la commande.

### Instrument

`Instrument` est la représentation publique, stable et minimale d'un instrument intégré.

Attributs possibles :

- `id` ;
- `name`.

`InstrumentId` est un type stable et opaque déclaré avec `Instrument` dans `domain/models/instrument/Instrument.ts`.

Une `Track` sauvegarde uniquement cet identifiant, et non une référence directe vers l’objet `Instrument`. Plusieurs pistes peuvent référencer le même instrument. Les notes d’un clip utilisent l’instrument de sa piste ; déplacer le clip vers une autre piste peut donc modifier cette association sonore sans changer son score.

`Instrument` appartient à un modèle de référence distinct de l'agrégat `Project`. Il ne contient ni configuration d'échantillons, ni état de voix, ni objet du moteur audio.

Les instruments sont définis avant la compilation et ne sont pas éditables par l'utilisateur. À l’ouverture d’un fichier, un `InstrumentId` absent du catalogue produit une erreur applicative explicite ; aucun instrument de remplacement n’est choisi implicitement.

### Temps musical

Le temps musical canonique utilise des ticks entiers avec une résolution fixe de 960 ticks par noire.

Les battements, mesures et secondes sont des représentations dérivées. La grille visible peut quantifier un geste, mais ne réduit pas les positions valides du domaine à ses subdivisions affichées.

| Value Object | Représentation | Règles principales |
| --- | --- | --- |
| `Tick` | entier | De `0` à `MAX_TICK` inclus ; son contexte d'emploi détermine s'il est global ou local |
| `Duration` | `ticks` | De `1` à `MAX_TICK` inclus |
| `TimeRange` | `start`, `duration` | Intervalle local dont `start` est un `Tick`, utilisé notamment par une note |
| `Tempo` | `bpm` | Une décimale, de `20.0` à `999.9` BPM inclus |
| `Meter` | `beatsPerMeasure`, `beatUnit` | Numérateur de `1` à `32` ; dénominateur parmi `1, 2, 4, 8, 16, 32, 64` |

Avec une résolution de 960 ticks par noire :

```text
ticksPerMeasure = beatsPerMeasure * (4 / beatUnit) * 960
durationSeconds = (durationTicks / 960) * (60 / project.tempo.bpm)
```

Pour la répétition d'indice `i`, commençant à zéro, la position globale d'une note est :

```text
noteGlobalStart = clip.start
                + i * score.duration
                + note.range.start
```

La métrique du score n'intervient pas dans cette conversion. Elle structure les mesures et les repères locaux sans créer une horloge indépendante.

`TimeRange` facilite notamment la détection des chevauchements ainsi que les opérations de déplacement et de redimensionnement.

### Hauteur et harmonie

Ces objets décrivent la hauteur des notes et leur contexte harmonique local. Ils n'appartiennent pas au calcul du temps musical.

| Value Object | Représentation | Règles principales |
| --- | --- | --- |
| `Pitch` | `midiNumber` | Le nom et l'octave peuvent être dérivés |
| `RootNote` | `letter`, `accidental` | Lettre de `A` à `G` ; altération `FLAT`, `NATURAL` ou `SHARP` ; classe chromatique dérivée |
| `Chord` | `root`, `typeId` | Accord possédant obligatoirement une fondamentale explicite |
| `Scale` | `root`, `typeId` | Gamme possédant obligatoirement une tonique explicite |
| `Harmony` | `kind`, `chord` ou `scale` | Contexte exclusif contenant un accord ou une gamme |

`RootNote` conserve l'orthographe sous la forme d'une lettre et d'une altération, sans octave. Sa classe de hauteur chromatique est dérivée et n'est pas sauvegardée séparément : `C_SHARP` et `D_FLAT` sont enharmoniquement équivalents, mais restent deux valeurs distinctes.

```ts
interface Chord {
  root: RootNote;
  typeId: ChordTypeId;
}

interface Scale {
  root: RootNote;
  typeId: ScaleTypeId;
}

type Harmony =
  | { kind: "CHORD"; chord: Chord }
  | { kind: "SCALE"; scale: Scale };
```

`Chord` et `Scale` possèdent toujours une `RootNote` explicite. Il n'existe ni référence par degré, ni contexte supérieur implicite permettant de la déduire. Les deux variantes de `Harmony` sont exclusives : une seule est active à un tick donné.

Le catalogue initial comprend :

```ts
type ChordTypeId =
  | "MAJOR"
  | "MINOR"
  | "DIMINISHED"
  | "AUGMENTED"
  | "DOMINANT_SEVENTH"
  | "MAJOR_SEVENTH"
  | "MINOR_SEVENTH";

type ScaleTypeId =
  | "CHROMATIC"
  | "IONIAN"
  | "DORIAN"
  | "PHRYGIAN"
  | "LYDIAN"
  | "MIXOLYDIAN"
  | "AEOLIAN"
  | "LOCRIAN"
  | "MAJOR_PENTATONIC"
  | "MINOR_PENTATONIC";
```

Les types d’accords et de gammes définissent leurs classes de hauteur à partir de la fondamentale ou de la tonique. Le choix d’une nouvelle valeur est explicite dans l’éditeur et n’est pas limité par l’harmonie précédente. Les suggestions de remplacements compatibles sont hors du premier périmètre.

### Chronologies locales du score

Un score possède deux collections ordonnées de changements :

| Changement | Valeur | Changement initial au tick `0` | Positions suivantes |
| --- | --- | --- | --- |
| `MeterChange` | `Meter` | Obligatoire | Début d’une nouvelle mesure locale ; peut tronquer la précédente |
| `HarmonyChange` | `Harmony` | Obligatoire, `SCALE · C CHROMATIC` par défaut | N'importe quel tick local du score |

Chaque changement possède une identité, une position locale et sa nouvelle valeur. Les règles communes sont :

- un seul changement d'un même type peut exister à une position ;
- la nouvelle valeur s'applique à partir de la position du changement, incluse ;
- un changement peut être déplacé ou modifié sans perdre son identité ;
- un changement ferme la section précédente du même type et commence la suivante ;
- un changement peut être placé de `0` à `score.duration` inclus ;
- aucun changement n'accepte `null` ni une variante `CLEAR`.

Le `HarmonyChange` initial au tick `0` ne peut être ni supprimé ni déplacé, mais sa valeur peut être remplacée par un accord ou une autre gamme. `SCALE · C CHROMATIC` est la valeur créée par défaut ; sa `RootNote` est conservée par cohérence de modèle même si elle ne modifie pas les douze classes de hauteur de la gamme chromatique.

Chaque nouveau `HarmonyChange` remplace indifféremment l’accord ou la gamme précédente. Il est donc impossible qu’un `Chord` et une `Scale` soient actifs simultanément ou que deux marqueurs harmoniques occupent le même tick.

Un changement placé exactement à `score.duration` est valide et persistant. Il n’affecte aucune note et ne produit aucun événement audio tant que la durée ne change pas. Si le score est allongé, il devient automatiquement le début de la nouvelle section terminale. Si un raccourcissement placerait un changement au-delà de la nouvelle durée, l’opération doit également déplacer ou supprimer ce changement, faute de quoi la validation échoue.

Les marqueurs visibles dans l'éditeur sont la représentation des changements existants. Ils ne forment pas un type métier générique supplémentaire.

### Sections dérivées

`MeterSection` et `HarmonySection` sont des vues locales dérivées. Chacune couvre l'intervalle entre un changement et le changement suivant du même type, ou entre ce changement et la fin du score.

Elles ne sont pas sauvegardées comme des objets autonomes.

| Section | Valeurs dérivées | Usage principal |
| --- | --- | --- |
| `MeterSection` | `start`, `end`, `meter`, découpage en mesures | Repères métriques et frontières de mesure |
| `HarmonySection` | `start`, `end`, `harmony` | Recherche de l’accord ou de la gamme active et analyse des notes |

Pour une `MeterSection` :

```text
fullMeasureCount = floor(sectionDuration / ticksPerMeasure)
trailingMeasureDuration = sectionDuration % ticksPerMeasure
```

Une valeur non nulle de `trailingMeasureDuration` représente une dernière mesure incomplète. Cette mesure tronquée est valide aussi bien à la fin du score qu’avant un `MeterChange`. Chaque `MeterChange` termine immédiatement la section précédente, même au milieu de sa mesure théorique, puis commence une nouvelle mesure complète dans la nouvelle métrique.

Une section dérivée peut être vide : un changement placé à `score.duration` produit l’intervalle semi-ouvert `[score.duration, score.duration)`. Les calculs d’intersection et la planification l’ignorent naturellement ; aucune valeur n’est active au tick final, situé hors de l’intervalle sonore du score.

Grâce au changement initial obligatoire, une `HarmonySection` couvre toujours chaque tick de `[0, score.duration)`. Pour analyser une note, les intervalles pertinents sont dérivés de l'union des frontières de `HarmonySection` et du `TimeRange` de la note. Chaque portion expose un rôle exclusif :

```ts
type NoteHarmonyRole =
  | "CHORD_TONE"
  | "SCALE_TONE"
  | "OUTSIDE_TONE";
```

### Frontière de l'agrégat

`Project` est la racine de l'unique agrégat constituant le document de composition sauvegardé.

Les entités internes conservent des identifiants stables afin d'être ciblées par l'éditeur et les cas d'usage. Elles ne possèdent cependant ni repository ni cycle de persistance autonomes.

L’indépendance du `Score` concerne son contenu et son utilisation : elle n’en fait pas une seconde racine de sauvegarde. Dans ce périmètre, le partage relie des clips du même projet à une entrée de `Project.scores`. Les scores sans clip sont conservés dans ce document au même titre que les scores utilisés.

Une opération peut être déléguée à un `Score` ou une `Track` pour préserver ses invariants locaux. Le projet garantit les références de piste et de score ainsi que l’ordre des pistes. Le `Project` valide ensuite le résultat complet avant publication : modifier la durée d’un score doit notamment préserver les bornes des fins globales calculées de tous ses clips. Les superpositions entre clips sont valides et ne demandent aucune résolution. Une validation locale réussie ne suffit donc pas à accepter l’édition de l’agrégat.

Les `Instrument` sont extérieurs à cet agrégat et sont fournis par un catalogue.

---

## Application

La couche applicative traduit les intentions de l'utilisateur en opérations sur le domaine et orchestre les interactions avec l'extérieur à travers des ports.

Elle possède les états transitoires des trois éditeurs et les états d'orchestration nécessaires à la lecture. Elle ne contient ni configuration `smplr`, ni banque d'échantillons, ni `AudioNode`, ni détail de stockage.

### États des éditeurs

#### Sélections

Deux sélections indépendantes correspondent aux deux éditeurs spécialisés.

```ts
type ScoreContentRef =
  | { kind: "NOTE"; noteId: NoteId }
  | { kind: "METER_CHANGE"; meterChangeId: MeterChangeId }
  | { kind: "HARMONY_CHANGE"; harmonyChangeId: HarmonyChangeId };

interface ScoreContentSelection {
  items: readonly ScoreContentRef[];
}

interface ClipSelection {
  clipIds: readonly ClipId[];
}
```

`ScoreContentSelection` contient les notes et changements du score source actuellement édité. Elle est portée par le même `ScoreEditorState` que l'identité et la tête de lecture locale du score. Elle est vidée lorsque ce score change ou est fermé.

`ClipSelection` contient les clips sélectionnés dans l’éditeur d’arrangement. Elle est portée par `ArrangementEditorState` et sert notamment au déplacement temporel ou vertical des clips, à leur duplication et à leur suppression. Dupliquer cette sélection crée par défaut de nouveaux clips référençant les mêmes scores ; l’intention explicite de duplication indépendante crée les nouveaux scores décrits dans le domaine.

```ts
interface ArrangementEditorState {
  playhead: Tick;
  selection: ClipSelection;
}

interface ScoreEditorState {
  scoreId: ScoreId;
  auditionTrackId?: TrackId;
  playhead: Tick;
  selection: ScoreContentSelection;
}

interface GlobalEditorState {
  // État propre à la vue globale, indépendante du projet ouvert.
}
```

Ces trois états sont indépendants et possèdent chacun leur fichier. `GlobalEditorState` appartient à la vue globale toujours présente, quel que soit le statut de l’application. Il ne contient ni `ArrangementEditorState` ni `ScoreEditorState`. Ses propriétés concrètes seront déclarées lorsque cette vue aura des données applicatives à conserver ; les détails purement visuels restent dans la présentation.

`ArrangementEditorState` existe lorsqu’un projet est ouvert dans la grille. Il possède la tête globale de la composition et la sélection de clips. Son champ `playhead` est global par son propriétaire : aucun préfixe `project` redondant n’est nécessaire.

`ScoreEditorState` existe lorsqu’un score est ouvert dans le piano roll. Regrouper `scoreId`, la tête locale, la piste d’écoute et la sélection empêche qu’un état local subsiste sans score édité. Il peut cibler un score sans clip, mais toujours un score du projet ouvert.

Le score source édité et la sélection de clips expriment des faits différents. Un geste d'interface peut mettre à jour `ScoreEditorState` et `ArrangementEditorState` dans une même transaction, sans imposer de lien implicite entre leurs sélections. Le point d’assemblage applicatif conserve séparément l’unique `GlobalEditorState`, l’éventuel `ArrangementEditorState` et l’éventuel `ScoreEditorState` ; aucun quatrième modèle d’éditeur ne les enveloppe.

| Situation | `GlobalEditorState` | `ArrangementEditorState` | `ScoreEditorState` |
| --- | --- | --- | --- |
| Aucun projet ouvert | Présent | Absent | Absent |
| Projet ouvert | Présent | Présent | Absent |
| Score ouvert dans le piano roll | Présent | Présent | Présent |

Le changement de `scoreId` d’un clip n’affecte pas automatiquement le `ScoreEditorState` existant : celui-ci reste attaché au score ouvert. Ouvrir explicitement la copie remplace l’état local par celui du nouveau score et vide la sélection locale ; aucune référence de note ou de changement de l’original n’est réutilisée dans la copie. `ArrangementEditorState.selection` conserve en revanche l’identité d’un clip rendu indépendant puisqu’il garde son `ClipId`.

La couche applicative choisit explicitement la sélection correspondant à l'action, résout ses références et transmet au domaine les identifiants concernés. Le domaine ne connaît jamais la notion de sélection.

#### Piste d’écoute du piano roll

`ScoreEditorState.auditionTrackId` désigne la piste dont l’instrument sert à `playScore`, `previewPitch` et `previewSelection`. Ce choix applicatif n’est ni une propriété du score, ni un clip supplémentaire ; il n’entre pas dans la sauvegarde ou l’historique du projet.

Ouvrir un clip initialise le contexte depuis son `scoreId` et son `trackId`. Pour un nouvel éditeur, la référence de piste est disponible immédiatement ; sa banque peut encore être en préparation et aucun son ne démarre avant sa disponibilité. Ouvrir directement un score source, sans passer par un clip, laisse la piste d’écoute absente, jusqu’à un choix explicite. Le contenu reste éditable et la tête locale déplaçable sans piste, mais les appels produisant du son retournent `NO_AUDITION_TRACK`. Aucune piste ou instrument de remplacement n’est choisi implicitement.

`PlaybackService.setAuditionTrack(trackId)` valide la piste et prépare son instrument via le même port que les transports. Tant que la préparation dure, le choix précédent reste effectif et le nouveau choix apparaît en attente. En cas d’échec, le choix précédent est conservé. Une fois la demande encore courante prête, le choix devient effectif ; si le score édité est encore joué par le transport `SCORE`, son `trackId` bascule avec le plan accepté à une borne sûre, sans changer la session ni la position locale. Si l’instrument effectif est identique, aucun contexte ni aucune voix n’est recréé. Une session `PROJECT` ou une session `SCORE` attachée à un autre score reste inchangée.

Ouvrir un autre clip du même score conserve ses notes sélectionnées et sa tête, mais applique explicitement ce changement de piste d’écoute. Ouvrir un autre score crée un nouvel état local et laisse le transport existant attaché à son ancien couple score/piste. Déplacer ultérieurement le clip d’origine ne modifie pas automatiquement la piste d’écoute : l’éditeur conserve une référence de piste, pas un lien vivant vers ce bloc.

Chaque préécoute capture `scoreId`, `trackId` et l’instrument résolu. Changer le score édité, fermer le piano roll, changer effectivement sa piste d’écoute ou l’instrument de cette piste arrête les handles concernés, y compris leurs préparations, sans réattaque automatique. Les nouveaux gestes utilisent le nouveau contexte. Une simple réorganisation des pistes n’invalide aucune audition.

#### GridResolution

`GridResolution` représente une précision de quantification utilisée pendant l'édition.

Attribut possible :

- `snapStepTicks`.

`Settings` appartient à l’application et constitue la source de vérité persistante des résolutions :

```ts
interface Settings {
  arrangement: {
    gridResolution: GridResolution;
  };

  scores: ReadonlyMap<
    ScoreId,
    {
      gridResolution: GridResolution;
    }
  >;
}
```

Les deux espaces possèdent des réglages indépendants :

| Espace | État | Valeur initiale |
| --- | --- | ---: |
| Arrangement | `Settings.arrangement.gridResolution` | `960` ticks, soit une noire |
| Piano roll | `Settings.scores.get(scoreId).gridResolution` | `240` ticks, soit une double croche |

La résolution de l’arrangement sert au déplacement et au redimensionnement des clips ainsi qu’au positionnement quantifié de sa tête. Elle travaille dans le référentiel global du projet et ne dépend d’aucune métrique locale. Pour le redimensionnement d’un clip, la position quantifiée du bord est ensuite convertie en un `repeatCount` entier ; le bord effectif s’aligne donc sur la frontière de répétition complète la plus proche.

La résolution locale sert à créer, déplacer et redimensionner les notes, à déplacer les changements de métrique ou d’harmonie et à positionner la tête locale. Elle travaille dans le référentiel du score.

Modifier une résolution ne modifie jamais l’autre. Il n’existe ni lien automatique, ni conversion, ni option de synchronisation entre elles dans le premier périmètre. La création d’un score ajoute son réglage avec `240` ticks ; une duplication indépendante crée également un réglage distinct, initialisé avec la résolution du score source. Supprimer un score retire son réglage lors de la publication de la même édition applicative.

Les deux espaces utilisent le même Value Object et la même unité `Tick`, sans pour autant partager leur valeur. `EditService` résout la grille depuis `ProjectState.settings` selon l’éditeur concerné ; les états d’éditeur ne dupliquent pas cette configuration. Une modification de résolution met à jour `Settings` directement, sans `ProjectEditCommand`, projet transitoire, replanification audio ou entrée dans `ProjectHistory`. Elle marque néanmoins le fichier comme modifié afin d’être sauvegardée.

`Settings` est validé par l’application : la résolution de l’arrangement est obligatoire, chaque `ScoreId` courant possède exactement un réglage et aucun réglage ne cible un score absent. Une création, une duplication ou une suppression de score publie atomiquement le nouveau `Project` et les réglages correspondants. Les réglages restent hors du domaine musical et de ses transformations.

#### Têtes de lecture

Les positions mémorisées `ArrangementEditorState.playhead` et `ScoreEditorState.playhead` sont des ticks applicatifs, initialisés à `0` et non sauvegardés dans le projet. Elles déterminent le départ d’une portée inactive.

Pendant la lecture, `PlaybackService` est la seule autorité sur la position de la portée active. Il la dérive de l’horloge de session et d’un ancrage temps/tick ; il ne conserve pas un second compteur `playhead` avançant indépendamment. La tête affichée du projet ou du même score est une projection de cette position. À l’arrêt, au remplacement ou à la fin naturelle, le dernier tick atteint est mémorisé dans l’état de l’éditeur correspondant, si cet espace existe encore.

| Situation | Source de la tête affichée |
| --- | --- |
| Transport `PROJECT` actif | Position globale dérivée du transport |
| Transport `SCORE` actif sur le score ouvert | Position locale dérivée du transport |
| Portée inactive ou autre score ouvert | Position mémorisée dans `ArrangementEditorState` ou `ScoreEditorState` |

L’ancrage interne conserve la précision temporelle nécessaire, y compris une fraction de tick. La position publique en `Tick` est le tick entier atteint (partie entière, bornée par la portée) ; elle n’est pas aimantée à `GridResolution`. Un changement de tempo prend effet à la borne acceptée de replanification : jusqu’à cette borne, l’ancien ancrage reste utilisé ; à partir d’elle, le nouvel ancrage conserve exactement la continuité de position. Ces données d’exécution ne constituent pas une chronologie persistante de tempo.

Pour une portée de fin `endTick`, une tête accepte `[0, endTick]`, mais la lecture exige un départ dans `[0, endTick)`. Un tick supérieur produit une erreur de validation. `playProject(tick)` et `playScore(tick)` positionnent immédiatement une tête inactive ; si leur portée est déjà active, le tick demandé reste une destination provisoire jusqu’au remplacement réussi, comme pour un seek. Un échec ne fait pas sauter la tête sonore existante.

Les deux espaces restent indépendants. Une position globale ne détermine pas une position locale, car un score peut avoir plusieurs clips et répétitions. Une synchronisation à l’ouverture d’un bloc demande une action explicite de présentation.

Un seek sur la portée active remplace gracieusement la session lorsqu’il est prêt. Un seek sur une portée inactive modifie seulement sa position mémorisée. Aller exactement à la fin termine le transport sans ouvrir de nouvelle session. `stop(GRACEFUL | IMMEDIATE)` conserve la position atteinte ; la fin naturelle mémorise exactement `endTick` et libère le rôle d’`ActiveTransport`, indépendamment des tails.

Une session `SCORE` reste attachée au couple `scoreId` / `trackId` choisi à son ouverture, sauf changement explicite de piste d’écoute pour ce même score. Fermer le piano roll ou ouvrir un autre score ne l’arrête pas : sa position continue d’être dérivée en interne, sans modifier la tête du nouvel éditeur. Rouvrir le même score pendant sa lecture affiche la position du transport ; lorsqu’il est inactif, son nouvel éditeur commence au tick `0`. Supprimer le score lu suit la politique d’arrêt explicite définie plus loin.

#### Intention d’édition et projet transitoire

`EditService` possède le cycle d’édition. La présentation traduit les événements bruts — pointeur, clavier ou commandes d’interface — en une `EditIntent` sémantique. Le service résout ensuite la sélection et la résolution concernées, quantifie l’intention et construit le `ProjectEditCommand` transmis aux opérations du domaine. La présentation ne construit donc directement ni commande métier, ni agrégat, ni projet transitoire.

`EditIntent` est une union applicative fermée décrivant l’action demandée et son espace d’édition, sans pixel ni objet du domaine déjà transformé. Ses variantes peuvent demander l’utilisation de `ScoreContentSelection` ou de `ClipSelection` ; `EditService` capture alors les identifiants concernés au début du geste. Les positions ou deltas qu’elle transporte sont exprimés dans le référentiel sémantique global ou local, puis quantifiés par le service.

```ts
interface EditSession {
  id: EditSessionId;
  baseProject: Project;
  command: ProjectEditCommand;
  commandRevision: number;
  phase: EditSessionPhase;
}

type EditSessionPhase =
  | { status: "EDITING" }
  | {
      status: "AWAITING_DECISION";
      pendingDecision: EditDecisionRequest;
    };

interface PendingEditDecision<
  Kind extends string,
  Details,
  Choice
> {
  id: EditDecisionId;
  kind: Kind;
  details: Details;
  choices: readonly Choice[];
}

type EditDecisionRequest =
  | PendingEditDecision<
      "NOTE_OVERLAP",
      { scoreId: ScoreId; overlaps: readonly NoteOverlap[] },
      NoteOverlapResolution
    >;

interface EditDecision<Kind extends string, Choice> {
  decisionId: EditDecisionId;
  kind: Kind;
  choice: Choice;
}

type SubmittedEditDecision =
  | EditDecision<"NOTE_OVERLAP", NoteOverlapResolution>;

interface PendingEditPreparation {
  id: EditPreparationId;
  editSessionId: EditSessionId;
  commandRevision: number;
  instrumentIds: readonly InstrumentId[];
  status: "PREPARING";
}

interface ProjectState {
  project: Project;
  settings: Settings;
  editSession?: EditSession;
  transientProject?: TransientProject;
  pendingEditPreparation?: PendingEditPreparation;
  history: ProjectHistory;
  effectiveProjectRevision: number;
}

const effectiveProject: Project | TransientProject =
  state.transientProject ?? state.project;
```

`project` est la version musicale courante validée faisant autorité, éventuellement non encore sauvegardée. `settings` contient les configurations persistantes associées à ce fichier. `EditSession.baseProject` référence la version immuable du projet au début du geste. Le mécanisme couvre toutes les modifications musicales du document : contenu local d’un score dans le piano roll, clips dans la grille, pistes instrumentales et propriétés générales du projet. L’éditeur de score ne possède donc ni session ni projet transitoire séparés.

`ProjectEditCommand` est l’union applicative des commandes métier élémentaires que `EditService` sait composer et rejouer comme une seule transaction. Les commandes élémentaires restent déclarées près des opérations du domaine qui les exécutent ; l’union n’appartient pas à `Project`, car son exhaustivité décrit les capacités du cas d’usage d’édition. Ce nom désigne la portée transactionnelle de la commande, pas son origine dans la grille. Les paramètres expriment une transformation cumulée depuis `baseProject`, jamais depuis le brouillon précédent. Les identifiants des créations ordinaires sont alloués une fois et conservés dans la commande pendant le geste. Ceux des fragments de collision sont alloués seulement à la résolution définitive.

Prévisualisation et validation réutilisent les mêmes calculs purs de transformation, déclarés auprès des entités du domaine. Ces calculs peuvent produire des données candidates sans construire un agrégat valide ; la publication d’un `Project` ajoute toujours la validation complète. Le domaine ignore les gestes, les sélections, l’audio et la notion applicative de `TransientProject`.

Les fichiers `*Transformations.ts` sont classés selon le type qu’ils produisent, et non selon l’élément principalement ciblé par la commande. Une fonction qui retourne un `Score` appartient ainsi à `ScoreTransformations.ts`. Toute fonction qui retourne un `Project` appartient à `ProjectTransformations.ts`, y compris lorsqu’elle ajoute, déplace, redimensionne, duplique ou supprime un clip, ou lorsqu’elle transforme une piste. Il n’existe donc pas de `ClipTransformations.ts` tant qu’aucune opération du domaine ne retourne un `Clip` isolé.

Cette convention s’applique aussi aux copies :

```ts
// domain/operations/composition/ScoreTransformations.ts
duplicateScore(
  score: Score,
  command: DuplicateScoreCommand
): Result<Score, ScoreValidationError>;

// domain/operations/composition/ProjectTransformations.ts
addScore(
  project: Project,
  command: AddScoreCommand
): Result<Project, ProjectEditError>;

duplicateClips(
  project: Project,
  command: DuplicateClipsCommand
): Result<Project, ProjectEditError>;

makeClipIndependent(
  project: Project,
  command: MakeClipIndependentCommand
): Result<Project, ProjectEditError>;
```

`DuplicateScoreCommand` fournit la nouvelle identité du score, son nom et la correspondance complète des nouvelles identités de notes et de changements. Ces identités sont allouées une seule fois par l’application et conservées dans la commande ; les fonctions pures ne génèrent aucun identifiant aléatoire. `AddScoreCommand` insère un score valide dans l’agrégat. Dupliquer un score seul compose donc `duplicateScore` et `addScore` dans la même transaction applicative.

`DuplicateClipsCommand` précise les clips sources, les nouveaux `ClipId`, les placements et le choix explicite `REFERENCE` ou `INDEPENDENT`. La branche `INDEPENDENT` fournit un `DuplicateScoreCommand` par clip copié ; la branche `REFERENCE` n’en fournit aucun. `MakeClipIndependentCommand` cible un clip existant et fournit la commande de copie de son score. Les opérations retournant un projet délèguent la copie locale à `ScoreTransformations`, puis valident ensemble les nouveaux scores et les références des clips. Le choix de duplication appartient uniquement à la commande : il ne devient pas une propriété persistante du clip ou du score.

`TransientProject` est la projection applicative complète de ces données candidates. Les chevauchements de notes de même hauteur constituent la seule relaxation d’invariant du premier périmètre. Les bornes numériques, durées positives, références, identités, limites locales des notes et chronologies restent valides. Une intention violant une autre règle ne remplace pas la dernière projection admissible et retourne une erreur. Aucun `Project` invalide n’est construit.

`transientProject` est un cache de projection de la commande, jamais une deuxième intention à modifier indépendamment. `effectiveProject` est dérivé et constitue la source commune du document affiché et du rendu sonore ; ni l’un ni l’autre ne peut être sauvegardé. Un repère de geste en attente de préparation peut être affiché séparément, sans prétendre être le contenu effectif.

Cette projection contient une seule entrée par `ScoreId`, résolue par tous ses clips : une édition locale partagée n’est pas recopiée dans chaque bloc. Une copie indépendante introduit une nouvelle entrée avec ses propres identités, stables pendant toutes les actualisations du geste. Annuler ce geste retire simultanément ses créations et rétablit les références initiales ; aucun clip orphelin ni score créé par un geste annulé ne subsiste.

Une seule édition du document est ouverte à la fois, y compris pendant une décision attendue ou un chargement requis par cette édition. Toute autre édition, annulation d’historique, rétablissement ou ouverture de fichier retourne `EDIT_IN_PROGRESS` ; l’utilisateur termine ou annule d’abord le geste. Les commandes de transport et la sauvegarde du dernier `project` validé restent disponibles. Aucun retour asynchrone ne remplace la base d’un geste en cours.

`EditSession` reste le même objet pendant tout le geste. Son champ `phase` porte l’état courant : `EDITING` ou `AWAITING_DECISION`. `EDITING` est donc bien une valeur d’état et non un type de session. L’union discriminée `EditSessionPhase` garantit qu’une décision n’existe que pendant la phase qui l’attend.

`PendingEditDecision<Kind, Details, Choice>` est une structure générique : elle ne connaît aucun…9438 tokens truncated…project.duration`, l’appel retourne une erreur de validation.

#### Lecture du score édité

`playScore()` exige un `ScoreEditorState` avec une piste d’écoute existante et capture son `scoreId` et son `auditionTrackId` comme `trackId` de session. Il utilise le tick local dérivé si ce même score est en cours de lecture, sinon la position mémorisée `ScoreEditorState.playhead`. Si cette tête se trouve à `score.duration`, l’appel la replace au tick `0` avant de préparer la lecture. Un score possède toujours une durée strictement positive.

`playScore(tick)` valide explicitement le tick dans `[0, score.duration]`. Il positionne une tête inactive ; sur le transport du même score actif, la destination reste provisoire jusqu’au remplacement réussi. Si `tick === score.duration`, aucune session n’est ouverte et un transport du même score actif est terminé à cette destination ; si `tick > score.duration`, l’appel retourne une erreur de validation.

Ce transport :

- lit uniquement le contenu du score édité ;
- ignore les clips, leurs positions et leurs `repeatCount` ;
- utilise l’instrument de sa piste d’écoute et le tempo unique du projet ;
- s'arrête structurellement à `score.duration` ;
- ouvre un seul `PlaybackContext` pour ce score.

`PROJECT` et `SCORE` sont deux portées d'un même transport exclusif. Démarrer l'une remplace gracieusement l'autre sans modifier la tête inactive.

Avant d'ouvrir la session et de faire avancer sa tête, le service résout tous les `InstrumentId` nécessaires à la portée demandée et attend le chargement de leurs échantillons. Pour `PROJECT`, il considère les clips susceptibles d’être lus entre le tick de départ et la fin du projet ; pour `SCORE`, seulement l’instrument de la piste d’écoute capturée. Les pistes sans clip dans la portée ne sont pas préparées pour `PROJECT`. Chaque instrument requis est obtenu depuis le `trackId` du clip, et les identifiants d’instrument sont dédupliqués avant la préparation. La promesse se résout lorsque le transport a effectivement démarré, ou immédiatement lorsqu’aucune session ne doit être ouverte. Aucun transport ne commence avec une banque requise manquante.

Sur une session `SCORE` active du même score, `seekScore` conserve le `trackId` de cette session, même si le score a été rouvert directement sans piste d’écoute dans l’éditeur. Le choix de piste relève de `setAuditionTrack` ou d’un nouveau `playScore`, pas du seek.

`seekProject(tick)` et `seekScore(tick)` acceptent la fin correspondante mais refusent toute valeur supérieure. Si la tête appartient au transport actif — et, pour `SCORE`, au même `scoreId` — un déplacement avant la fin crée une requête asynchrone de même portée, soumise à la même barrière de préparation ; un déplacement exactement à la fin termine naturellement le transport et retourne `ok("NO_CONTENT")`. Lorsque la portée est inactive, le seek déplace seulement la tête et retourne `ok("POSITIONED")`, même si le score édité n’a pas encore de piste d’écoute.

#### Fin de portée après modification

Lorsqu’une édition raccourcit la portée, sa tête est ramenée dans les nouvelles bornes :

```text
newPlayhead = min(currentPlayhead, newEndTick)
```

Si un transport actif reste strictement avant `newEndTick`, il continue avec ses commandes et `ContextCompletion` replanifiées.

Si sa position à la borne acceptée se trouve à la nouvelle fin ou au-delà, le service prépare d’abord la fin du plan ; après acceptation :

- l’état applicatif place immédiatement la tête sur `newEndTick` ;
- le service annule les attaques futures remplaçables ;
- il relâche les voix actives à la première borne `safeAt` disponible ;
- les contextes passent à `DRAINING` s’ils possèdent encore des releases ou tails ;
- l’`ActiveTransport` est supprimé ; la session est fermée lorsque sa fin sonore acceptée est atteinte, puis libérée après drainage.

Si la portée est inactive, seul le clamp de sa tête est nécessaire. L’allongement ultérieur d’une portée ne déplace jamais automatiquement sa tête.

#### Suppression du score attaché à une session

Un score ne peut être supprimé du domaine que s’il n’est référencé par aucun `Clip`. Le cas d’usage valide d’abord la suppression et l’ensemble de la commande, sans publier le projet. Si ce score est actuellement joué par une session `SCORE`, il orchestre ensuite à la publication :

1. l’arrêt `GRACEFUL` de la session et l’annulation de ses attaques futures ;
2. le relâchement de ses voix actives et le drainage éventuel de ses contextes ;
3. la suppression de l’`ActiveTransport` ;
4. l’arrêt de ses `PITCH_PREVIEW` et `SELECTION_PREVIEW` ;
5. l’invalidation des préparations devenues sans objet ; une édition concurrente reste interdite ;
6. la fermeture de `ScoreEditorState` s’il cible encore ce score ;
7. la suppression du score dans le nouveau `project`.

La tête locale appartient au `ScoreEditorState` supprimé : elle n’est ni conservée sans score, ni transférée au prochain score ouvert. Les tails de l’ancienne session peuvent continuer à se drainer après la suppression sans maintenir le score dans l’agrégat.

Supprimer un score non placé pendant un transport `PROJECT` n’a aucun effet sur cette session, puisqu’aucun clip ne peut le rendre audible.

#### Suppression d’une piste utilisée pour l’écoute

Le projet refuse une piste encore référencée par des clips. Une piste vide peut toutefois être utilisée par une session `SCORE` ou des préécoutes. Lorsqu’une édition, une annulation de brouillon ou un undo/redo retire effectivement cette piste, l’application arrête gracieusement la session `SCORE` liée, mémorise la tête locale si ce score est ouvert, termine les handles concernés et invalide les préparations visant la piste. Elle retire `auditionTrackId` des états qui la référencent, tout en conservant le score ouvert, ses notes sélectionnées et sa tête.

Ces effets sont coordonnés avec la publication du nouveau projet et ne se produisent pas si la suppression est refusée. Restaurer ensuite la piste via l’historique ne restaure ni une audition arrêtée ni un choix applicatif effacé. Une suppression collective qui retire aussi des clips suit en plus la réconciliation ordinaire de `PROJECT`.

#### Préécoutes du piano roll

Les deux préécoutes utilisent l’instrument de `ScoreEditorState.auditionTrackId`, ne déplacent aucune tête de lecture et peuvent coexister avec le transport actif. Elles n’exposent aucun identifiant de session ou de contexte audio à la présentation.

##### Préécoute d’une hauteur

`previewPitch(pitch, velocity?)` joue une hauteur explicite, qu’une `Note` correspondante existe ou non dans le score. La vélocité reçoit une valeur de préécoute par défaut lorsqu’elle est omise.

L’audition est soutenue jusqu’à `PreviewPitchHandle.release()` ou jusqu’à une durée maximale de sécurité. `release()` est idempotente et relâche uniquement la voix créée par cet appel ; sa release et son tail peuvent ensuite se terminer naturellement. La présentation conserve le handle entre `pointerdown` et `pointerup` ou `pointercancel`.

##### Préécoute d’une sélection

`previewSelection(noteIds)` résout les notes dans le score édité depuis l’`effectiveProject`, ignore leurs positions et leurs durées, déduplique leurs `Pitch` et attaque simultanément l’ensemble obtenu avec une vélocité de préécoute fixe. Chaque attaque est brève et produit automatiquement ses `NOTE_OFF` après une durée applicative fixe : aucune note n’est soutenue jusqu’à la fin du geste.

Le `PreviewSelectionHandle` reste toutefois actif pendant le geste afin de suivre les transformations. À chaque remplacement de l’`effectiveProject`, le service compare la hauteur de chaque `NoteId` sélectionné avec sa valeur précédente :

- si aucune hauteur sélectionnée n’a changé, notamment pendant un déplacement seulement temporel, aucune attaque n’est produite ;
- dès que la hauteur d’au moins une note sélectionnée change, toute attaque précédente encore active est relâchée, puis l’ensemble complet des hauteurs actuelles est de nouveau dédupliqué et réattaqué brièvement ;
- les hauteurs inchangées sont donc elles aussi rejouées ;
- plusieurs notes partageant désormais le même `Pitch` ne produisent qu’une seule voix.

Le déclenchement dépend des hauteurs portées par les identités sélectionnées, et non seulement de l’ensemble dédupliqué final. Ainsi, une note peut changer de hauteur et imposer une réattaque complète même si les doublons produisent finalement le même ensemble de `Pitch`.

Les mises à jour reçues dans un même cycle sûr de planification sont coalescées : seule la projection la plus récente déclenche l’attaque. `PreviewSelectionHandle.stop()` est idempotente, cesse d’observer le geste, annule toute attaque encore en attente et relâche les voix brèves encore actives.

##### Préparation de l’instrument

`PlaybackService` demande le préchargement de l’instrument à l’ouverture du piano roll lorsqu’une piste d’écoute est définie, ou lors de son choix explicite. Sans piste, aucune banque n’est préparée. `previewPitch` et `previewSelection` effectuent d’abord leur validation synchrone et retournent `err(PreviewValidationError)` sans handle lorsque l’entrée est invalide.

Lorsqu’un handle est retourné, il l’est immédiatement, même si la banque n’est pas encore disponible. Sa propriété `ready` expose l’issue asynchrone de la préparation :

- après chargement, `previewPitch` ne produit son `NOTE_ON` que si son handle n’a pas déjà été relâché ;
- pour `previewSelection`, seule la projection de hauteurs la plus récente est attaquée si le handle est encore actif ;
- un `release()` ou un `stop()` antérieur à la fin du chargement fait résoudre `ready` avec `ok("CANCELLED")` et empêche toute attaque tardive ;
- un échec de chargement fait résoudre `ready` avec `err(InstrumentPreparationError)` et ne crée aucune voix ;
- une attaque effectivement créée fait résoudre `ready` avec `ok("STARTED")`.

Le handle représente ainsi une intention immédiatement annulable, tandis que `ready` décrit son démarrage technique différé.

#### Planification selon la portée

Pour un transport `PROJECT`, le service obtient directement l’intervalle global de chaque clip depuis son `start`, son `repeatCount` et la durée du score référencé :

```text
clipInterval = [clip.start,
                      clip.start + score.duration * clip.repeatCount)
projectEnd = max(clipInterval.end), ou 0 sans clip
```

Les intervalles sont semi-ouverts. Tous ceux qui se recouvrent sont planifiés simultanément, indépendamment de leurs pistes. Chaque répétition recommence au tick local `0` avec les valeurs initiales de métrique et d’harmonie du score.

Pour un transport `SCORE`, le service parcourt les événements du `scoreId` attaché à la session entre son curseur local et `score.duration`, avec l’instrument de son `trackId`. Aucun placement global ni `repeatCount` n'intervient.

Le tempo unique du projet convertit les ticks en secondes dans les deux portées :

```text
PROJECT:
command.at = ((eventGlobalTick - sessionStartProjectTick) / 960)
           * (60 / project.tempo.bpm)

SCORE:
command.at = ((eventLocalTick - sessionStartScoreTick) / 960)
           * (60 / project.tempo.bpm)
```

La planification peut rester glissante et bornée pour limiter le volume de commandes préparées.

Si le tempo est modifié pendant le transport, le service ancre la nouvelle conversion à la borne de replanification. Le temps déjà écoulé n'est pas recalculé et aucune chronologie de tempo n'est créée :

```text
command.at = replanAt
           + ((eventTick - replanTick) / 960)
           * (60 / effectiveProject.tempo.bpm)
```

`eventTick` et `replanTick` sont interprétés dans le référentiel du transport actif. La borne conserve donc le tick global atteint pour `PROJECT`, ou le tick local atteint pour `SCORE`.

Lorsqu’un transport commence au milieu d’un clip ou d’un score, le service applique une note chase minimale : toute note dont l'intervalle couvre la tête est réattaquée à l'ouverture de la session, puis relâchée à sa fin restante. Il ne tente pas de reconstruire une enveloppe ou un état de voix antérieur. Pour `PROJECT`, le service calcule d'abord la position locale dans chaque clip en tenant compte de sa répétition.

#### Identités d'exécution

La lecture utilise trois niveaux d'identité opaques et transitoires :

| Identité | Portée |
| --- | --- |
| `PlaybackSessionId` | Un transport de projet, un transport de score ou une préécoute |
| `PlaybackContextId` | Une unité audio isolée appartenant à une session |
| `NoteOccurrenceId` | Une attaque précise dans un contexte |

Une nouvelle occurrence de note est créée à chaque attaque, y compris lors des répétitions et des réattaques complètes d’une sélection. Deux notes issues de clips utilisant le même instrument, avec la même hauteur et le même instant, restent ainsi indépendantes, sauf la déduplication explicite propre à `previewSelection`.

Un nouveau contexte est créé à chaque activation d’un clip dans un transport `PROJECT`, pour le score isolé d’un transport `SCORE`, pour chaque `previewPitch` et pour une préécoute de sélection. Les répétitions d’un même clip réutilisent son contexte et son instance d'instrument, mais produisent de nouvelles occurrences de notes. Deux clips simultanés référençant le même `ScoreId` possèdent toujours des contextes distincts.

Les identifiants persistants `ScoreId`, `ClipId` et `NoteId` restent connus du domaine et du service. Ils ne sont pas transmis au moteur audio.

Ces identités d'exécution appartiennent au langage interne du port `AudioEngine` et ne sont jamais exposées à la présentation. `PreviewPitchHandle` et `PreviewSelectionHandle` exposent uniquement le contrôle nécessaire à leur audition.

#### Sessions et concurrence

```ts
type PlaybackSessionKind =
  | "PROJECT"
  | "SCORE"
  | "PITCH_PREVIEW"
  | "SELECTION_PREVIEW";

interface TransportAnchor {
  at: number;     // secondes relatives à la session
  tick: number;   // position continue interne, globale ou locale
  tempo: Tempo;
}

type ActiveTransport =
  | { kind: "PROJECT"; sessionId: PlaybackSessionId; anchor: TransportAnchor }
  | { kind: "SCORE"; sessionId: PlaybackSessionId; scoreId: ScoreId; trackId: TrackId; anchor: TransportAnchor };
```

| Catégorie | Kind | Règle de concurrence |
| --- | --- | --- |
| Transport global | `PROJECT` | Mutuellement exclusif avec `SCORE` |
| Transport local | `SCORE` | Mutuellement exclusif avec `PROJECT` |
| Touche du piano roll | `PITCH_PREVIEW` | Plusieurs sessions peuvent coexister entre elles et avec le transport |
| Sélection manipulée | `SELECTION_PREVIEW` | Au plus une session de sélection ; peut coexister avec le transport et les préécoutes de hauteur |

`PlaybackSessionKind` appartient à `PlaybackService` : il décrit les catégories de cas d’usage et leurs règles de concurrence. Il n’est pas transmis à `AudioEngine`, dont toutes les sessions suivent le même contrat technique.

Le service conserve au plus un `ActiveTransport`. Démarrer `playProject` ou `playScore` retire ce rôle au transport précédent et annule ses attaques futures lorsque la nouvelle portée est prête à démarrer. Ses contextes peuvent néanmoins subsister jusqu'à la fin de leurs releases et tails ; cela ne constitue pas un second transport actif. Les deux têtes de lecture restent indépendantes.

`previewPitch` et `previewSelection` ne remplacent jamais le transport. Démarrer une nouvelle préécoute de sélection arrête la précédente. Relâcher ou arrêter leurs handles termine structurellement leur session sans affecter les autres auditions.

`stop(mode)` invalide d’abord toute `PendingTransportRequest`, puis arrête l'unique transport actif, qu'il soit `PROJECT` ou `SCORE`, immobilise sa tête à la position courante et n'affecte aucune préécoute. Ni `GRACEFUL` ni `IMMEDIATE` ne réinitialise l'une des deux têtes. Le service transmet au moteur l'identifiant de la session correspondante. Sans transport actif, l’opération annule encore la requête de transport en préparation ; sans transport ni requête en attente, elle est sans effet.

Le mode par défaut est `GRACEFUL` :

- les attaques futures sont annulées ;
- les occurrences de note actives sont relâchées immédiatement ;
- les contextes laissent leurs releases et tails se terminer.

`IMMEDIATE` détruit sans délai les contextes ciblés et leur sortie sonore. Le remplacement d’un transport suit la politique `GRACEFUL`.

### Ports applicatifs

#### AudioEngine

`AudioEngine` accepte des identités d'exécution, des commandes sonores et des bornes de cycle de vie sans exposer `smplr`, les définitions ou instances techniques d'instrument, ni les objets Web Audio. Il ne reçoit pas la catégorie applicative de la session. Le `PlaybackService` résout le `Track.instrumentId` depuis le clip pour `PROJECT` ou depuis la piste d’écoute capturée pour `SCORE` et les préécoutes, puis le copie dans chaque commande `NOTE_ON` ; la commande reste ainsi autonome au moment de son exécution sans attribuer l'instrument à la note persistante.

```ts
type AudioCommand = {
  contextId: PlaybackContextId;
  at: number;
} & (
  | {
      kind: "NOTE_ON";
      occurrenceId: NoteOccurrenceId;
      instrumentId: InstrumentId;
      pitch: Pitch;
      velocity: Velocity;
    }
  | {
      kind: "NOTE_OFF";
      occurrenceId: NoteOccurrenceId;
    }
);

interface ContextCompletion {
  contextId: PlaybackContextId;
  at: number;
}

interface PlaybackSchedule {
  audioCommands: readonly AudioCommand[];
  contextCompletions: readonly ContextCompletion[];
}

interface ScheduleUpdate extends PlaybackSchedule {
  from: number;
}

interface PlaybackClock {
  now: number;
  safeAt: number;
}

type ScheduleError = {
  kind: "SCHEDULE_ERROR";
  code: "SCHEDULE_TOO_LATE";
  safeAt: number;
};

interface AudioEngine {
  prepareInstruments(
    instrumentIds: readonly InstrumentId[]
  ): Promise<Result<"READY", InstrumentPreparationError>>;

  openSession(sessionId: PlaybackSessionId): void;

  openContext(
    sessionId: PlaybackSessionId,
    contextId: PlaybackContextId
  ): void;

  getClock(sessionId: PlaybackSessionId): PlaybackClock;

  schedule(
    sessionId: PlaybackSessionId,
    schedule: PlaybackSchedule
  ): Result<void, ScheduleError>;

  replaceSchedule(
    sessionId: PlaybackSessionId,
    update: ScheduleUpdate
  ): Result<void, ScheduleError>;

  closeSession(sessionId: PlaybackSessionId): void;
  stopContext(contextId: PlaybackContextId, mode: StopMode): void;
  stopSession(sessionId: PlaybackSessionId, mode: StopMode): void;
}
```

Tous les champs `at`, ainsi que `PlaybackClock.now`, `PlaybackClock.safeAt` et `ScheduleUpdate.from`, sont exprimés en secondes relativement au début de la session. Le moteur possède l’horloge monotone et la conversion vers son horloge technique interne ; l’application n’utilise ni `Date.now()` ni directement `AudioContext.currentTime`.

`now` représente la position audio actuelle de la session. `safeAt` est la première borne que l’application peut encore remplacer ou programmer de façon fiable selon le lookahead, la latence et le cycle du moteur. Le moteur garantit `safeAt >= now`. Les événements antérieurs à `safeAt` sont considérés comme engagés.

`prepareInstruments` résout et charge toutes les ressources demandées. `PlaybackService` en est l’unique appelant applicatif et attend `ok("READY")` avant toute continuation sonore. Pour une édition nécessitant de nouvelles banques, il transmet ce résultat à l’orchestration d’`EditService` avant la publication du nouvel `effectiveProject` et de son plan sonore. Le chargement reste ainsi technique sans déplacer dans l’application les définitions `smplr`.

Un échec attendu de banque est retourné directement par `err(InstrumentPreparationError)` ; il ne rejette pas la promesse et ne demande aucune conversion supplémentaire dans les services. Les défauts de programmation et les défaillances techniques non prévues restent des exceptions. L’obsolescence, `CANCELLED` et `SUPERSEDED` n’appartiennent pas au port : ils sont déterminés par le propriétaire applicatif de la demande après réception du résultat.

`openContext` enregistre une seule fois la relation entre le contexte et sa session propriétaire. Chaque `AudioCommand` et `ContextCompletion` transporte donc uniquement son `contextId`.

`ContextCompletion` n’est pas une commande sonore. Elle fixe la fin structurelle planifiée d’un contexte : aucune nouvelle attaque de ce contexte n’est acceptée à partir de `at`, ses voix encore actives sont relâchées, puis il passe à `DRAINING` ou directement à `DISPOSED`. Les commandes et fins nécessaires placées avant cette borne restent exécutées.

`replaceSchedule` retire, pour la session ciblée, les `AudioCommand` et `ContextCompletion` non encore exécutés dont `at >= from`, puis installe atomiquement les deux nouvelles collections. `from` doit être supérieur ou égal à la borne sûre encore valable lors de l’acceptation par le moteur, pas seulement à celle lue avant le calcul. Si cette borne a été dépassée, le moteur retourne `err({ kind: "SCHEDULE_ERROR", code: "SCHEDULE_TOO_LATE", safeAt })` sans retirer ni installer aucun événement. Le service recalcule les voix et commandes à une nouvelle borne avec la dernière projection candidate ; il ne décale pas simplement les anciennes commandes. Ce résultat reste interne à l’orchestration. `schedule` vérifie de même les ajouts devenus tardifs. L’identité, l’origine temporelle et la continuité du transport restent inchangées.

La borne retournée peut devenir dépassée à son tour ; le service peut choisir une marge supplémentaire bornée. Une replanification ne doit jamais prolonger un `NOTE_OFF` déjà engagé avant sa borne. Le moteur conserve donc les événements remplaçables dans sa propre file et ne transmet à `smplr` que la portion engagée ; les possibilités d’annulation et le lookahead du moteur d’instrument déterminent la borne annoncée.

`openSession` crée une session ouverte sans lancer une horloge audible à vide. La première planification acceptée fixe son origine technique avec une marge suffisante pour jouer les événements `at = 0`. Avant cette origine, la tête reste au tick de départ ; l’horloge peut exposer un `now` négatif pour cette courte attente technique. Les conversions et le démarrage visuel utilisent la même origine.

À un même instant `at`, le moteur garantit l’ordre suivant sur l’ensemble de la session :

1. tous les `NOTE_OFF` ;
2. toutes les `ContextCompletion` ;
3. tous les `NOTE_ON`.

Une fin de contexte interdit ainsi ses propres attaques simultanées, tandis qu’un `NOTE_ON` appartenant à un nouveau contexte reste accepté. Entre événements d’une même catégorie et du même instant, l’ordre d’insertion est stable mais ne porte aucune signification musicale.

`closeSession` marque la fin structurelle naturelle décidée par le service lorsque sa borne est atteinte. Elle interdit de nouveaux contextes ou plans sans avancer les fins déjà acceptées ; les derniers contextes se terminent puis la session est libérée. Elle est idempotente. `stopContext` et `stopSession` restent les opérations d’interruption demandées immédiatement selon un `StopMode` ; `stopSession` ferme aussi la session aux nouveaux plans. Elles sont distinctes d’une `ContextCompletion` planifiée et ne servent pas à replanifier un transport qui continue.

`InstrumentDefinition`, `InstrumentInstance`, `AudioNode` et `AudioContext` ne traversent jamais ce port.

#### InstrumentCatalog

`InstrumentCatalog` est un port de consultation permettant :

- de lister les `Instrument` disponibles ;
- d'obtenir un `Instrument` à partir de son `InstrumentId` ;
- de vérifier si un identifiant peut être résolu.

Il retourne directement les objets `Instrument` du domaine. Aucune configuration `smplr`, banque d'échantillons, instance technique ou donnée de chargement ne traverse ce port.

#### ProjectFileStore

```ts
interface ProjectFileStore {
  read(): Promise<Result<
    { project: Project; settings: Settings } | undefined,
    ProjectFileError
  >>;

  write(
    project: Project,
    settings: Settings
  ): Promise<Result<"SAVED" | "CANCELLED", ProjectFileError>>;
}
```

`undefined` dans une lecture réussie représente l’annulation du choix de fichier. `ProjectFileError` compose les erreurs de lecture/écriture, de format JSON, de réglages invalides et de version non prise en charge (`UNSUPPORTED_FILE_VERSION`) avec les unions de validation déjà définies par le domaine ; il ne redéclare pas ces dernières. Le port reçoit ou retourne un `Project` validé avec ses `Settings` applicatifs ; ses types d’erreur sont structurés, sans message d’interface. Le contrôle de disponibilité des `InstrumentId` appartient à `ProjectFileService`, après reconstitution et avant publication.

Le port ne reçoit aucun des trois états d’éditeur ni projet transitoire. Il connaît `Settings`, car ces configurations font partie du fichier sans appartenir au domaine musical. Les chemins, dialogues de fichiers, chaînes JSON et données brutes restent à l’infrastructure. Le service propage l’échec attendu par `Result` ; les défauts de programmation restent des exceptions.

---

## Présentation

La présentation distingue une vue globale toujours présente, un éditeur d’arrangement pour le projet ouvert et un éditeur de score pour le contenu local. Leurs états applicatifs suivent la même séparation et ne sont pas imbriqués.

### Éditeur global

L’éditeur global existe pendant tout le cycle de l’application et observe `GlobalEditorState`. Il fournit le cadre commun dans lequel l’utilisateur peut ouvrir ou fermer un projet, puis accéder aux éditeurs spécialisés. Il ne possède ni la sélection des clips, ni la tête de l’arrangement, ni le score ouvert. Les états d’affichage sans portée applicative restent dans les stores de présentation.

### Éditeur d’arrangement

L’éditeur d’arrangement observe `ArrangementEditorState` et affiche une grille temporelle bidimensionnelle contenant directement les `Clip` persistants. L'axe horizontal représente des `Tick` depuis le début du projet. Il est commun à tous les scores et continu sur toute la composition. L’axe vertical présente les pistes dans l’ordre de `Project.tracks`, avec leur nom et leur instrument.

```mermaid
block-beta
    columns 5
    t0["0–2 s"] t1["2–4 s"] t2["4–5 s"] t3["5–6 s"] t4["6–8 s"]
    intro["Piano · Introduction"] space:3 conclusion["Piano · Conclusion"]
    space grooveA["Percussions · Groove A"] grooveB["Percussions · Groove B"]:2 space
    space bass["Basse"]:2 space:2
```

Ce schéma reprend le cas 3 ; ses colonnes représentent des intervalles de durées différentes et ne constituent pas une échelle proportionnelle.

Dans cette représentation :

| Élément visuel | Signification |
| --- | --- |
| Position horizontale | `Clip.start` sauvegardé |
| Longueur d'un bloc | `score.duration * clip.repeatCount`, convertie par le tempo du projet |
| Position verticale | Rang de la piste référencée par `Clip.trackId` dans `Project.tracks` |
| Blocs chevauchants, sur une même piste ou non | Clips lus simultanément |
| Tête globale verticale | Position dérivée du transport actif, sinon `ArrangementEditorState.playhead` |

Tous les clips d’une piste utilisent son instrument. Déplacer un clip verticalement change sa piste et peut donc changer le son ; la superposition avec un bloc de la piste cible reste valide. Réordonner les pistes conserve au contraire toutes les affectations instrumentales. La présentation permet de distinguer et sélectionner les blocs superposés ; leur ordre de dessin n’introduit aucune priorité audio.

La présentation affiche toujours l’`effectiveProject`. Ouvrir un bloc dans le piano roll résout son `scoreId` et son `trackId`, initialise la piste d’écoute et crée, si le score change, le `ScoreEditorState` avec une position mémorisée au tick `0` et édite le contenu référencé ; si ce score est déjà lu isolément, sa tête affichée suit la position dérivée du transport ; tous les clips correspondants reflètent immédiatement la modification. Pendant un geste, `EditService` dérive `transientProject` de la commande quantifiée. Après les éventuelles préparations et l’acceptation du plan, cette projection devient visible et audible à la borne sûre, même si une collision provisoire empêche encore d’en faire un `Project` valide. Les coordonnées acceptées au terme du geste appartiennent au domaine ; le pointeur brut, les pixels, le zoom et le défilement restent des états de présentation.

Les actions de duplication distinguent clairement le partage du score et la création d’une copie indépendante. Le nombre de clips utilisant un score peut être dérivé de leurs `scoreId` pour informer l’utilisateur de la portée d’une édition. Cette information n’est ni un champ du document ni un mode d’édition : modifier le score ouvert modifie toujours ce score et tous les clips qui le référencent.

### Éditeur de score

L’éditeur de score observe `ScoreEditorState` et présente le contenu local du score ouvert dans un piano roll. Il peut afficher simultanément des notes de hauteurs différentes. Après quantification d'une création ou d'une transformation, une collision n'existe que si deux notes de même hauteur se chevauchent avec une durée strictement positive.

Lorsque `commitEdit` retourne `ok("DECISION_REQUIRED")`, la présentation lit `EditSession.phase.pendingDecision` après discrimination sur `phase.status`. Pour la variante `NOTE_OVERLAP`, elle utilise `details.overlaps` et affiche les choix `SLICE` et `MERGE`. Aucun mode n'est choisi par défaut ni mémorisé implicitement. La réponse appelle `submitEditDecision` avec l’identité de la décision et le choix explicite ; l’éditeur observe ensuite le projet publié si la validation réussit.

Pour les autres erreurs de validation, la présentation effectue une correspondance exhaustive sur `error.code` et construit elle-même le message localisé. Elle ne reçoit jamais une chaîne métier déjà formatée par le domaine.

---

## Infrastructure

L'infrastructure contient les implémentations techniques des ports applicatifs. Elle peut être remplacée sans modifier le cœur de la composition.

### Infrastructure audio

L'infrastructure audio possède :

- le catalogue concret des instruments intégrés ;
- leurs définitions techniques et leurs sources `smplr` ;
- le chargeur et les échantillons partagés ;
- les sessions et contextes d'exécution ;
- les instances d'instrument ;
- le moteur Web Audio concret.

Le moteur et son `AudioContext` Web Audio sont globaux. Un `PlaybackContext` constitue un périmètre logique et une chaîne audio isolée, pas un nouvel `AudioContext` natif.

### BuiltInCatalog

`BuiltInCatalog` est l’adaptateur concret du port `InstrumentCatalog`. Il expose uniquement les `Instrument` publics à la couche applicative et permet de résoudre un `InstrumentId` vers la définition technique correspondante pour le moteur audio.

Il consomme la collection immuable déclarée par `SmplrInstruments`. Cette relation conserve une seule source de vérité sans introduire de second port ni de registre parallèle.

### SmplrInstruments

Le module `infrastructure/audio/instruments/SmplrInstruments.ts` contient ce qui dépend directement de `smplr` :

- le type technique `InstrumentDefinition` ;
- la collection des définitions intégrées ;
- leurs sources d’échantillons ;
- leurs factories `createInstance`.

Chaque `InstrumentDefinition` associe l’`Instrument` public à sa factory technique. `createInstance` reçoit l’`AudioContext` global, le bus de sortie du `PlaybackContext` et le chargeur d’échantillons partagé, puis crée une `InstrumentInstance` fondée sur [`smplr`](https://github.com/danigb/smplr).

`InstrumentDefinition` peut être exporté pour le typage interne de l’infrastructure, mais ni ce type, ni les données `smplr` ne traversent un port applicatif.

La définition choisit l'instrument ou le preset `smplr` employé. Elle ne décrit aucune chaîne de traitement ni politique d'allocation des voix propre à Pianola. Ces détails ne traversent jamais le port `InstrumentCatalog`.

### PlaybackSession

`PlaybackSession` est l'état technique transitoire d’un transport de projet, d’un transport de score, d’une préécoute de hauteur ou d’une préécoute de sélection. Elle possède les contextes ouverts pour cette opération et permet leur arrêt collectif.

Une session ouverte reste vivante pendant les silences, même sans contexte vivant ; la planification glissante peut ouvrir des contextes plus tard. Une session est libérée seulement après sa fermeture structurelle (`closeSession` ou `stopSession`) et après la destruction de tous ses contextes. Une session remplacée ne devient pas elle-même `DRAINING` : elle est fermée et subsiste comme propriétaire de contextes éventuellement en drainage.

### PlaybackContext

`PlaybackContext` est une unité de lecture audio isolée dans une session.

Il possède notamment :

- un bus de sortie propre ;
- l’`InstrumentId` résolu depuis la piste et son unique `InstrumentInstance`, créés paresseusement au premier `NOTE_ON` ;
- une table `NoteOccurrenceId -> VoiceHandle` ;
- les commandes programmées qui doivent pouvoir être annulées ;
- un état `SCHEDULED`, `ACTIVE`, `DRAINING` ou `DISPOSED`.

`DRAINING` appartient exclusivement au cycle de vie du contexte. Il commence après la fin structurelle ou un arrêt gracieux, lorsque des releases ou tails restent audibles.

#### Propriétaire et transitions d'état

La machine d'état de `PlaybackContext` appartient au moteur audio concret. Le `PlaybackService` reste l'autorité temporelle : il décide de la fin structurelle, de la replanification et des arrêts, puis les exprime à travers `AudioEngine`.

```mermaid
stateDiagram-v2
    [*] --> SCHEDULED: openContext
    SCHEDULED --> ACTIVE: première commande exécutée
    ACTIVE --> DRAINING: achèvement ou arrêt gracieux avec résidu sonore
    SCHEDULED --> DISPOSED: achèvement ou arrêt
    ACTIVE --> DISPOSED: achèvement ou arrêt sans résidu
    DRAINING --> DISPOSED: silence ou délai maximal
    SCHEDULED --> DISPOSED: arrêt immédiat
    ACTIVE --> DISPOSED: arrêt immédiat
    DRAINING --> DISPOSED: arrêt immédiat
```

| État | Signification | Nouvelles commandes |
| --- | --- | --- |
| `SCHEDULED` | Le contexte est ouvert, mais aucune de ses commandes n'a encore été exécutée. | Acceptées. |
| `ACTIVE` | Au moins une commande a été exécutée et la lecture structurelle peut encore produire des attaques. | Acceptées. |
| `DRAINING` | La lecture structurelle est terminée ; seules les occurrences de note relâchées et les tails subsistent. | Toute nouvelle attaque est refusée. |
| `DISPOSED` | La chaîne audio, les instances et les références du contexte ont été libérées. | Toute commande devient sans effet. |

Une `ContextCompletion` exprime la fin structurelle décidée par le `PlaybackService`. À son instant planifié, le moteur refuse les nouvelles attaques de ce contexte, relâche ses occurrences de note encore actives et passe à `DRAINING` si un signal peut encore être produit ; sinon il passe directement à `DISPOSED`.

Un arrêt `GRACEFUL` suit la même sortie vers `DRAINING`, mais peut survenir avant la fin structurelle. Le remplacement préparé d’un instrument utilise également cette sortie pour l’ancien contexte, tandis que le nouveau contexte est ouvert dans la même session. Un arrêt `IMMEDIATE` annule les commandes futures, coupe la sortie et conduit directement à `DISPOSED` depuis tout état non détruit.

Le moteur réalise seul la transition `DRAINING -> DISPOSED`, lorsqu'aucune voix ni aucun tail ne peut encore produire de signal, ou lorsque la durée maximale de sécurité est atteinte. Il notifie alors la session propriétaire, qui n’est détruite que si elle est structurellement fermée et si tous ses contextes sont `DISPOSED`.

Les opérations de cycle de vie sont idempotentes. Un contexte `DRAINING` ne peut pas redevenir `ACTIVE` : si une replanification exige de nouvelles attaques après son achèvement, le `PlaybackService` doit ouvrir un nouveau contexte. `replaceSchedule` peut déplacer une fin encore future, mais ne change pas l'état d'un contexte toujours `SCHEDULED` ou `ACTIVE`.

Si un clip est déplacé au-delà de la tête alors que son ancien contexte est déjà `DRAINING`, ce contexte conserve uniquement ses releases et tails jusqu'au silence. Il n'est ni réactivé ni coupé. Si le nouveau placement requiert des attaques futures, le service ouvre un autre contexte indépendant.

Un contexte de lecture correspond soit à l’activation audio d’un `Clip` dans un transport `PROJECT`, soit à la lecture isolée du score attaché à un transport `SCORE`. Dans le premier cas, les identifiants `ClipId`, `ScoreId` et `TrackId` référencés ainsi que leur correspondance avec le contexte restent une connaissance du `PlaybackService`. Dans le second, le service conserve l’association entre le couple `ScoreId` / `TrackId` et le contexte courant de la session. Deux clips du même score ouverts simultanément reçoivent toujours des contextes et des instances d’instrument indépendants. Le remplacement de l’instrument suit la préparation sonore des éditions et crée un nouveau contexte dans la même session ; l’ancien peut encore se drainer.

Les deux formes de préécoute utilisent le même type de contexte. Le `PlaybackService` conserve les associations internes entre leurs handles publics, leurs sessions, leurs contextes et leurs occurrences sonores ; aucun descripteur supplémentaire n'est nécessaire.

### InstrumentInstance

`InstrumentInstance` adapte une instance `smplr` au cycle de vie audio de Pianola.

Une instance appartient exclusivement à un `PlaybackContext` et dirige sa sortie vers le bus propre à ce contexte. Un contexte utilise un seul instrument résolu depuis une piste et crée au plus une instance, paresseusement. Deux clips sur la même piste possèdent néanmoins des instances indépendantes ; il en va de même pour deux pistes utilisant le même instrument.

Lors d'un `NOTE_ON`, l'instance déclenche la note à l'instant `at`. Le contrôle d'arrêt retourné par `smplr` est associé au `NoteOccurrenceId` par le contexte, afin qu'un `NOTE_OFF` puisse relâcher exactement la bonne occurrence.

`smplr` prend en charge la lecture et le cycle de vie interne de ses voix. Pianola ne modélise ni oscillateurs, ni enveloppes, ni allocation de voix propre à l'instrument.

### Ressources d'échantillons partagées

Les banques `smplr` font partie des ressources statiques distribuées et versionnées avec chaque version de l’application. Pianola ne dépend d’aucun catalogue distant ni d’un téléchargement dynamique depuis un fournisseur externe pour résoudre un instrument intégré.

Chaque `InstrumentDefinition` référence uniquement les chemins internes des échantillons livrés avec l’application. Une mise à jour de banque est donc publiée comme une nouvelle version de l’application et reste cohérente avec le catalogue compilé correspondant.

Le moteur possède un chargeur `smplr` partagé. Le chargement des ressources distribuées et leur décodage sont mutualisés entre les instances, tandis que leurs voix et leurs connexions de sortie restent isolées par contexte.

`prepareInstruments` applique un contrat universel, indépendamment de la demande qui l’a déclenché :

- les `InstrumentId` répétés sont dédupliqués ;
- une banque déjà prête produit immédiatement `ok("READY")` ;
- les demandes concurrentes d’une même banque partagent le même chargement et le même décodage ;
- chaque appel attend l’ensemble des banques distinctes qu’il a demandé ;
- l’obsolescence ou l’annulation d’une demande applicative n’invalide pas une ressource déjà chargée et n’interdit pas au chargement partagé de terminer ;
- une réussite tardive peut alimenter le cache, mais ne déclenche par elle-même ni publication, ni session, ni attaque.

Le propriétaire de chaque parcours reste responsable de vérifier que sa demande est encore courante après la préparation : `PendingTransportRequest` pour un transport, `EditSessionId + commandRevision` pour une édition, et l’état du handle pour une préécoute. Ces contrôles ne dupliquent pas le chargement ; ils définissent des continuations applicatives différentes autour du même résultat technique.

La préparation constitue une barrière de démarrage : toutes les banques nécessaires à la portée sont chargées et décodées avant l’ouverture de la session. Le cache mémoire partagé évite de recommencer le décodage lors des lectures suivantes. Le cache HTTP éventuel des ressources statiques relève du mécanisme ordinaire de distribution de l’application et non d’un catalogue de banques téléchargées à la demande.

### WebAudioEngine

Le moteur audio concret implémente `AudioEngine`, crée les instances `smplr` propres aux contextes et produit leur mixage dans l'`AudioContext` global.

`smplr` est utilisé uniquement comme moteur d'instrument. Son séquenceur n'est pas utilisé : le `PlaybackService` reste l'unique autorité qui transforme soit les clips placés, soit le contenu local du score attaché au transport, en commandes horodatées.

Le premier périmètre repose sur les nœuds Web Audio natifs employés par `smplr` et ne nécessite aucun `AudioWorklet`. Tout l'état du moteur est transitoire et n'est jamais sauvegardé dans le `Project`.

### Persistance

`JsonProjectFileStore` implémente `ProjectFileStore`. Il lit et écrit un fichier JSON dont l’enveloppe est :

```ts
interface ProjectFileData {
  schemaVersion: 1;
  project: ProjectData;
  settings: SettingsData;
}

interface SettingsData {
  arrangement: {
    gridResolution: GridResolutionData;
  };

  scores: Record<
    ScoreId,
    {
      gridResolution: GridResolutionData;
    }
  >;
}

interface GridResolutionData {
  snapStepTicks: number;
}
```

`ProjectData` est la représentation sérialisable de l’agrégat : identifiants, nom et métadonnées du projet, tempo en BPM, collection ordonnée `tracks`, collections `scores` et `clips`. Les pistes contiennent `id`, `name` et `instrumentId` ; leur ordre dans la collection est sauvegardé. Chaque `ScoreData` contient `id`, `name`, `duration` en ticks, `notes`, `meterChanges` et `harmonyChanges`, avec les identités locales de ces entités. Chaque `ClipData` contient `id`, `scoreId`, `trackId`, `start` et `repeatCount` ; `scoreId` est toujours obligatoire. Les Value Objects y sont représentés par leurs valeurs primitives validables. Les références et identités sont conservées exactement, sans dupliquer les contenus partagés.

Un score est sérialisé une seule fois dans `scores`, qu’il soit référencé par zéro, un ou plusieurs clips. Une copie indépendante occupe une seconde entrée avec un autre `id`, même si son contenu est identique. Il n’existe aucun contenu musical inline, champ de liaison optionnel, indicateur de partage ni déduplication par valeur à la lecture. L’ouverture conserve donc à la fois le partage volontaire et l’indépendance des copies ; elle ne génère pas de nouveaux identifiants.

`SettingsData.arrangement` conserve la résolution de l’éditeur d’arrangement. `SettingsData.scores` contient exactement une entrée par `ScoreId` de `ProjectData.scores` et conserve la résolution locale de chacun, y compris lorsqu’il n’est pas ouvert ou qu’aucun clip ne le référence. `GridResolutionData` reprend la valeur primitive validable de `GridResolution` sans faire entrer le schéma JSON dans l’application.

Le fichier ne contient ni sections dérivées, ni secondes, ni rôles harmoniques calculés, ni `GlobalEditorState`, ni `ArrangementEditorState`, ni `ScoreEditorState`, ni sélection, ni piste d’écoute du piano roll, ni tête, ni historique, ni ressources audio. Le schéma initial `1` décrit ce modèle de scores référencés, de pistes instrumentales et de réglages de grille ; toute autre version est refusée. Une enveloppe de version `1` dont les champs ne respectent pas cette structure est également refusée, sans alias de champs ni conversion implicite.

Le décodage vérifie l’enveloppe et la forme des données, puis reconstitue l’agrégat avec les mêmes factories et validations que la création interactive. Il reconstitue aussi chaque `GridResolution` par sa factory et vérifie la correspondance exacte entre les clés de `SettingsData.scores` et les `ScoreId` du projet. Une incohérence métier ou applicative produit une erreur de validation sans objet partiellement valide. Un problème de syntaxe, de version ou d’accès reste distinct. `ProjectFileService` contrôle ensuite le catalogue avant de publier ensemble le projet et ses réglages.

La sauvegarde porte exclusivement sur la version validée du projet et les réglages capturés par le cas d’usage. Un geste musical en cours, même audible, n’est jamais adopté implicitement.

---

## Dépendances architecturales

| Depuis | Dépend de | Ne connaît pas |
| --- | --- | --- |
| Domaine | Aucun élément extérieur | sélection, pixels, cas d'usage, ports, Web Audio, stockage |
| Application | Domaine et ports qu'elle définit | implémentations concrètes, `smplr`, banques d'échantillons, `AudioNode` |
| Infrastructure | Domaine et ports applicatifs | composants et état de présentation |
| Présentation | API applicative et modèles d'affichage | définitions et instances audio internes |

## Questions ouvertes

Les invariants de composition et les contrats de publication ci-dessus sont définis. Les choix suivants restent à préciser sans ajouter de nouveaux concepts au domaine :

- Comment distinguer et sélectionner les clips superposés sur une même piste dans la présentation ?
- Quelles valeurs techniques retenir pour la marge de planification, la durée maximale des préécoutes et tails et la capacité de l’historique ? Ces paramètres ne doivent pas modifier les règles de propriété ou de concurrence.
- Quelle interaction proposer pour quitter ou remplacer un document modifié non sauvegardé ? L’ouverture réussie reste atomique et la sauvegarde porte toujours sur une version validée.

## Arborescence cible

Cette arborescence sépare explicitement les données métier et leurs invariants des algorithmes qui les transforment ou les analysent. `models/` contient les représentations immuables, leurs identités, leurs factories et leurs validations intrinsèques. `operations/` contient des fonctions pures et sans état qui reçoivent des modèles valides et retournent soit une nouvelle version validée, soit une information dérivée. Les dépendances extérieures restent assemblées au point d’entrée de l’application ; les services ne construisent pas leurs adaptateurs.

```text
src/
├── domain/
│   ├── Result.ts
│   ├── models/
│   │   ├── composition/
│   │   │   ├── Project.ts
│   │   │   ├── Track.ts
│   │   │   ├── Score.ts
│   │   │   ├── Clip.ts
│   │   │   ├── Note.ts
│   │   │   └── Velocity.ts
│   │   ├── time/
│   │   │   ├── Tick.ts
│   │   │   ├── Duration.ts
│   │   │   ├── TimeRange.ts
│   │   │   ├── Tempo.ts
│   │   │   ├── Meter.ts
│   │   │   └── MeterChange.ts
│   │   ├── pitch/
│   │   │   ├── Pitch.ts
│   │   │   └── RootNote.ts
│   │   ├── harmony/
│   │   │   ├── Chord.ts
│   │   │   ├── Scale.ts
│   │   │   ├── Harmony.ts
│   │   │   └── HarmonyChange.ts
│   │   └── instrument/
│   │       └── Instrument.ts
│   └── operations/
│       ├── composition/
│       │   ├── ProjectTransformations.ts
│       │   ├── ScoreTransformations.ts
│       │   └── NoteOverlap.ts
│       ├── time/
│       │   └── MeterTimeline.ts
│       └── harmony/
│           ├── HarmonyTimeline.ts
│           └── NoteRoleAnalysis.ts
├── application/
│   ├── GlobalEditorState.ts
│   ├── ArrangementEditorState.ts
│   ├── ScoreEditorState.ts
│   ├── Settings.ts
│   ├── ProjectState.ts
│   ├── EditSession.ts
│   ├── ProjectHistory.ts
│   ├── Selection.ts
│   ├── Grid.ts
│   ├── use-cases/
│   │   ├── EditService.ts
│   │   ├── PlaybackService.ts
│   │   └── ProjectFileService.ts
│   └── ports/
│       ├── AudioEngine.ts
│       ├── InstrumentCatalog.ts
│       └── ProjectFileStore.ts
├── infrastructure/
│   ├── audio/
│   │   ├── instruments/
│   │   │   ├── BuiltInCatalog.ts
│   │   │   └── SmplrInstruments.ts
│   │   └── engine/
│   │       ├── WebAudioEngine.ts
│   │       ├── PlaybackSession.ts
│   │       ├── PlaybackContext.ts
│   │       └── InstrumentInstance.ts
│   └── persistence/
│       └── JsonProjectFileStore.ts
└── presentation/
    ├── components/
    └── stores/
```

### Propriétaires des types et comportements

| Module | Contenu |
| --- | --- |
| `domain/Result.ts` | `Result`, helpers et forme générique de `ValidationError` ; les erreurs concrètes restent auprès du modèle ou de l'opération qui les produit |
| `domain/models/composition/Project.ts` | `Project`, `ProjectId`, factory, reconstitution, `MAX_TRACK_COUNT`, ordre des pistes, collection de scores et validations de référence, dont `SCORE_NOT_FOUND`, `SCORE_IN_USE` et `DUPLICATE_SCORE_ID` |
| `domain/models/composition/Track.ts` | `Track`, `TrackId`, factory et invariants propres à une piste |
| `domain/models/composition/Score.ts` | `Score`, `ScoreId`, références `ScoreContentRef`, factory, `ScoreValidationError` et invariants propres au contenu musical local |
| `domain/models/composition/Clip.ts` | `Clip`, `ClipId`, références obligatoires de score et de piste, limite de répétitions et invariants de placement, dont `INVALID_SCORE_ID` |
| `domain/models/composition/Note.ts` | `Note`, `NoteId`, factory et invariants d'une note isolée |
| `domain/models/composition/Velocity.ts` | `Velocity`, bornes et validation |
| `domain/models/time/Tick.ts` | `Tick` et `MAX_TICK` |
| `domain/models/time/Duration.ts` | `Duration` et validation de sa valeur en ticks |
| `domain/models/time/TimeRange.ts` | `TimeRange`, intervalle semi-ouvert et validation de ses bornes composées |
| `domain/models/time/Tempo.ts` | `Tempo`, bornes, précision et conversion temporelle élémentaire |
| `domain/models/time/Meter.ts` | `Meter` et ses invariants intrinsèques |
| `domain/models/time/MeterChange.ts` | `MeterChange`, son identité et sa position locale persistante |
| `domain/models/pitch/Pitch.ts` | `Pitch` et validation du numéro MIDI |
| `domain/models/pitch/RootNote.ts` | `RootNote`, `NoteLetter`, `Accidental` et cohérence de la classe chromatique |
| `domain/models/harmony/Chord.ts` | `Chord` et catalogue des types d'accord |
| `domain/models/harmony/Scale.ts` | `Scale` et catalogue des types de gamme |
| `domain/models/harmony/Harmony.ts` | `Harmony`, union exclusive entre accord et gamme avec une `RootNote` explicite |
| `domain/models/harmony/HarmonyChange.ts` | `HarmonyChange`, son identité et sa position locale persistante |
| `domain/models/instrument/Instrument.ts` | `Instrument` public et `InstrumentId` |
| `domain/operations/composition/ProjectTransformations.ts` | Toutes les commandes et fonctions pures retournant un nouveau `Project` via `Result`, y compris `addScore`, `duplicateClips`, `makeClipIndependent` et les transformations des pistes, scores ou clips intégrées à l’agrégat |
| `domain/operations/composition/ScoreTransformations.ts` | Toutes les commandes et fonctions pures retournant un nouveau `Score` via `Result`, notamment `duplicateScore` et les transformations de ses notes, de sa durée et de ses chronologies |
| `domain/operations/composition/NoteOverlap.ts` | `NoteOverlap`, `NoteOverlapError`, `NoteOverlapResolution`, détection et résolution `SLICE` ou `MERGE` |
| `domain/operations/time/MeterTimeline.ts` | Ordonnancement des `MeterChange`, résolution de la métrique active et production des `MeterSection` dérivées |
| `domain/operations/harmony/HarmonyTimeline.ts` | Ordonnancement des `HarmonyChange`, résolution de l'harmonie active et production des `HarmonySection` dérivées |
| `domain/operations/harmony/NoteRoleAnalysis.ts` | Segmentation d'une note et dérivation de ses rôles `CHORD_TONE`, `SCALE_TONE` ou `OUTSIDE_TONE` |
| `application/ProjectState.ts` | `ProjectState`, `TransientProject`, `Settings` courants, métadonnées observables de préparation et résolution dérivée d’`effectiveProject` ; aucune promesse ni tâche asynchrone |
| `application/EditSession.ts` | `EditSession`, `EditSessionPhase`, conteneurs génériques `PendingEditDecision` et `EditDecision`, unions `EditDecisionRequest` et `SubmittedEditDecision`, identifiants et `PendingEditPreparation` descriptif |
| `application/ProjectHistory.ts` | Versions validées, bornage et parcours de l’historique, sans orchestration audio ni persistance |
| `application/GlobalEditorState.ts` | `GlobalEditorState` et état applicatif propre à la vue globale toujours présente ; ne possède aucun état d’un éditeur spécialisé |
| `application/ArrangementEditorState.ts` | `ArrangementEditorState`, tête globale du projet et `ClipSelection` |
| `application/ScoreEditorState.ts` | `ScoreEditorState`, score ouvert, tête locale, piste d’écoute optionnelle et `ScoreContentSelection` |
| `application/Settings.ts` | `Settings`, réglage de grille de l’arrangement et réglages indexés par `ScoreId`, avec leurs invariants de correspondance au projet |
| `application/Selection.ts` | `ScoreContentSelection`, `ClipSelection` ; réutilise les références du domaine |
| `application/Grid.ts` | `GridResolution`, sa factory et la quantification des intentions dans leur référentiel ; aucune politique de persistance |
| `application/use-cases/EditService.ts` | `EditIntent`, union et composition `ProjectEditCommand`, tâches privées de préparation, cycle d’édition, publication, undo/redo, `EditOutcome`, validations et erreurs applicatives |
| `application/use-cases/PlaybackService.ts` | Transport, préparation et planification ; unique appelant de `AudioEngine.prepareInstruments`, propriétaire de `PlaybackSessionKind`, `ActiveTransport`, `TransportAnchor`, requêtes en attente, résultats publics et handles de préécoute ; importe et réexpose `StopMode` |
| `application/use-cases/ProjectFileService.ts` | Ouverture/sauvegarde, contrôle du catalogue et remplacement atomique du projet avec ses réglages |
| `application/ports/AudioEngine.ts` | `StopMode`, identités audio, `AudioCommand`, `ContextCompletion`, plans, horloge, contrat universel de préparation, `ScheduleError`, `InstrumentPreparationError` et contrat moteur ; ne connaît ni `PlaybackSessionKind`, ni l’obsolescence applicative |
| `application/ports/InstrumentCatalog.ts` | Contrat de consultation des `Instrument` publics et résolution des `InstrumentId` |
| `application/ports/ProjectFileStore.ts` | Contrat abstrait de sélection, lecture et écriture d’un `Project` avec ses `Settings`, erreurs techniques et composition avec les erreurs de validation du domaine et de l’application ; aucun schéma JSON |
| `infrastructure/audio/instruments/BuiltInCatalog.ts` | Adaptateur concret du port `InstrumentCatalog` et résolution des définitions techniques |
| `infrastructure/audio/instruments/SmplrInstruments.ts` | `InstrumentDefinition`, collection intégrée, sources d’échantillons et factories propres à `smplr` |
| `infrastructure/audio/engine/WebAudioEngine.ts` | Implémentation du port, horloge technique, planification et mixage Web Audio |
| `infrastructure/audio/engine/PlaybackSession.ts` | État technique transitoire et propriété des contextes d’une session |
| `infrastructure/audio/engine/PlaybackContext.ts` | Chaîne audio isolée, commandes programmées, voix et cycle `SCHEDULED → ACTIVE → DRAINING → DISPOSED` |
| `infrastructure/audio/engine/InstrumentInstance.ts` | Adaptation d’une instance `smplr` au cycle de vie d’un contexte |
| `infrastructure/persistence/JsonProjectFileStore.ts` | Adaptateur, `ProjectFileData`, `ProjectData`, `SettingsData`, `GridResolutionData`, `ScoreData`, `ClipData`, encodage et reconstitution du format versionné |
| `presentation/components/` | Rendu de la grille, du piano roll, des décisions et des états de chargement |
| `presentation/stores/` | État strictement visuel et adaptation réactive de l’état applicatif, sans duplication du document ni des tâches |

Les modules de `models/` décrivent des données immuables et empêchent leur construction dans un état invalide. Les modules de `operations/` ne les modifient jamais en place : une transformation reçoit un modèle valide et retourne une nouvelle version avec `Result`, tandis qu'une timeline ou une analyse produit uniquement une vue dérivée. Le terme « transformation » décrit donc un changement métier, et non une mutation de l'objet reçu.

Les commandes et erreurs propres à une transformation restent dans son module. Les erreurs de création restent auprès du modèle qui protège l'invariant correspondant. `MeterSection`, `HarmonySection` et les segments de rôle sont dérivés par les opérations et ne sont pas persistés. `Clip` demeure possédé directement par `Project` malgré son fichier distinct. Enfin, `ScoreContentRef` reste une adresse typée d'entité locale et non une sélection ; `Selection.ts` l'emploie sans déplacer sa propriété hors du domaine.

Les services de `use-cases/` sont les points d’entrée applicatifs. `ports/` décrit uniquement les capacités sortantes réalisées par l’infrastructure. `EditService` demande à `PlaybackService` la coordination sonore des publications ; `ProjectFileService` coordonne ses publications avec ces services. `PlaybackService` ne dépend pas en retour d’`EditService` et ne modifie pas l’historique. Les stores de présentation observent séparément `GlobalEditorState`, `ArrangementEditorState` et `ScoreEditorState` et conservent les détails d’interface ; ils ne les regroupent pas dans un autre modèle d’éditeur et ne dupliquent ni l’agrégat, ni la commande courante, ni l’horloge active.
