# Domaines

Ce document recense les premiers objets fondamentaux du domaine pour une application de piano roll avec instruments modulaires.

L'objectif est de poser un vocabulaire metier stable avant de penser interface, stockage, Web Audio API ou implementation React.

## Vision generale

Un projet contient deux grands espaces volontairement decouples :

- le domaine d'arrangement, qui decrit l'organisation musicale dans le temps ;
- le domaine audio, qui decrit la fabrication et le comportement sonore des instruments.

Le domaine d'arrangement ne produit pas directement de son. Il exprime des intentions musicales sous forme de pistes, clips et evenements. Le domaine audio interprete ces intentions a travers des instruments construits comme des patchs modulaires.

```mermaid
flowchart TD
    Project --> Arrangement
    Project --> AudioSystem
    Arrangement --> Track
    Track --> Clip
    Clip --> MusicalEvent
    AudioSystem --> ModularInstrument
    ModularInstrument --> ModularPatch
```

## Decisions actees

- Un `Arrangement` est l'ensemble ordonne des pistes du projet.
- Une `Track` est un conteneur de clips ordonnes dans le temps, lie a un instrument arbitraire par son identifiant.
- Le temps du domaine est pense comme un espace continu. L'utilisateur pourra toutefois placer, deplacer et redimensionner des evenements a l'aide d'une grille quantifiee (la quantification appartient d'abord a l'experience d'edition : elle guide les gestes de l'utilisateur sans obliger le modele musical a devenir une grille rigide).

## Domaine d'arrangement

Le domaine d'arrangement decrit la structure musicale du projet dans le temps. Il repond a des questions comme :

- quelles pistes existent ?
- quels clips sont places sur ces pistes ?
- quels evenements musicaux existent dans un clip ?
- a quel moment ces evenements doivent-ils etre joues ?

Il reste independant de la maniere dont le son est produit.

### Entities

Une entity possede une identite propre. Elle peut changer au cours du temps tout en restant le meme objet du point de vue du domaine.

### Project

Represente le document musical complet ouvert dans l'application.

Attributs possibles :

- `id`
- `name`
- `arrangement`
- `audioSystem`
- `tempo`
- `timeSignature`
- `createdAt`
- `updatedAt`

Responsabilites :

- servir de racine de sauvegarde ;
- contenir les grands sous-domaines du projet ;
- porter les reglages globaux du morceau.

### Arrangement

Represente l'organisation musicale globale du projet.

Attributs possibles :

- `id`
- `tracks`
- `length`
- `gridResolution`

Responsabilites :

- organiser les pistes dans le temps ;
- definir la duree globale editable ;
- fournir le cadre temporel commun aux clips.

### Track

Represente un conteneur de clips ordonnes dans le temps.

Une piste peut etre associee a un instrument, mais elle ne contient pas l'instrument lui-meme. Elle reference l'instrument qui interpretera ses evenements.

Attributs possibles :

- `id`
- `name`
- `instrumentId`
- `color`
- `isMuted`
- `isSolo`
- `clips`

Responsabilites :

- contenir et ordonner des clips selon leur position temporelle ;
- porter les reglages propres a la piste ;
- faire le lien entre des intentions musicales et un instrument audio sans fusionner avec lui.

### Clip

Represente une unite musicale editable, deplacable, copiable et potentiellement bouclable.

Attributs possibles :

- `id`
- `name`
- `range`
- `loop`
- `events`

Responsabilites :

- contenir des evenements musicaux ;
- definir une region temporelle sur une piste ;
- permettre l'edition locale d'un motif, d'une phrase ou d'une cellule musicale.

### MusicalEvent

Represente un evenement musical abstrait contenu dans un clip.

`MusicalEvent` peut etre pense comme une famille d'evenements plus specialises.

Types possibles :

- `NoteEvent`
- `AutomationEvent`
- `ControlEvent`

Responsabilites :

- decrire ce qui doit arriver musicalement ;
- rester independant du moteur audio ;
- etre interpretable par un instrument ou par le systeme de lecture.

### NoteEvent

Represente une note placee dans un clip.

Attributs possibles :

- `id`
- `pitch`
- `range`
- `velocity`

Responsabilites :

- definir une hauteur ;
- definir une position temporelle relative au clip ;
- definir une duree ;
- porter des parametres d'interpretation simples.

### AutomationEvent

Represente une variation de parametre dans le temps.

Attributs possibles :

- `id`
- `target`
- `time`
- `value`
- `curve`

Responsabilites :

- exprimer une modulation composee ou dessinee ;
- permettre au domaine d'arrangement d'agir sur des parametres audio sans connaitre leur implementation ;
- decrire des changements reproductibles dans le temps musical.

### Selection

Represente l'ensemble courant des objets selectionnes par l'utilisateur.

Question ouverte : `Selection` appartient peut-etre davantage a l'etat applicatif de l'editeur qu'au domaine metier pur.

Attributs possibles :

- `id`
- `selectedTrackId`
- `selectedClipIds`
- `selectedEventIds`

Responsabilites :

- conserver l'intention d'edition courante ;
- permettre les operations de groupe ;
- separer la logique de selection de la representation graphique.

## Domaine audio

Le domaine audio decrit les instruments et leur architecture interne. Il repond a des questions comme :

- quels instruments existent dans le projet ?
- quels modules composent un instrument ?
- comment ces modules sont-ils connectes ?
- quels parametres peuvent etre controles par les clips ou l'utilisateur ?

Il reste independant de l'interface de piano roll et de la disposition graphique des clips.

### AudioSystem

Represente l'ensemble des ressources audio du projet.

Attributs possibles :

- `id`
- `instruments`
- `masterOutput`

Responsabilites :

- contenir les instruments disponibles ;
- definir la sortie audio globale ;
- fournir les instruments references par les pistes.

### ModularInstrument

Represente un instrument fabrique a partir d'un patch modulaire.

Attributs possibles :

- `id`
- `name`
- `patch`
- `parameters`

Responsabilites :

- recevoir des evenements musicaux ;
- exposer des parametres controlables ;
- produire un signal audio a partir d'un patch.

### ModularPatch

Represente le graphe interne d'un instrument modulaire.

Attributs possibles :

- `id`
- `modules`
- `connections`
- `outputModuleId`

Responsabilites :

- organiser les modules audio ;
- garantir la coherence des connexions ;
- decrire le parcours du signal et des modulations.

### AudioModule

Represente un module audio ou de controle.

Types possibles :

- `VCO`
- `Envelope`
- `Filter`
- `LFO`
- `VCA`
- `Mixer`
- `Output`

Attributs possibles :

- `id`
- `type`
- `parameters`
- `inputs`
- `outputs`

Responsabilites :

- fournir une fonction sonore ou de controle ;
- declarer ses entrees et sorties ;
- exposer des parametres modulables.

### ModuleConnection

Represente une connexion entre deux ports de modules.

Attributs possibles :

- `id`
- `sourcePort`
- `targetPort`

Responsabilites :

- relier deux modules ;
- distinguer signal audio et signal de controle si necessaire ;
- permettre la validation du graphe modulaire.

## Value Objects

Un value object ne possede pas d'identite propre. Il est defini par ses valeurs. Deux value objects ayant les memes valeurs sont equivalents.

### Pitch

Represente une hauteur musicale.

Attributs possibles :

- `midiNumber`
- `name`
- `octave`

Exemples :

- C4
- F#3
- MIDI 60

Regles possibles :

- le numero MIDI doit rester dans une plage valide ;
- le nom de note peut etre derive du numero MIDI.

### TimePosition

Represente une position dans le temps musical continu.

Attributs possibles :

- `ticks`
- `beats`
- `seconds`

Responsabilites :

- positionner un evenement dans le temps musical ;
- permettre les conversions entre temps musical et temps reel ;
- accepter des valeurs non quantifiees lorsque l'edition ou l'import le necessite.

### Duration

Represente une duree musicale.

Attributs possibles :

- `ticks`
- `beats`
- `seconds`

Regles possibles :

- une duree doit etre strictement positive ;
- une duree peut etre libre dans le domaine ;
- une duree peut etre quantifiee lors d'une operation d'edition.

### TimeRange

Represente un intervalle musical entre un debut et une duree.

Attributs possibles :

- `start`
- `duration`

Responsabilites :

- decrire l'emplacement temporel d'un clip ou d'un evenement ;
- detecter les chevauchements ;
- faciliter les operations de deplacement et de redimensionnement.

### Velocity

Represente l'intensite d'une note.

Attributs possibles :

- `value`

Regles possibles :

- valeur comprise entre 0 et 127 si l'on suit le modele MIDI ;
- valeur par defaut possible : 100.

### Tempo

Represente la vitesse globale du projet.

Attributs possibles :

- `bpm`

Regles possibles :

- le BPM doit rester dans une plage musicalement exploitable ;
- le tempo permet de convertir le temps musical en temps reel.

### TimeSignature

Represente la mesure musicale.

Attributs possibles :

- `beatsPerMeasure`
- `beatUnit`

Exemples :

- 4/4
- 3/4
- 6/8

Responsabilites :

- organiser les reperes en mesures ;
- influencer l'affichage et les reperes visuels.

### GridResolution

Represente la precision de la grille utilisee pendant l'edition.

Attributs possibles :

- `ticksPerBeat`
- `snapStep`

Responsabilites :

- definir les pas de quantification proposes a l'utilisateur ;
- controler la finesse du placement et du redimensionnement pendant l'edition ;
- convertir un geste utilisateur vers une position ou une duree quantifiee.

### Loop

Represente le comportement de repetition d'un clip.

Attributs possibles :

- `enabled`
- `length`

Responsabilites :

- definir si un clip boucle ;
- distinguer la duree visible du clip et la duree du motif repete.

### ParameterId

Represente l'identifiant stable d'un parametre controlable.

Exemples :

- `filter.cutoff`
- `vco.frequency`
- `envelope.attack`

Responsabilites :

- permettre a l'arrangement de cibler un parametre sans connaitre l'objet technique qui l'implemente ;
- stabiliser les liens entre automation et instrument.

### ParameterValue

Represente la valeur d'un parametre audio ou de controle.

Attributs possibles :

- `value`
- `unit`
- `min`
- `max`

Responsabilites :

- encapsuler une valeur controlable ;
- permettre la validation d'une plage ;
- exprimer des unites differentes comme Hz, dB, pourcentage ou temps.

### ModulePort

Represente une entree ou une sortie de module.

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

## Relations entre les domaines

Le couplage entre arrangement et audio doit rester minimal.

| Depuis | Vers | Nature du lien |
| --- | --- | --- |
| `Track.instrumentId` | `ModularInstrument.id` | Une piste choisit l'instrument qui interprete ses clips. |
| `NoteEvent` | `ModularInstrument` | Une note declenche l'instrument pendant la lecture. |
| `AutomationEvent.target` | `ParameterId` | Une automation cible un parametre expose par un instrument. |
| `Project` | `Arrangement` et `AudioSystem` | Le projet coordonne les deux sous-domaines. |

## Premiers agregats possibles

### Project comme aggregate root

`Project` peut etre considere comme la racine principale. Il garantit la coherence globale du document musical.

Il contient :

- un `Arrangement` ;
- un `AudioSystem` ;
- des reglages globaux comme `Tempo` et `TimeSignature` ;
- des informations de sauvegarde.

### Arrangement comme aggregate

`Arrangement` garantit la coherence temporelle des pistes et des clips.

Regles possibles :

- une piste appartient a un seul arrangement ;
- l'arrangement definit l'ordre de ses pistes ;
- une piste ordonne ses clips selon leur position temporelle ;
- une piste reference un instrument arbitraire sans le contenir ;
- un clip appartient a une seule piste ;
- les clips peuvent se chevaucher ou non selon le choix d'edition ;
- les positions des clips sont exprimees dans le temps global du projet.

### Clip comme aggregate secondaire

`Clip` garantit la coherence de ses propres evenements.

Regles possibles :

- un evenement appartient a un seul clip ;
- les evenements sont positionnes relativement au debut du clip ;
- les notes peuvent etre triees par position ;
- les positions et durees peuvent rester continues dans le modele ;
- la quantification est appliquee par les operations d'edition quand l'utilisateur active ou utilise la grille ;
- selon le choix musical, on peut autoriser ou interdire les chevauchements sur une meme hauteur.

### ModularInstrument comme aggregate

`ModularInstrument` garantit la coherence de son patch.

Regles possibles :

- un module appartient a un seul patch ;
- une connexion relie deux ports compatibles ;
- un patch doit posseder une sortie audio valide ;
- les parametres exposes doivent avoir des identifiants stables.

## Questions ouvertes

- Le terme `Arrangement` convient-il pour nommer le domaine temporel, ou faut-il preferer `Composition`, `Timeline`, `Score` ou `Session` ?
- Une `NoteEvent` doit-elle etre une entity, ou un value object contenu dans un clip ?
- Les clips doivent-ils etre uniquement des conteneurs de notes, ou peuvent-ils contenir d'autres types d'evenements comme des automations et des controles ?
- La selection appartient-elle vraiment au domaine, ou plutot a l'etat applicatif de l'editeur ?
- Le domaine audio doit-il modeliser seulement la definition des instruments, ou aussi leur etat d'execution pendant la lecture ?
- Comment representer proprement les parametres exposes par un instrument modulaire pour que l'arrangement puisse les automatiser sans connaitre le patch en detail ?

## Intuition de depart

Pour une architecture clean, le domaine devrait rester independant de l'interface graphique, du moteur Web Audio et du stockage.

Les objets d'arrangement comme `Track`, `Clip`, `NoteEvent`, `TimeRange` ou `GridResolution` doivent pouvoir exister sans connaitre React, canvas ou Web Audio.

Les objets audio comme `ModularInstrument`, `ModularPatch`, `AudioModule` ou `ModuleConnection` doivent pouvoir exister sans connaitre le piano roll. Ils decrivent une architecture sonore ; le moteur audio concret viendra plus tard interpreter cette architecture.