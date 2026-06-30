# SDK_camUSB_Elp

SDK fourni par ELP (Shenzhen Ailipu Technology) pour une caméra USB UVC supportant
l'encodage matériel H264/MJPEG. Ce dépôt regroupe le driver Linux, plusieurs
applications de test en C, des outils Windows, et la documentation constructeur.

Ce README sert de point d'entrée pour revenir rapidement dans le dossier et pour
réutiliser tel ou tel composant comme contexte de prompt (Claude / autre LLM)
sans devoir ré-explorer toute l'arborescence à chaque fois.

---

## Vue d'ensemble de l'arborescence

```
.doc/
├── AILIPU H264Camera-Linux开发文档.pdf       # Doc constructeur (dev Linux)
├── ELP Decoding H264 video using VLC.docx     # Tuto décodage H264 via VLC
├── 文件介绍( Files introduction).docx          # Explication des dossiers (chinois)
├── H264_Driver_AP_SDK_V11/
│   ├── Linux_uvc_driver/
│   │   ├── uvc_2.6.36/                        # Driver kernel UVC patché (vieux kernel)
│   │   └── uvc_3.3.8/                         # Driver kernel UVC patché (kernel plus récent)
│   └── Linux_UVC_TestAP/                      # App de test complète (référence XU)
├── OSD-Linux_H264_AP_0724/                    # Variante TestAP avec overlay OSD
├── UCAM-DEMO/UCAM-DEMO/                       # Démo minimaliste, bon point de départ
└── H264_Preview_AP_1_2_0/                     # Visualiseur Windows prêt à l'emploi (.exe + DLL)
```

---

## 1. Contrôle XU — `h264_xu_ctrls.c/h` (cœur du protocole propriétaire)

**Emplacement** : `.doc/H264_Driver_AP_SDK_V11/Linux_UVC_TestAP/h264_xu_ctrls.{c,h}`
(identique en substance dans la variante OSD)

C'est le fichier le plus important du SDK : il définit le protocole de
communication propriétaire ELP au-dessus des commandes UVC standard, via
l'**Extension Unit (XU)** et l'ioctl `UVCIOC_CTRL_QUERY` (ou `UVCIOC_CTRL_SET/GET`
sur les anciens kernels).

Deux unités XU sont exposées (identifiées par leur GUID) :
- `XU_RERVISION_SYS_ID` (0x03) — contrôles "système" bas niveau (ASIC, flash)
- `XU_RERVISION_USR_ID` (0x04) — contrôles "utilisateur" haut niveau

Fonctions disponibles, regroupées par domaine :

| Domaine | Fonctions clés | Usage |
|---|---|---|
| Encodage H264 | `XU_H264_Set/Get_Mode`, `_BitRate`, `_QP`, `_GOP`, `_IFRAME`, `_SEI` | Ajuster qualité/débit à la volée |
| MJPEG | `XU_MJPG_Set/Get_Bitrate` | Contrôle du flux MJPEG |
| OSD (incrustation écran) | `XU_OSD_Set/Get_RTC`, `_Size`, `_Color`, `_Enable`, `_String`, `_Start_Position` | Horodatage, texte, position à l'écran |
| Détection de mouvement | `XU_MD_Set/Get_Mode`, `_Threshold`, `_Mask`, `_RESULT` | Détection matérielle de mouvement |
| Image | `XU_IMG_Set/Get_Mirror`, `_Flip`, `_Color` | Effets miroir/flip/couleur |
| Multi-stream | `XU_Multi_Set/Get_*` | Plusieurs flux simultanés, paramètres indépendants |
| GPIO | `XU_GPIO_Ctrl_Set/Get` | Pilotage de broches matérielles (relais, LED...) |
| Bas niveau | `XU_Asic_Read/Write`, `XU_SF_Read` | Accès direct registres ASIC / flash SPI |
| Mode dashcam | `XU_OSD_Set/Get_Speed`, `_Coordinate1/2` (si `CARCAM_PROJECT==1`) | OSD vitesse/GPS pour caméra embarquée véhicule |

**Prompt de reprise suggéré** :
> "Voici le fichier h264_xu_ctrls.h du SDK caméra ELP (liste des fonctions XU
> disponibles : encodage H264, OSD, détection de mouvement, GPIO...). Aide-moi
> à écrire un script/programme qui [activer la détection de mouvement et
> déclencher un enregistrement / ajuster dynamiquement le bitrate / piloter le
> GPIO X...]."

---

## 2. Driver UVC Linux — `Linux_uvc_driver/`

**Emplacement** : `.doc/H264_Driver_AP_SDK_V11/Linux_uvc_driver/{uvc_2.6.36,uvc_3.3.8}/`

Sources du module noyau Linux `uvcvideo` **patché par ELP** pour supporter le
H264 matériel (le driver UVC standard du noyau Linux ne gère que MJPEG/YUV).
Deux versions fournies selon l'âge du kernel cible :
- `uvc_2.6.36` — kernels anciens / cartes embarquées historiques
- `uvc_3.3.8` — kernels plus récents (préférer celui-ci sur système actuel,
  avec adaptation probable nécessaire pour kernels très récents 5.x/6.x)

Fichiers clés : `uvc_driver.c` (entrée du module), `uvc_video.c` (gestion du
flux vidéo), `uvc_ctrl.c` (contrôles UVC standards), `uvc_queue.c` (buffers).
Compilation via `Makefile_PC` (poste Linux classique) ou `Makefile_Embedded`
(cross-compilation carte embarquée).

**Prompt de reprise suggéré** :
> "Voici les sources du driver UVC patché ELP (uvc_driver.c, uvc_video.c...).
> Aide-moi à le compiler/adapter pour un kernel Linux X.Y, ou à comprendre
> pourquoi [tel symptôme] se produit au chargement du module."

---

## 3. Démo minimaliste — `UCAM-DEMO/UCAM-DEMO/`

**Emplacement** : `.doc/UCAM-DEMO/UCAM-DEMO/`

Point d'entrée : `main.c`. C'est l'exemple **le plus simple et le plus lisible**
du SDK — bon point de départ pour comprendre le cycle de vie V4L2 (open device,
négociation de format, mmap des buffers, boucle de capture) sans la complexité
multi-format de TestAP. Utilise `v4l2uvc.c` pour les fonctions V4L2 génériques
et `rervision_xu_ctrls.c` (équivalent simplifié de `h264_xu_ctrls`) pour les
extensions XU (notamment la détection de mouvement, visible dès le haut du
fichier avec `md_mask`/`md_result`).

**Prompt de reprise suggéré** :
> "Voici main.c et v4l2uvc.c de la démo UCAM-DEMO du SDK ELP (capture V4L2 de
> base + détection de mouvement XU). Aide-moi à transformer cette base en
> [appli de streaming réseau / sauvegarde sur détection de mouvement / script
> Python équivalent via ctypes]."

---

## 4. Application de test complète — `Linux_UVC_TestAP/` et variante OSD

**Emplacement** :
- `.doc/H264_Driver_AP_SDK_V11/Linux_UVC_TestAP/` (base)
- `.doc/OSD-Linux_H264_AP_0724/` (même base + gestion OSD avancée)

Point d'entrée : `H264_UVC_TestAP.c` (version `v1.0.14.0_H264_UVC_TestAP_Multi`).
Application complète en ligne de commande gérant : énumération des formats
H264 disponibles (`H264_GetFormat`, `cap_desc_parser.c`), capture multi-buffer,
parsing des NALU H264 (`nalu.c`), et tous les contrôles XU vus en section 1.
C'est la référence à consulter pour voir des **exemples d'appel concrets** des
fonctions `h264_xu_ctrls` (comment les paramètres sont réellement construits
et envoyés).

La variante `OSD-Linux_H264_AP_0724` ajoute un focus particulier sur l'usage
des fonctions OSD (texte, position, échelle automatique).

**Prompt de reprise suggéré** :
> "Voici H264_UVC_TestAP.c (appli de test complète du SDK ELP) qui montre comment
> appeler les fonctions XU en pratique. Aide-moi à extraire uniquement la logique
> de [récupération des formats disponibles / parsing NALU / gestion OSD] pour
> l'intégrer dans mon propre programme."

---

## 5. Outils Windows — `H264_Preview_AP_1_2_0/`

**Emplacement** : `.doc/H264_Preview_AP_1_2_0/`

Exécutable `H264_Preview.exe` + DLLs FFmpeg (avcodec, avformat, avutil,
swscale, xvidcore, pthreadGC2) : visualiseur H264 prêt à l'emploi sous Windows,
utile pour **valider rapidement** que la caméra fonctionne avant de coder quoi
que ce soit de personnalisé. Pas de source fournie (binaire uniquement).

---

## 6. Documentation constructeur

- `AILIPU H264Camera-Linux开发文档.pdf` — documentation officielle de
  développement Linux (référence du protocole XU, à consulter en cas de doute
  sur un paramètre précis).
- `ELP Decoding H264 video using VLC.docx` — méthode pour visualiser le flux
  H264 brut via VLC (utile pour debug rapide sans coder de décodeur).
- `文件介绍( Files introduction).docx` — explication (en chinois) du rôle de
  chaque dossier du SDK.

---

## Comment réutiliser ce dépôt efficacement

Pour reprendre le travail sans perdre de temps : indiquer quel chapitre
ci-dessus est concerné (1 à 6), coller ou faire lire le(s) fichier(s)
correspondant(s), et préciser l'objectif (lecture, contrôle d'un paramètre
spécifique, portage, debug). Les sections 1, 3 et 4 sont les plus susceptibles
de servir de base à du développement ; les sections 2 et 5 sont plutôt des
briques d'infrastructure/outillage.
