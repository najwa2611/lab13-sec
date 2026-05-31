
# LAB 13 : Bypass de la Détection de Root Android avec Objection

---

## Prérequis

- Python 3.8+ et pip
- ADB (Android Platform Tools) : [Télécharger](https://developer.android.com/tools/releases/platform-tools)
- Appareil Android 8.0+ avec Options développeur + Débogage USB activés
- Frida (PC) et frida-server (Android) - versions alignées
- APK cible

**Vérifications rapides :**
```bash
python --version
pip --version
adb devices
frida --version
```

---

## Étape 1 — Installer Objection

```bash
# Méthode recommandée via pipx (isolation)
pip install --user pipx
pipx ensurepath
pipx install objection

# Ou via pip classique
pip install --upgrade objection
```

**Vérification :**
```bash
objection --version
objection --help
```
---

## Étape 2 — Préparer l’appareil et démarrer frida-server

**1. Identifier l'ABI :**
```bash
adb shell getprop ro.product.cpu.abi
```

**2. Télécharger et installer frida-server :**
```bash
# Télécharger depuis https://github.com/frida/frida/releases
adb push frida-server /data/local/tmp/
adb shell chmod 755 /data/local/tmp/frida-server
adb shell "/data/local/tmp/frida-server -l 0.0.0.0"
```

**3. Forward des ports (optionnel) :**
```bash
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043
```

**4. Vérifier la visibilité :**
```bash
frida-ps -Uai
```

---

## Étape 3 — Démarrer Objection sur l’app cible

### Méthode 1 : Spawn (recommandée)
```bash
objection -g com.example.rootcheck explore --startup-command "android root disable"
```

### Méthode 2 : Attach (si l'app résiste)
```bash
# Lancez l'app manuellement sur le téléphone, puis :
objection -g com.example.rootcheck explore
# Dans la console Objection :
android root disable
```

---

## Étape 4 — Que fait `android root disable` ?

Objection installe des hooks Java qui :
- Force `android.os.Build.TAGS` à retourner `release-keys`
- Neutralise `java.io.File.exists()` sur les chemins sensibles (`/system/xbin/su`, etc.)
- Bloque `Runtime.getRuntime().exec()` pour les commandes `su`/`which su`
- Désactive les méthodes de détection des librairies comme RootBeer

---

## Étape 5 — Valider le bypass

1. **Sans Objection** : Lancez l'app → "Root detected"
2. **Avec Objection** : Exécutez `android root disable` → "Not rooted" ou fonctionnement normal

**Commandes d'exploration utiles :**
```bash
android hooking search classes root
android hooking search methods isRoot
android intent launch_activity <ActivityName>
```

---

## Étape 6 — Automatiser l'injection au démarrage

```bash
objection -g com.example.rootcheck explore \
  --startup-command "android root disable" \
  --startup-command "android sslpinning disable" \
  --startup-command "android hooking search classes root"
```

---

## Étape 7 — Checks natifs (C/C++)

### Option A : Hooking au niveau Java (le plus simple)
```bash
android hooking watch class com.example.RootCheck
android hooking set return_value com.example.RootCheck isRooted false
```

### Option B : Script Frida dédié
```bash
frida -U -n "NomDuProcessus" -l bypass_native.js
```

### Option C : Découverte avec frida-trace
```bash
frida-trace -U -i open -i access -i stat -i openat com.example.rootcheck
```

---

## Dépannage (FAQ)

| Problème | Solution |
|----------|----------|
| `objection: command not found` | Ajoutez Python Scripts au PATH ou utilisez `python -m pip install objection` |
| `unable to connect to remote frida-server` | Vérifiez `adb devices` et que frida-server tourne (`adb shell ps \| grep frida`) |
| L'app détecte encore le root | Vérifiez l'exécution de `android root disable` ; essayez l'attache après démarrage |
| Anti-instrumentation Frida | Attachez après lancement plutôt que spawn, réduisez les logs |


---

## Références utiles

- [Objection](https://github.com/sensepost/objection)
- [Frida](https://frida.re/)
- [RootBeer](https://github.com/scottyab/rootbeer)
- [Platform Tools (ADB)](https://developer.android.com/tools/releases/platform-tools)

---
