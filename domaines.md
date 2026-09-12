# Domaines

Ce document recense les premiers objets fondamentaux du domaine pour une application de piano roll. L'objectif est de poser un vocabulaire metier stable avant de penser interface, stockage ou moteur audio.

## Domaine principal : edition musicale

Le coeur de l'application est l'edition d'une sequence musicale organisee dans le temps. Le piano roll permet de placer, modifier, deplacer et supprimer des evenements musicaux sur une grille temporelle.

## Entities

Une entity possede une identite propre. Elle peut changer au cours du temps tout en restant le meme objet du point de vue du domaine.

### Project

Represente le document musical complet ouvert dans l'application.

Attributs possibles :

- `id`
- `name`
- `score`
- `tempo`
- `timeSignature`
- `createdAt`
- `updatedAt`

Responsabilites :

- contenir l'etat musical principal ;
- servir de racine de sauvegarde ;
- porter les reglages globaux du morceau.

### Score

Represente l'organisation musicale generale du projet.

Attributs possibles :

- `id`
- `tracks`
- `length`
- `gridResolution`

Responsabilites :

- organiser les pistes ;
- definir la duree globale editable ;
- fournir le cadre temporel commun.

### Track

Represente une voix, un instrument ou une ligne musicale editable dans le piano roll.

Attributs possibles :

- `id`
- `name`
- `instrument`
- `color`
- `isMuted`
- `isSolo`
- `notes`

Responsabilites :

- contenir les notes d'une voix ;
- porter les reglages propres a cette voix ;
- permettre l'edition separee de plusieurs parties musicales.

### Note

Represente un evenement musical place sur la grille.

Attributs possibles :

- `id`
- `pitch`
- `range`
- `velocity`

Responsabilites :

- definir une hauteur ;
- definir une position temporelle et une duree ;
- porter des parametres d'interpretation simples.

### Selection

Represente l'ensemble courant des objets selectionnes par l'utilisateur.

Attributs possibles :

- `id`
- `selectedNoteIds`
- `selectedTrackId`

Responsabilites :

- conserver l'intention d'edition courante ;
- permettre les operations de groupe ;
- separer la logique de selection de la representation graphique.

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

Represente une position dans le temps musical.

Attributs possibles :

- `tick`
- `beat`
- `measure`

Responsabilites :

- positionner un evenement sur la grille ;
- permettre les conversions entre ticks, temps et mesures.

### Duration

Represente une duree musicale.

Attributs possibles :

- `ticks`
- `beats`

Regles possibles :

- une duree doit etre strictement positive ;
- une duree peut etre quantifiee selon la resolution de la grille.

### TimeRange

Represente un intervalle musical entre un debut et une duree.

Attributs possibles :

- `start`
- `duration`

Responsabilites :

- decrire l'emplacement temporel d'une note ;
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

- organiser la grille en mesures ;
- influencer l'affichage et les reperes visuels.

### GridResolution

Represente la precision d'edition de la grille.

Attributs possibles :

- `ticksPerBeat`
- `snapStep`

Responsabilites :

- definir les pas de quantification ;
- controler la finesse du placement et du redimensionnement.

### Instrument

Represente le timbre ou la source sonore associee a une piste.

Attributs possibles :

- `type`
- `name`
- `parameters`

Exemples :

- synthese simple Web Audio ;
- sampler ;
- instrument MIDI externe.

## Premiers agregats possibles

### Project comme aggregate root

`Project` peut etre considere comme la racine principale. Il garantit la coherence globale du document musical.

Il contient :

- un `Score` ;
- des reglages globaux comme `Tempo` et `TimeSignature` ;
- des informations de sauvegarde.

### Track comme aggregate secondaire

`Track` peut garantir la coherence de ses propres notes.

Regles possibles :

- une note appartient a une seule piste ;
- les notes d'une piste peuvent etre triees par position ;
- selon le choix musical, on peut autoriser ou interdire les chevauchements sur une meme hauteur.

## Questions a trancher

- Une `Note` doit-elle etre une entity, ou un value object contenu dans une piste ?
- Le piano roll vise-t-il principalement une ecriture MIDI classique, ou un modele plus libre adapte a la composition algorithmique ?
- Les pistes representent-elles des instruments, des voix musicales, ou les deux ?
- Faut-il penser la partition comme une grille fixe, ou comme un espace temporel continu avec quantification optionnelle ?
- La selection appartient-elle vraiment au domaine, ou plutot a l'etat applicatif de l'editeur ?

## Intuition de depart

Pour une architecture clean, le domaine devrait rester independant de l'interface graphique, du moteur Web Audio et du stockage. Les objets comme `Note`, `Track`, `Pitch`, `Duration` ou `TimeRange` doivent pouvoir exister sans React, sans canvas et sans navigateur.