# Etudes de cas

Ce document illustre les règles de composition et de lecture définies dans [architecture.md](architecture.md). Les exemples ne décrivent aucune position globale sauvegardée : tous les instants de départ et de fin sont dérivés de l'arbre de `Group` et de `Clip`.

## Conventions de calcul

Chaque clip utilise sa propre chronologie de tempo. Pour un intervalle dont le tempo est constant :

```text
durationSeconds = (durationTicks / 960) * (60 / bpm)
```

Lorsque le tempo change dans le clip, sa duree reelle correspond a la somme des durees de ses `TempoSection`.

Chaque cas présente une projection chronologique Mermaid. Les diagrammes utilisent des intervalles proportionnels lorsque les durées sont connues. Lorsqu'un scénario décrit surtout des états d'exécution sans fournir de durée, le diagramme représente leur ordre sans imposer d'échelle temporelle.

La duree de lecture d'un element suit les regles suivantes :

```text
clip contourne                    = 0
clip                              = duree d'une lecture * repeatCount
groupe SEQUENTIAL                 = somme des durees de ses enfants
groupe SIMULTANEOUS               = maximum des durees de ses enfants
groupe vide                       = 0
```

## Cas 1 - Sequence, repetition et bypass

Le groupe racine contient trois clips et utilise le mode `SEQUENTIAL`.

```mermaid
flowchart TD
    Root["RootGroup - SEQUENTIAL"] --> A["Ouverture"]
    Root --> B["Transition - bypass"]
    Root --> C["Motif - repeatCount 3"]
```

| Clip | Duree d'une lecture | Etat | Contribution |
| --- | ---: | --- | ---: |
| `Ouverture` | 2 s | actif, `repeatCount = 1` | 2 s |
| `Transition` | 2 s | `isBypassed = true` | 0 s |
| `Motif` | 1 s | actif, `repeatCount = 3` | 3 s |

La lecture obtenue est la suivante :

| Temps reel | Lecture |
| --- | --- |
| 0 a 2 s | `Ouverture` |
| 2 a 3 s | premiere lecture de `Motif` |
| 3 a 4 s | deuxieme lecture de `Motif` |
| 4 a 5 s | troisieme lecture de `Motif` |

Le clip `Transition` reste dans l'arbre. Son `repeatCount` est conserve, mais il ne produit aucun son et ne retarde pas le clip suivant tant qu'il est contourne.


### Chronologie dérivée

```mermaid
flowchart LR
    Ouverture["0–2 s · Ouverture"] --> Transition["2 s · Transition bypassée · 0 s"]
    Transition --> Motif1["2–3 s · Motif 1"]
    Motif1 --> Motif2["3–4 s · Motif 2"]
    Motif2 --> Motif3["4–5 s · Motif 3"]
```

## Cas 2 - Deux clips simultanes aux horloges independantes

Le groupe racine utilise le mode `SIMULTANEOUS`.

```mermaid
flowchart TD
    Root["RootGroup - SIMULTANEOUS"] --> Rythme["Rythme"]
    Root --> Basse["Ligne de basse"]
```

| Clip | Duree | Metrique | Tempo | Duree reelle |
| --- | ---: | ---: | ---: | ---: |
| `Rythme` | 3840 ticks | 4/4 | 120 BPM | 2 s |
| `Ligne de basse` | 5760 ticks | 3/4 | 90 BPM | 4 s |

Les deux clips commencent a l'instant zero. `Rythme` se termine apres deux secondes, tandis que `Ligne de basse` continue jusqu'a quatre secondes. Le groupe dure donc quatre secondes.

La metrique 3/4 et le tempo de 90 BPM de la basse ne modifient ni les reperes ni la chronologie du rythme en 4/4 a 120 BPM. Les clips partagent uniquement leur instant de depart reel.


### Chronologie dérivée

```mermaid
block-beta
    columns 2
    t0["0–2 s"] t1["2–4 s"]
    rythme["Rythme"] space
    basse["Ligne de basse"]:2
```

## Cas 3 - Imbrication d'une sequence dans une superposition

La composition comporte une introduction, un ensemble simultane, puis une conclusion. A l'interieur de l'ensemble, deux clips rythmiques sont lus en sequence pendant qu'une ligne de basse suit sa propre chronologie.

```mermaid
flowchart TD
    Root["RootGroup - SEQUENTIAL"] --> Intro["Introduction"]
    Root --> Ensemble["Ensemble - SIMULTANEOUS"]
    Root --> Conclusion["Conclusion"]
    Ensemble --> Rythme["Rythme - SEQUENTIAL"]
    Ensemble --> Basse["Ligne de basse"]
    Rythme --> GrooveA["Groove A"]
    Rythme --> GrooveB["Groove B"]
```

| Clip | Duree | Metrique | Tempo | Duree reelle |
| --- | ---: | ---: | ---: | ---: |
| `Introduction` | 3840 ticks | 4/4 | 120 BPM | 2 s |
| `Groove A` | 3840 ticks | 4/4 | 120 BPM | 2 s |
| `Groove B` | 3840 ticks | 4/4 | 120 BPM | 2 s |
| `Ligne de basse` | 5760 ticks | 3/4 | 90 BPM | 4 s |
| `Conclusion` | 2880 ticks | 3/4 | 90 BPM | 2 s |

Les nombres de ticks ne sont comparables qu'apres conversion par le tempo propre a chaque clip. Les deux grooves totalisent bien davantage de ticks que la basse, mais leur tempo plus rapide compense exactement cette difference :

```text
dureeRythme = ((3840 + 3840) / 960) * (60 / 120) = 4 s
dureeBasse  = (5760 / 960) * (60 / 90)            = 4 s
```

Les deux branches de l'ensemble ont donc la meme duree reelle, malgre leurs nombres de ticks differents.

La lecture se deroule comme suit :

| Temps reel | Lecture |
| --- | --- |
| 0 a 2 s | `Introduction` |
| 2 a 4 s | `Groove A` et premiere partie de `Ligne de basse` |
| 4 a 6 s | `Groove B` et seconde partie de `Ligne de basse` |
| 6 a 8 s | `Conclusion` |

Le groupe `Rythme` dure quatre secondes, car il additionne deux clips de deux secondes. Le groupe `Ensemble` dure egalement quatre secondes : ses deux enfants commencent a deux secondes et se terminent a six secondes.

Le debut de `Conclusion` a six secondes est entierement derive du parcours de l'arbre : deux secondes pour l'introduction, puis quatre secondes pour l'enfant le plus long du groupe simultane.


### Chronologie dérivée

```mermaid
block-beta
    columns 4
    t0["0–2 s"] t1["2–4 s"] t2["4–6 s"] t3["6–8 s"]
    intro["Introduction"] grooveA["Groove A"] grooveB["Groove B"] conclusion["Conclusion"]
    space basse["Ligne de basse"]:2 space
```

## Cas 4 - Mute et solo par instrument

Deux clips simultanes utilisent plusieurs instruments. Des notes de piano apparaissent dans les deux clips.

```mermaid
flowchart TD
    Root["RootGroup - SIMULTANEOUS"] --> A["Clip A"]
    Root --> B["Clip B"]
    A --> AP["Piano"]
    A --> AR["Rythme"]
    B --> BP["Piano"]
    B --> BB["Basse"]
```

Lorsque `piano` est mute, ses notes sont silencieuses dans les deux clips. Les notes de rythme et de basse restent audibles. Lorsque `piano` est le seul instrument solo, ses notes restent audibles dans les deux clips et les autres instruments sont silencieux.

Dans les deux cas, les clips conservent leurs positions, leurs durees et leurs contextes. Le groupe se termine donc au meme instant qu'en l'absence de mute ou de solo. Ces reglages appartiennent au projet et ciblent l'`InstrumentId`, non un clip ou une instance particuliere.

Si `piano` est a la fois mute et solo, le mute est prioritaire. Une preecoute de note utilisant cet instrument applique la meme regle d'audibilite.


### Chronologie dérivée

```mermaid
block-beta
    columns 1
    periode["Début → fin structurelle inchangée"]
    clipA["Clip A · rythme audible · piano filtré"]
    clipB["Clip B · basse audible · piano filtré"]
```

Cette projection illustre le cas où `piano` est muté. Un solo utilise exactement la même géométrie et change seulement les instruments audibles.

## Cas 5 - Deux clips utilisent le meme instrument

Deux clips lus simultanement contiennent chacun une note de meme hauteur jouee par le meme instrument.

```mermaid
flowchart TD
    Root["RootGroup - SIMULTANEOUS"] --> A["Clip A"]
    Root --> B["Clip B"]
    A --> NA["Note A - piano, do"]
    B --> NB["Note B - piano, do"]
```

La note A commence a zero seconde et se termine a quatre secondes. La note B commence a une seconde et se termine a deux secondes. Le `PlaybackService` produit deux occurrences distinctes :

| Occurrence | Source | Instrument | Hauteur | Debut | Fin |
| --- | --- | --- | --- | ---: | ---: |
| `occurrence-a` | note A du clip A | piano | do | 0 s | 4 s |
| `occurrence-b` | note B du clip B | piano | do | 1 s | 2 s |

A deux secondes, le moteur relache uniquement `occurrence-b`. La voix correspondant a `occurrence-a` continue jusqu'a quatre secondes. Une commande de relachement identifiee seulement par l'instrument et la hauteur serait insuffisante, car elle risquerait d'interrompre les deux voix.

Les deux notes ne sont jamais fusionnees implicitement. Chaque activation de clip possede son propre `PlaybackContext` et sa propre `InstrumentInstance` du piano. La politique `InstrumentDefinition.voiceAllocation` s'applique donc separement dans chaque instance.

Meme si le piano est monophonique, la note du clip B n'interrompt pas celle du clip A. La monophonie limite les notes concurrentes a l'interieur d'un meme contexte ; elle n'est pas globale a tous les clips utilisant le meme `InstrumentId`.


### Chronologie dérivée

```mermaid
block-beta
    columns 4
    t0["0–1 s"] t1["1–2 s"] t2["2–3 s"] t3["3–4 s"]
    noteA["Note A · piano"]:4
    space noteB["Note B · piano"] space:2
```

## Cas 6 - Remplacement du transport et preecoute concurrente

Le clip `Motif` est actif dans une session `PROJECT` lorsque l'utilisateur relance la lecture avec `play(motif.id)`.

Le nouvel appel place la tete au debut global derive de `Motif` et ouvre une nouvelle session `PROJECT`. Le `PlaybackService` retire immediatement le role de transport a `project-session-a`, annule ses attaques futures, relache ses occurrences actives selon le mode `GRACEFUL` et ouvre `project-session-b`. Il n'existe jamais deux transports actifs.

| Etape | Session | Type | Etat |
| --- | --- | --- | --- |
| Lecture initiale | `project-session-a` | `PROJECT` | transport actif |
| Nouvel appel a `play` | `project-session-a` | `PROJECT` | inactive ; contextes eventuellement `DRAINING` |
| Nouvel appel a `play` | `project-session-b` | `PROJECT` | nouveau transport actif |

Les deux activations successives du meme `ClipId` recoivent des `PlaybackContextId` differents. L'ancienne session reste inactive tandis que ses contextes passent eventuellement en `DRAINING` : leurs releases peuvent rester audibles pendant le debut de la nouvelle session, mais aucune nouvelle attaque n'est planifiee. La session est detruite lorsque tous ses contextes sont `DISPOSED`. Avec un arret `IMMEDIATE`, ses contextes sont detruits sans delai.

Pendant ce transport, une note peut etre preecoutee independamment :

```ts
const notePreviewSession = preview(noteId);
```

Cette operation ouvre une session `NOTE_PREVIEW` sans remplacer `project-session-b`, puis lui rattache un contexte possedant son propre `PlaybackContextId`. La commande `NOTE_ON` porte ce `contextId` ainsi que l'`InstrumentId` resolu par le `PlaybackService`, mais aucun identifiant de source persistante ne traverse le port audio. Plusieurs preecoutes de notes peuvent se chevaucher, et arreter `notePreviewSession` n'affecte pas le transport actif. Le mute et le solo de l'instrument restent applicables.


### Chronologie dérivée

```mermaid
flowchart LR
    A["project-session-a · ACTIVE"] --> Commande["play(motif.id)"]
    Commande --> Remplacement["session-a · DRAINING · session-b · ACTIVE"]
    Remplacement --> Fin["session-a · DISPOSED · session-b poursuit"]
```

Les sessions `NOTE_PREVIEW` peuvent apparaître pendant la troisième phase sans modifier cette succession du transport.

## Cas 7 - Repetitions et occurrences de notes

Un clip contient une note `note-a` et possede `repeatCount = 3`. Les trois lectures reutilisent le meme `PlaybackContextId` et la meme `InstrumentInstance`, mais elles produisent trois occurrences distinctes.

| Repetition | Note persistante | Occurrence d'execution |
| ---: | --- | --- |
| 1 | `note-a` | `occurrence-a-1` |
| 2 | `note-a` | `occurrence-a-2` |
| 3 | `note-a` | `occurrence-a-3` |

Chaque `NoteOff` cible son `NoteOccurrenceId`. Une release de la premiere repetition peut donc continuer au debut de la deuxieme. Si l'instrument est monophonique, la nouvelle attaque peut appliquer sa politique de retrigger ou de vol de voix a l'occurrence precedente, puisqu'elles appartiennent a la meme instance.

Si une voix a deja ete volee, le `NoteOff` programme pour son ancienne occurrence devient une operation sans effet. Le moteur doit donc traiter les relachements comme des commandes idempotentes.


### Chronologie dérivée

```mermaid
flowchart LR
    R1["Répétition 1 · occurrence-a-1"] --> R2["Répétition 2 · occurrence-a-2"]
    R2 --> R3["Répétition 3 · occurrence-a-3"]
```

Les trois phases utilisent le même `PlaybackContextId`, mais chaque attaque reçoit son propre `NoteOccurrenceId`.

## Cas 8 - Fin structurelle et tail audio

Un clip `Nappe` possede une duree structurelle de deux secondes, mais son instrument produit une release et une reverberation qui restent audibles une seconde supplementaire. `Nappe` est suivi du clip `Conclusion` dans un groupe sequentiel.

| Temps reel | Evenement structurel | Etat audio de `Nappe` |
| --- | --- | --- |
| 0 s | debut de `Nappe` | `ACTIVE` |
| 2 s | fin de `Nappe`, debut de `Conclusion` | `DRAINING` |
| 3 s | aucune modification de la structure | `DISPOSED` apres extinction du tail |

La fin structurelle, calculee a partir des ticks et du tempo, determine le depart de `Conclusion`. Le tail ne rallonge donc pas le groupe et peut se superposer au clip suivant. Le contexte de `Nappe` refuse toute nouvelle attaque apres deux secondes, mais conserve ses instances jusqu'au silence ou jusqu'a une duree maximale de securite.


### Chronologie dérivée

```mermaid
flowchart LR
    Debut["0 s · Nappe ACTIVE"] --> Passage["2 s · Conclusion démarre · Nappe DRAINING"]
    Passage --> Drainage["3 s · Nappe DISPOSED · Conclusion indépendante"]
```

## Cas 9 - Lecture globale depuis un noeud

Le groupe racine est sequentiel. Son deuxieme enfant, `Ensemble`, est un groupe simultane.

```mermaid
flowchart TD
    Root["RootGroup - SEQUENTIAL"] --> Intro["Introduction"]
    Root --> Ensemble["Ensemble - SIMULTANEOUS"]
    Root --> Conclusion["Conclusion"]
    Ensemble --> Piano["Piano"]
    Ensemble --> Basse["Basse"]
```

`Piano`, `Basse` et leur groupe parent possedent le meme instant de depart global derive. Les commandes suivantes expriment les positions choisies sans changer la portee du transport :

| Commande | Resultat |
| --- | --- |
| `play()`, tete au debut | `Introduction`, puis `Piano` et `Basse` ensemble, puis `Conclusion`. |
| `play(rootGroup.id)` | Replace la tete au debut et produit le meme parcours complet. |
| `play(ensemble.id)` | Place la tete au debut d'`Ensemble`, lit `Piano` et `Basse` ensemble, puis `Conclusion`. |
| `play(piano.id)` | Meme position globale et meme resultat que `play(ensemble.id)`. |
| `play(basse.id)` | Meme position globale et meme resultat que `play(ensemble.id)`. |
| `play(conclusion.id)` | Place la tete au debut de `Conclusion` et lit la fin du projet. |

Dans l'interface, chaque `Group` et chaque `Clip` peut donc presenter un bouton `play`. Une note possede uniquement une action `preview(noteId)`, qui ne deplace pas la tete et ne modifie pas le transport.

L'identifiant passe a `play` sert uniquement a resoudre une position. La racine de la session reste toujours le projet et aucune frontiere de preecoute structurelle n'est creee.

Dans une structure simultanee plus complexe, un element peut commencer alors qu'une autre branche est deja en cours. Demarrer depuis cet element reprend toutes les branches actives a cette position globale. Le traitement des notes commencees avant cette position est laisse ouvert.


### Chronologie dérivée

```mermaid
block-beta
    columns 3
    phase1["Phase initiale"] phase2["Position partagée"] phase3["Phase finale"]
    intro["Introduction"] piano["Piano"] conclusion["Conclusion"]
    space basse["Basse"] space
```

`play(piano.id)`, `play(basse.id)` et `play(ensemble.id)` placent tous la tête dans la colonne centrale.

## Consequences pour le PlaybackService

Le calcul structurel peut etre interprete par une operation recursive :

```ts
calculateDuration(item: GroupItem): number
```

La projection temporelle associe ensuite chaque element a un intervalle global derive et chaque clip actif a une activation. Elle peut etre planifiee par fenetres pour limiter le volume de commandes preparees, meme si toutes les durees sont finies.

- un `Clip` calcule ses evenements depuis son instant de depart en utilisant ses propres `TempoSection` et possede une fin structurelle ;
- un groupe `SEQUENTIAL` transmet la fin de chaque enfant comme debut du suivant ;
- un groupe `SIMULTANEOUS` transmet le meme debut a tous ses enfants et retourne la fin la plus tardive ;
- `repeatCount` est toujours un entier strictement positif, de sorte que toutes les positions et durees derivees sont finies ;
- `play()` commence a la position actuelle de la tete de lecture ;
- `play(itemId)` place la tete au debut global derive du groupe ou du clip, puis lit le projet depuis cette position ;
- `play(rootGroup.id)` constitue le redemarrage explicite depuis le debut ;
- tout groupe ou clip peut servir de repere, y compris sous un ancetre `SIMULTANEOUS` ;
- toutes les branches actives a la position choisie participent au transport ;
- `preview(noteId)` auditionne uniquement une note, sans deplacer la tete ni remplacer le transport ;
- chaque lecture globale ouvre une session `PROJECT` transitoire ;
- une seule session `PROJECT` peut constituer le transport actif ; une nouvelle lecture remplace la precedente, qui ne subsiste que comme proprietaire de contextes eventuellement `DRAINING` ;
- les sessions `NOTE_PREVIEW` peuvent coexister entre elles et avec le transport actif ;
- les mute et solo persistants filtrent les notes par `InstrumentId` dans tous les contextes, sans modifier la timeline ;
- un clip bypassé avant son activation ne contribue pas a la duree ; s'il est bypassé pendant une iteration, celle-ci se termine et aucune repetition supplementaire n'est lancee ;
- `stop(sessionId, mode)` arrete une session precise, tandis que `stopTransport(mode)` n'arrete que le transport actif ;
- un arret `GRACEFUL` relache les occurrences actives et conserve leurs tails, tandis qu'un arret `IMMEDIATE` detruit les contextes sans delai ;
- chaque activation de clip et chaque preecoute de note recoivent un `PlaybackContextId` transitoire distinct des identifiants persistants ;
- chaque `AudioCommand` porte ce `contextId`, tandis que la relation entre contexte et session n'est enregistree qu'une fois, a l'ouverture du contexte ;
- chaque attaque, y compris lors d'une repetition, recoit un `NoteOccurrenceId` unique ;
- la fin structurelle permet au parcours de continuer pendant que l'infrastructure conserve eventuellement le contexte en `DRAINING` ;
- aucune position globale ni aucun identifiant d'execution n'est ajoute au modele sauvegarde.
