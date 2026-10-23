# Notes techniques — Derja

Comment chaque fonctionnalité est codée, et pourquoi. À relire avant toute modification.
Mise à jour : 23/10/2026 (app v2.5, banc v28).

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
- **Identification des personnes (v17).**
  - Gemini remplit seulement une fiche à cases fermées par réplique : genre, âge, hauteur, texture, débit.
  - Il a pour consigne d'ignorer le volume, la distance, l'émotion, le chuchotement ou les cris, et le contenu.
  - C'est l'**app** qui attribue l'identité (`classify`) : même trio genre + âge + hauteur = même personne.
  - Texture et débit sont seulement affichés, car ils varient trop pour une même personne.
  - Pourquoi : quand Gemini choisissait lui-même l'identifiant, il réutilisait P1 pour des voix différentes. L'analyse du signal côté app a été jugée trop complexe.
  - Limites : deux personnes au trio identique sont fusionnées, et une hésitation de Gemini sur l'âge coupe une personne en deux.
- **Affichage façon messagerie.**
  - Les bulles sont à gauche, teintées à 34 % de la couleur de la personne.
  - Couleurs : bleus et verts pour les hommes (`#7cc4ff`, `#7fe0a8`, `#9b9cff`), tons chauds pour les femmes (`#ff9fc4`, `#ffad85`, `#f0cf5a`), gris `#9a9ca6` pour une voix inconnue.
  - Le bleu de l'app `#3d8bff` est réservé à tes propres répliques, à droite, en couleur pleine (futur bouton « Répondre »).
- **Mesures.**
  - Plus de boutons d'appréciation (Juste/Approx./Faux, Locuteur ✓/✗) : le test porte maintenant sur la tenue dans la durée. Le bouton ▶︎ de réécoute de chaque extrait est conservé.
  - Le volet sous chaque extrait ne montre que le bouton de réécoute et la fiche de chaque voix. La durée de préparation (20 ms au plus, tout activé) n'est plus affichée ni mesurée dans le tableau.
  - Réécoute : un seul extrait à la fois. En lancer un autre arrête le premier, et toucher « ■ Arrêter » coupe la lecture.
  - Le coût est calculé à partir de `usageMetadata`.
  - Les statistiques sont regroupées par configuration : nettoyé ou brut, réduction iPhone, personnes.
  - Stockage : `localStorage["derja-bench-listen"]`.

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

## Tarifs (pour estimer)
- Gemini 3.8 Flash : 0,75 $ par million de jetons en entrée, 3,75 $ par million en sortie.
- Synthèse vocale : Flash 9 $, Lite 6 $ par million de jetons en sortie.
- Les prix doublent au 01/01/2027.

## Essais abandonnés
- **Affichage retourné / toujours vertical (banc v21 à v25, retiré en v26 = v20).**
  - Les iPhone avec Face ID n'affichent jamais l'écran tête en bas, et une web app ne peut pas verrouiller l'orientation.
  - On a essayé de lire l'inclinaison (`deviceorientation`) et de faire pivoter la page en CSS, y compris pour contrer le paysage.
  - Retourné en portrait : fonctionnel. En paysage depuis l'écran d'accueil : le contenu restait tassé sur la moitié de l'écran, malgré des tailles correctes.
  - Autre inconvénient : iOS redemande l'autorisation des capteurs de mouvement à chaque ouverture.
  - Décision : abandonné. À reconsidérer seulement dans une app native.

## À venir
- Écoute continue portée dans l'app comme 3ᵉ mode, une fois validée au banc.
- Renommer ou corriger une personne à la main.
