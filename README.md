# LAB 18 : FireStorm - Résolution détaillée


## 📌 Contexte du challenge

L'application Android **FireStorm** contient une fonction `Password()` qui génère un mot de passe pour se connecter à une base de données Firebase. 

**Problème :** Cette fonction n'est **jamais appelée** dans le flux normal de l'application. Elle existe dans le code mais reste inactive

## 📌 Objectif du challenge

1. Forcer l'exécution de `Password()` avec Frida
2. Récupérer le mot de passe généré
3. S'authentifier sur Firebase
4. Récupérer le flag
   
---

## 📊 Processus complet du challenge

| Étape | Action | Outil | Résultat |
|-------|--------|-------|----------|
| **1** | Analyser l'APK pour trouver `Password()` et les strings | JADX GUI | Localisation de la méthode et des identifiants Firebase |
| **2** | Injecter un script Frida pour forcer l'appel de `Password()` | Frida | Récupération du mot de passe généré |
| **3** | S'authentifier sur Firebase avec email + mot de passe | Python + Firebase REST API | Obtention d'un token d'authentification |
| **4** | Récupérer le flag depuis la base de données Firebase | Python + Firebase REST API | Flag final |

### Détail des étapes

| Étape | Description détaillée |
|-------|----------------------|
| **Analyse statique** | Ouverture de l'APK avec JADX, localisation de `MainActivity` et de la méthode `Password()`, extraction des strings depuis `strings.xml`, récupération des identifiants Firebase |
| **Injection Frida** | Création d'un script JavaScript, utilisation de `Java.choose()` pour trouver l'instance `MainActivity`, appel forcé de `instance.Password()`, récupération du mot de passe |
| **Authentification Firebase** | Utilisation de l'API REST Firebase, authentification avec email + mot de passe, obtention d'un token |
| **Récupération du flag** | Requête GET sur la base de données Firebase, lecture du flag stocké |



## 🎯 Ce que nous avons appris

| Concept | Définition |
|---------|-------------|
| **Java.choose()** | Fonction Frida pour trouver des instances Java en mémoire |
| **Instance d'activité** | Objet vivant d'une classe Activity créé par Android |
| **Méthode native** | Fonction implémentée en C/C++ dans une bibliothèque `.so` |
| **Firebase REST API** | Interface pour interagir avec Firebase sans SDK |
| **Pyrebase / Requests** | Bibliothèques Python pour communiquer avec Firebase |

---

## 🛠️ Environnement et outils utilisés

| Élément | Valeur |
|---------|--------|
| **VM** | Mobexler |
| **Outil d'analyse** | JADX GUI |
| **Instrumentation** | Frida v16.7.19 |
| **APK analysé** | FireStorm.apk |
| **Package** | com.pwnsec.firestorm |

---


---

## 🔧 Analyse statique avec JADX

### Strings.xml (valeurs clés)

| Clé | Valeur |
|-----|--------|
| `Friday_Night` | `It's Friday, and PwnSec CTF is here!!!!!` |
| `Author` | `TK757567` |
| `JustRandomString` | `or_is_it_random???` |
| `URL` | `https://pwnsec.xyz/flag?auth=` |
| `IDKMaybethepasswordpassowrd` | `v1n4of.5EY?%0z` |
| `Token` | `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...` |
| `firebase_email` | `TK757567@pwnsec.xyz` |
| `firebase_database_url` | `https://firestorm-9d3db-default-rtdb.firebaseio.com` |
| `google_api_key` | `AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY` |

### Méthode Password() dans MainActivity

```java
public String Password() {
    String s1 = getString(R.string.Friday_Night);
    String s2 = getString(R.string.Author);
    String s3 = getString(R.string.JustRandomString);
    String s4 = getString(R.string.URL);
    String s5 = getString(R.string.IDKMaybethepasswordpassowrd);
    String s6 = getString(R.string.Token);
    
    sb.append(s1.substring(5, 9));    // "Frid"
    sb.append(s4.substring(1, 6));    // "ttps:"
    sb.append(s2.substring(2, 6));    // "7575"
    sb.append(s5.substring(5, 8));    // "f.5"
    sb.append(s3);                     // "or_is_it_random???"
    sb.append(s6.substring(18, 26));   // extrait du token
    
    return generateRandomString(String.valueOf(sb));
}
```


## 🚀 Script Frida

### frida_firestorm.js

```javascript
Java.perform(function() {

    function getPassword() {
        console.log("[*] Recherche d'instance de MainActivity...");

        Java.choose('com.pwnsec.firestorm.MainActivity', {
            onMatch: function(instance) {
                console.log("[+] MainActivity instance trouvée");
                try {
                    var password = instance.Password();
                    console.log("[+] Mot de passe généré : " + password);
                } catch(e) {
                    console.log("[-] Erreur : " + e);
                }
            },
            onComplete: function() {
                console.log("[*] Recherche terminée");
            }
        });
    }

    console.log("[*] Script chargé, attente 3 secondes...");
    setTimeout(getPassword, 3000);
});
```

### Execution 

```bash

# Trouver le PID de Firestorm
frida-ps -U | grep -i firestorm

# Attacher au processus (remplacer 2028 par le PID trouvé)
frida -U -p 2028 -l frida_firestorm.js

```

### Résultat attendu

```bash

[+] Mot de passe généré : C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC

```

## 🐍 Script Python pour Firebase

### get_flag.py

```python
import requests
import json

api_key = "AIzaSyAXsK0qsx4RuLSA9C8IPSWd0eQ67HVHuJY"
email = "TK757567@pwnsec.xyz"
password = "C7_dotpsC7t7f_._In_i.IdttpaofoaIIdIdnndIfC"

# Authentification
url = f"https://identitytoolkit.googleapis.com/v1/accounts:signInWithPassword?key={api_key}"
payload = {
    "email": email,
    "password": password,
    "returnSecureToken": True
}

print("[*] Authentification...")
response = requests.post(url, json=payload)
data = response.json()

if "idToken" in data:
    id_token = data["idToken"]
    print("[+] Connexion réussie !")
    
    # Récupération du flag
    db_url = "https://firestorm-9d3db-default-rtdb.firebaseio.com/.json"
    params = {"auth": id_token}
    
    print("[*] Récupération du flag...")
    response = requests.get(db_url, params=params)
    flag_data = response.json()
    
    print("\n[+] FLAG récupéré :")
    print(json.dumps(flag_data, indent=2))
else:
    print("[-] Erreur:", data.get("error", {}).get("message", "Unknown error"))

```

### Execution 

```bash

python3 get_flag.py

```

### Résultat attendu

```bash

[*] Authentification...
[+] Connexion réussie !
[*] Récupération du flag...

[+] FLAG récupéré :
"PWNSEC{C0ngr4ts_Th4t_w45_4N_345y_P4$$w0rd_t0_G3t!!!_0R_!5_!t???}"

```


## 📸 Screenshots

| # | Description |
|---|-------------|
| <img width="1598" height="809" alt="Etap2" src="https://github.com/user-attachments/assets/c371f65c-1289-4b09-b6e6-b6dd33e68250" />| JADX - Méthode `Password()` |
| <img width="1600" height="792" alt="Etap2_1" src="https://github.com/user-attachments/assets/503722ae-8316-4084-978a-7ba582363bff" />| JADX - `strings.xml` (valeurs) |
| <img width="932" height="601" alt="Etap3" src="https://github.com/user-attachments/assets/fa6213c0-b7f0-4592-ad31-33a50108da66" />| Frida - Script d'injection |
| <img width="948" height="516" alt="Etap3_1" src="https://github.com/user-attachments/assets/3989136a-a59d-4e4e-8118-d2ad3f7692a7" />| Frida - Injection et mot de passe |
| <img width="1421" height="817" alt="Etap4" src="https://github.com/user-attachments/assets/fa5c5f9d-b04a-49a8-a9e3-3b0561d24eb1" />| Python - Scrip d'authentification Firebase |
| <img width="909" height="338" alt="Etap4_1" src="https://github.com/user-attachments/assets/fd3ec901-23ae-42fd-ab08-56d1086ca039" />| Python - Flag récupéré |




## 👤 Auteur

**El Hachimi Abdelhamid**  
Date : 2026-04-23  
Cours : Sécurité des applications mobiles







