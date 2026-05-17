# LAB-12-Bypass-de-la-D-tection-de-Root-Android-avec-Medusa

Avertissement et objectifs
Objectif: réaliser, pas à pas (niveau débutant), un bypass de la détection de root Android en utilisant l’outil Medusa (ensemble d’outils/scripts d’instrumentation, généralement basés sur Frida), puis valider que l’application ne « voit » plus le root.
Usage éthique: n’utilisez ces techniques que sur des apps/appareils que vous êtes autorisé à tester.
Ce guide fournit aussi un plan B avec Frida pur si Medusa n’est pas disponible ou échoue.

---

Prérequis simples
PC Windows/macOS/Linux avec Internet et droits admin/sudo.
Python 3.8+ et pip:
# python --version
 # pip --version 
ADB (Android Platform Tools): https://developer.android.com/tools/releases/platform-tools
Un appareil Android (8.0+ recommandé) avec:
Options développeur activées
Débogage USB activé
Câble USB
Connaître (ou retrouver) le package de l’app cible, ex: com.example.rootcheck.
Astuce Windows: utilisez PowerShell. Sur Linux/macOS: bash/zsh.

![Import OVA](https://github.com/user-attachments/assets/e8279451-618e-4a77-9a5e-02b5a589211d)

---

Étape 1 — Préparer l’environnement Android et Frida
Même si Medusa automatise beaucoup, il repose généralement sur Frida côté PC et un service côté appareil.
1-Installer Frida côté PC:

pip install --upgrade frida frida-tools
frida --version
python -c "import frida; print(frida.__version__)"
Installer ADB et vérifier l’appareil:
adb version
adb devices
Vous devez voir votre appareil avec l’état device (et pas unauthorized). Si unauthorized, rebranchez et acceptez la demande sur le téléphone.
Démarrer frida-server sur l’appareil (recommandé pour compatibilité maximale)
Identifiez l’ABI CPU:
adb shell getprop ro.product.cpu.abi
Depuis les releases Frida, téléchargez frida-server-<même_version>-android-<arch>.xz, décompressez puis poussez-le:
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"   # laissez tourner (ou utilisez nohup pour le background)
Option de redirection de ports (selon appareil):
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
Vérification:
frida-ps -Uai
Vous devez voir la liste des apps.
Remarque: certains modules Medusa peuvent fonctionner sans lancer manuellement frida-server (par ex. via Frida Gadget intégré), mais pour un débutant, frida-server est la voie la plus simple.

---
Étape 2 — Installer Medusa (outil d’instrumentation)
Medusa est généralement distribué comme un projet Python avec des scripts/modules Frida.
Clonage de la boîte à outils Medusa (exemple générique):
# Option 1: via git
git clone <URL_du_depot_Medusa>
cd Medusa

# Option 2: via package (si publié sur pip)
pip install medusa-android  # seulement si la distribution existe sur pip
Installez les dépendances Python si le dépôt fournit un requirements.txt:
pip install -r requirements.txt
Vérifiez l’aide/CLI de Medusa (les commandes exactes peuvent varier selon la distribution):
python medusa.py --help
# ou
medusa --help
Attendez-vous à voir des sous-commandes comme --package, --attach, --root-bypass, --ssl-bypass, etc. L’intitulé exact peut différer (parfois "modules").
Si vous ne trouvez pas le bon dépôt Medusa ou si la CLI n’est pas reconnue, passez directement au « Plan B — Frida pur » plus bas; vous pourrez toujours revenir à Medusa plus tard.

---

Étape 3 — Comprendre la détection de root (vue débutant)
Côté Java: l’app peut lire android.os.Build.TAGS (recherche test-keys), vérifier des chemins (/system/xbin/su, busybox) avec java.io.File.exists(), ou essayer Runtime.getRuntime().exec("su").
Côté natif (C/C++): elle peut appeler open, openat, access, stat, etc., sur des chemins suspects, ou lire /proc/mounts.
Idée du bypass: « hooker » ces appels pour renvoyer des réponses “non root” (par ex. dire que su n’existe pas, empêcher su d’être exécuté, etc.).
Medusa fournit généralement un module prêt à l’emploi pour neutraliser ces vérifications (souvent appelé root-bypass, anti-root, rootcloak ou similaire).



