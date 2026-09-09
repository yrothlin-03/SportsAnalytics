# Notes de conception — pipeline pose + foundation model temporel

## Objectif
S'inspirer de Pose2Sim/OpenCap et ajouter un modèle de fondation pour séries temporelles en aval, pour faire de l'évaluation de performance et de l'analyse de mouvement à partir de vidéo.

## Pipeline envisagé
1. **Extraction cinématique** — keypoints via rtmlib (RTMPose Wholebody), déjà prototypé sur `bench_ludo.mov`.
2. **Encodeur foundation model** — extraire un embedding par fenêtre/rep à partir des séries d'angles articulaires. Candidats : MOMENT (AutonLab, orienté classification/anomalie — favori), Chronos (Amazon), Moirai (Salesforce), TimesFM (Google).
3. **Tête de tâche** — zero-shot par similarité : comparer l'embedding d'une rep à une banque de reps de référence (pas besoin de gros dataset labellisé). Sert à la fois pour :
   - évaluation de performance (score = similarité à la référence la plus proche)
   - analyse de mouvement (même distance lue localement pour situer où ça déraille)

## Question ouverte : 2D ou 3D ?
Pose2Sim est fondamentalement un pipeline 3D (calibration multi-caméra + triangulation DLT + scaling/IK OpenSim), précisément parce qu'un angle articulaire mesuré en 2D dépend du point de vue caméra — pas fiable pour comparer des reps entre elles si l'angle de vue change.

Options si on reste en monoculaire :
- (a) repasser à 2 caméras minimum — le plus fidèle à l'esprit Pose2Sim, triangulation réelle
- (b) lifter 2D→3D appris (type MotionBERT) — reste monoculaire, 3D approximative
- (c) rester en 2D pur — viable seulement si la caméra garde une position fixe/identique entre référence et évaluation

**Pas encore tranché.**

## Données
- Prototype existant : rtmlib/RTMPose Wholebody sur `bench_ludo.mov` (exercice bench press)
- Dataset externe repéré : [Mayank022/bench-press-deadlift-exercises](https://huggingface.co/datasets/Mayank022/bench-press-deadlift-exercises)
  - 93 vidéos (75 train / 9 val / 9 test), bench press + deadlift, plusieurs angles de caméra
  - MP4 H.264, seulement 8 FPS, 12.7 MB au total
  - Labels = type d'exercice uniquement (bench vs deadlift), **pas de label de qualité de forme**
  - Utile pour tester la robustesse au point de vue caméra / diversité de sujets, pas pour superviser la qualité de forme
  - Limite : 8 FPS est bas pour capturer la dynamique fine d'une rep rapide, à garder en tête

## Prochaines étapes
- Trancher 2D vs 3D (voir ci-dessus)
- Construire le module de segmentation en reps + extraction des angles depuis les sorties rtmlib
- Constituer une petite banque de reps de référence (variée : plusieurs sujets/reps, pas juste un individu)
- Tester MOMENT en zero-shot sur les premières fenêtres extraites

## Mise à jour — détection de mouvements dangereux (2026-09-09)

**Option 2D/3D n°4 trouvée dans la littérature** : au lieu de reconstruire une vraie 3D, le papier [View-Aware Pose Analysis](https://www.mdpi.com/2673-2688/7/1/7) classifie d'abord le point de vue caméra (face/dos/profil) avec un modèle dédié, puis interprète les angles articulaires selon des seuils de ROM (Range of Motion) biomécaniques propres à ce point de vue. 87% de réussite pour identifier des postures dangereuses, en monoculaire, sans triangulation. À ajouter aux options (a)/(b)/(c) déjà listées plus haut.

**Validation empirique de l'approche caméra fixe** : [Automated Deadlift Techniques Assessment](https://www.mdpi.com/2673-2688/6/7/148) classifie la forme d'un deadlift (dos rond, hyperextension, lever de hanche précoce = les "mouvements dangereux" typiques) à partir d'une séquence de 17 keypoints (MoveNet) en vue latérale fixe, via CNN 2+1D ou LSTM. F1 jusqu'à 1.00 sur la classification de forme, avec seulement 2 sujets et un dataset custom filmé au smartphone. Confirme que l'option (c) (2D + caméra fixe) est viable pour ce type de tâche précis.

**Proposition pour le modèle "mouvements dangereux"** — approche hybride plutôt que 100% zero-shot :
1. Garder l'embedding foundation model (MOMENT) comme signal général "à quel point cette rep est statistiquement inhabituelle" par rapport à la banque de référence — capte l'imprévu, sans labels.
2. Ajouter en parallèle des règles biomécaniques explicites calculées sur les angles articulaires (flexion lombaire excessive, valgus du genou, décélération brutale) — capte les patterns dangereux *connus*, interprétable, pas de boîte noire pour un usage sécurité.
3. Envisager une petite tête supervisée (LSTM/CNN léger, comme le papier deadlift) entraînée sur une poignée de reps labellisées si on a le temps de filmer/labelliser — la littérature montre que même 2 sujets suffisent à un F1 très élevé sur ce type de classification de forme.

## Datasets — mouvements compétitifs (recherche)

- [AthletePose3D](https://github.com/calvinyeungck/AthletePose3D) — 12 mouvements sportifs (athlétisme, patinage artistique, course), 8 athlètes niveau national/international, multi-caméra synchronisé, ~1.3M frames, 3D par mocap optique marqueurs. Pas de musculation, mais bonne référence méthodologique multi-vues.
- [AthleticsPose](https://github.com/SZucchini/AthleticsPose) — 8 épreuves d'athlétisme (sprint, haies, lancers...), 23 athlètes compétitifs, 8 caméras synchronisées, évalue spécifiquement la précision de l'estimation 3D **monoculaire** sur mouvements sportifs réels — directement utile pour trancher la question 2D/3D.
- **SportsPose** (cité dans une revue de littérature) — 176 000 poses 3D, 24 athlètes, dataset généraliste multi-sports.
- **Injury Ski II** — dataset de chutes/blessures annotées en ski alpin de compétition (1463 frames, skieurs World Cup) — utile comme exemple de méthodologie pour construire un dataset "mouvement dangereux" propre à un sport, pas réutilisable directement pour la musculation.

**Constat général** : pas de gros dataset public "mouvements compétitifs de musculation" trouvé — la littérature confirme une pénurie de datasets annotés spécifiques à chaque discipline. Les papiers sur la musculation (deadlift, powerlifting judging) utilisent tous des datasets custom filmés par les auteurs, souvent avec très peu de sujets. Ça valide l'approche déjà prévue : filmer notre propre petite banque de référence plutôt que chercher un gros dataset externe pour la musculation spécifiquement.
