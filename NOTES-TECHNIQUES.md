# Notes techniques — Derja

Comment chaque fonctionnalité est codée, et pourquoi. À relire avant toute modification.
Mise à jour : 24/10/2026 (app v2.5, banc v38).

## Fichiers
- `index.html` : l'application (web app, page unique, sans build). Ajoutée à l'écran d'accueil de l'iPhone.
- `bench.html` : banc d'essai. Toute nouvelle fonctionnalité y est mise au point avant d'aller dans l'app.
  Il sert actuellement à l'écoute continue tunisien → français.
- Hébergement : GitHub Pages (`nicolas-sallon/Derja`). Le dépôt est **public**, donc jamais de clé API dans les fichiers.
  Les clés sont saisies dans les réglages et ne sont gardées que dans le `localStorage` de l'iPhone.

## Mise à jour et version
- Chaque page affiche une constante `VERSION` (heure réelle de Tunis).
- Le détecteur relit la page avec `?check=<horodatage>` et compare `VERSION`. Si elle a changé, il propose de recharger.
- Le rechargement utilise `?v=<horodatage>` : une adresse jamais vue force GitHub Pages et iOS à contourner leur cache.

## Moteurs (choix issus des essais au banc)
- **Tunisien → français : Gemini 3.8 Flash uniquement.**
  - Au banc, 3/3 phrases justes contre 0/3 pour Gemini 3.6 Flash, 3.5 Flash-Lite et Whisper + gpt-oss.
  - Whisper invente de l'arabe standard. Groq et Whisper ont été retirés complètement.
  - Gemini 2.5 Pro répond 404 (fermé aux nouveaux comptes).
- **Réglages des appels.**
  - `thinkingConfig.thinkingLevel: "low"` et réponses « essentielles » : 15,8 s → 4,4 s de délai.
  - `responseMimeType: "application/json"` pour des réponses lisibles directement.
- **Erreur 503 (saturation)** : on réessaie le **même** modèle. Jamais de repli vers un modèle plus léger, car ils ne comprennent pas le tunisien.
- **Français → tunisien.**
  - La parole française est transcrite par la reconnaissance d'Apple (`webkitSpeechRecognition`, fr-FR). C'est gratuit, et le texte s'affiche en direct.
  - En secours, l'audio est envoyé à Gemini.
  - iOS coupe parfois la reconnaissance en pleine phrase : elle est relancée tant que le bouton reste maintenu.
  - La traduction se fait avec Gemini ou Claude, au choix dans les réglages.
- **L'arabizi** (3 = ع, 7 = ح, 9 = ق, 5 = خ, 8 = غ) est la forme écrite du tunisien partout. Les mots français restent en français.

## Audio envoyé à Gemini
- Format : WAV 16 kHz mono en base64, envoyé directement dans la requête. Gemini compte 25 jetons par seconde d'audio.
- `toWav` retire **uniquement** le silence au début et à la fin. Les pauses entre les mots restent, pour ne pas abîmer le sens.
- **Micro.** Pour écouter le tunisien, c'est toujours le micro de l'iPhone (`findPhoneMic`), jamais celui des AirPods, qui est de trop mauvaise qualité.
  Les noms des micros ne sont visibles qu'après l'autorisation du micro.
- En écoute, la réduction de bruit d'iOS est coupée par défaut : elle efface les voix éloignées.

## Voix tunisienne (synthèse)
- Modèle : `gemini-3.8-flash-lite-tts`, avec la voix Orus ou Charon (réglage).
  - Lite plutôt que Flash : 6 $ au lieu de 9 $ par million de jetons, pour une qualité jugée équivalente.
  - Les 120 voix de la bibliothèque Google ont été écartées : aucune n'est tunisienne.
  - Les voix générées ne sont pas mémorisées.
- **Lecture en continu.** On utilise `:streamGenerateContent?alt=sse`. Les morceaux PCM (L16, 24 kHz) sont décodés et joués au fil de l'eau dans Web Audio. On gagne plus d'1 s.
- **Contraintes d'iOS.**
  - L'`AudioContext` doit être créé ou relancé pendant un geste (`freshCtx`/`ensureCtx`) : toucher Traduire, Écouter, Tester…, jamais pendant que le micro écoute.
  - Web Audio est coupé par le bouton silencieux. Il y a donc un simple rappel sous la zone de saisie.
- **Pas de `navigator.audioSession`.** La bascule play-and-record/playback a été essayée puis retirée en v2.5 :
  - le son sortait faiblement par l'écouteur interne ;
  - le micro était perdu en mode silencieux.
- **Secours** : les voix de l'iPhone (`speakIos`). Une web app n'accède qu'aux voix de base, donc Majed (ar-001).

## Écoute continue (banc, pas encore dans l'app)
- **Capture.** `ScriptProcessor` de 2048 échantillons.
  - Option « Nettoyage » : un filtre passe-bande 100 Hz – 7 kHz s'applique **en direct**, donc sans délai à l'envoi.
  - Avant l'envoi, le volume est normalisé (RMS) pour que les voix lointaines soient aussi audibles que les proches. Le gain est limité à ×8 pour ne pas amplifier le bruit.
- **Seuil automatique.** C'est le 20ᵉ percentile des 120 derniers niveaux (environ 5 s), plus 8 dB. Il s'adapte au bruit de la pièce.
  Il se règle aussi au doigt : poser le doigt sur la jauge et glisser place un seuil manuel (trait orange, entre -65 et -10 dB). Ce seuil reste fixe, car le bruit de fond peut monter d'un coup et le seuil automatique suit mal. Le bouton « Auto » rétablit le réglage automatique. Le seuil manuel ne dure que la session, car le bruit change d'une pièce à l'autre. Le curseur de seuil a été retiré des réglages.
- **Découpe sur les pauses uniquement.**
  - Après 0,8 s de pause (0,5 s coupait des phrases), on envoie si le passage dure au moins 2 s.
  - Après 2,5 s de silence, on envoie dès 0,6 s.
  - Maximum 20 s par passage.
  - 0,2 s de marge avant et après le passage.
  - Moins de 2 s, c'est trop court pour que Gemini comprenne. Plus de 20 s, le délai devient trop long.
- **Contexte.** Les répliques (arabizi + français) des 3 derniers extraits sont envoyées en **texte**, jamais en audio : c'est moins cher et plus rapide.
- **Plusieurs personnes dans un extrait.** Gemini découpe l'extrait en répliques.
- **Identification des personnes.** Remplacée en v29 par la galerie de voix (section dédiée plus bas).
  - Historique v17 : une fiche descriptive remplie par Gemini (genre, âge, hauteur, texture, débit), avec un rapprochement par l'app sur le trio genre + âge + hauteur.
  - Abandonnée parce que la description variait pour une même personne, ce qui donnait des dédoublements et des fusions à tort.
- **Affichage façon messagerie.**
  - Les bulles sont à gauche, teintées à 34 % de la couleur de la personne.
  - Couleurs : bleus et verts pour les hommes (`#7cc4ff`, `#7fe0a8`, `#9b9cff`), tons chauds pour les femmes (`#ff9fc4`, `#ffad85`, `#f0cf5a`), gris `#9a9ca6` pour une voix inconnue.
  - Le bleu de l'app `#3d8bff` est réservé à tes propres répliques, à droite, en couleur pleine (futur bouton « Répondre »).
- **Mesures.**
  - Plus de boutons d'appréciation (Juste/Approx./Faux, Locuteur ✓/✗) : le test porte maintenant sur la tenue dans la durée. Le bouton ▶︎ de réécoute de chaque extrait est conservé.
  - Le volet sous chaque extrait ne montre que le bouton de réécoute et, pour chaque réplique, la voix retenue, le code renvoyé par Gemini, la hauteur mesurée, l'écart de timbre et un éventuel refus du garde-fou. La durée de préparation (20 ms au plus, tout activé) n'est plus affichée ni mesurée dans le tableau.
  - Réécoute : un seul extrait à la fois. En lancer un autre arrête le premier, et toucher « ■ Arrêter » coupe la lecture.
  - v36 : le tableau des mesures est retiré, remplacé par les « Statistiques d'utilisation » (section Discussions).
  - Réglages : `localStorage["derja-bench-listen"]`.

## Deux boutons et réponse « Je parle » (banc v27)
- **Disposition.** Deux boutons de même taille sous le sélecteur à 3 cases, qui reste en place pour l'instant.
  - À gauche, « J'écoute » : un toucher lance ou arrête l'écoute continue. Il devient rouge, « Arrêter l'écoute », quand elle est active.
  - À droite, « Je parle » : à **maintenir** pendant la phrase. Bleu clair au repos, bleu vif pendant l'appui.
- **Un seul circuit audio pour la session** (`ensureCtx`) : créé au premier geste, jamais fermé. Il sert à capter et à jouer la voix.
  - Le micro s'ouvre et se ferme à part (`openMic`/`closeMic`).
  - Le micro est **fermé pendant la voix tunisienne**. Micro ouvert, iOS passe en mode lecture-et-enregistrement et joue le son dans l'écouteur, faiblement (constaté sur l'app avant la v2.5).
- **Capture du français : Gemini audio** (choix validé), avec le micro déjà ouvert pour l'écoute.
  - Pas de dictée d'Apple : elle risquait d'entrer en conflit avec le micro de l'écoute continue sur iOS.
  - Ce qu'on perd : le texte français ne s'affiche pas pendant qu'on parle.
- **Appui sur Je parle.**
  - Si un passage était en cours chez l'interlocuteur, il part en traduction (au moins 0,6 s). Puis la découpe s'arrête.
  - Les images du micro vont dans le tampon de la réponse (`talkBuf`). Elles n'entrent pas dans la mesure du bruit de fond.
  - Si l'écoute était arrêtée, le micro s'ouvre juste le temps de la phrase.
- **Relâchement.**
  - Le micro se ferme.
  - Moins de 0,5 s, ou moins de 0,3 s de voix au-dessus du seuil : annulé, avec un message.
  - Sinon, le son est préparé comme les extraits (`prepWav`), puis envoyé à Gemini (`replyPrompt`), qui renvoie `francais`, `arabizi` et `arabe`.
  - La traduction est au masculin. Le registre vient de l'app. Le contexte reprend les 3 derniers échanges.
- **Voix.**
  - Elle utilise la voix choisie dans l'app (Orus ou Charon), lue depuis `localStorage["derja-express-v1"]` : même site, donc stockage partagé.
  - Elle est lue en continu, comme dans l'app. Le son reçu est gardé dans un WAV pour la réécoute, sans nouvel appel.
- **Réécoute (v28).** Le volet sous ma bulle a deux boutons, « ▶︎ Français » et « ▶︎ Tunisien ». Un seul son est joué à la fois (`playingId` = `id:fr` ou `id:tn`).
  - Français : mon enregistrement, tel qu'envoyé à Gemini (`urlFr`).
  - Tunisien : la voix de Gemini (`url`).
  - Utile pour savoir si une erreur vient de la compréhension de ma voix ou de la traduction.
  - Toucher Je parle pendant la voix l'arrête.
- **Reprise.** 0,3 s après la fin de la voix, l'écoute reprend si elle était active à l'appui (`resumeAfter`).
- **Bouton de gauche pendant ma réponse (v28)** : il est **inactif**, du premier appui jusqu'à la fin de la voix (choix validé).
  - Il affiche « Écoute en pause » en gris si l'écoute reprendra ensuite.
  - Sinon il affiche « J'écoute ».
  - Il redevient « Arrêter l'écoute » en rouge à la reprise.
- **Contexte.** Mes réponses entrent dans le contexte écrit des extraits suivants, sous la forme « Moi : arabizi (= français) ».
- **Affichage.** Ma bulle est à droite, en bleu plein `#3d8bff`, avec le français en gros et l'arabizi en dessous. Sous la bulle : heure, durée, « voix après X s » (du relâchement au début de la voix) et le volet de réécoute.
- **Mesures.** La ligne « Réponses (Je parle) » donne le nombre de réponses et le délai moyen avant la voix. Le coût inclut la voix : 6 $ par million de tokens audio.

## Contexte et galerie des voix (banc v29)
Choix validés avec Nicolas le 23/10 après plusieurs itérations écrites.
- **Contexte libre** (v29 : `cfg.ctxText` dans les réglages ; v36 : propre à chaque discussion, en haut de la page) : décrit la conversation (lieu, personnes, sujet). Il est envoyé avec chaque extrait et chaque réponse « Je parle » (`ctxBlock`).
- **Galerie permanente**, dans l'iPhone (IndexedDB `derja-voices`), 20 voix au plus.
  - **Contenu d'une voix :**
    - un code fixe (`V3`) ;
    - un nom, proposé par Gemini (`suggest`, affiché « V3 · grand-mère ? ») puis modifiable. Une fois nommée par l'utilisateur (`userNamed`), Gemini ne la renomme plus ;
    - le genre et la couleur. La première couleur est proposée automatiquement : tons froids pour un homme, chauds pour une femme, la moins utilisée du groupe. Cette répartition ne sert qu'à ce premier choix ;
    - une référence audio de 5 s (Int16, 16 kHz) ;
    - la hauteur médiane (`f0`) et l'empreinte de timbre (`mfcc`) ;
    - un score de qualité (`voicedSec`, `snr`) et des marqueurs (`provisoire`, `importee`, `doubt`) ;
    - la référence Google (`file`) et sa version (`refVer`).
  - **Numéros des voix (v36).**
    - Ils sont **uniques pour toujours** : V1, V2… ne sont jamais réutilisés, et les correspondances de fusion (`meta.aliases`) sont permanentes.
    - Pourquoi : les discussions archivées font référence aux codes des voix. La renumérotation à l'ouverture (v33) a été retirée, car elle aurait rattaché d'anciennes répliques à la mauvaise personne.
    - Chaque réplique garde un instantané (`snap` : nom et couleur), affiché en gris si la voix a été supprimée ou n'a jamais été gardée.
    - Les références chez Google ne dépendent pas du numéro.
  - **Galerie pleine :** la voix sans nom entendue il y a le plus longtemps sort. Si toutes ont un nom, la voix nouvelle reste temporaire.
  - **Ouverture depuis l'écran d'accueil impérative :** Safari peut effacer les données d'un site non visité pendant 7 jours, et le stockage d'un onglet est séparé de celui de l'app de l'écran d'accueil.
- **Références chez Google (API Files).**
  - Envoi « multipart » ; à défaut, « resumable ».
  - Si l'API Files est inaccessible depuis la page, repli automatique (`filesMode = "inline"`) : les références sont jointes à chaque appel. Les réglages affichent le mode en cours.
  - Une référence est renvoyée si elle est absente ou expire dans moins de 10 min (`ensureFile`). Les fichiers durent 48 h chez Google.
  - Une ancienne référence n'est effacée que **5 min après** le basculement (`deleteFileLater`), car un appel en cours peut encore l'utiliser.
- **Sélection des références,** recalculée **à chaque appel** (`selectRefs`, 6 au plus) :
  - tant qu'aucune voix n'est présente : les voix nommées dans le contexte, puis les plus récentes ;
  - ensuite : seulement les voix présentes et celles du contexte. C'est moins cher, et Gemini a moins de voix à départager.
  - Un nouveau venu est rattrapé par la vérification des doublons.
- **Coche « présente » (v36 : propre à chaque discussion, voir plus bas) :** mise automatiquement à la reconnaissance, ou à la main. Avant la v36, tout était décoché à l'arrêt de l'écoute continue, pas pendant « Je parle ».
- **Appel d'un extrait.**
  - Contenu : chaque référence précédée de « Voix de référence V3 (« Mamie ») : », puis « Extrait à traduire : » et l'audio.
  - Gemini renvoie pour chaque réplique : `voix` (code ou « nouvelle », jugé sur le timbre), `nom`, `genre`, `debut`/`fin` (en secondes), `entendu`, `francais`.
- **Attribution par l'app** (`attribute`) :
  - **Code connu :** garde-fou sur la **hauteur seulement**. Refus, réplique en « ? », si la hauteur mesurée est à plus de 5 demi-tons de celle de la voix et que le morceau ne contient qu'une voix. L'écart de timbre est seulement affiché, pour calibrer plus tard.
  - **« Nouvelle » :** une voix est créée si le morceau isolé ne contient qu'une voix et au moins 2 s de voix.
    - Extrait à une seule réplique : on prend l'extrait entier.
    - Extrait à plusieurs répliques : on découpe le morceau d'après `debut`/`fin`, en retirant 0,5 s de chaque côté, et la référence est **provisoire**.
    - Sinon, la réplique affiche le nom proposé ou « ? ».
  - **Mesures locales** (`analyze`, environ 10 à 50 ms pour 5 s, sans latence ajoutée) :
    - énergie par trame de 10 ms ;
    - hauteur YIN à 8 kHz (70 à 470 Hz), médiane ;
    - contrôle « une seule voix » : au moins 70 % des trames à moins de 4 demi-tons de la médiane ;
    - MFCC sur 12 coefficients, 26 filtres mel de 100 Hz à 7 kHz ;
    - meilleure fenêtre de 5 s : celle qui contient le plus de voix.
    - Validées sur des voix synthétiques : 120, 220 et 300 Hz bien mesurés, un mélange 120 + 230 Hz est bien refusé.
  - **Genre à vérifier :** signalé si le genre contredit la hauteur (homme au-dessus de 185 Hz, femme en dessous de 140 Hz, enfant en dessous de 190 Hz).
- **Traitements de fond** (`bgQueue`) : ils ne passent que lorsqu'aucun extrait ni aucune réponse n'est en traduction (compteur `inflight`).
  - **Doublons :** à chaque voix nouvelle, on cherche des voix de hauteur proche (à 4 demi-tons près), classées par écart de timbre, 12 au plus. Aucun appel s'il n'y en a pas. Sinon, Gemini répond {"meme":"V…"/"aucune"}, et une réponse positive déclenche la fusion.
  - **Fusion** (`mergeVoice`) : une correspondance (`meta.aliases`, V8 → V3) garde justes les bulles déjà affichées et les réponses qui arrivent après. Si la cible est provisoire, elle reprend la meilleure référence. Le nom donné par l'utilisateur est conservé.
  - **Remplacement d'une référence** (`considerReplace`) :
    - Conditions : un extrait à une seule voix, attribué à cette voix, avec au moins 3 s de voix et nettement meilleur (+3 dB de clarté, ou +1 s de voix).
    - Fréquence : au plus un remplacement toutes les 10 min, sauf pour les références provisoires ou importées.
    - Confirmation par Gemini : {"meme":true/false}.
    - La nouvelle version est envoyée chez Google **avant** le basculement, qui est instantané. Le code de la voix ne change pas, donc aucun appel en cours ne peut voir apparaître une voix « nouvelle ».
- **Galerie dans les réglages :** écouter, renommer, cocher « présente », changer la couleur, fusionner (⋯ → « = autre voix »), supprimer.
  - **Genre (v34)** : un toucher sur la petite ligne sous le nom ouvre Homme / Femme / Enfant.
    - Le genre choisi est marqué confirmé (`genreOk`), et « genre à vérifier » ne s'affiche plus.
    - S'il contredit la hauteur mesurée, une confirmation est demandée une seule fois.
    - La couleur ne change pas.
    - Le genre choisi lors d'un ajout manuel compte aussi comme confirmé.
  - **« Nouvel extrait… » (v34, menu ⋯)** : même panneau que l'ajout, titré « Nouvel extrait pour X », avec le nom et le genre préremplis.
    - La référence est remplacée sans avis de Gemini, puisque c'est le choix de l'utilisateur (`replaceRef`).
    - Confirmation si la hauteur est à plus de 5 demi-tons de l'ancienne référence.
    - L'extrait est marqué « importée » s'il vient d'un fichier.
- **Couleurs (v31).** Un toucher sur le rond ouvre un nuancier sous la ligne, sans titres de catégories, dans cet ordre :
  - hommes : bleu ciel, vert menthe, pervenche, turquoise, vert d'eau ;
  - femmes : rose poudré, pêche, jaune doré, corail, lilas ;
  - enfants : jaune citron, sable ;
  - puis les couleurs personnalisées déjà utilisées, et « + », qui ouvre la roue de couleurs de l'iPhone (`input type=color`).
  - Un petit point marque une couleur déjà prise par une autre voix ; on peut la choisir quand même. Le choix s'applique tout de suite, dans la galerie et dans les bulles.
  - Le nom affiché au-dessus des bulles est éclairci si la couleur est trop sombre pour le fond noir (`readable`). La bulle garde la couleur choisie.
- **Ajouter une voix à la main (« + »)** :
  - soit enregistrer en maintenant le bouton, l'écoute étant arrêtée ;
  - soit importer un fichier audio ou vidéo, décodé par `decodeAudioData`. Le sélecteur n'a **aucun filtre de type** : avec `accept="audio/*,video/*"`, iOS grisait certains fichiers (même correctif que pour l'app) ;
  - l'app choisit les 5 meilleures secondes (avec « Passage suivant » pour en changer), le nom est obligatoire et le genre est prérempli d'après la hauteur ;
  - un fichier importé est marqué « importée », parce que son micro et sa pièce sont différents : il peut être remplacé par un bon extrait de conversation.
  - **Vérification après l'ajout (v31, `checkManualDup`)** : immédiate, sans passer par la file de fond, puisque c'est l'utilisateur qui l'a lancée. Même tri par hauteur et même question à Gemini que pour les doublons.
    - Pas de fusion automatique : un bandeau « Ressemble à X : [Fusionner] [Garder les deux] » s'affiche sur la ligne.
    - Fusionner : le **nom de la voix existante** est gardé, puisque c'est celui que l'utilisateur connaît déjà (choix du 23/10). Le nouveau nom ne sert que si la voix existante n'en avait pas. La meilleure des deux références est gardée.
    - Garder les deux : la paire est mémorisée (`notDup`), et la question n'est plus posée.
    - Pendant la conversation, rien ne change. Une voix nouvelle n'a jamais de nom donné par l'utilisateur, donc la fusion y reste automatique et garde le nom déjà donné.
- **Menu sur le nom ou la bulle d'une réplique (v34)** : un toucher sur le nom au-dessus des bulles ou sur une bulle ouvre un menu sous la bulle.
  - **Voix de la galerie :**
    - « Accepter « nom proposé » » ;
    - « Renommer… » ;
    - « C'est quelqu'un d'autre… » (v35), puis le choix d'une voix ou de « Inconnu ».
    - Pour une autre voix, une question de portée suit :
      - « Seulement cette réplique » : correction de cette seule réplique ;
      - « Toujours : V5 est Mamie » : fusion des deux voix, la voix choisie garde son nom.
    - Pourquoi : en v34, « C'est en fait… » (fusion) et « Ce n'est pas X, c'est… » (correction) ouvraient la même liste. Le risque était de fusionner par erreur deux personnes.
  - **Réplique sans voix (nom en gris) :**
    - « C'est… » ;
    - « Créer la voix « nom » » : seulement si le morceau dure au moins 1,5 s. Référence provisoire si l'extrait contient plusieurs répliques, ou si le morceau a peu de voix ou plusieurs voix.
  - **Une correction vaut confirmation** (`improveFromCorrection`) : si le morceau ne contient qu'une voix, avec au moins 3 s de voix, et qu'il est meilleur que la référence actuelle (ou que celle-ci est provisoire ou importée), il la remplace, sans demander à Gemini.
  - **Audio conservé :** celui des 60 derniers extraits (`it.x16`), pour ces corrections et créations.
  - **Saisie protégée :** le texte tapé dans le menu est conservé quand le fil se met à jour, et le fil ne défile pas tant que le menu est ouvert.
- **Mesures :** la ligne « Voix (session) » donne la moyenne des références jointes par appel, les voix nouvelles, les fusions, les remplacements, les « ? », les refus par la hauteur, et le nombre et le coût des appels de fond.

## Discussions (banc v36)
Choix validés avec Nicolas le 24/10.
- **Accueil (`conv = null`).**
  - La liste des discussions, la plus récente en haut (selon le dernier message, sinon la création).
  - Une carte : titre, « Créée … · durée d'audio · poids · dernier message … », ou « aucun message ».
  - Liste vide : un mot d'accueil et « L'historique de tes discussions apparaîtra ici. »
  - En bas, à la place des boutons J'écoute et Je parle : « Démarrer une nouvelle discussion ».
- **Nouvelle discussion.**
  - Un panneau « De quoi va-t-on parler ? », avec le contexte et un seul bouton, « Commencer ».
  - La discussion est **archivée même vide**, pour qu'on la retrouve avec son contexte. La date affichée est celle de la création.
- **Titre local uniquement** (`titleFrom`) : le début du contexte, jusqu'au premier « : . ; , », 6 mots au plus. Sans contexte : « Discussion du jj/mm à hh:mm ».
  - Il est modifiable en le touchant (`titleUser`). Sinon, il suit le contexte.
  - Il n'est jamais envoyé à Gemini : le contexte suffit.
- **Page de discussion.**
  - En haut : « ‹ Discussions », le titre et le contexte sur une ligne. Un toucher sur le contexte le déplie pour le modifier ; le nouveau contexte s'applique aux appels suivants.
  - Le contexte n'est plus dans les réglages. `ctxBlock(cid)` et `contextLines(id, liste)` utilisent la discussion du message.
- **Stockage (IndexedDB `derja-convs`)** :
  - `c` : une fiche par discussion (contexte, titre, voix présentes, `audioSec`, `bytes`, `nMsg`, `lastAt`) ;
  - `m` : un enregistrement par message (texte JSON, sans audio) ;
  - `a` : l'audio — `x` l'extrait, `fr` mon français, `tn` la voix tunisienne.
  - On demande à iOS de garder les données de façon persistante (`navigator.storage.persist`).
- **Sauvegarde au fil de l'eau**, car iOS peut fermer l'app sans prévenir :
  - un extrait est enregistré dès son envoi, son audio avant l'appel à Gemini, puis le texte au retour ;
  - ma réponse est enregistrée au relâchement, avec mon français, puis la traduction, puis la voix tunisienne ;
  - le contexte, le titre, les coches et les corrections du menu sont enregistrés à chaque changement.
- **Traduction interrompue :** un message resté « en cours » est affiché « Traduction interrompue » à la réouverture, avec un bouton « Traduire ».
  - Pour un extrait : relancé depuis l'audio enregistré (`retryItem`).
  - Pour ma réponse : relancée depuis mon français enregistré, puis dite (`retryMe`), seulement écoute arrêtée.
- **Reprise d'une discussion :** le fil et l'audio sont rechargés, donc la réécoute et les menus fonctionnent. On peut continuer la discussion.
  - Les messages chargés restent en mémoire pour la session (`convItems`) : une traduction en cours se termine et s'enregistre même si on quitte la discussion.
  - Les identifiants de messages sont uniques et croissants (`newId`).
- **Voix présentes par discussion** (`PresentSet`, enregistrées dans la fiche) :
  - aucune coche dans une nouvelle discussion, donc sélection de départ (contexte, puis voix récentes) ;
  - cochées automatiquement à la reconnaissance, ou à la main ;
  - **arrêter l'écoute ne décoche plus rien** ;
  - quitter la discussion arrête l'écoute et garde les coches dans la discussion ;
  - à la réouverture, les coches sont restaurées, avec les fusions suivies et les voix supprimées retirées ;
  - hors discussion, les coches sont masquées dans la galerie.
- **Dates dans la galerie :**
  - « entendue » = dernière réplique attribuée (`lastHeard`). C'est elle qui sert aux « voix récentes » ;
  - « extrait du » = date de la référence (`refAt`).
- **Supprimer une discussion :** glisser la carte vers la gauche, toucher la poubelle rouge, confirmer. Les messages, l'audio et la fiche sont effacés ; la galerie n'est pas touchée.
- **Poids :** environ 1,9 Mo par minute d'audio envoyé, et 2,9 Mo par minute de voix tunisienne. Une discussion d'une heure fait environ 40 à 50 Mo.
- **Statistiques d'utilisation** (réglages, compteurs permanents `cfg.usage`) :
  - discussions, messages, audio envoyé, poids, place occupée (`navigator.storage.estimate`), voix ;
  - jetons par type : écoute, réponses, voix tunisienne, appels de fond ;
  - coût estimé au tarif du moment (doublé à partir du 01/01/2027), à 0,92 € le dollar.
- **Solde estimé :** l'utilisateur recopie le solde affiché dans AI Studio. L'app retient la date et déduit sa propre consommation depuis (`cfg.balance`).
  - Aucune API ne permet de lire le solde avec une clé Gemini.
- **Crédit épuisé :** une erreur 429 ou 402 qui mentionne prepay, credit, billing ou depleted affiche « Crédit Gemini épuisé : recharge dans AI Studio » (`apiErr`).

## Tarifs (pour estimer)
- Gemini 3.8 Flash : 0,75 $ par million de jetons en entrée, 3,75 $ par million en sortie.
- Synthèse vocale : Flash 9 $, Lite 6 $ par million de jetons en sortie.
- Les prix doublent au 01/01/2027.

## Zoom et orientation (banc v37)
- **Pas de zoom :**
  - balise viewport avec `maximum-scale=1, user-scalable=no` ;
  - `touch-action: manipulation` contre le zoom au double toucher ;
  - `gesturestart` et `gesturechange` bloqués contre le pincement ;
  - champs de saisie à 16 px au moins, car iOS zoome automatiquement sur un champ plus petit quand on le touche.
- **Orientation :** iOS ignore le champ `orientation` du manifeste. `screen.orientation.lock("portrait")` n'est que partiellement pris en charge (16.4+) : il est tenté au premier toucher, sans effet si iOS refuse. La rotation par CSS a été abandonnée (v21 à v25).
- **iPhone sur le côté (v38) :** un écran plein « Remets ton iPhone en vertical » s'affiche par-dessus la page (`#rotate`).
  - Il est déclenché en CSS seul (`orientation: landscape` et hauteur de 500 px au plus), donc sans effet sur un iPad ou un écran large.
  - L'écoute et les traductions continuent dessous.

## Essais abandonnés
- **Affichage retourné / toujours vertical (banc v21 à v25, retiré en v26 = v20).**
  - Les iPhone avec Face ID n'affichent jamais l'écran tête en bas, et une web app ne peut pas verrouiller l'orientation.
  - On a essayé de lire l'inclinaison (`deviceorientation`) et de faire pivoter la page en CSS, y compris pour contrer le paysage.
  - Retourné en portrait : fonctionnel. En paysage depuis l'écran d'accueil : le contenu restait tassé sur la moitié de l'écran, malgré des tailles correctes.
  - Autre inconvénient : iOS redemande l'autorisation des capteurs de mouvement à chaque ouverture.
  - Décision : abandonné. À reconsidérer seulement dans une app native.

## À venir
- Fusion du banc et de l'app : une seule page de discussion, plus un mode « apprendre » relié aux discussions (mot à mot, variantes, notes culturelles, fiches, exercices).
- Écoute continue portée dans l'app comme 3ᵉ mode, une fois validée au banc.
