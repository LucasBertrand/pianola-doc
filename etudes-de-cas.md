# Études de cas

Ce document illustre les règles de composition et de lecture définies dans [architecture.md](architecture.md). Un `Clip` désigne un contenu musical local éditable dans le piano roll ; un bloc de la grille est une `ClipOccurrence` persistante qui référence ce contenu. Sauf indication contraire, chaque occurrence nommée dans les exemples référence un clip source de même nom.

## Conventions de calcul

Sauf indication contraire, le projet utilise un tempo unique de 120,0 BPM et une résolution de 960 ticks par noire. Un `Tempo` accepte une décimale entre 20,0 et 999,9 BPM inclus :

```text
durationSeconds = (durationTicks / 960) * (60 / 120)
```

La métrique et la chronologie harmonique restent locales à chaque clip. La métrique fournit des repères de mesures, mais ne modifie pas la conversion des ticks en secondes.

La durée structurelle d'une occurrence et celle du projet suivent les règles suivantes :

```text
occurrenceEnd = occurrence.start + clip.duration * occurrence.repeatCount
projectEnd    = maximum des occurrenceEnd, ou 0 si le projet ne contient aucune occurrence
```

Les intervalles sont semi-ouverts. Deux occurrences sont simultanées si leurs intervalles globaux se recouvrent avec une durée strictement positive. Ce recouvrement est valide sur une même ligne comme sur des lignes différentes. La ligne n'intervient jamais dans le calcul audio.

## Cas 1 — Placement explicite, espace vide et répétition

Trois occurrences sont placées sur la ligne `0` :

| Occurrence | Début global | Durée du clip référencé | `repeatCount` | Fin globale |
| --- | ---: | ---: | ---: | ---: |
| `Ouverture` | 0 | 3840 ticks | 1 | 3840 |
| `Motif` | 5760 | 1920 ticks | 2 | 9600 |
| `Conclusion` | 11520 | 3840 ticks | 1 | 15360 |

La position de `Motif` n'est pas calculée depuis la fin d'`Ouverture`. L'intervalle `[3840, 5760)` reste silencieux, puis les deux lectures contiguës du motif occupent `[5760, 9600)`. Un second silence sépare le motif de la conclusion.

```mermaid
block-beta
    columns 8
    ouverture["0–2 s · Ouverture"]:2 gap1["2–3 s · silence"] motif1["3–4 s · Motif 1"] motif2["4–5 s · Motif 2"] gap2["5–6 s · silence"] conclusion["6–8 s · Conclusion"]:2
```

Déplacer ou supprimer une occurrence ne rapproche jamais automatiquement les autres occurrences. Leurs coordonnées persistantes restent inchangées. Supprimer l'occurrence ne supprime pas son clip source, même s'il s'agissait de sa dernière occurrence : seule une suppression manuelle distincte peut ensuite supprimer ce clip non référencé.

## Cas 2 — Chevauchement de clips aux métriques indépendantes

Deux occurrences référencent des clips soumis au même tempo de projet, mais possédant des métriques locales différentes :

| Occurrence | Ligne | Début | Durée du clip | Métrique locale | Intervalle réel |
| --- | ---: | ---: | ---: | ---: | ---: |
| `Rythme` | 0 | 0 | 3840 ticks | 4/4 | 0 à 2 s |
| `Ligne de basse` | 2 | 1920 ticks | 5760 ticks | 3/4 | 1 à 4 s |

Les deux clips jouent simultanément entre une et deux secondes. La basse continue seule jusqu'à quatre secondes.

```mermaid
block-beta
    columns 4
    t0["0–1 s"] t1["1–2 s"] t2["2–3 s"] t3["3–4 s"]
    rythme["L0 · Rythme"]:2 space:2
    space basse["L2 · Ligne de basse"]:3
```

La basse conserve ses mesures en 3/4, mais elle n'utilise ni tempo propre ni horloge indépendante. Son décalage provient uniquement de son début global sauvegardé.

## Cas 3 — Composition sur plusieurs lignes

La grille contient des occurrences d'introduction, de deux grooves consécutifs, d'une basse superposée et d'une conclusion :

| Occurrence | Ligne | Début global | Durée du clip | Intervalle réel |
| --- | ---: | ---: | ---: | ---: |
| `Introduction` | 0 | 0 | 3840 ticks | 0 à 2 s |
| `Groove A` | 1 | 3840 | 3840 ticks | 2 à 4 s |
| `Groove B` | 1 | 7680 | 3840 ticks | 4 à 6 s |
| `Ligne de basse` | 2 | 3840 | 5760 ticks | 2 à 5 s |
| `Conclusion` | 0 | 11520 | 3840 ticks | 6 à 8 s |

```mermaid
block-beta
    columns 8
    t0["0–1"] t1["1–2"] t2["2–3"] t3["3–4"] t4["4–5"] t5["5–6"] t6["6–7"] t7["7–8 s"]
    intro["L0 · Introduction"]:2 space:4 conclusion["L0 · Conclusion"]:2
    space:2 grooveA["L1 · Groove A"]:2 grooveB["L1 · Groove B"]:2 space:2
    space:2 basse["L2 · Basse"]:3 space:3
```

`Groove A` et `Groove B` partagent une ligne, mais leur succession résulte exclusivement de leurs placements. `Ligne de basse` chevauche les deux grooves, puis s'arrête une seconde avant `Groove B`. La conclusion commence au tick `11520` parce que cette valeur est sauvegardée, non parce qu'un élément précédent la déclenche.

## Cas 4 — Lignes sans association instrumentale

L'occurrence `Couplet`, ligne `0`, référence un clip associé au piano. L'occurrence `Contrechant`, ligne `3`, référence un clip associé aux cordes. Leurs intervalles globaux se chevauchent et les deux parties peuvent donc être audibles simultanément.

Déplacer `Contrechant` sur la ligne `0` sans modifier son intervalle est valide : les deux blocs se superposent et continuent de jouer simultanément dans leurs contextes indépendants. Changer sa ligne ne modifie ni ses notes, ni son instrument, ni le son. La présentation doit permettre de distinguer et sélectionner les deux blocs sans créer de priorité audio.

Une ligne est uniquement une coordonnée d'organisation bornée par une limite fixe de sécurité. Elle n'impose pas d'instrument, ne crée ni bus, ni filtre, ni contexte audio commun et ne fait l'objet d'aucune commande utilisateur d'ajout ou de suppression.

## Cas 5 — Deux clips utilisent le même instrument

Deux occurrences qui se chevauchent référencent chacune un clip associé au piano et contenant une note de même hauteur.

| Occurrence | Source | Instrument | Hauteur | Début | Fin |
| --- | --- | --- | --- | ---: | ---: |
| `occurrence-a` | clip A · note A | piano | do | 0 s | 4 s |
| `occurrence-b` | clip B · note B | piano | do | 1 s | 2 s |

À deux secondes, le moteur relâche uniquement `occurrence-b`. La voix correspondant à `occurrence-a` continue jusqu'à quatre secondes. Une commande identifiée seulement par l'instrument et la hauteur serait insuffisante.

Chaque occurrence active possède son propre `PlaybackContext` et sa propre `InstrumentInstance` `smplr` du piano. Lors de chaque `NOTE_ON`, le moteur associe au `NoteOccurrenceId` le contrôle d'arrêt retourné par l'instance concernée. Les occurrences ne sont jamais fusionnées implicitement.

```mermaid
block-beta
    columns 4
    t0["0–1 s"] t1["1–2 s"] t2["2–3 s"] t3["3–4 s"]
    noteA["Note A · piano"]:4
    space noteB["Note B · piano"] space:2
```

## Cas 6 — Transports et deux formes de préécoute

Le projet est lu depuis sa tête globale dans une session `PROJECT`. Le clip source `Motif` est parallèlement ouvert dans le piano roll ; sa tête locale est restée au tick `960`.

Lorsque l'utilisateur appelle `playClip()`, le `PlaybackService` prépare l’instrument de `Motif`, puis, une fois celui-ci disponible, retire le rôle de transport à `project-session-a`, annule ses attaques futures et relâche ses occurrences actives selon le mode `GRACEFUL`. Il ouvre ensuite `clip-session-b` de type `CLIP` au tick local `960`. La tête globale conserve sa position.

| Étape | Session | Type | État |
| --- | --- | --- | --- |
| Lecture de l'arrangement | `project-session-a` | `PROJECT` | transport actif |
| Appel à `playClip()` | `project-session-a` | `PROJECT` | inactive ; contextes éventuellement `DRAINING` |
| Appel à `playClip()` | `clip-session-b` | `CLIP` | nouveau transport actif |

La session `CLIP` lit directement le contenu local de `Motif` jusqu'à sa durée structurelle. Elle ignore toutes ses occurrences, leurs positions globales et leurs répétitions.

Pendant ce transport, l’utilisateur maintient la touche `F♯4` du piano roll, même si aucune note de cette hauteur n’existe dans le clip :

```ts
const pitchResult = previewPitch(Pitch.F_SHARP_4);

if (pitchResult.ok) {
  const pitchPreview = pitchResult.value;
  pitchPreview.release(); // pointerup ou pointercancel
}
```

`previewPitch` valide immédiatement l’entrée. Lorsque son `Result` contient un handle, celui-ci est disponible avant la banque et sa propriété `ready` décrit ensuite `STARTED`, `CANCELLED` ou un échec de préparation. La session `PITCH_PREVIEW` ne remplace jamais `clip-session-b` et soutient la voix jusqu’à `release()`. Un relâchement antérieur au chargement fait résoudre `ready` avec `ok("CANCELLED")` et empêche toute attaque tardive.

L’utilisateur commence ensuite une transposition de quatre notes sélectionnées, dont deux possèdent initialement la même hauteur :

```ts
const selectionResult = previewSelection(selectedNoteIds);

if (!selectionResult.ok) {
  // La présentation traduit PreviewValidationError.
}

const selectionPreview =
  selectionResult.ok ? selectionResult.value : undefined;
```

Le service ignore leurs positions et leurs durées, déduplique leurs hauteurs et produit une attaque brève simultanée lorsque `selectionPreview.ready` peut démarrer l’audition. Lorsqu’une seule des quatre notes change de hauteur, l’attaque précédente encore active est relâchée et toutes les hauteurs actuelles de la sélection sont dédupliquées puis réattaquées. Les hauteurs restées identiques sont donc elles aussi rejouées. Un déplacement seulement temporel ne provoque aucune nouvelle attaque.

`selectionPreview?.stop()` cesse de suivre le geste, fait résoudre une préparation encore en attente avec `ok("CANCELLED")`, annule les attaques futures et relâche les voix brèves encore actives. La session `SELECTION_PREVIEW` n’a jamais remplacé le transport `CLIP`.

Un appel ultérieur à `playProject(5760)` remplace à son tour le transport `CLIP`, déplace uniquement la tête globale au tick demandé et reprend la lecture de l'arrangement. La tête locale conserve son dernier tick.

## Cas 7 — Répétitions et occurrences de notes

Un clip contient une note `note-a` et possède une durée locale de 1920 ticks. Une occurrence qui le référence possède un `repeatCount` de `3` et occupe donc 5760 ticks globaux à partir de son `start`.

| Répétition | Note persistante | Occurrence d'exécution |
| ---: | --- | --- |
| 1 | `note-a` | `occurrence-a-1` |
| 2 | `note-a` | `occurrence-a-2` |
| 3 | `note-a` | `occurrence-a-3` |

Les trois répétitions de cette même occurrence de clip réutilisent le même `PlaybackContextId` et la même `InstrumentInstance`, mais chaque attaque reçoit un `NoteOccurrenceId` distinct. Une release peut continuer au début de la répétition suivante sans confondre les deux occurrences de note.

Si une voix a déjà été volée, le `NOTE_OFF` programmé pour son ancienne occurrence devient une opération sans effet. Les relâchements sont donc idempotents.

## Cas 8 — Fin structurelle et tail audio

L'occurrence `Nappe` occupe `[0, 3840)`, soit deux secondes. Le clip qu'elle référence produit une release et une réverbération qui restent audibles une seconde supplémentaire. L'occurrence `Conclusion` est placée explicitement au tick `3840`.

| Temps réel | Événement structurel | État audio de `Nappe` |
| --- | --- | --- |
| 0 s | début de `Nappe` | `ACTIVE` |
| 2 s | fin de `Nappe`, début de `Conclusion` | `DRAINING` |
| 3 s | aucun changement de placement | `DISPOSED` après extinction du tail |

Le tail ne modifie ni la fin globale de l'occurrence `Nappe`, ni le `start` de l'occurrence `Conclusion`. Il peut se superposer à l'occurrence suivante. Le contexte de `Nappe` refuse toute nouvelle attaque après deux secondes, mais conserve ses instances jusqu'au silence ou jusqu'à une durée maximale de sécurité.

## Cas 9 — Lecture globale depuis un tick explicite

La grille contient les occurrences suivantes :

| Occurrence | Ligne | Intervalle réel |
| --- | ---: | ---: |
| `Introduction` | 0 | 0 à 2 s |
| `Piano` | 1 | 2 à 6 s |
| `Basse` | 2 | 2 à 5 s |
| `Percussions` | 3 | 1 à 4 s |
| `Conclusion` | 0 | 6 à 8 s |

Les commandes suivantes choisissent une position sans changer la portée du transport :

| Commande | Résultat |
| --- | --- |
| `playProject()`, tête globale à 0 | Lit tout le projet depuis son début. |
| `playProject(3840)` | Place la tête globale à 2 s ; `Piano`, `Basse` et la partie encore active de `Percussions` participent au transport. |
| `playProject(11520)` | Place la tête globale à 6 s et lit la fin du projet. |

`playProject` reçoit directement un `Tick` global et aucune identité d'occurrence. `playClip(tick?)` utilise au contraire un tick local au clip édité. `previewPitch` et `previewSelection` ne déplacent aucune tête et ne modifient pas le transport.

Si une note de `Percussions` a commencé avant deux secondes mais couvre encore cette position, la note chase minimale la réattaque au démarrage et programme son relâchement pour sa durée restante. Aucun état antérieur d'enveloppe ou de voix n'est reconstruit.

## Cas 10 — Alternance exclusive entre gamme et accord

Un clip de 7680 ticks possède les changements harmoniques suivants :

| Tick local | `Harmony` |
| --- | --- |
| `0` | `SCALE · C IONIAN` |
| `1920` | `CHORD · D MINOR_SEVENTH` |
| `3840` | `SCALE · A MINOR_PENTATONIC` |

| Intervalle local | Harmonie active |
| --- | --- |
| `[0, 1920)` | do ionien |
| `[1920, 3840)` | `Dm7` |
| `[3840, 7680)` | la pentatonique mineure |

Chaque `HarmonyChange` remplace la variante précédente. Aucun accord et aucune gamme ne sont actifs simultanément.

Le placement global des occurrences et le tempo du projet n’affectent pas cette chronologie locale.

## Cas 11 — Remplacement harmonique explicite

La section active contient `SCALE · C IONIAN`. L’utilisateur remplace cette valeur par `CHORD · D♭ MAJOR` : la transformation est valide, même si les notes de cet accord ne sont pas toutes dans la gamme précédente.

Le premier périmètre présente le catalogue pour un choix explicite, sans suggestion ni classement par compatibilité. Le remplacement modifie la variante du même `HarmonyChange` ; il ne crée aucun contexte harmonique simultané et ne transforme aucune note.

## Cas 12 — Gammes modales et pentatoniques

Un clip possède successivement les changements harmoniques suivants :

| Tick local | `Harmony` |
| --- | --- |
| `0` | `SCALE · D DORIAN` |
| `3840` | `SCALE · A MINOR_PENTATONIC` |

Les deux gammes sauvegardent une `RootNote` explicite. Leur résolution ne dépend d’aucun contexte supérieur implicite. La première `HarmonySection` couvre `[0, 3840)` et la seconde `[3840, clip.duration)`.

Les occurrences du clip peuvent être placées n’importe où dans la grille. Leur ligne et leur éventuel chevauchement avec d’autres occurrences ne changent pas ces analyses locales.

## Cas 13 — Note tenue à travers plusieurs variantes d’harmonie

Une note `E4` commence au tick local `960` et se termine au tick `5280`. Le clip possède `SCALE · C IONIAN` au tick `0`, `CHORD · C MAJOR` au tick `1920`, puis `SCALE · A MINOR_PENTATONIC` au tick `3840`.

| Intervalle analysé | Harmonie active | Rôle de `E4` |
| --- | --- | --- |
| `[960, 1920)` | do ionien | `SCALE_TONE` |
| `[1920, 3840)` | do majeur | `CHORD_TONE` |
| `[3840, 5280)` | la pentatonique mineure | `SCALE_TONE` |

Les frontières des `HarmonySection` sont réunies avec celles du `TimeRange` pour produire ces portions. La note persistante conserve néanmoins un seul intervalle et n’est jamais découpée.

Si une occurrence du clip commence au tick global `10000`, ces bornes locales correspondent aux ticks globaux `10960`, `11920`, `13840` et `15280`. Les objets locaux ne sont pas réécrits pour autant.

## Cas 14 — Replanification du projet transitoire

Un transport `PROJECT` est actif. À cinq secondes depuis le début de sa session, un même geste déplace globalement une occurrence déjà active et déplace un changement local situé plus loin dans le clip qu'elle référence. La commande fournit une nouvelle projection candidate. Elle deviendra l’`effectiveProject` partagé par la présentation et l’audio après acceptation de sa replanification.

Le `PlaybackService` consulte l’horloge de la session :

```ts
const { now, safeAt } = audioEngine.getClock(transportSessionId);
```

Si `safeAt` vaut `5.04` secondes, il calcule le tick de réconciliation avec l’ancien ancrage et l’état des voix prévu par l’ancien plan à cette borne. Il recalcule alors le transport sans ouvrir une nouvelle session, puis demande :

```ts
const updateResult = audioEngine.replaceSchedule(transportSessionId, {
  from: safeAt,
  audioCommands: replacementCommands,
  contextCompletions: replacementCompletions
});
```

Si la borne est encore sûre, le moteur conserve les événements antérieurs, remplace atomiquement les commandes et fins de contexte à partir de cette borne et retourne `ok(undefined)`. Si elle est dépassée, `updateResult` contient `SCHEDULE_TOO_LATE` et aucune collection n’est modifiée ; le service recalcule depuis une nouvelle borne avant publication. Dans la suite de ce cas, « tête » désigne le tick de cette borne acceptée.

Si l'ancien et le nouveau début global de la note restent avant la tête, tandis que sa fin reste après, l'occurrence de note audible est conservée et seul son `NOTE_OFF` est replanifié. Si le déplacement de la `ClipOccurrence` place l'attaque après la tête, l'occurrence de note reçoit un `NOTE_OFF` à la borne et sa future attaque est replanifiée. Une note auparavant inactive qui couvre désormais la tête reçoit un `NOTE_ON` à cette borne.

Une modification de hauteur ou de vélocité de la note impose une relâche puis une réattaque lorsqu'elle reste couverte. Un changement d’instrument utilise la préparation sonore commune aux éditions : l’ancien instrument continue de jouer jusqu’à ce que la nouvelle banque soit prête, puis chaque occurrence active bascule vers un nouveau contexte tandis que l’ancien se draine. Un changement du tempo du projet replanifie les instants futurs sans réattaquer une note dont les données sonores sont inchangées.

Le changement local appartient à la même édition atomique, mais ne produit aucun événement sonore en lui-même. Le `PlaybackSessionId` et l’origine temporelle ne changent pas ; la position poursuit son avance continue. Le document est publié après acceptation du plan, et le son le rejoint à la borne sûre.

Si le transport actif était `CLIP` sur ce même clip, le service ignorerait le déplacement de la `ClipOccurrence` et réconcilierait uniquement le contenu local depuis la tête du clip. Une modification d’un autre clip n’affecterait pas cette session ; le tempo du projet continuerait en revanche à s’y appliquer. La portée reste le clip attaché à la session, même si le piano roll en affiche un autre.

## Cas 15 — Plusieurs occurrences liées au même clip

Le clip source `Ostinato` possède une durée de 1920 ticks et contient `note-a`. Deux blocs de la grille le référencent :

| `ClipOccurrence` | `clipId` | Ligne | Début | `repeatCount` |
| --- | --- | ---: | ---: | ---: |
| `ostinato-a` | `ostinato` | 0 | 0 | 2 |
| `ostinato-b` | `ostinato` | 2 | 960 | 1 |

Les deux occurrences se chevauchent et sont planifiées dans deux `PlaybackContext` distincts. Les attaques issues de `note-a` reçoivent des `NoteOccurrenceId` distincts ; elles ne partagent ni voix ni instance audio malgré leur `ClipId` commun.

Ouvrir l'un ou l'autre bloc dans le piano roll édite le même clip `Ostinato`. Transposer `note-a`, modifier sa vélocité, déplacer un changement local ou redimensionner le clip met immédiatement à jour les deux occurrences. Un changement d’instrument ne devient effectif pour aucune des deux avant la préparation réussie de sa banque ; elles basculent ensuite ensemble dans le projet, et le `PlaybackService` réconcilie séparément leurs contextes actifs et leurs commandes futures.

Déplacer `ostinato-b`, changer sa ligne ou son `repeatCount` ne modifie pas `ostinato-a`, car ces propriétés appartiennent à chaque `ClipOccurrence`. Dupliquer `ostinato-a` crée une troisième occurrence liée au même `clipId`. Le premier périmètre ne propose aucune commande pour rendre cette occurrence unique ou la délier.

## Cas 16 — Harmonie chromatique initiale

À sa création, un clip possède obligatoirement un `HarmonyChange` au tick `0` dont la valeur est `SCALE · C CHROMATIC`. Ce changement initial ne peut être ni supprimé ni déplacé, mais il peut être remplacé par une autre gamme ou par un accord.

Si un changement `CHORD · F MAJOR_SEVENTH` est ajouté au tick `3840` :

| Intervalle local | Harmonie active |
| --- | --- |
| `[0, 3840)` | do chromatique |
| `[3840, clip.duration)` | `Fmaj7` |

Dans la première section, toute note est `SCALE_TONE`. Dans la seconde, une note de l’accord est `CHORD_TONE` et toute autre note est `OUTSIDE_TONE`.

Aucun marqueur n’accepte `null` ou `CLEAR`. La dernière harmonie déclarée reste active jusqu’à la fin du clip.

## Cas 17 — Préchargement, seek et fermeture du piano roll

Le projet est arrêté avec une tête globale au tick `1920`. `playProject()` recense les instruments nécessaires entre ce tick et la fin du projet, demande leur préparation au moteur, puis attend leur chargement. La session `PROJECT` et l'avancement de la tête ne commencent qu'après la disponibilité de toutes les banques requises.

Pendant la lecture, `seekProject(7680)` crée une requête identifiée, prépare les instruments requis à partir du tick `7680`, puis remplace gracieusement la session par une nouvelle session `PROJECT` à ce tick lorsque les banques sont disponibles. Un `stop(IMMEDIATE)` ultérieur coupe le son mais laisse la tête globale au tick atteint ; la tête locale n'est pas modifiée.

Dans un autre scénario, `playClip()` démarre le clip `Motif`, puis l'utilisateur ferme le piano roll ou ouvre `Couplet`. La session `CLIP` continue sur `Motif` avec son propre curseur d'exécution. La tête locale du nouvel éditeur ne suit pas ce transport. Un nouvel appel à `playClip()` remplace la session et cible alors `Couplet`.

## Cas 18 — Collision de notes de même hauteur

Un clip contient une note existante `note-a`, de hauteur `C4`, vélocité `70` et intervalle `[0, 1920)`. Une note `note-m`, de même hauteur et de vélocité `100`, est manipulée jusqu'à l'intervalle quantifié `[960, 1440)`. Une note `E4` recouvre également cette zone, mais sa hauteur différente l'exclut de la collision.

Pendant le geste, `transientProject` montre et fait entendre `note-m` et `note-a` simultanément dans leur position provisoire. Aucun fragment n’est encore créé. Au relâchement, `await commitEdit()` ne modifie pas `project` et retourne `{ ok: false, error }` avec le code `NOTE_OVERLAP`. `error.details.collisions` associe `note-m` à `note-a`. Le brouillon final reste affiché et audible tandis que la présentation demande `SLICE`, `MERGE` ou l’annulation.

Avec `SLICE`, `note-m` reste inchangée et `note-a` est soustraite autour d'elle :

| Note résultante | Hauteur | Intervalle | Identité | Vélocité |
| --- | --- | --- | --- | ---: |
| Fragment gauche | `C4` | `[0, 960)` | conserve `note-a` | 70 |
| Note manipulée | `C4` | `[960, 1440)` | conserve `note-m` | 100 |
| Fragment droit | `C4` | `[1440, 1920)` | nouveau `NoteId` | 70 |

Avec `MERGE`, `note-a` est absorbée et supprimée. `note-m` devient `[0, 1920)` tout en conservant son identité et sa vélocité `100`. Dans les deux modes, la note `E4` reste intacte et le résultat complet est appliqué comme une seule transformation. Le nouveau `NoteId` du fragment droit de `SLICE` est généré seulement à cet instant. Une annulation aurait simplement supprimé `transientProject` et restauré `project` à l’écran comme dans l’audio.

Une note `C4` commençant exactement au tick `1920` serait seulement contiguë au résultat : les intervalles semi-ouverts ne déclenchent alors ni question ni résolution automatique.

## Cas 19 — Validation par `Result`

Une tentative de création de tempo à `1000.0` BPM appelle la factory du Value Object :

```ts
const result = Tempo.create(1000.0);
// {
//   ok: false,
//   error: {
//     kind: "VALIDATION_ERROR",
//     code: "TEMPO_OUT_OF_RANGE",
//     details: { received: 1000.0, min: 20.0, max: 999.9, decimals: 1 }
//   }
// }
```

Aucun `Tempo` invalide n'est construit. Le cas d'usage conserve le tempo précédent et la présentation traduit le code stable vers son propre message.

De même, déplacer une occurrence vers la ligne `128` retourne un `ProjectEditError`, car les indices valides vont de `0` à `127`. Cette intention ne remplace ni le `Project` d’origine ni la dernière projection transitoire admissible. En revanche, un déplacement valide qui superpose deux occurrences sur la même ligne est accepté sans résolution de collision. Une opération valide du domaine retourne `{ ok: true, value: updatedProject }` ; seule cette valeur peut remplacer `project`.

La lecture d'une sauvegarde suit le même chemin de validation. Une référence vers un clip absent produit une erreur métier typée, tandis qu'un échec d'accès au stockage reste une erreur technique distincte.

## Cas 20 — Changement d’instrument pendant la lecture

Un transport `PROJECT` joue deux occurrences actives du même clip `Ostinato`, actuellement associé au piano. L’utilisateur choisit un vibraphone dont la banque n’est pas encore chargée.

`EditService` ouvre une édition portant la commande de changement d’instrument. Sa préparation est suivie par `PendingEditPreparation`, liée à l’identité et à la révision de cette commande. `PlaybackService` demande au moteur :

```ts
await audioEngine.prepareInstruments([vibraphoneId]);
```

Pendant le chargement :

- `project` et `effectiveProject` conservent le piano ;
- les deux contextes actifs continuent leurs attaques avec le piano ;
- la présentation indique que le vibraphone est en préparation ;
- une annulation ou un autre choix rend cette demande obsolète.

Lorsque la banque est prête et la commande toujours courante, le service calcule un plan commun aux deux occurrences à la même borne sûre. Il ouvre leurs nouveaux contextes de vibraphone et inclut les relâchements et fins des anciens contextes dans `replaceSchedule`. Après acceptation, l’édition est publiée ; à la borne choisie, les pianos passent à `DRAINING` et les notes couvrant cette borne sont réattaquées au vibraphone. Les attaques futures sont planifiées dans les nouveaux contextes. Un refus temporel conserve l’ancien plan et provoque un nouveau calcul, sans appliquer le changement d’instrument ni relâcher prématurément les pianos.

Les releases et tails du piano peuvent donc coexister temporairement avec les nouvelles voix de vibraphone. Aucun nouveau transport n’est créé et la tête globale ne se déplace pas.

Si la préparation échoue, aucun contexte n’est remplacé et l’`InstrumentId` du clip reste celui du piano.

## Cas 21 — Réconciliation après modification d’une répétition

Une occurrence commence au tick global `0`, référence un clip de `3840` ticks et possède un `repeatCount` de `3`.

| `repeatIndex` | Intervalle initial |
| ---: | --- |
| 0 | `[0, 3840)` |
| 1 | `[3840, 7680)` |
| 2 | `[7680, 11520)` |

À la tête globale `7000`, les voix en cours appartiennent à la répétition `1`. La durée du clip est ensuite réduite à `3000` ticks :

| `repeatIndex` | Intervalle recalculé |
| ---: | --- |
| 0 | `[0, 3000)` |
| 1 | `[3000, 6000)` |
| 2 | `[6000, 9000)` |

Les voix de la répétition `1` ne sont jamais renommées en répétition `2`. Elles sont conservées uniquement si leur propre intervalle de note recalculé couvre encore la tête ; sinon elles sont relâchées. Les notes de la répétition `2` qui couvrent désormais le tick `7000` produisent de nouvelles occurrences sonores identifiées par `(occurrenceId, 2, noteId)`.

Si seul `repeatCount` passe ensuite de `3` à `2`, les répétitions `0` et `1` ne changent ni de frontière ni d’indice. La répétition terminale `2` est supprimée, ses commandes futures sont annulées et ses éventuelles voix actives sont relâchées.

Une augmentation ultérieure de `repeatCount` ajoute au contraire de nouveaux indices terminaux sans modifier ceux qui existent déjà.

## Cas 22 — Horloge sûre et fin simultanée d’un contexte

Une session est en cours à `now = 4.98` secondes. Le moteur retourne `safeAt = 5.04` : les événements antérieurs à cette borne sont déjà engagés et ne peuvent plus être remplacés de façon fiable.

Le service fournit une mise à jour contenant :

```ts
{
  from: 5.04,
  audioCommands: [
    { kind: "NOTE_OFF", contextId: oldContext, at: 6.0, occurrenceId: oldVoice },
    { kind: "NOTE_ON", contextId: newContext, at: 6.0, occurrenceId: newVoice, /* … */ }
  ],
  contextCompletions: [
    { contextId: oldContext, at: 6.0 }
  ]
}
```

À `6.0` secondes, le moteur exécute dans cet ordre :

1. le `NOTE_OFF` de `oldVoice` ;
2. la fin structurelle de `oldContext`, qui peut alors passer à `DRAINING` ;
3. le `NOTE_ON` de `newVoice` dans `newContext`.

Un `NOTE_ON` visant `oldContext` au même instant serait refusé, puisque sa fin structurelle précède les attaques. Le nouveau contexte reste en revanche indépendant et son attaque est valide.

Si une édition déplace ultérieurement cette fin et que son ancienne borne est toujours supérieure ou égale au nouveau `safeAt`, `replaceSchedule` remplace atomiquement l’ancienne `ContextCompletion`. Si cette borne est déjà engagée ou exécutée, le contexte ne peut pas être réactivé : le service ouvre un nouveau contexte.

## Cas 23 — Fin naturelle et départ aux bornes

Un projet possède une fin structurelle au tick `11520`. Lorsque sa session atteint cette borne, la position dérivée vaut exactement `11520` et est mémorisée dans la tête globale. Le service appelle `closeSession`, l’`ActiveTransport` disparaît et les contextes peuvent continuer en `DRAINING` pendant leurs tails. La session n’est libérée qu’après leur destruction.

- `playProject()` sans argument replace alors la tête à `0` et redémarre la lecture ;
- `playProject(11520)` conserve la tête à la fin et n’ouvre aucune session ;
- `playProject(11521)` retourne une erreur de validation ;
- un projet vide conserve sa tête à `0` et n’ouvre aucune session.

Le même comportement s’applique à `playClip()` et `playClip(tick)` avec `clip.duration`, à l’exception qu’un clip possède toujours une durée strictement positive.

## Cas 24 — Raccourcissement derrière la tête

Un transport `CLIP` se trouve au tick local `6000`. Une édition valide réduit `clip.duration` de `7680` à `4800` ticks.

Le service prépare la fin du plan à une borne sûre. Après acceptation, l’application publie la durée et mémorise la tête locale à `4800`. Le plan annule les attaques futures remplaçables et relâche les voix actives à la borne acceptée. L’`ActiveTransport` est supprimé ; la session est fermée à cette borne puis libérée après drainage. Les contextes possédant encore des releases ou tails passent à `DRAINING`.

Si la nouvelle durée avait été `7000`, la tête serait restée à `6000` et le transport aurait continué jusqu’à sa nouvelle fin replanifiée. Si aucun transport n’avait été actif, seul le clamp de la tête aurait été nécessaire.

## Cas 25 — Suppression du clip lu isolément

Le clip non placé `Esquisse` est ouvert dans le piano roll et joué par une session `CLIP`. Comme aucune `ClipOccurrence` ne le référence, l’utilisateur peut demander sa suppression.

Le cas d’usage produit d’abord un candidat validé, sans le publier. Au remplacement effectif du projet, il :

1. arrête gracieusement la session `CLIP` ;
2. annule ses attaques futures et relâche ses voix actives ;
3. supprime l’`ActiveTransport` ;
4. arrête les préécoutes liées à `Esquisse` ;
5. invalide ses préparations devenues sans objet ; aucune édition concurrente n’est acceptée ;
6. ferme son `ClipEditorState` et fait disparaître sa tête locale ;
7. supprime enfin le clip du projet.

Les contextes peuvent terminer leurs tails en `DRAINING`, mais la session ne continue pas à lire une copie orpheline du clip. Une tête locale n’est jamais transférée au prochain clip ouvert.

Supprimer un clip non placé pendant un transport `PROJECT` n’affecte pas ce transport, car aucune occurrence de ce clip ne participe à sa portée.

## Cas 26 — Bornes du domaine et changement au tick final

Le premier périmètre accepte notamment :

- la ligne `127`, mais refuse la ligne `128` ;
- les hauteurs MIDI `0` et `127`, mais refuse `-1` et `128` ;
- les vélocités `1` et `127`, mais refuse `0` et `128` ;
- une métrique `32/64`, mais refuse `0/4`, `33/4` et `4/3` ;
- `FLAT`, `NATURAL` et `SHARP`, mais pas encore les doubles altérations ;
- un `repeatCount` de `65_535`, sous réserve que la fin globale calculée ne dépasse pas `MAX_TICK`.

Une occurrence est refusée même lorsque ses valeurs individuelles sont valides si le calcul suivant dépasse `2_147_483_647` :

```text
occurrence.start + clip.duration * occurrence.repeatCount
```

Un clip se termine au tick `7680`. L’utilisateur ajoute un `HarmonyChange` exactement à cette position. Le changement est valide et sauvegardé, mais sa section vaut `[7680, 7680)` : elle n’affecte aucune note et ne produit aucun événement audio.

Si le clip est ensuite allongé jusqu’au tick `11520`, ce même changement devient le début de la section `[7680, 11520)` sans être déplacé ni recréé.

Un `MeterChange` placé à la fin suit la même règle. À l’inverse, raccourcir le clip en dessous d’un changement existant est refusé, sauf si le même geste déplace ou supprime également ce changement.

## Cas 27 — Préparation remplacée et projet modifié

Le projet est arrêté au tick `1920`. Un premier `playProject()` crée la requête `request-a` et commence à préparer le piano nécessaire à cette portée.

Avant la fin du chargement, l’utilisateur appelle `playProject(7680)`. Le service crée `request-b`, rend `request-a` obsolète et résout sa promesse avec `ok("SUPERSEDED")`. Même si le piano termine ensuite son chargement pour `request-a`, cette ancienne requête ne peut ouvrir aucune session.

Pendant la préparation de `request-b`, l’utilisateur modifie `effectiveProject` et ajoute après le tick `7680` une occurrence utilisant un vibraphone. `effectiveProjectRevision` change. Lorsque la préparation courante se termine, le service détecte cette différence, recalcule la portée, réutilise le piano déjà prêt et prépare en plus le vibraphone. Il ne planifie la session qu’après un nouveau contrôle sur la dernière révision.

Si `stop()` intervient pendant cette seconde préparation :

- `request-b` se résout avec `ok("CANCELLED")` ;
- aucun résultat tardif ne peut démarrer le transport ;
- l’éventuel transport déjà actif est arrêté selon le `StopMode` demandé.

Dans une variante avec un transport déjà actif, `seekProject(7680)` laisse la tête sonore continuer à avancer et affiche le tick `7680` comme destination provisoire en chargement. La tête effective ne saute à cette position qu’au remplacement effectif de la session. Un échec de banque retourne `err(InstrumentPreparationError)` et conserve l’ancien transport.

Les préécoutes suivent une forme différente parce que leur contrôle doit être immédiat : `previewPitch` ou `previewSelection` retourne synchroniquement un `Result` contenant éventuellement un handle, tandis que `handle.ready` porte l’issue asynchrone du chargement.

## Cas 28 — Intention unique, préparation et annulation

Un geste déplace une note et un changement harmonique. `EditService.beginEdit` capture le projet validé dans `EditSession.baseProject`. Chaque `updateEdit` remplace le delta total de la commande ; les transformations sont recalculées depuis cette base avec les mêmes fonctions pures que la validation finale.

Un delta qui placerait une note avant `0` est refusé sans changer la dernière projection admissible. Un delta qui crée seulement une collision de même hauteur peut être prévisualisé ; au commit, `NOTE_OVERLAP` conserve le brouillon et fige la commande jusqu’au choix de résolution. Une seconde édition ou `undo()` pendant cette attente retourne `EDIT_IN_PROGRESS`.

Le choix `SLICE` produit un seul nouveau projet et une seule entrée d’historique. Une annulation aurait supprimé le brouillon sans rien inscrire. Si une préparation audio était en cours, `cancelEdit` l’aurait rendue obsolète ; sa réponse tardive ne pourrait ni modifier le projet ni démarrer des notes.

## Cas 29 — Placement d’un clip dont la banque n’est pas prête

Le transport `PROJECT` lit les occurrences d’un piano. Un clip de vibraphone existe dans le document sans occurrence ; sa banque n’a donc pas été requise au démarrage. L’utilisateur le place sur la ligne `0`, dans la portée encore à lire, éventuellement par-dessus une occurrence de piano.

La superposition est valide. La banque manquante déclenche toutefois `PendingEditPreparation` avant publication de cette nouvelle projection. L’ancien projet effectif continue de jouer ; le placement demandé dispose d’un repère en chargement. Aucun `NOTE_ON` de vibraphone n’est envoyé prématurément.

À la fin du chargement, le service vérifie la commande et la portée actuelles, recalcule depuis la nouvelle borne sûre et publie après acceptation du plan. Les notes du clip qui couvrent cette borne sont poursuivies par une attaque minimale ; celles déjà entièrement passées ne sont pas rejouées. Annuler ou modifier le geste rend la préparation précédente obsolète. Un échec conserve l’ancien projet effectif et n’ajoute rien à l’historique.

## Cas 30 — Note terminée avant la borne de réconciliation

À `now = 4.98`, une note joue encore, mais son `NOTE_OFF` à `5.00` est déjà engagé. Le moteur annonce `safeAt = 5.04`. L’utilisateur allonge la note jusqu’à `6.00`.

L’ancienne voix n’est plus active à `5.04`. Le service ne prétend donc pas prolonger son arrêt déjà engagé : le nouvel intervalle couvrant cette borne produit une nouvelle attaque à `5.04`, avec un nouveau `NoteOccurrenceId`, puis un arrêt à `6.00`.

Si le calcul arrive trop tard et que le moteur exige désormais `from >= 5.08`, `replaceSchedule` retourne `SCHEDULE_TOO_LATE` sans modification partielle. Le service recalcule l’état des voix et les intersections à `5.08` ; il ne se contente pas de décaler les commandes du plan refusé.

## Cas 31 — Silence sans destruction du transport

Une occurrence se termine à deux secondes et la suivante commence à dix secondes. La planification glissante n’a pas encore ouvert le contexte de la seconde. Après extinction des tails, la session peut n’avoir aucun contexte vivant entre ces deux blocs.

Elle reste ouverte : son horloge continue d’avancer et le service peut programmer le contexte suivant. Seule la fin structurelle du projet ou un arrêt ferme cette session. Sa destruction exige alors que tous ses contextes soient également détruits.

## Cas 32 — Historique et fichier de projet

Une transposition collective puis un déplacement d’occurrence produisent deux versions validées. `undo()` restaure la version avant le déplacement et réconcilie la lecture ; un second undo restaure les notes avant transposition. `redo()` avance dans ces mêmes versions, sans recréer leurs identifiants. Les têtes de transport ne reviennent pas à leur position historique.

Une sauvegarde pendant un nouveau geste écrit seulement le dernier projet validé, dans une enveloppe `schemaVersion: 1`. Le brouillon, les têtes et l’historique en sont absents. Une édition validée pendant cette écriture demeure une modification non sauvegardée.

Rouvrir ce fichier reconstitue et valide les clips ainsi que leurs références. Si tous les instruments sont connus, le remplacement ferme les anciennes auditions, remet la tête globale à `0` et vide l’historique. Un instrument absent, un format invalide ou une version non prise en charge laisse au contraire le document courant intact.

## Cas 33 — Durée partagée et superpositions

Un clip `Motif` de `1920` ticks possède deux occurrences sur la ligne `0`, commençant respectivement à `0` et `1920`, avec `repeatCount = 1`. Allonger le clip à `2880` ticks produit les intervalles `[0, 2880)` et `[1920, 4800)`.

Cette édition est valide : les deux occurrences se superposent pendant `960` ticks, conservent leurs débuts et jouent simultanément dans des contextes indépendants. Aucun déplacement en cascade ni choix `SLICE`/`MERGE` n’est demandé. Ces modes concernent exclusivement les collisions de notes de même hauteur à l’intérieur d’un clip.

Changer ensuite la métrique de `4/4` en `3/4` conserve cette durée de `2880` ticks. Elle représente désormais une mesure complète ; ni les notes ni les placements ne bougent. Le résultat est identique si le clip est vide : seule une commande explicite change sa durée.

## Référence des contrats

Les règles communes, les signatures des services et ports et les questions encore ouvertes sont centralisées dans [architecture.md](architecture.md). Les cas ci-dessus illustrent ces règles sans constituer une seconde spécification.

