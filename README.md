# LAB12

# 🔐 LAB 12 — Bypass de la Détection de Root Android
 

 
---
 

 
## 1. Prérequis et installation
 
### 1.1 Vérification de Python, pip et Frida
 
Avant de commencer, on vérifie que l'environnement Python est prêt et que Frida est installé avec des versions cohérentes entre le CLI et le module Python.
 
```
python --version
pip --version
frida --version
python -c "import frida; print(frida.__version__)"
```
 
<img width="1168" alt="Preuve installation Frida" src="https://github.com/user-attachments/assets/22016891-0cab-44de-8611-8352b2c67290" />  


✅ **Frida 17.8.0** installé et cohérent entre CLI et module Python.
 
---
 
## 2. Vérification de l'émulateur et de Frida
 
### 2.1 Détection ADB de l'émulateur
 
```
adb version
adb devices
```
 
<img width="613" alt="adb devices" src="https://github.com/user-attachments/assets/d895151c-03ed-4605-b6d8-11c4a803b65b" />  

✅ L'émulateur `emulator-5554` est bien détecté avec le statut `device`.
 
---
 
### 2.2 Déploiement de frida-server sur l'émulateur
 
On identifie d'abord l'architecture CPU de l'émulateur, puis on pousse le bon binaire `frida-server`.
 
```
adb shell getprop ro.product.cpu.abi
# → x86_64
 
adb push frida-server /data/local/tmp/frida-server
adb shell chmod 755 /data/local/tmp/frida-server
adb root
adb shell "/data/local/tmp/frida-server &"
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```
 
<img width="722" alt="frida-server démarrage" src="https://github.com/user-attachments/assets/e46e5b94-0ded-47a7-be4e-aae797c00a1c" />  

---
 
### 2.3 Vérification de frida-server et liste des applications
 
```

adb shell "ps -A | grep frida"
frida-ps -Uai
```
 
<img width="1073" alt="ps grep frida" src="https://github.com/user-attachments/assets/923dfb39-e82a-4b50-b7d9-57b12acad931" />
<img width="893" alt="frida-ps -Uai" src="https://github.com/user-attachments/assets/6fb508d3-6b13-4bc3-9571-293e3bd22a11" />

✅ **frida-server** tourne en tant que `root`. **19 applications** listées, dont notre cible `owasp.mstg.uncrackable3`.
 
---
 
## 3. Installation de Medusa
 
Medusa est un framework d'instrumentation Android basé sur Frida, avec des modules prêts à l'emploi pour le bypass de protections.
 
```
git clone https://github.com/Ch0pin/medusa.git
cd medusa
pip install -r requirements.txt
pip install cmd2
python3.12 medusa.py --help
```
 
**Note :** Il faut utiliser `python3.12` car les dépendances Frida sont installées pour Python 3.12.
 
<img width="893" alt="Medusa installation" src="https://github.com/user-attachments/assets/167960e1-e06a-4a4f-932d-8197c7199c09" />  

<img width="1857" alt="Medusa help output" src="https://github.com/user-attachments/assets/b7f0ee35-909d-45ac-b81b-50d9b8f61775" />  

✅ **Medusa opérationnel** — 124 modules disponibles.
 
---
 
## 4. Application cible — Uncrackable Level 3
 
**Package :** `owasp.mstg.uncrackable3`    

**Source :** OWASP Mobile Security Testing Guide
 
### Comportement sans bypass
 
Au lancement sur l'émulateur rooté, l'application détecte immédiatement l'environnement et affiche :
 
*"Rooting or tampering detected. This is unacceptable. The app is now going to exit."*
 
<img width="441" alt="Root detected - avant bypass" src="https://github.com/user-attachments/assets/6a897ca8-4bde-47b6-973d-fc3c49e64ed5" />
**Cause :** L'app lit la propriété système `ro.build.tags = test-keys`, caractéristique des builds de débogage et émulateurs rootés.
 
---
 
## 5. Bypass avec Medusa
 
### 5.1 Lancement de Medusa et connexion à l'émulateur

   
```
python3.12 medusa.py -p owasp.mstg.uncrackable3 -d emulator-5554
# → Sélectionner : 3) Device(id="emulator-5554")
```
 
<img width="1374" alt="Medusa connexion émulateur" src="https://github.com/user-attachments/assets/346ed101-e81d-43ac-85f2-ffac1c2b4cb9" />  

<img width="789" alt="Medusa propriétés device" src="https://github.com/user-attachments/assets/6db39791-986b-4e00-86c7-18703e1e6678" />  

**Propriété clé détectée :**
```
[ro.build.tags]: [test-keys]
```
>C'est exactement ce que l'application vérifie pour détecter le root.
 
---
 
### 5.2 Recherche et chargement du module root bypass
 
```
medusa> search root
```
 
<img width="746" alt="search root modules" src="https://github.com/user-attachments/assets/7b2aecc4-8ccf-46e9-b598-769d03bd00c0" />  

**4 modules disponibles :**
 
| # | Module | Usage |
|---|---|---|
| 1 | `root_detection/universal_root_detection_bypass` | ✅ Le plus complet — utilisé ici |
| 2 | `root_detection/rootbeer_detection_bypass` | Pour apps utilisant RootBeer |
| 3 | `root_detection/rootbeer_detection_bypass_no_obfuscation` | RootBeer sans obfuscation |
| 4 | `root_detection/jailMonkey_react_native` | Pour apps React Native |
 
```
medusa> use root_detection/universal_root_detection_bypass
medusa> run -f owasp.mstg.uncrackable3 --fallback
```
 
---
 
## 6. Erreur Medusa — Conflit d'architecture
 
### Erreur rencontrée
 
<img width="1920" alt="Erreur gadget arm64" src="https://github.com/user-attachments/assets/86908cd8-4ac9-4bbd-b8fb-9dd52200b18c" />  

```
Failed to spawn: need Gadget to attach on jailed Android;
its default location is: /home/ennoukra/.cache/frida/gadget-android-arm64.so
```
 
**Cause :** Medusa/Frida cherche automatiquement un gadget `arm64`, alors que l'émulateur est `x86_64`. Il s'agit d'un conflit de détection d'architecture.
 
### Tentative de correction
 
Relancement de Medusa en forçant le device USB :
 
```
python3.12 medusa.py -p owasp.mstg.uncrackable3 -d emulator-5554
```
 
<img width="1920" alt="Relancement Medusa" src="https://github.com/user-attachments/assets/0e0ab57b-cd99-4264-9b45-90b706be7d3a" />  

<img width="1912" alt="Erreur persistante" src="https://github.com/user-attachments/assets/c2c2fd25-3b5d-4753-a808-589df45837e2" />
> ❌ **Le problème persiste.** Medusa ne résout pas automatiquement le conflit d'architecture pour les émulateurs x86_64. → **Passage au Plan B.**
 
---
 
## 7. Plan B — Bypass avec Frida pur
 
Comme prévu dans le lab, on utilise directement des scripts Frida pour reproduire le bypass sans Medusa.
 
### 7.1 Script `bypass_root.js`
 
<img width="1912" alt="Script bypass_root.js" src="https://github.com/user-attachments/assets/8f5c6772-a794-4b31-9e91-648622d0a10c" />  

  
```
// bypass_root.js — Neutralise Build.TAGS, File.exists, Runtime.exec
 
const suspiciousPaths = [
  "/system/bin/su", "/system/xbin/su", "/sbin/su", "/system/su",
  "/system/app/Superuser.apk", "/system/app/SuperSU.apk",
  "/system/bin/.ext/.su", "/system/usr/we-need-root/",
  "/system/xbin/daemonsu", "/system/etc/init.d/99SuperSUDaemon",
  "/system/bin/busybox", "/system/xbin/busybox"
];
 
Java.perform(function () {
 
  // 1. Hook Build.TAGS → retourner "release-keys" au lieu de "test-keys"
  try {
    const Build = Java.use('android.os.Build');
    Object.defineProperty(Build, 'TAGS', {
      get: function() { return 'release-keys'; }
    });
    console.log('[+] Build.TAGS -> release-keys');
  } catch (e) {}
 
  // 2. Hook File.exists() → masquer les chemins de binaires root suspects
  try {
    const File = Java.use('java.io.File');
    File.exists.implementation = function () {
      const p = this.getAbsolutePath();
      if (suspiciousPaths.indexOf(p) !== -1) {
        console.log('[+] File.exists bypass for', p);
        return false;
      }
      return this.exists.call(this);
    };
  } catch (e) {}
 
  // 3. Hook Runtime.exec() → bloquer les tentatives d'exécution de "su"
  try {
    const Runtime = Java.use('java.lang.Runtime');
    const JString = Java.use('java.lang.String');
    function blockIfSus(x) {
      const s = Array.isArray(x) ? x.join(' ') : ('' + x);
      const t = s.toLowerCase().trim();
      if (t.startsWith('su') || t.includes(' which su') ||
          t.includes(' busybox') || t.includes(' su '))
        return ['sh', '-c', 'echo'];
      return null;
    }
    Runtime.exec.overload('java.lang.String').implementation = function(cmd) {
      const r = blockIfSus(cmd);
      return r ? this.exec(JString.$new(r.join(' '))) : this.exec(cmd);
    };
    console.log('[+] Runtime.exec hooks installed');
  } catch(e) {}
 
  console.log('[+] Java bypass installed');
});
```
 
 
### 7.2 Lancement du bypass
 
```
# Redémarrer frida-server proprement
adb shell "pkill -9 -f frida-server"
adb shell "/data/local/tmp/frida-server &"
adb forward tcp:27042 tcp:27042
 
# Lancer le bypass
frida -U -f owasp.mstg.uncrackable3 -l ~/Downloads/bypass_root.js
```
 
<img width="1173" alt="Frida bypass lancé" src="https://github.com/user-attachments/assets/b37a80c3-1286-4d43-9305-7a6213509f6a" />
---
 
### 7.3 Résultat — Hooks actifs et app spawnée
 
**Logs Frida obtenus :**
 
```
Connected to Android Emulator 5554 (id=emulator-5554)
Spawned `owasp.mstg.uncrackable3`. Resuming main thread!
[Android Emulator 5554::owasp.mstg.uncrackable3 ]-> [+] Build.TAGS -> release-keys
[+] Runtime.exec hooks installed
[+] Java bypass installed
```
 
**Émulateur — Application lancée sans détection de root :**
 
<img width="506" alt="App lancée après bypass" src="https://github.com/user-attachments/assets/8b942b58-a637-47e6-a9de-76083b0e8028" />  

✅ **Bypass réussi** — Les hooks sont injectés, l'application se lance sans déclencher la détection de root.
 
---
 
## 8. Résultats et validation
 
### Tableau récapitulatif des livrables
 
| Livrable | Commande / Preuve | Statut |
|---|---|---|
| Python + pip installés | `python --version` / `pip --version` | ✅ |
| ADB fonctionnel | `adb devices` | ✅ |
| Émulateur détecté | `emulator-5554 device` | ✅ |
| Frida 17.8.0 (CLI + Python) | `frida --version` | ✅ |
| frida-server déployé | `ps -A \| grep frida` | ✅ |
| frida-ps liste les apps | 19 apps listées | ✅ |
| Medusa installé | 124 modules disponibles | ✅ |
| Module root bypass trouvé | `search root` → 4 modules | ✅ |
| Preuve avant bypass | Screenshot "Root detected" | ✅ |
| Script bypass_root.js créé | `~/Downloads/bypass_root.js` | ✅ |
| Hook `Build.TAGS` actif | `[+] Build.TAGS -> release-keys` | ✅ |
| Hook `Runtime.exec` actif | `[+] Runtime.exec hooks installed` | ✅ |
| Hook Java bypass actif | `[+] Java bypass installed` | ✅ |
| App spawnée sans crash root | Screenshot app ouverte | ✅ |
 
---
 
### Comparaison avant / après bypass
 
| État | Comportement de l'app |
|---|---|
| **Sans bypass** | Affiche *"Rooting or tampering detected"* et quitte |
| **Avec bypass Frida** | Se lance normalement, hooks actifs, root masqué |
 
---
 
## 9. Conclusion
 
### Ce que ce lab démontre
 
**Détection de root côté Java** — Les applications Android utilisent plusieurs mécanismes pour détecter un environnement rooté :
 
- Lecture de `android.os.Build.TAGS` (valeur `test-keys` = build non officiel/rooté)
- Vérification de chemins suspects via `File.exists()` (`/system/xbin/su`, `/system/bin/busybox`, etc.)
- Tentatives d'exécution de `su` via `Runtime.exec()`
**Mécanisme du bypass** — En hookant ces fonctions au niveau du runtime Java avec Frida, on retourne de fausses valeurs à l'application :
 
| Hook | Comportement original | Valeur retournée après bypass |
|---|---|---|
| `Build.TAGS` | `test-keys` | `release-keys` |
| `File.exists("/system/xbin/su")` | `true` | `false` |
| `Runtime.exec("su")` | Exécute `su` | Remplacé par `echo` |

   
**Medusa vs Frida pur** — Medusa est un excellent framework pour automatiser l'instrumentation, mais peut présenter des limitations de compatibilité selon l'architecture de l'émulateur (ici x86_64 vs arm64). Dans ce cas, le Plan B avec des scripts Frida directement offre plus de contrôle et de flexibilité.
 
### Outils utilisés
 
| Outil | Version | Rôle |
|---|---|---|
| Frida | 17.8.0 | Instrumentation dynamique |
| frida-server | 17.8.0 | Service côté émulateur |
| Medusa | dev | Framework d'analyse Android |
| ADB | 1.0.41 | Communication PC ↔ émulateur |
| Android Emulator | API 30 x86_64 | Environnement de test |
 
---
 
**Avertissement éthique :** Les techniques présentées dans ce lab sont utilisées exclusivement dans un cadre légal d'audit de sécurité mobile, sur des applications et appareils de test. Toute utilisation sur des applications tierces sans autorisation explicite est illégale.
 
---
 









