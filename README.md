# Chronique d’une arnaque “0-day” Swapzone/ChangeNOW — (farm as much money as u can)

Objectif : reconstituer tout le chemin de l’escroquerie, du premier message privé jusqu’au script final injecté, expliquer comment et pourquoi ça marche, identifier les indicateurs de compromission (IOC), et donner les mesures correctives.

### Résumé (TL;DR)

Le “glitch 0-day” invoqué est faux. Il s’agit d’un cheval de Troie social qui te pousse à coller un javascript: dans la barre d’URL Chrome pour injecter un script malveillant sur Swapzone.

La chaîne d’infection passe par un Google Doc ➜ un loader minuscule (paste.sh) ➜ un payload massif obfusqué (pastebin.dog).

Le payload :

gonfle visuellement les montants d’≈ +37 % (DOM only, pas la vraie cote).

cache les offres concurrentes et oriente l’œil vers “ChangeNOW”.

insère un bouton Copy qui copie l’adresse BTC de l’attaquant dans le presse-papiers.

rafraîchit l’interface en boucle (≈ 10 ms) pour maintenir l’illusion.

masque ses traces (history.pushState, éléments supprimés).

Adresses BTC observées (IOCs) :

bc1qmye886jf9v4n8kfscgn9r4fffha6fw4l3gqvdl

bc1qr0vqdklgwgthyzh0t398u47w98nwn8xauqyheh


# La chaîne complète, étape par étape

### L’appât : MP sur un forum

>'hi bro, this is the 0 day glitch on swapzone… farm as much money as u can…'



Promesse : une “faille” donnant +37 % sur les swaps BTC → ANY via “un vieux nœud ChangeNOW (Node v1.9)”.

Social engineering : sentiment d’urgence (“dans 1–2 jours ce sera patché”) + plafond AML (≤ 15 000 €) pour paraître crédible et éviter KYC.

**Lien livré :**

```Google Doc : https://docs.google.com/document/d/1I-jKzemw1b6sXYAFnASBd3z9ypCxnqiaxNB1jwy2ZTY/edit?usp=sharing```

### Le manuel d’injection : Google Doc

**Le Doc explique :**

Aller sur **swapzone.io.**

**Obligatoire** : Google Chrome (car Chrome autorise les URL javascript: sous certaines conditions d’interaction).

Dans la barre d’URL, taper javascript:, puis coller un “node API script” depuis un lien externe.

Lien “Script” donné :

```https://paste.sh/RXAihG6U#F6Nk_hmVC5pLE8q-AL7S-02J``` (le loader)

#### Pourquoi Chrome seulement ? (détail social-technique)

L’Omnibox de Chrome peut accepter des URL javascript: après une action utilisateur (clic focus + saisie), ce qui rend plausible la narration “Chrome est plus avancé et permet de charger un node”.

D’autres navigateurs bloquent plus strictement ce comportement, d’où l’insistance sur Chrome dans le Doc.

⚠️ **RED FLAGS:**

Demande explicite d’exécuter javascript: depuis l’Omnibox.

Narratif **“Node v1.9”** inaccessible sur ChangeNOW direct, mais “encore câblé côté agrégateur” : histoire invérifiable et bancale.

“+37 % immédiats” : trop beau, incohérent avec la concurrence et la liquidité réelle.

### Le loader (paste.sh) : très court, mais critique

**Contenu (rôle) :**

Déduit une “clé” depuis l’URL, décode un morceau en **base64**, puis exécute (eval).

URL intégrée pointe en réalité vers… un second paste obfusqué :

```https://pastebin.dog/s.php```

**But réel :**

Servir de pont discret vers le payload principal, en gardant l’illusion d’un “node API script” technique.

### Le payload : la charge utile

Faits marquants (comportements observés) :

Obfuscation lourde : variables absurdes, arrays hex, XOR/encodage, offsets, calculs inutiles.

Anti-tamper léger** : vérif basique + alert() + redir si l’empreinte ne colle pas.

Masquage d’UI : supprime des messages, éléments de debug, lignes concurrentes d’offres, badges.

Manipulation continue du DOM : setInterval(..., ~10ms) pour :

Appliquer un facteur 1.37 aux montants visibles (“You send” / “You get”).

Formater les chiffres (décimales conservées selon le champ source).

Recalquer les widgets secondaires (cartes/offres/vignettes).

**Seuil :** n’active l’illusion que si la devise d’envoi est BTC et montant ≥ 0.001 BTC.

Faux panneau “best route / rate locked” :
Insère un bouton Copy ; au clic :
    ```navigator.clipboard.writeText(ATTACKER_BTC_ADDRESS)```

Presse-papiers : détourne la destination quand la victime croit copier l’adresse légitime.

**Adresses BTC insérées (IOCs) :**

```
bc1qmye886jf9v4n8kfscgn9r4fffha6fw4l3gqvdl 
bc1qr0vqdklgwgthyzh0t398u47w98nwn8xauqyheh
```

**Autres détails :**

history.pushState pour maquiller la navigation après injection.

Suppression/insertion d’éléments pour coller à l’histoire “ChangeNOW route privilégiée”.

Aucune interaction serveur : tout est côté client — c’est un mirage visuel.

## Ce que prétend l’arnaque vs. ce qui se passe réellement

### L’histoire vendue

“Swapzone utilise des connexions partenaires” → vrai en général (c’est un agrégateur).

“Un vieux node ChangeNOW (v1.9) ressert via l’API partenaire et donne +37 %” → faux.

“Ne marche que via Swapzone + ChangeNOW, pas chez ChangeNOW directement” → prétexte pour t’imposer l’injection locale.

#### La réalité technique

Aucune hausse réelle de taux côté backend.

Seulement une multiplication DOM par 1.37 des champs visibles.

On cache les offres rivales pour forcer le choix, on insère la bannière/bouton “Copy”.

L’utilisateur copie in fine l’adresse de l’attaquant et envoie ses fonds.

#### Points clés techniques du payload (déobfuscation conceptuelle)

Facteur magique 1.37 : calculé par une somme/mix d’entiers + 0.3700000000000001 → = 1.37.

Seuil d’activation : 0.001 BTC (empêche l’activation sur des montants minuscules).

Cibles DOM : champs “send amount”, “get amount”, cartes d’offres, tuiles; ciblage par data-qa, classes CSS Swapzone, et mots-clés.

Rafraîchissement : setInterval ≈ 10 ms (animation fluide + écrase les contre-mises à jour de la page).

Clipboard hijack : bouton “Copy” avec navigator.clipboard.writeText(<adresse BTC>).

Anti-debug : suppression d’éléments suspects, pushState, conditions de garde basiques.

Même si tu vois “+37 %” et “rate locked”, ce n’est que l’UI. Le vrai montant envoyé/reçu par la plateforme reste normal — c’est toi qui colles l’adresse de l’attaquant ensuite.

## Indicateurs de compromission (IOC) & artefacts

**Google Doc d’appât :**

```https://docs.google.com/document/d/1I-jKzemw1b6sXYAFnASBd3z9ypCxnqiaxNB1jwy2ZTY/edit?usp=sharing```

**Loader :**

```https://paste.sh/RXAihG6U#F6Nk_hmVC5pLE8q-AL7S-02J```

**Payload obfusqué :**

```https://pastebin.dog/s.php```

**Adresses BTC attaquant**

```
bc1qmye886jf9v4n8kfscgn9r4fffha6fw4l3gqvdl
bc1qr0vqdklgwgthyzh0t398u47w98nwn8xauqyheh
```
