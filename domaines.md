# Domaines

Ce document recense les premiers objets fondamentaux de Pianola, une application de piano roll avec instruments modulaires.

L'objectif est de poser un vocabulaire metier stable et de rendre visibles les frontieres architecturales avant de penser stockage, Web Audio API ou implementation React.

## Navigation

- [Vision generale](#vision-generale)
- [Decisions actees](#decisions-actees)
- [Domaine de composition](#domaine-de-composition)
  - [Entities](#entities)
  - [Value Objects de la composition](#value-objects-de-la-composition)
  - [Agregats](#agregats)
- [Etat applicatif de l'editeur](#etat-applicatif-de-lediteur)
- [Couche applicative et lecture](#couche-applicative-et-lecture)
- [Infrastructure audio](#infrastructure-audio)
- [Relations architecturales](#relations-architecturales)
- [Arborescence cible](#arborescence-cible)
- [Questions ouvertes](#questions-ouvertes)
- [Principes directeurs](#principes-directeurs)

## Vision generale

L'application est centree sur un domaine de composition. Le `Project` y organise une sequence de clips necessairement consecutifs.

Le projet exprime des intentions musicales sous forme de clips et de notes. Chaque note reference l'instrument qui doit l'interpreter. Le projet ne contient ni les patchs des instruments, ni l'etat d'execution du moteur audio, ni l'etat transitoire de l'editeur.

Il n'existe pas de vue d'arrangement multipiste dans laquelle des clips seraient places librement sur plusieurs pistes instrumentales. Les moyens necessaires a l'ecoute sont places autour du domaine :

- une couche applicative qui interprete la sequence de clips ;
- des ports qui definissent ce dont l'application a besoin pour produire du son ;
- une infrastructure audio qui contient le catalogue d'instruments integre, leurs patchs et le moteur audio.

```mermaid
flowchart LR
    Domain["Domaine de composition"] --> App["Application et lecture"]
    Editor["Etat de l'editeur"] --> App
    App --> Ports["Ports audio"]
    Ports --> Infra["Infrastructure audio"]
```

## Decisions actees

- Le `Project` organise directement une sequence ordonnee de clips.
- Les clips sont necessairement consecutifs : ils ne possedent pas de position dans une timeline globale et leur ordre determine l'ordre de lecture.
- Un clip peut etre contourne pendant la lecture. Cet etat est conserve dans le clip afin que la sauvegarde preserve la structure courante de la sequence.
- Chaque clip possede un `repeatCount`, independant du bypass, qui indique son nombre total de lectures. Sa valeur est un entier strictement positif ou `infinite`.
- Chaque `Clip` porte ses propres chronologies locales de tempo, de metrique et de contexte de hauteurs, positionnees en ticks.
- Les chronologies de tempo et de metrique possedent obligatoirement un changement initial au tick `0` ; ils remplacent les anciennes proprietes scalaires `tempo` et `meter` du clip.
- Un changement de tempo peut intervenir sur n'importe quel tick. Un changement de metrique est insere sur une frontiere de mesure ; il peut ensuite fermer une mesure devenue incomplete si la metrique precedente est modifiee sans deplacer le marqueur.
- Les `TempoSection`, `MeterSection` et `PitchSection` ne sont pas stockees directement : elles sont derivees des intervalles entre deux changements de leur chronologie respective, ou entre le dernier changement et la fin du clip.
- Modifier une metrique propose deux intentions explicites : conserver la duree en ticks ou conserver le nombre de mesures.
- Par defaut, un clip vide conserve son nombre de mesures ; une section contenant deja des notes ou suivie d'autres sections conserve sa duree.
- Un `Clip` contient uniquement des `Note` comme contenu musical dans le premier perimetre fonctionnel. Les changements de tempo, de metrique et de contexte de hauteurs sont des donnees structurelles, et non des automations ou des evenements de controle.
- Le contexte de hauteurs est descriptif : il permet de mettre en evidence les notes qui appartiennent a un ensemble de hauteurs, sans interdire les notes exterieures.
- Une `Note` est une entity appartenant a un `Clip`. Elle conserve son identite lorsqu'elle est modifiee et recoit une nouvelle identite lorsqu'elle est copiee.
- Chaque `Note` reference l'instrument qui doit l'interpreter au moyen d'un `InstrumentId`. Un meme clip peut donc contenir plusieurs instruments.
- Le temps musical canonique est represente par des ticks entiers, avec une resolution fixe de 960 ticks par noire.
- Les positions des notes sont locales au clip et restent libres a l'echelle de ces ticks : une position n'a pas besoin d'etre alignee sur la grille visible.
- La quantification appartient d'abord a l'experience d'edition : elle guide les gestes de l'utilisateur sans transformer le modele musical en grille rigide.
- La selection appartient a l'etat applicatif de l'editeur. Elle reference temporairement des objets du domaine sans faire partie de la composition.
- L'application est destinee a l'ecriture et au processus initial de composition, pas a la production audio.
- Les instruments et leurs patchs sont definis dans le code avant la compilation. L'utilisateur choisit un instrument pour les notes, mais ne peut ni creer ni modifier son patch.
- Le catalogue d'instruments fait partie de l'infrastructure audio.
- Le moteur audio est indispensable a l'ecoute, mais il appartient a l'infrastructure et non au modele metier editable.
- Les automations, les evenements de controle et l'exposition de parametres audio ne font pas partie du premier perimetre fonctionnel.
- Le domaine ne connait de l'audio que les `InstrumentId` stables associes aux notes. Les ports audio appartiennent a la couche applicative.
- Les services applicatifs sont ranges dans `application/use-cases/`. Aucun service d'edition generique n'est cree avant que ses responsabilites soient definies.
- Aucun dossier generique `application/contracts/` n'est necessaire : les ports portent leurs propres modeles d'echange et les types metier restent dans le domaine.

## Domaine de composition

Le domaine de composition decrit une succession de sections musicales. Le `Project` ordonne directement les clips et repond a des questions comme :

- quels clips composent la sequence ?
- dans quel ordre doivent-ils etre lus ?
- quels clips doivent etre contournes ?
- quelles notes existent dans un clip ?
- quel instrument doit interpreter chaque note ?
- a quel moment relatif du clip chaque note doit-elle etre jouee ?
- quels tempo, quelle metrique et quel contexte de hauteurs sont actifs a un tick donne ?
- comment le clip est-il decoupe en sections temporelles, metriques et de hauteurs ?
- quelles notes appartiennent au contexte de hauteurs actif ?

Il reste independant de la maniere dont le son est produit et de la maniere dont l'utilisateur manipule visuellement les objets.

### Entities

Une entity possede une identite propre. Elle peut changer au cours du temps tout en restant le meme objet du point de vue du domaine.

#### Project

Represente le document musical complet ouvert dans l'application et la sequence de clips qui le compose.

Attributs possibles :

- `id`
- `name`
- `clips`
- `createdAt`
- `updatedAt`

Responsabilites :

- servir de racine de sauvegarde ;
- contenir et ordonner la sequence de clips ;
- garantir que les clips sont lus consecutivement ;
- permettre de reorganiser la composition sans recourir a des pistes ni a une timeline libre.

Le projet ne porte ni tempo ni metrique globaux : ces proprietes appartiennent a chaque clip.

#### Clip

Represente une section musicale editable, copiable, reordonnable et repetable.

Un clip ne possede pas de position globale. Il commence lorsque le clip precedent se termine, sauf s'il est contourne pendant la lecture.

Attributs possibles :

- `id`
- `name`
- `duration`
- `isBypassed`
- `repeatCount`
- `notes`
- `tempoChanges`
- `meterChanges`
- `pitchContextChanges`

Responsabilites :

- contenir et ordonner des notes selon leur position locale ;
- definir sa duree canonique en ticks ;
- contenir et ordonner les changements locaux de tempo, de metrique et de contexte de hauteurs ;
- fournir le contexte musical actif a n'importe quelle position ;
- conserver son etat de bypass et son nombre de lectures dans la sauvegarde ;
- permettre l'edition locale d'un motif, d'une phrase ou d'une section musicale ;
- permettre a plusieurs instruments de coexister dans une meme section par l'intermediaire des notes.

#### Note

Represente une note placee dans un clip et associee a un instrument. Elle possede une identite propre afin de conserver sa continuite lorsqu'elle est deplacee, redimensionnee, transposee ou modifiee.

Une `Note` n'est pas une racine d'agregat : elle appartient a un `Clip`, qui controle sa creation, sa modification et sa suppression. Le nom `Note` est retenu parce que cet objet occupe un intervalle musical ; les evenements instantanes `NoteOn` et `NoteOff` seront produits plus tard par le service de lecture.

Attributs possibles :

- `id`
- `pitch`
- `range`
- `velocity`
- `instrumentId`

Responsabilites :

- definir une hauteur ;
- definir une position temporelle relative au clip ;
- definir une duree ;
- porter des parametres d'interpretation simples ;
- identifier l'instrument charge de l'interpreter ;
- conserver son identite au fil de ses modifications.

Regles possibles :

- la position de debut est positive ou nulle ;
- la duree est strictement positive ;
- la note doit se terminer au plus tard a la fin du clip ;
- la hauteur et la velocite doivent rester dans leurs plages valides ;
- un `InstrumentId` est toujours present ;
- modifier une note conserve son identite, tandis que la copier ou la dupliquer en cree une nouvelle ;
- modifier le tempo ou la metrique ne deplace pas la note : sa position et sa duree restent exprimees dans les ticks canoniques du clip ;
- l'appartenance au `PitchContext` actif est calculee a partir de la position de debut et n'est pas stockee dans la note ;
- une note exterieure au contexte de hauteurs actif reste valide.

#### TempoChange

Represente un changement de tempo place sur la chronologie locale d'un clip.

Attributs possibles :

- `id`
- `position`
- `tempo`

Regles possibles :

- le premier changement est obligatoirement place au tick `0` ;
- un changement peut etre place sur n'importe quel tick compris dans le clip ;
- un seul changement de tempo peut exister a une meme position ;
- le nouveau tempo s'applique a partir du tick du changement, inclus ;
- la position et le tempo peuvent etre modifies sans changer l'identite du changement.

#### MeterChange

Represente un changement de metrique place sur la chronologie locale d'un clip.

Attributs possibles :

- `id`
- `position`
- `meter`

Regles possibles :

- le premier changement est obligatoirement place au tick `0` ;
- un nouveau changement est insere sur une frontiere de mesure ;
- un seul changement de metrique peut exister a une meme position ;
- la nouvelle metrique s'applique a partir du tick du changement, inclus ;
- un changement ferme la section metrique precedente et commence la suivante.

#### PitchContextChange

Represente un changement de contexte de hauteurs place sur la chronologie locale d'un clip.

Attributs possibles :

- `id`
- `position`
- `context`

Regles possibles :

- un changement peut etre place sur n'importe quel tick compris dans le clip ;
- un seul changement de contexte de hauteurs peut exister a une meme position ;
- le nouveau contexte s'applique a partir du tick du changement, inclus ;
- un changement ferme la section de hauteurs precedente et commence la suivante ;
- avant le premier changement, aucun contexte de hauteurs n'est actif.

Les marqueurs visibles dans l'editeur sont la representation des `TempoChange`, des `MeterChange` et des `PitchContextChange`. Un changement de chaque type peut exister au meme tick.

### Value Objects de la composition

Un Value Object ne possede pas d'identite propre. Il est defini par ses valeurs et appartient a un contexte precis, plutot qu'a une categorie transversale commune a toute l'application.

#### Pitch

Represente une hauteur musicale.

Attributs possibles :

- `midiNumber`
- `name`
- `octave`

Regles possibles :

- le numero MIDI doit rester dans une plage valide ;
- le nom et l'octave peuvent etre derives du numero MIDI.

#### TimePosition

Represente une position dans le temps musical.

Representation canonique :

- `ticks`, sous la forme d'un entier positif ou nul ;
- 960 ticks representent une noire.

Responsabilites :

- positionner une note dans le temps musical local d'un clip ;
- accepter toute position en ticks, y compris lorsqu'elle n'est pas alignee sur la grille d'edition ;
- rester independant du temps reel.

Les battements, les mesures et les secondes sont des representations derivees. Les secondes sont calculees par le service de lecture en integrant les changements de tempo rencontres dans le clip.

#### Duration

Represente une duree musicale exprimee en ticks entiers.

Regles possibles :

- une duree doit etre strictement positive ;
- 960 ticks representent une noire ;
- une duree peut utiliser toute valeur entiere, independamment de la grille ;
- une duree peut etre quantifiee par une operation d'edition.

Comme pour `TimePosition`, les battements et les secondes sont derives de la valeur canonique en ticks.

#### InstrumentId

Represente l'identifiant stable et opaque de l'instrument associe a une `Note`.

Responsabilites :

- permettre au domaine de conserver et comparer l'instrument choisi ;
- ne reveler aucune information sur la definition technique ou le patch de l'instrument ;
- rester exploitable par les couches applicative et d'infrastructure sans inverser le sens des dependances.

La disparition d'un instrument entre deux versions de l'application est traitee au chargement par la couche applicative.

#### TimeRange

Represente un intervalle musical entre un debut et une duree.

Attributs possibles :

- `start`
- `duration`

Responsabilites :

- decrire l'emplacement temporel d'une note dans son clip ;
- detecter les chevauchements ;
- faciliter les operations de deplacement et de redimensionnement.

#### Velocity

Represente l'intensite d'une note.

Attributs possibles :

- `value`

Regles possibles :

- valeur comprise entre 0 et 127 si l'on suit le modele MIDI ;
- valeur par defaut possible : 100.

#### Tempo

Represente une valeur de vitesse musicale.

Attributs possibles :

- `bpm`

Regles possibles :

- le BPM doit rester dans une plage musicalement exploitable ;
- le tempo permet au service de lecture de convertir un intervalle de temps musical en temps reel ;
- sa periode de validite est determinee par les `TempoChange` du clip.

#### TempoSection

Represente une vue derivee de l'intervalle compris entre un `TempoChange` et le changement suivant, ou la fin du clip.

Attributs derives possibles :

- `start`
- `end`
- `tempo`

La section n'est pas sauvegardee comme un objet autonome. Elle permet notamment au `PlaybackService` de convertir chaque intervalle de ticks en temps reel avec le tempo qui lui est propre.

#### Meter

Represente une valeur de metrique.

Attributs possibles :

- `beatsPerMeasure`
- `beatUnit`

Exemples :

- 4/4
- 3/4
- 6/8

Responsabilites :

- organiser les reperes en mesures a l'interieur d'une section metrique ;
- influencer les reperes temporels proposes par l'editeur ;
- permettre de calculer la longueur d'une mesure en ticks.

Avec une resolution de 960 ticks par noire :

```text
ticksPerMeasure = beatsPerMeasure * (4 / beatUnit) * 960
```

#### MeterSection

Represente une vue derivee de l'intervalle compris entre un `MeterChange` et le changement suivant, ou la fin du clip.

Attributs derives possibles :

- `start`
- `end`
- `meter`
- `fullMeasureCount`
- `trailingMeasureDuration`

La section n'est pas sauvegardee comme un objet autonome. Sa duree est imposee par ses bornes en ticks, puis son decoupage est calcule :

```text
fullMeasureCount = floor(sectionDuration / ticksPerMeasure)
trailingMeasureDuration = sectionDuration % ticksPerMeasure
```

Une valeur non nulle de `trailingMeasureDuration` represente une derniere mesure incomplete. Le changement suivant constitue alors une frontiere explicite et commence une nouvelle mesure.

#### PitchContext

Represente un ensemble de classes de hauteurs utilise comme reference harmonique ou melodique dans une partie du clip.

Attributs possibles :

- `label`
- `pitchClasses`

Exemples :

- `Re dorien` : re, mi, fa, sol, la, si, do ;
- `Fa majeur 7` : fa, la, do, mi ;
- une gamme ou un ensemble de hauteurs libre defini par le programme.

Responsabilites :

- determiner si la hauteur d'une note appartient au contexte ;
- permettre a l'editeur de mettre visuellement en evidence les hauteurs interieures et exterieures ;
- representer indifferemment une gamme, un mode, un accord ou un ensemble arbitraire de classes de hauteurs.

Le contexte ne valide ni ne refuse les notes. Une note exterieure reste une `Note` parfaitement valide.

#### PitchSection

Represente une vue derivee de l'intervalle compris entre un `PitchContextChange` et le changement suivant, ou la fin du clip.

Attributs derives possibles :

- `start`
- `end`
- `context`

La section n'est pas sauvegardee comme un objet autonome. Elle sert a retrouver le contexte de hauteurs actif pour une note ou une position donnee.

### Agregats

#### Project comme aggregate root

`Project` est la racine principale. Il garantit la coherence globale du document musical et de sa sequence.

Il contient :

- une collection ordonnee de clips ;
- des informations de sauvegarde.

Regles possibles :

- un clip appartient a un seul projet ;
- l'ordre des clips definit integralement leur ordre de lecture ;
- les clips sont consecutifs et ne possedent pas de position temporelle globale ;
- reordonner un clip modifie la structure de la sequence ;
- un clip contourne reste present a sa place dans la sequence et son etat est sauvegarde ;
- pendant la lecture, un clip contourne est ignore et le clip suivant commence immediatement, quel que soit son `repeatCount` ;
- un `repeatCount` fini indique le nombre total de lectures du clip avant de passer au suivant ;
- un `repeatCount` egal a `infinite` repete le clip jusqu'a l'arret ou au deplacement manuel de la lecture et rend les clips suivants inaccessibles par progression automatique ;
- chaque repetition recommence au tick `0` avec les changements initiaux de tempo, de metrique et de contexte de hauteurs du clip.

#### Clip comme aggregate secondaire

`Clip` garantit la coherence de ses propres notes et de son contexte rythmique.

Regles possibles :

- une note appartient a un seul clip et ne possede pas de cycle de vie autonome ;
- une note modifiee conserve son identite ;
- une note copiee ou dupliquee recoit une nouvelle identite ;
- les notes sont positionnees relativement au debut du clip ;
- une note reference exactement un instrument ;
- plusieurs notes d'un meme clip peuvent referencer des instruments differents ;
- `repeatCount` vaut par defaut `1` ; une valeur finie est un entier strictement positif et `infinite` represente une repetition sans fin ;
- `isBypassed` reste independant de `repeatCount`, afin de pouvoir contourner puis reactiver un clip sans perdre son nombre de lectures ;
- les positions et durees peuvent rester continues dans le modele ;
- les changements de tempo, de metrique et de contexte de hauteurs sont ordonnes par position ;
- les chronologies de tempo et de metrique contiennent exactement un changement initial au tick `0` ;
- la chronologie de contexte de hauteurs peut etre vide ; avant son premier changement, aucun contexte n'est actif ;
- un changement s'applique a partir de sa position, incluse, jusqu'au changement suivant ;
- modifier une metrique en conservant la duree ne deplace ni les notes, ni les marqueurs, ni la fin du clip ;
- la quantification est appliquee par les operations d'edition ;
- selon le choix musical, les chevauchements sur une meme hauteur peuvent etre autorises ou interdits.

## Etat applicatif de l'editeur

L'etat applicatif de l'editeur decrit le contexte transitoire dans lequel l'utilisateur manipule la composition. Sa modification ne change pas, a elle seule, le contenu musical du projet.

### Selection

Represente l'ensemble courant des objets selectionnes. Elle ne possede pas d'identite propre et n'est ni une entity metier ni un agregat du domaine.

Attributs possibles :

- `selectedClipIds`
- `selectedNoteIds`
- `selectionAnchor`

Des informations comme `activeClipId` ou `focusedNoteId` permettent de distinguer l'objet actif de l'ensemble des objets selectionnes.

Responsabilites :

- conserver l'intention d'edition courante ;
- permettre les operations de groupe ;
- porter la logique de selection independamment de sa representation graphique ;
- transmettre aux cas d'usage les identifiants des objets concernes.

Lorsqu'une action est executee, la couche applicative transforme la selection en une commande explicite. Le domaine recoit les identifiants des objets a modifier et applique l'operation sans connaitre la notion de selection.

### GridResolution

Represente la precision de la grille utilisee pendant l'edition.

Attributs possibles :

- `snapStepTicks`

Responsabilites :

- definir les pas de quantification proposes a l'utilisateur ;
- controler la finesse du placement et du redimensionnement ;
- convertir un geste utilisateur vers une position ou une duree quantifiee.

La selection et la resolution de grille sont des donnees transitoires de l'editeur. Elles ne sont pas sauvegardees comme des donnees musicales du projet. Leur persistance eventuelle relevera des preferences ou de la restauration de session.

## Couche applicative et lecture

La couche applicative orchestre les actions de l'utilisateur et la lecture sans contenir les regles internes du moteur audio.

### Cas d'usage d'edition

Responsabilites :

- traduire les gestes de l'editeur en commandes explicites ;
- resoudre la selection vers les identifiants des objets concernes ;
- appliquer si necessaire la quantification avant d'appeler le domaine ;
- charger et sauvegarder le projet a travers des ports.

Exemples :

- deplacer des notes ;
- redimensionner un clip ;
- ajouter, deplacer ou supprimer un changement de tempo, de metrique ou de contexte de hauteurs ;
- modifier une metrique en indiquant explicitement s'il faut conserver la duree ou le nombre de mesures ;
- associer un contexte de hauteurs a une partie du clip sans contraindre les notes ;
- reordonner un clip, modifier son `repeatCount` ou son etat de bypass ;
- transposer plusieurs notes ;
- associer un instrument disponible a une ou plusieurs notes.

Le changement de metrique utilise une politique explicite, par exemple `PRESERVE_DURATION` ou `PRESERVE_MEASURE_COUNT`.

- `PRESERVE_DURATION` conserve les ticks des notes, des marqueurs et de la fin du clip. Le nombre de mesures est recalcule et la section peut se terminer par une mesure incomplete.
- `PRESERVE_MEASURE_COUNT` recalcule la borne de fin de la section selon la nouvelle longueur de mesure. Dans le premier perimetre, ce mode est utilise pour un clip vide ou une section terminale vide, afin de ne pas imposer de deplacement en cascade aux sections suivantes.

Lors de la creation d'un clip, le cas d'usage recoit un nombre de mesures, une metrique et un tempo, puis calcule immediatement la duree canonique en ticks. Pour un nouveau clip vide, modifier la metrique conserve par defaut le nombre de mesures. Des que la section contient des notes ou que d'autres sections la suivent, l'editeur propose par defaut de conserver la duree.

Aucun `EditorService` generique n'est introduit. Les futurs services d'edition seront nommes et ajoutes dans `application/use-cases/` lorsque leurs responsabilites precises seront etablies.

### PlaybackService

Le service de lecture fait le lien entre la composition et l'infrastructure audio.

Responsabilites :

- parcourir la sequence de clips dans son ordre ;
- ignorer les clips contournes sans modifier leur `repeatCount` ;
- repeter chaque clip selon son `repeatCount` avant de poursuivre la sequence ;
- recommencer les chronologies locales au tick `0` a chaque repetition ;
- construire les `TempoSection` de chaque clip ;
- convertir chaque section de temps musical en temps reel selon son tempo ;
- transformer les notes en commandes audio ;
- transmettre ces commandes a un `AudioEngine` abstrait.

### Ports

Les ports decrivent les capacites attendues par l'application sans imposer leur implementation.

#### AudioEngine

Port minimal permettant notamment :

- d'initialiser et d'arreter la lecture ;
- de planifier des commandes audio ;
- de controler le cycle de lecture ;
- de transmettre un `InstrumentId` sans connaitre le patch correspondant.

#### InstrumentCatalog

Port de consultation permettant notamment :

- de lister les instruments disponibles ;
- d'obtenir pour chacun un `InstrumentSummary` ;
- de verifier qu'un `InstrumentId` peut etre resolu.

`InstrumentSummary` est un modele de sortie minimal declare a cote du port :

- `id`, de type `InstrumentId` ;
- `name`.

Il ne contient aucune definition de patch ni aucun parametre audio. L'implementation concrete du port appartient a l'infrastructure audio.

## Infrastructure audio

L'infrastructure audio regroupe le catalogue integre, les definitions techniques des instruments et le moteur qui produit le son.

Les instruments et leurs patchs sont ecrits dans le code avant la compilation. Ils ne sont ni editables par l'utilisateur ni sauvegardes dans le projet.

### BuiltInInstrumentCatalog

Implementation concrete du port `InstrumentCatalog`. Il expose en lecture seule les instruments disponibles et resout leurs identifiants stables.

### InstrumentDefinition

Represente la definition technique complete d'un instrument integre.

Attributs possibles :

- `id`
- `name`
- `patch`

Responsabilites :

- associer un identifiant stable a une implementation sonore ;
- fournir le patch necessaire a l'instanciation.

### ModularPatch

Represente le graphe interne statique d'un instrument.

Attributs possibles :

- `modules`
- `connections`
- `outputModuleId`

Responsabilites :

- organiser les modules audio ;
- garantir la coherence technique des connexions ;
- decrire le parcours du signal et des modulations.

### AudioModule

Represente la definition d'un module audio ou de controle, comme un `VCO`, une enveloppe, un filtre, un `LFO`, un `VCA`, un mixer ou une sortie.

Attributs possibles :

- `id`
- `type`
- `parameters`
- `inputs`
- `outputs`

### ModuleConnection

Represente une connexion statique entre deux ports de modules.

Attributs possibles :

- `sourcePort`
- `targetPort`

### ModulePort

Value Object propre a l'infrastructure audio.

Attributs possibles :

- `moduleId`
- `portName`
- `signalType`

Types de signal possibles :

- `audio`
- `control`
- `gate`
- `trigger`

Responsabilites :

- identifier un point de connexion ;
- permettre la validation des connexions entre modules.

### Moteur audio concret

Le moteur audio instancie les definitions d'instruments et produit le son.

Il gere notamment :

- la resolution des `InstrumentId` dans le catalogue integre ;
- l'execution des patchs modulaires ;
- la planification temporelle des commandes ;
- le cycle de vie des voix sonores ;
- la sortie audio.

Son etat d'execution est transitoire et n'est pas sauvegarde dans le projet.

## Relations architecturales

| Depuis | Vers | Nature du lien |
| --- | --- | --- |
| `Project.clips` | `Clip` | Le projet conserve l'ordre persistant des sections musicales. |
| `Note.instrumentId` | `InstrumentId` | Chaque note conserve l'identifiant opaque de l'instrument qui doit l'interpreter. |
| `Clip.tempoChanges` | `TempoChange` | Les changements delimitent les `TempoSection` derivees du clip. |
| `Clip.meterChanges` | `MeterChange` | Les changements delimitent les `MeterSection` derivees du clip. |
| `Clip.pitchContextChanges` | `PitchContextChange` | Les changements delimitent les `PitchSection` utilisees pour analyser visuellement les notes. |
| Etat de l'editeur | Cas d'usage | La selection et la grille sont transformees en commandes explicites. |
| `PlaybackService` | `AudioEngine` | Le service transmet des commandes a travers un port abstrait. |
| `InstrumentCatalog` | `BuiltInInstrumentCatalog` | L'infrastructure implemente le port de consultation attendu par l'application. |
| Moteur audio concret | `InstrumentDefinition` | Le moteur resout l'identifiant, instancie le patch et produit le son. |

Le sens des dependances de code doit pointer vers l'interieur : l'application depend du domaine, et l'infrastructure depend des ports applicatifs ainsi que du domaine, jamais l'inverse.

## Arborescence cible

Cette arborescence est une cible de travail provisoire. Elle documente les frontieres actuellement retenues et evoluera avec les prochaines decisions.

```text
src/
├── domain/
│   ├── composition/
│   │   ├── Project.ts
│   │   ├── Clip.ts
│   │   └── Note.ts
│   ├── time/
│   │   ├── TimePosition.ts
│   │   ├── Duration.ts
│   │   ├── TimeRange.ts
│   │   ├── Tempo.ts
│   │   ├── TempoChange.ts
│   │   ├── TempoSection.ts
│   │   ├── Meter.ts
│   │   ├── MeterChange.ts
│   │   └── MeterSection.ts
│   ├── pitch/
│   │   ├── Pitch.ts
│   │   ├── PitchContext.ts
│   │   ├── PitchContextChange.ts
│   │   └── PitchSection.ts
│   ├── InstrumentId.ts
│   └── Velocity.ts
├── application/
│   ├── editor/
│   │   ├── EditorState.ts
│   │   ├── Selection.ts
│   │   └── GridResolution.ts
│   ├── use-cases/
│   │   └── PlaybackService.ts
│   └── ports/
│       ├── AudioEngine.ts
│       └── InstrumentCatalog.ts
├── infrastructure/
│   ├── audio/
│   │   ├── catalog/
│   │   │   ├── BuiltInInstrumentCatalog.ts
│   │   │   └── InstrumentDefinition.ts
│   │   ├── modular/
│   │   │   ├── ModularPatch.ts
│   │   │   ├── AudioModule.ts
│   │   │   ├── ModuleConnection.ts
│   │   │   └── ModulePort.ts
│   │   └── engine/
│   └── persistence/
└── presentation/
    ├── components/
    └── stores/
```

Le domaine est maintenant organise par concepts metier coherents plutot que par categories techniques comme les entities et les Value Objects :

- `composition/` contient les agregats qui organisent le document musical ;
- `time/` contient les positions, les durees, le tempo et la metrique ;
- `pitch/` contient les hauteurs et leurs contextes.

Les dependances doivent principalement partir de `composition/` vers `time/` et `pitch/`. Ces deux sous-domaines restent independants de `composition/` : par exemple, `Clip` peut connaitre `MeterChange`, mais `MeterChange` ne connait pas `Clip`.

`InstrumentId` et `Velocity` restent provisoirement a la racine de `domain/`. Des sous-dossiers ne seront crees pour eux que lorsqu'un ensemble de concepts suffisamment coherent apparaitra.

Cette structure exprime des responsabilites plutot qu'un decoupage definitif fichier par fichier. Elle ne doit pas conduire a creer prematurement un fichier pour chaque type si plusieurs concepts restent plus coherents dans un meme module.

## Questions ouvertes

Aucune pour le moment.

## Principes directeurs

Pour une architecture clean, le domaine doit rester independant de l'interface graphique, du moteur Web Audio et du stockage.

Les objets du domaine de composition comme `Project`, `Clip`, `Note`, `PitchContext` ou `TimeRange` doivent pouvoir exister sans connaitre React, canvas, Zustand ou Web Audio.

L'etat de l'editeur peut connaitre les identifiants du domaine, mais le domaine ne connait ni la selection, ni la grille, ni les outils de l'interface.

La couche applicative orchestre les cas d'usage et depend de ports abstraits. L'infrastructure audio implemente ces ports, contient le catalogue d'instruments et peut etre remplacee sans modifier le coeur de la composition.
