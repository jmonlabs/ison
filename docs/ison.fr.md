# ison : un instrument dit en JSON

ison, pour JSON Instrument Object Notation. Un ison, c'est aussi la note tenue sous la mélodie dans le chant byzantin : un bourdon qui porte. Le schéma est `algo/schemas/ison-schema.json` ; jmon/sound le lit (`src/ison.js`).

## Pourquoi

Aujourd'hui, un instrument arrive brut : un SFZ, un preset de SoundFont, un DSP Faust, un synthé Tone. Tout ce qui fait un jeu vivant est refait à la main dans chaque carnet : quel enregistrement pour quel mode, quand changer d'archet, à partir de quelle longueur mettre du vibrato, à quel niveau, avec quelle attaque, quel contrôleur fait quoi. `violinist` dans `essais/violons.js` est ce savoir, et il est dans un carnet d'essai au lieu d'être dans l'instrument.

SoundFont et SFZ disent ce qu'il y a à jouer : des échantillons, des zones, des enveloppes, des keyswitches. Ils ne disent pas comment on joue de l'instrument. Le format proposé ajoute cette couche, et seulement elle : il enveloppe un ou plusieurs sons existants, il ne les remplace pas.

## Les principes

- **Déclaratif.** Un objet JSON, comme une pièce jmon. Pas de code : des noms, des nombres, des listes. Un fichier `.json`, ou l'objet inline dans `synth`.
- **Dans les mots de jmon.** Les temps en temps, les niveaux en dB, les proportions de 0 à 1, les contrôleurs par nom, les mêmes champs de note (`variant`, `dynamics`, `bend`, `articulations`).
- **Plus souple qu'un SoundFont.** Un instrument peut prendre ses sons dans plusieurs sources : un mode dans un SF2, un autre dans un SFZ, un autre dans un DSP Faust. Les sources existantes restent telles quelles.
- **Le jeu est écrit une fois.** Les règles de jeu (coups d'archet, vibrato, niveaux) sont dans l'instrument ; une pièce écrit des notes, et l'instrument les joue comme il se joue.
- **Lisible par un compositeur.** Les mêmes conventions que les carnets : des noms qui disent le rôle, une ligne par réglage, une description.

## Un exemple : le violon LDK

```json
{
  "format": "ison",
  "version": "1.0",
  "name": "Violon LDK",
  "description": "Un violon solo enregistré dans tous ses modes de jeu. Aucun enregistrement ne tient plus de 4 s : une note longue se joue en coups d'archet.",
  "family": "bowed strings",
  "range": [55, 100],

  "sources": {
    "ldk": { "sf2": "/samples/soundfonts/Violin_LDK1609.sf2" },
    "harmonics": { "sfz": "/samples/soundfonts/violin-soundfonts/harmonics.sfz" }
  },

  "modes": {
    "normal": { "source": "ldk", "program": 0, "holds": 5 },
    "vibrato": { "source": "ldk", "program": 11, "holds": 4, "vibrato": { "onset": 0.5, "depth": 9, "rate": 5 } },
    "tremolo": { "source": "ldk", "program": 4, "holds": 4 },
    "pizzicato": { "source": "ldk", "program": 15, "holds": 1.5, "oneShot": true },
    "martellato": { "source": "ldk", "program": 19, "holds": 3.5 },
    "harmonic": { "source": "harmonics", "holds": 3 }
  },

  "play": {
    "bow": { "max": 4, "gap": 0.15, "change": 0.85 },
    "vibrato": { "from": 2, "mode": "vibrato" },
    "stroke": { "attack": 0.2, "swell": 0.15, "peak": 0.5, "fade": 0.2 }
  },

  "level": { "db": -6, "velocity": "sqrt" },
  "envelope": { "attack": 0.05, "release": 0.3 },

  "controls": {
    "vibrato": { "cc": 1, "default": 0, "description": "la largeur du vibrato de synthèse, par-dessus l'enregistrement" },
    "brightness": { "cc": 74, "default": 0.5 }
  }
}
```

Dans une pièce :

```js
import violon from "/instruments/violon-ldk.ison.json" with { type: "json" }; // ou fetch(...).json()
{ label: "Violon 1", synth: violon, notes }
{ pitch: 69, duration: 8, time: 0 }                       // deux coups d'archet, vibrato
{ pitch: 76, duration: 1, time: 9, variant: "pizzicato" } // une note qui choisit son mode
```

## Les champs

- `format`, `version`, `name`, `description`, `family`, `range` : qui c'est. `family` sert aux réglages par défaut (les cordes frottées entrent plus doucement).
- `sources` : les sons, par nom. Chacun est ce que jmon/sound sait déjà jouer : `{ sf2 }`, `{ sfz }`, `{ faust }`, `{ gm }`, un synthé Tone. Rien de nouveau ici.
- `modes` : les manières de jouer, par nom. Chacun dit sa source, et ce qu'il faut pour l'atteindre (`program` et `bankSelect` d'un SF2, un `keyswitch` ou un `controls` d'un SFZ, des `params` Faust). Puis ce que le son fait : `holds`, combien de secondes il tient (0 ou absent : indéfiniment) ; `oneShot` ; `vibrato` décrit (début, profondeur en cents, vitesse), pour que le jeu sache ce qu'il a déjà. Le premier mode est celui par défaut ; `variant` sur une note en choisit un autre. C'est `variants` d'aujourd'hui, généralisé à toutes les sources.
- `play` : comment on joue de l'instrument. Les règles sont appliquées aux notes avant qu'elles soient envoyées au son, du côté de jmon/sound, ou par `jm.performance` si on les veut dans le carnet. Au départ, celles qu'on connaît : `bow` (un archet dure `max` temps : une note plus longue est jouée en coups, avec `gap` entre eux, le suivant à `change` de la vélocité), `vibrato` (à partir de `from` temps, le coup passe au mode nommé), `stroke` (la forme d'un coup, les réglages de `jm.performance.bow`). D'autres viendront des instruments : un souffle (`breath`), une pédale, une résonance.
- `level` : le niveau calibré en dB, mesuré pour qu'un instrument joué à vélocité 0,6 sorte comme les autres ; et la courbe de vélocité (`linear`, `sqrt`).
- `envelope` : l'attaque et le relâchement, en secondes, s'ils remplacent ceux de la source.
- `controls` : les contrôleurs continus par nom, vers un CC, un paramètre Faust ou un `controls` SFZ, avec une plage, une valeur de départ, une description. Une pièce écrit `automation.tracks.Violon.vibrato` au lieu de `midi.cc1`.

## Ce que ça change dans jmon

- jmon/sound lit le format : il crée les sources, assigne un canal ou une voix par mode, applique `play`, `level`, `envelope`, et traduit `controls`. `variants` d'aujourd'hui en est le premier morceau.
- Le schéma jmon gagne une forme de `synth` : `{ "…": "url" }` ou l'objet inline.
- `compositions/instruments/` devient une collection de ces fichiers, avec le même en-tête (pièces, mots) dans `description` ou à côté.
- Les règles de `play` peuvent être appliquées hors du navigateur (un export MIDI vers Reaper garde les coups d'archet et les changements de mode).

## Ce qu'il n'est pas

- Pas un nouveau format d'échantillons : il ne contient pas de sons, il les désigne.
- Pas un VST : pas de code, pas de DSP ; le DSP, c'est Faust, qui est une source parmi d'autres.
- Pas obligatoire : `{ sf2, program }` et `{ sfz }` continuent de marcher tels quels.

## Où en est l'implémentation

Fait (jmon/sound 1.4.0, jmon/show, vérifié au rendu sur le quatuor LDK, le vent Faust et la baleine Tone) : `sources` (sf2, sfz, faust, gm, un synthé Tone par `type`), `modes` (un SoundFont joue ses modes comme les `variants` d'un seul instrument ; une autre source, un nœud par mode), `play` (`bow`, `vibrato`, `stroke`), `level` (dB, courbe de vélocité), `envelope` (pour un SFZ), les valeurs de départ des `controls`. Le lecteur et le rendu WAV appellent `sound.performPiece(piece)` avant de programmer les notes.

Une source Tone (`type`) : un synthé à une voix est reconnu (il relâche sans nommer la note), et un `bend` passe par son `detune` ; ses `dynamics` ne sont pas encore appliquées.

À faire : les `controls` par nom dans les lignes d'automation (aujourd'hui une ligne vise encore `midi.cc1`) ; `{ ison: "url" }` chargé par jmon/sound lui-même (aujourd'hui le carnet charge le JSON et le met dans `synth`) ; `holds` pris en compte par le son lui-même quand il n'y a pas de règle `bow`.
