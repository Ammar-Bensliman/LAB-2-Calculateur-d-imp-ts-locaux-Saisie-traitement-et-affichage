# LAB-2-Calculateur-d-imp-ts-locaux-Saisie-traitement-et-affichage
# LAB-13 : Bypass de la Détection de Root Android avec Objection

> **Avertissement de sécurité :** Ce lab est réalisé dans un environnement contrôlé à des fins strictement pédagogiques. L'application cible appartient à la famille OWASP UnCrackable.

## 📝 Contexte et Introduction
Ce lab s'inscrit dans la continuité du **LAB 11 sur Frida**. Cette fois-ci, nous franchissons une étape supérieure en utilisant **Objection**, un toolkit d'exploration mobile basé sur le framework Frida. 

Alors que Frida nécessite l'écriture manuelle de scripts d'injection JavaScript (hooks), Objection fournit une interface en ligne de commande interactive capable d'automatiser les tâches de rétro-ingénierie et de test de pénétration les plus courantes. Parmi ces automatisations, le contournement (bypass) des mécanismes de détection de root est l'une des fonctionnalités les plus puissantes.

L'application cible reste dans la famille **OWASP UnCrackable (Level 1)**, et l'architecture globale repose sur un modèle client-serveur : un agent léger (`frida-server`) s'exécute sur l'appareil Android cible tandis qu'Objection pilote l'analyse depuis la machine de l'auditeur.

---

## 🎯 Objectifs du Laboratoire
1. **Installer** Objection et la suite d'outils Frida sur le poste de travail.
2. **Déployer et exécuter** le binaire `frida-server` adapté à l'architecture de l'émulateur Android.
3. **Établir et valider** la communication bidirectionnelle entre le PC et l'appareil mobile via ADB.
4. **Injecter le gadget Objection** au sein du processus de l'application cible dès son initialisation.
5. **Exécuter la routine automatisée** de bypass de la détection de root.
6. **Vérifier le succès de l'opération** et analyser le comportement de l'application avant/après injection.
7. **Comprendre les limitations** du hook au niveau de la couche d'abstraction Java et appréhender l'utilité de Frida pour le code natif (C/C++).

---

## 🛠️ Environnement de Travail

| Élément | Spécification / Détail |
| :--- | :--- |
| **Système PC** | Windows (PowerShell ou CMD avec privilèges requis) |
| **Appareil Android** | Émulateur Android (ex: Genymotion ou Android Studio AVD avec accès Root) |
| **Outils Requis** | Android Debug Bridge (ADB), Python 3.x, Frida, frida-server, Objection |
| **Langages sous-jacents** | Python (outils), JavaScript (moteur d'injection de Frida) |
| **Cible du Test** | Application Android OWASP UnCrackable App (avec détection active de Root) |

---

## 🚀 Guide Opérationnel Étape par Étape

### Étape 1 — Installation d'Objection

L'approche moderne et recommandée pour installer Objection consiste à utiliser `pipx`. Cet outil isole l'application et ses dépendances dans un environnement virtuel dédié, évitant ainsi de corrompre l'installation globale de Python sur votre système.

```bash
# Installation de pipx via pip
pip install --user pipx

# Configuration des variables d'environnement (PATH) pour pipx
pipx ensurepath

# Installation d'Objection dans son environnement isolé
pipx install objection
```

*Alternative — Installation directe via le gestionnaire de paquets global de Python (pip) :*
```bash
pip install --upgrade objection
```

**Validation de l'installation :**
```bash
# Vérification de la version installée
objection --version

# Affichage du menu d'aide général
objection --help
```

---

### Étape 2 — Préparation et Configuration de l'Émulateur Android

#### 1. Vérification de la connexion ADB
```bash
adb devices
```
*💡 **Note importante :** L'émulateur doit renvoyer le statut `device`. S'il affiche `unauthorized`, vous devez explicitement accepter l'empreinte de la clé de débogage USB sur l'écran de l'émulateur.*

#### 2. Identification de l'architecture matérielle
```bash
adb shell getprop ro.product.cpu.abi
```

#### 3. Alignement des versions Frida sur le PC
```bash
pip install --upgrade frida-tools
frida --version
```

#### 4. Déploiement et exécution de `frida-server`
```bash
# Transfert du fichier vers un espace accessible en exécution
adb push frida-server /data/local/tmp/

# Attribution des droits d'exécution (chmod 755 ou +x)
adb shell chmod 755 /data/local/tmp/frida-server

# Lancement du serveur en tâche de fond
adb shell "/data/local/tmp/frida-server &"
```

#### 5. Redirection de port et validation de la liaison
```bash
# Redirection du port de communication par défaut de Frida
adb forward tcp:27042 tcp:27042

# Énumération des applications installées et en cours d'exécution
frida-ps -Uai
```

---

### Étape 3 — Lancement d'Objection et Bypass du Root

#### 1. Attacher Objection à l'application cible
```bash
objection --gadget "owasp.mstg.uncrackable1" explore
```

#### 2. Exécuter la commande de bypass du root
Une fois que vous avez obtenu le prompt interactif `owasp.mstg.uncrackable1 on (android) [usb] #`, saisissez la commande d'automatisation suivante :
```bash
android root disable
```

---
---

# LAB-14 : Contournement de la détection Root Android avec Frida, Objection et Hooks natifs

> **Avertissement de sécurité :** Ce lab est réalisé dans un environnement contrôlé à des fins strictement pédagogiques. L'application cible appartient à la famille OWASP UnCrackable.

## 📝 Contexte et Objectifs
Ce lab va un cran plus loin que les précédents : on ne se contente plus d'un simple bypass Java ou d'une commande automatisée sous Objection. Nous allons combiner les deux approches, ajouter des **hooks natifs**, et tracer les appels système pour comprendre ce que l'application fait réellement sous le capot.

---

## 🛠️ Environnement de Travail

| Élément | Détail / Spécification |
| :--- | :--- |
| **Système** | Windows (PowerShell) |
| **Émulateur** | Android Emulator 5554 |
| **Frida** | Version 17.8.0 |
| **Objection** | Version 1.12.4 |
| **OS Cible** | Android 11 |
| **Package Cible** | `owasp.mstg.uncrackable1` |

---

## 🚀 Guide d'Exécution

### Étape 1 — Vérification de l'injection Frida
Avant d'écrire des scripts complexes, vérifions que la "plomberie" fonctionne avec un script minimal (`hello.js`).

```bash
frida -U -f owasp.mstg.uncrackable1 -l hello.js
```

### Étape 2 — Comportement par défaut (Sans intervention)
Sans aucune intervention de notre part, l'application se ferme toute seule dès le lancement en affichant ce message :
> **Root detected!**
> This is unacceptable. The app is now going to exit.

### Étape 3 — Premier bypass : neutraliser les vérifications Java
```bash
frida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js
```

### Étape 4 — Quand Java ne suffit pas : l'ajout des hooks natifs
Certaines applications passent directement par le code natif (C/C++ via le NDK). Elles appellent des fonctions système de bas niveau (`open`, `stat`, `access`, `readlink`) pour chercher des traces de root.

```bash
frida -U -f owasp.mstg.uncrackable1 -l bypass_root_basic.js -l bypass_native.js
```

### Étape 5 — La version rapide avec Objection
Pour comparer notre approche manuelle avec une méthode automatisée, testons Objection en une seule ligne de commande :

```bash
objection -g owasp.mstg.uncrackable1 explore --startup-command "android root disable"
```

### Étape 6 — Observer avant d'agir avec `frida-trace`
```bash
frida-trace -U -i open -i access -i stat -i openat -i fopen -i readlink Uncrackable1
```

---
---

# LAB-15 : Inspection du Trafic TLS / HTTP(S) Android avec Frida et Burp Suite

> **Avertissement de sécurité :** Ce lab est réalisé dans un environnement de test contrôlé à des fins strictement pédagogiques. L'application cible est InsecureBankv2.

## 📝 Contexte et Présentation
Dans ce laboratoire, nous allons configurer une chaîne d'outils d'audit pour intercepter, analyser et modifier le trafic réseau chiffré d'une application Android. Nous utilisons **Burp Suite** comme proxy d'interception local et **Frida** pour injecter un script de contournement (bypass) du mécanisme de *SSL Pinning*.

---

## 🛠️ Environnement de Travail

| Élément | Spécification / Détail |
| :--- | :--- |
| **Système PC** | Windows (PowerShell avec privilèges) |
| **Émulateur** | Android Studio Emulator 5554 |
| **Frida**| Version 17.8.0 |
| **Proxy**| Burp Suite Community Edition |
| **Backend Serveur** | AndroLabServer (Exécuté sous Python 2.7) |
| **Package Cible** | `com.android.insecurebankv2` |

---

## 🚀 Guide Opérationnel Étape par Étape

### Étape 1 — Vérification de la visibilité de l'application
```bash
frida-ps -Uai
```

### Étape 2 — Test d'injection rapide
```bash
frida -U -f com.android.insecurebankv2 -l hello.js
```

### Étape 3 — Configuration de l'écouteur Burp Suite
1. Ouvrez Burp Suite et accédez à **Proxy** > **Proxy settings** > **Proxy listeners**.
2. Modifiez ou ajoutez un écouteur :
   * **Bind to port :** `8080`
   * **Bind to address :** Sélectionner `All interfaces`

### Étape 4 — Redirection du trafic de l'émulateur vers Burp
1. Dans l'émulateur, allez dans **Settings** > **Network & Internet** > **Wi-Fi**.
2. Modifiez les options avancées du Proxy :
   * **Proxy :** `Manual`
   * **Proxy hostname :** `192.168.11.130` *(Votre IP Windows)*
   * **Proxy port :** `8080`

### Étape 5 — Installation du certificat CA de Burp sur Android
1. Ouvrez le navigateur Web de l'émulateur et accédez à : `http://burp`
2. Cliquez sur **CA Certificate** pour télécharger `cacert.der`.
3. Installez-le via : **Settings** > **Security** > **Encryption & credentials** > **Install a certificate** > **CA certificate**.

### Étape 6 — Lancement du serveur Backend (`AndroLabServer`)
```bash
cd D:\Desktop\Android-InsecureBankv2\AndroLabServer
py -2 app.py
```

### Étape 7 — Appairage de l'application et du serveur
Dans InsecureBankv2, configurez :
* **Server IP :** `192.168.11.130`
* **Server Port :** `8888`

### Étape 8 — Analyse et observation des requêtes interceptées
Dans Burp Suite, sous **Proxy** > **HTTP history**, observez les requêtes. Vous verrez les identifiants en clair :
```http
POST /login HTTP/1.1
Host: 192.168.11.130:8888
username=jack&password=********
```

### Étape 9 — Contournement du SSL Pinning avec Frida
```bash
frida -U -f com.android.insecurebankv2 -l sslpin_bypass_universal.js
```

---
---

# LAB-16 : Interception de Trafic HTTPS sur Android avec Objection et Burp Suite

> **Avertissement de sécurité :** Ce lab est réalisé dans un environnement de test contrôlé à des fins strictement pédagogiques. L'application cible est InsecureBankv2.

## 📝 Contexte et Objectifs
L'objectif principal de ce laboratoire est de maîtriser l'interception et l'analyse des communications réseau chiffrées (HTTPS) d'une application Android en désactivant le **SSL Pinning** avec **Objection**.

---

## 🚀 Guide Opérationnel Étape par Étape

### Étape 1 — Configuration du Proxy sur l'Émulateur Android
* **Proxy :** `Manual`
* **Proxy Hostname :** `192.168.11.130`
* **Proxy Port :** `8080`

### Étape 2 — Configuration de l'Écouteur dans Burp Suite
* **Bind to port :** `8080`
* **Bind to address :** `All interfaces`

### Étape 3 & 4 — Téléchargement et Installation du Certificat CA
* Naviguer vers `http://burp`, télécharger `cacert.der`.
* Installer via **Settings** → **Security** → **Encryption & credentials** → **Install a certificate** → **CA certificate**.

### Étape 5 — Initialisation de l'Application via Objection
```bash
objection -g com.android.insecurebankv2 explore
```

### Étape 6 — Désactivation Dynamique du SSL Pinning
Dans le shell interactif d'Objection :
```bash
android sslpinning disable
```

### Étape 7 — Validation et Inspection du Trafic dans Burp Suite
Effectuez une action dans l'application et observez le flux déchiffré dans l'onglet **HTTP history** de Burp Suite.

---
---

# LAB-17 : Cracker OWASP Uncrackable Android Level 3 (Analyse Statique)

> **Avertissement de sécurité :** Ce lab est réalisé dans un environnement de test contrôlé à des fins pédagogiques.

## 📝 Contexte et Objectifs
L'application demande un mot de passe secret. Notre mission est de le retrouver uniquement par analyse statique du code Java (JADX) et de la bibliothèque native C/C++ (Ghidra), puis de déchiffrer le flag (CyberChef).

---

## 🚀 Guide Opérationnel Étape par Étape

### Étape 1 — Analyser la couche Java avec JADX
En explorant la classe `MainActivity` dans JADX, on trouve une chaîne clé :
```java
String xorkey = "pizzapizzapizzapizzapizz";
```

### Étape 2 — Retrouver la logique de vérification dans le code natif
Dans Ghidra, analysez la bibliothèque `libfoo.so` et trouvez la fonction :
```c
Java_sg_vantagepoint_uncrackable3_CodeCheck_bar
```

### Étape 3 — Contourner l'obfuscation du flux de contrôle
La fonction `FUN_001012c0` est obfusquée. En scrollant jusqu'à la fin de la fonction décompilée dans Ghidra, le vrai payload hexadécimal apparaît.

### Étape 4 — Corriger l'Endianness (le sens des octets)
Les valeurs extraites par Ghidra doivent être lues "à l'envers" (Little Endian).

| Valeur Ghidra (Big Endian) | Valeur corrigée (Little Endian) |
| :--- | :--- |
| `0x1549170f1311081d` | `1d0811130f174915` |
| `0x15131d5a1903000d` | `0d0003195a1d1315` |
| `0x14130817005a0e08` | `080e5a0017081314` |

*Payload complet :* `1d0811130f1749150d0003195a1d1315080e5a0017081314`

### Étape 5 — Déchiffrer le flag avec CyberChef
1. **From Hex** : Input = payload hexadécimal.
2. **XOR** : Key = `pizzapizzapizzapizzapizz` (UTF-8).
*Résultat (Le Flag) :* `making owasp great again`

---
---

# LAB-19 : Exploitation SnakeYAML sur Android avec Patching Smali

> **Avertissement de sécurité :** Ce lab est réalisé dans un environnement de test contrôlé. L'application cible est `Snake.apk`.

## 📝 Contexte et Objectifs
L'objectif est de contourner la détection de root en patchant le bytecode Smali de l'APK, puis d'exploiter une vulnérabilité de désérialisation YAML pour extraire un flag.

---

## 🚀 Guide Opérationnel Étape par Étape

### Étape 1 à 5 — Analyse et Patch Smali
* Utilisez **Jadx-GUI** pour identifier que `isDeviceRooted()` bloque l'app.
* Décompilez l'APK avec **Apktool** : `java -jar apktool.jar d Snake.apk -o snake_smali`
* Ouvrez `MainActivity.smali` et patchez la fonction pour qu'elle renvoie toujours false (`0x0`) :
```smali
.method public static isDeviceRooted(Landroid/content/Context;)Z
    .locals 1
    const/4 v0, 0x0
    return v0
.end method
```

### Étape 6 & 7 — Recompilation et Installation
```bash
java -jar apktool.jar b snake_smali -o snake_patched.apk
adb uninstall com.pwnsec.snake
java -jar uber-apk-signer.jar --apks snake_patched.apk
adb install snake_patched-aligned-debugSigned.apk
```

### Étape 8 à 11 — Payload SnakeYAML
* Dans Jadx, on voit que `BigBoss` attend la chaîne `Snaaaaaaaaaaaaaake`.
* Créez `Skull_Face.yml` :
```yaml
!!com.pwnsec.snake.BigBoss ["Snaaaaaaaaaaaaaake"]
```
* Envoyez-le sur l'appareil :
```bash
adb shell mkdir -p /sdcard/snake
adb push Skull_Face.yml /sdcard/snake/Skull_Face.yml
```

### Étape 12 & 13 — Déclenchement et Capture
```bash
# Lancement de l'Intent
adb shell am start -n com.pwnsec.snake/.MainActivity -e SNAKE BigBoss

# Capture du flag
adb logcat | Select-String -Pattern "PWNSEC"
```

---
---

# LAB-20 : Création d'une application Android basique (Toast & Compteur)

## 📝 Contexte et But du Lab
Créer une petite application avec un bouton affichant un "Toast" et un bouton incrémentant un compteur.

---

## 🚀 Guide Opérationnel Étape par Étape

### Étape 1 & 2 — Création de l'interface (XML)
Dans `res/layout/activity_main.xml` :
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)"
    android:layout_width="match_parent" android:layout_height="match_parent"
    android:orientation="vertical" android:gravity="center" android:padding="16dp">

    <TextView android:id="@+id/textView_counter"
        android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="0" android:textSize="64sp" android:layout_marginBottom="32dp"/>

    <Button android:id="@+id/button_toast"
        android:layout_width="match_parent" android:layout_height="wrap_content"
        android:text="Afficher un Toast" android:layout_marginBottom="16dp"/>

    <Button android:id="@+id/button_count"
        android:layout_width="match_parent" android:layout_height="wrap_content"
        android:text="Incrémenter le compteur"/>
</LinearLayout>
```

### Étape 3 — Écriture de la logique (Java)
Dans `MainActivity.java` :
```java
package com.example.toastandcountapp;

import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity {
    private int mCount = 0;
    private TextView mShowCount;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mShowCount = findViewById(R.id.textView_counter);
        Button btnToast = findViewById(R.id.button_toast);
        Button btnCount = findViewById(R.id.button_count);

        btnToast.setOnClickListener(v -> Toast.makeText(this, "Bonjour ! Voici un Toast 🍞", Toast.LENGTH_SHORT).show());
        btnCount.setOnClickListener(v -> {
            mCount++;
            mShowCount.setText(Integer.toString(mCount));
        });
    }
}
```

---
---

# LAB 2 : Calculateur d'impôts locaux : Saisie, traitement et affichage

## 📝 Contexte et But du Lab
Créer une application permettant de calculer le montant total des impôts locaux selon : le nom, la surface (m²), le nombre de pièces, et la présence d'une piscine.

---

## 🚀 Guide Opérationnel Étape par Étape

### Étape 1 — Création de l'interface (XML)
Dans `res/layout/activity_main.xml` :
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="[http://schemas.android.com/apk/res/android](http://schemas.android.com/apk/res/android)"
    android:layout_width="match_parent" android:layout_height="match_parent"
    android:orientation="vertical" android:padding="20dp">

    <TextView android:layout_width="match_parent" android:layout_height="wrap_content"
        android:text="Calcul des impôts locaux" android:textSize="24sp" android:textStyle="bold" android:gravity="center" android:layout_marginBottom="24dp"/>

    <EditText android:id="@+id/edit_nom" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Nom du propriétaire" android:layout_marginBottom="12dp"/>
    <EditText android:id="@+id/edit_surface" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Surface (m²)" android:inputType="numberDecimal" android:layout_marginBottom="12dp"/>
    <EditText android:id="@+id/edit_pieces" android:layout_width="match_parent" android:layout_height="wrap_content" android:hint="Nombre de pièces" android:inputType="number" android:layout_marginBottom="12dp"/>
    <CheckBox android:id="@+id/check_piscine" android:layout_width="wrap_content" android:layout_height="wrap_content" android:text="Présence d'une piscine" android:layout_marginBottom="24dp"/>
    
    <Button android:id="@+id/btn_calculer" android:layout_width="match_parent" android:layout_height="wrap_content" android:text="Calculer l'impôt" android:layout_marginBottom="24dp"/>
    <TextView android:id="@+id/txt_resultat" android:layout_width="match_parent" android:layout_height="wrap_content" android:textSize="18sp" android:textStyle="bold" android:textColor="#D32F2F" android:gravity="center"/>
</LinearLayout>
```

### Étape 2 — Écrire le code Java
Dans `MainActivity.java` :
```java
package com.example.calculateurimpots;

import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.widget.Button;
import android.widget.CheckBox;
import android.widget.EditText;
import android.widget.TextView;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        EditText editNom = findViewById(R.id.edit_nom);
        EditText editSurface = findViewById(R.id.edit_surface);
        EditText editPieces = findViewById(R.id.edit_pieces);
        CheckBox checkPiscine = findViewById(R.id.check_piscine);
        Button btnCalculer = findViewById(R.id.btn_calculer);
        TextView txtResultat = findViewById(R.id.txt_resultat);

        btnCalculer.setOnClickListener(v -> {
            String nom = editNom.getText().toString();
            if (nom.isEmpty() || editSurface.getText().toString().isEmpty() || editPieces.getText().toString().isEmpty()) {
                Toast.makeText(this, "Veuillez remplir tous les champs", Toast.LENGTH_SHORT).show();
                return;
            }
            try {
                double surface = Double.parseDouble(editSurface.getText().toString());
                int pieces = Integer.parseInt(editPieces.getText().toString());
                
                double impotBase = surface * 15.0;
                double impotPieces = pieces * 50.0;
                double impotPiscine = checkPiscine.isChecked() ? 200.0 : 0.0;
                
                double total = impotBase + impotPieces + impotPiscine;
                txtResultat.setText("Propriétaire : " + nom + "\nTotal : " + String.format("%.2f", total) + " €");
            } catch (Exception e) {
                Toast.makeText(this, "Erreur de saisie", Toast.LENGTH_SHORT).show();
            }
        });
    }
}
```
