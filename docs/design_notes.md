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
