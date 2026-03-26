# 🧠 Wayzix RP System (s&box)

Système RP complet pour s&box inspiré de DarkRP, avec gestion de personnages, identité, argent, job et slots.

---

# 🚀 Fonctionnalités

✔ Système de personnages (SteamID + slots)  
✔ Identité RP (FirstName / LastName / WyzRpName)  
✔ Argent (Money)  
✔ Jobs (Job)  
✔ Sauvegarde automatique  
✔ Cache optimisé  
✔ Persistance via FileSystem (pas de SQLite, pas de JSON)  
✔ Commandes console complètes  
✔ Compatible UI / HUD  

---

# 🗂 Structure des données

Chaque joueur possède :

- `CharacterId` → ID unique du personnage  
- `SteamId` → ID Steam du joueur  
- `Slot` → slot du personnage (multi personnages)  
- `FirstName` → prénom RP  
- `LastName` → nom RP  
- `WyzRpName` → nom complet généré  
- `Money` → argent  
- `Job` → métier  

📁 Stockage :
```
data/rp/characters/steamid_slot.txt
```

---

# 🎮 Commandes Console

## 👤 Identité

### Définir identité
```
rp_set_identity Prenom Nom
```

### Voir nom RP
```
rp_name_get
```

### Vérifier identité
```
rp_has_identity
```

---

## 💰 Argent

### Voir argent
```
rp_money_get
```

### Définir argent
```
rp_set_money 5000
```

### Ajouter argent
```
rp_add_money 1000
```

### Retirer argent
```
rp_take_money 500
```

---

## 💼 Job

### Voir job
```
rp_job_get
```

### Définir job
```
rp_set_job Policier
```

---

## 👤 Personnage

### Voir toutes les infos
```
rp_me
```

### Sauvegarder
```
rp_save
```

### Recharger
```
rp_reload
```

---

## 📂 Slots (multi personnages)

### Créer slot
```
rp_slot_create 2
```

### Voir slot
```
rp_slot_info 2
```

### Utiliser slot
```
rp_slot_use 2
```

### Copier perso actuel vers slot
```
rp_slot_copy_active_to 3
```

---

## 🧪 Debug

### Profil test
```
rp_debug_set_demo
```

### Reset personnage
```
rp_debug_reset
```

---

# 🧠 Fonctionnement interne

## 🔁 Cache optimisé

- Les données sont mises en cache en mémoire
- Évite les accès disque inutiles
- Sync automatique via `Save()`

---

## 🧹 Nettoyage des noms

Fonction utilisée :
```csharp
CleanNamePart(string value, int maxLen)
```

✔ Supprime caractères invalides
✔ Limite la taille
✔ Évite spam / injection

---

## 🧩 Génération du nom RP

```csharp
BuildRpName(firstName, lastName)
```

Résultat :

```
Prenom Nom
```

---

# 🧪 Fonctions principales

## 🔹 Récupérer personnage

```csharp
RpCharacter.GetOrCreateActive(steamId)
```

## 🔹 Sauvegarder

```csharp
RpCharacter.Save(data)
```

## 🔹 Définir identité

```csharp
RpCharacter.TrySetIdentity(steamId, firstName, lastName)
```

## 🔹 Argent

```csharp
RpCharacter.SetMoney(steamId, amount)
RpCharacter.AddMoney(steamId, amount)
RpCharacter.GetMoney(steamId)
```

## 🔹 Job

```csharp
RpCharacter.SetJob(steamId, job)
RpCharacter.GetJob(steamId)
```

---

# 🖥 UI / HUD

Le système est compatible avec :

✔ Prompt identité
✔ HUD RP
✔ Mise à jour dynamique

---

# ⚠️ Important

* Le slot actif n'est pas encore persisté après restart
* Le système fonctionne sans SQLite (plus simple / compatible s&box)
* Chaque joueur a 1 slot par défaut

---

# 🔥 Exemple d'utilisation

```
rp_set_identity Naruto Uzumaki
rp_set_job Hokage
rp_add_money 10000
rp_me
```

---

# 🛠 Tech

* s&box Sandbox API
* FileSystem.Data
* Cache mémoire optimisé
* Architecture modulaire