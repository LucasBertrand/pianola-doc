# Etudes de cas

Ce document illustre les regles de composition et de lecture definies dans [domaines.md](domaines.md). Les exemples ne decrivent aucune position globale sauvegardee : tous les instants de depart et de fin sont derives de l'arbre de `Group` et de `Clip`.

## Conventions de calcul

Chaque clip utilise sa propre chronologie de tempo. Pour un intervalle dont le tempo est constant :

```text
durationSeconds = (durationTicks / 960) * (60 / bpm)
```

Lorsque le tempo change dans le clip, sa duree reelle correspond a la somme des durees de ses `TempoSection`.

La duree de lecture d'un element suit les regles suivantes :

```text
clip contourne                    = 0
clip fini                         = duree d'une lecture * repeatCount
clip avec repeatCount = infinite  = infinite
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

La lecture se deroule comme suit :

| Temps reel | Lecture |
| --- | --- |
| 0 a 2 s | `Introduction` |
| 2 a 4 s | `Groove A` et premiere partie de `Ligne de basse` |
| 4 a 6 s | `Groove B` et seconde partie de `Ligne de basse` |
| 6 a 8 s | `Conclusion` |

Le groupe `Rythme` dure quatre secondes, car il additionne deux clips de deux secondes. Le groupe `Ensemble` dure egalement quatre secondes : ses deux enfants commencent a deux secondes et se terminent a six secondes.

Le debut de `Conclusion` a six secondes est entierement derive du parcours de l'arbre : deux secondes pour l'introduction, puis quatre secondes pour l'enfant le plus long du groupe simultane.

## Cas 4 - Repetition infinie dans une branche simultanee

Une nappe repetee indefiniment joue en meme temps qu'une sequence rythmique finie. Une conclusion est placee apres leur groupe parent.

```mermaid
flowchart TD
    Root["RootGroup - SEQUENTIAL"] --> Intro["Introduction"]
    Root --> Ensemble["Ensemble - SIMULTANEOUS"]
    Root --> Conclusion["Conclusion"]
    Ensemble --> Nappe["Nappe - repeatCount infinite"]
    Ensemble --> Rythme["Rythme fini"]
```

Apres l'introduction, la nappe et le rythme commencent ensemble. Le rythme peut se terminer, mais la branche de la nappe n'a jamais d'instant de fin. La duree du groupe `Ensemble` est donc `infinite` et la `Conclusion` est inaccessible par progression automatique.

Si la nappe est contournee, sa contribution devient nulle. Le groupe se termine alors avec le rythme fini et la lecture peut atteindre la conclusion. Arreter la lecture ou deplacer manuellement la tete de lecture permet egalement de quitter la repetition infinie.


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

## Cas 6 - Lecture et preecoute simultanees du meme clip

Le clip `Motif` est deja actif dans la lecture du projet lorsque l'utilisateur en lance une preecoute.

| Operation | Session | Contexte | Source persistante |
| --- | --- | --- | --- |
| Lecture du projet | `project-session` | `clip-playback-a` | `ClipId = motif` |
| Preecoute | `preview-session` | `clip-playback-b` | `ClipId = motif` |

Les deux descripteurs sont strictement de type `CLIP` et portent le meme `ClipId`, mais des `ClipPlaybackId` differents. Chaque contexte cree ses propres instances. Arreter la preecoute detruit seulement `clip-playback-b` et ne relache aucune voix de `clip-playback-a`.

Une preecoute de note isolee utilise au contraire un descripteur de type `NOTE_PREVIEW`. Elle porte un `NotePreviewPlaybackId` et un `InstrumentId`, mais aucun `ClipId` optionnel : elle ne pretend pas etre la lecture d'un clip.

## Cas 7 - Repetitions et occurrences de notes

Un clip contient une note `note-a` et possede `repeatCount = 3`. Les trois lectures reutilisent le meme `ClipPlaybackId` et la meme `InstrumentInstance`, mais elles produisent trois occurrences distinctes.

| Repetition | Note persistante | Occurrence d'execution |
| ---: | --- | --- |
| 1 | `note-a` | `occurrence-a-1` |
| 2 | `note-a` | `occurrence-a-2` |
| 3 | `note-a` | `occurrence-a-3` |

Chaque `NoteOff` cible son `NoteOccurrenceId`. Une release de la premiere repetition peut donc continuer au debut de la deuxieme. Si l'instrument est monophonique, la nouvelle attaque peut appliquer sa politique de retrigger ou de vol de voix a l'occurrence precedente, puisqu'elles appartiennent a la meme instance.

Si une voix a deja ete volee, le `NoteOff` programme pour son ancienne occurrence devient une operation sans effet. Le moteur doit donc traiter les relachements comme des commandes idempotentes.

## Cas 8 - Fin structurelle et tail audio

Un clip `Nappe` possede une duree structurelle de deux secondes, mais son instrument produit une release et une reverberation qui restent audibles une seconde supplementaire. `Nappe` est suivi du clip `Conclusion` dans un groupe sequentiel.

| Temps reel | Evenement structurel | Etat audio de `Nappe` |
| --- | --- | --- |
| 0 s | debut de `Nappe` | `ACTIVE` |
| 2 s | fin de `Nappe`, debut de `Conclusion` | `DRAINING` |
| 3 s | aucune modification de la structure | `DISPOSED` apres extinction du tail |

La fin structurelle, calculee a partir des ticks et du tempo, determine le depart de `Conclusion`. Le tail ne rallonge donc pas le groupe et peut se superposer au clip suivant. Le contexte de `Nappe` refuse toute nouvelle attaque apres deux secondes, mais conserve ses instances jusqu'au silence ou jusqu'a une duree maximale de securite.

## Consequences pour le PlaybackService

Le calcul structurel peut etre interprete par une operation recursive :

```ts
calculateDuration(item: GroupItem): number
```

Une portion finie peut ensuite etre planifiee a partir d'un instant de depart. La planification effective utilise une fenetre d'anticipation bornee afin de ne jamais tenter de developper entierement une repetition infinie.

- un `Clip` calcule ses evenements depuis son instant de depart en utilisant ses propres `TempoSection` et possede une fin structurelle ;
- un groupe `SEQUENTIAL` transmet la fin de chaque enfant comme debut du suivant ;
- un groupe `SIMULTANEOUS` transmet le meme debut a tous ses enfants et retourne la fin la plus tardive ;
- une duree `infinite` se propage aux groupes ancetres selon les memes regles ;
- chaque operation globale ouvre une `PlaybackSession` transitoire ;
- chaque activation de clip recoit un `ClipPlaybackId` distinct de son `ClipId` ;
- chaque attaque, y compris lors d'une repetition, recoit un `NoteOccurrenceId` unique ;
- la fin structurelle permet au parcours de continuer pendant que l'infrastructure conserve eventuellement le contexte en `DRAINING` ;
- aucun de ces calculs ni identifiants d'execution n'ajoute de position temporelle globale ou d'etat audio au modele sauvegarde.
