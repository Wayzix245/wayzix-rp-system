Oui, avec ça je peux te faire une **vraie API doc/dev doc** pour ton système.
Et ton README actuel est beaucoup trop vide par rapport à ce que ton code expose déjà. 

Le plus important : ce que tu appelles “API” ici, ce n’est pas une API web HTTP.
C’est une **API interne C# s&box** + des **ConCmd serveur** que tes potes pourront utiliser pour brancher leurs propres systèmes dessus.

---

# README / API DOC — Wayzix RP System

## Vue d’ensemble

Le **Wayzix RP System** est un système RP de base pour s&box qui gère :

* l’identité RP du joueur
* le nom complet affiché en jeu
* l’argent
* le job
* les slots de personnages
* le chargement / la sauvegarde
* le cache actif du personnage
* un pont simple entre l’UI et les données gameplay

Le système est pensé pour que d’autres développeurs puissent créer leurs propres modules autour de lui :

* HUD
* jobs avancés
* inventaire
* téléphone
* banque
* factions
* système de spawn
* menus de création / sélection de personnage
* admin tools

---

# Architecture générale

Le système repose principalement sur 3 blocs :

## 1. `RpCharacter`

C’est le **coeur du système**.

Il gère :

* les données persistantes d’un personnage
* la création
* le chargement
* la sauvegarde
* les slots
* le cache actif
* le nettoyage des noms
* les getters/setters gameplay

## 2. `RpIdentity`

C’est le **helper côté UI / client**.

Il sert à :

* récupérer le prénom / nom / nom complet local
* gérer une preview locale avant sauvegarde définitive
* envoyer la commande serveur pour sauvegarder l’identité
* faire le lien entre interface et `RpCharacter`

## 3. `RpCommands`

C’est l’interface de debug / interaction via **console commands**.

Il sert à :

* lire les données
* modifier le personnage
* changer le slot
* sauvegarder / reload
* tester rapidement le système

---

# Données persistées

## Classe `RpCharacterData`

Cette classe représente un personnage RP.

```csharp
public sealed class RpCharacterData
{
    public long CharacterId { get; set; }
    public long SteamId { get; set; }
    public int Slot { get; set; } = 1;

    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
    public string WyzRpName { get; set; } = "";

    public long Money { get; set; } = 0;
    public string Job { get; set; } = "Citoyen";
}
```

## Champs

### `CharacterId`

ID unique du personnage.

### `SteamId`

SteamID64 du joueur propriétaire.

### `Slot`

Numéro du slot du personnage.

### `FirstName`

Prénom RP nettoyé.

### `LastName`

Nom RP nettoyé.

### `WyzRpName`

Nom complet final affiché en jeu.
Construit automatiquement à partir de `FirstName + LastName`.

### `Money`

Argent du personnage.
Le système empêche les valeurs négatives.

### `Job`

Job actuel du personnage.
Valeur par défaut : `"Citoyen"`.

---

# Stockage des fichiers

Le système stocke les personnages dans `FileSystem.Data`.

## Arborescence

```txt
rp/
├── next_character_id.txt
└── characters/
    ├── STEAMID_SLOT.txt
    ├── 7656119..._1.txt
    ├── 7656119..._2.txt
```

## Fichiers utilisés

### `rp/next_character_id.txt`

Stocke le prochain `CharacterId` à attribuer.

### `rp/characters/{steamId}_{slot}.txt`

Stocke les données d’un personnage pour un joueur et un slot donné.

Exemple :

```txt
CharacterId=1
SteamId=76561198000000000
Slot=1
FirstName=Wayzix
LastName=Sabaku
WyzRpName=Wayzix Sabaku
Money=2500
Job=Citoyen
```

---

# API principale : `RpCharacter`

## Initialisation

### `EnsureInitialized()`

Initialise le système si ce n’est pas déjà fait.

Crée automatiquement :

* `rp/`
* `rp/characters/`
* `rp/next_character_id.txt`

À appeler au démarrage de tout système qui dépend du RP.

Exemple :

```csharp
RpCharacter.EnsureInitialized();
```

---

## Gestion du personnage actif

### `GetOrCreateActive(long steamId)`

Retourne le personnage actif du joueur depuis le cache.

Si aucun personnage n’est chargé :

* lit le slot actif
* tente de charger le personnage
* sinon crée un nouveau personnage automatiquement

Exemple :

```csharp
var data = RpCharacter.GetOrCreateActive( steamId );
```

### `Unload(long steamId)`

Retire le personnage actif du cache mémoire.

Exemple :

```csharp
RpCharacter.Unload( steamId );
```

---

## Slots

### `GetActiveSlot(long steamId)`

Retourne le slot actuellement actif en mémoire.

Important :

* le slot actif est **en mémoire seulement**
* il n’est **pas persisté après restart** dans cette version

Exemple :

```csharp
int slot = RpCharacter.GetActiveSlot( steamId );
```

### `SetActiveSlot(long steamId, int slot)`

Définit le slot actif et charge ce personnage dans le cache.

Si le slot n’existe pas, il est créé.

Exemple :

```csharp
RpCharacter.SetActiveSlot( steamId, 2 );
```

### `CreateCharacter(long steamId, int slot)`

Crée un personnage sur un slot précis si ce slot est vide.

Si un personnage existe déjà sur ce slot, il est simplement retourné.

Exemple :

```csharp
var data = RpCharacter.CreateCharacter( steamId, 1 );
```

### `LoadBySteamIdAndSlot(long steamId, int slot)`

Charge directement un personnage depuis un slot.

Si rien n’existe, retourne un `RpCharacterData` vide.

Exemple :

```csharp
var data = RpCharacter.LoadBySteamIdAndSlot( steamId, 3 );
```

---

## Sauvegarde / chargement

### `Save(RpCharacterData data)`

Sauvegarde les données du personnage sur disque.

Effets :

* sanitize les données
* attribue un `CharacterId` si besoin
* écrit le fichier
* met à jour le cache actif
* met à jour le slot actif

Exemple :

```csharp
var data = RpCharacter.GetOrCreateActive( steamId );
data.Job = "Police";
RpCharacter.Save( data );
```

---

## Identité

### `HasIdentity(long steamId)`

Retourne `true` si le personnage a un prénom et un nom valides.

Exemple :

```csharp
if ( RpCharacter.HasIdentity( steamId ) )
{
    // le joueur a déjà une identité
}
```

### `TrySetIdentity(long steamId, string firstName, string lastName)`

Définit l’identité du personnage.

Retourne `false` si le prénom ou le nom sont invalides.

Exemple :

```csharp
bool ok = RpCharacter.TrySetIdentity( steamId, "Wayzix", "Sabaku" );
```

### `GetFullName(long steamId)`

Retourne le nom RP complet.

Retourne `"Nom inconnu"` si vide.

Exemple :

```csharp
string fullName = RpCharacter.GetFullName( steamId );
```

---

## Jobs

### `GetJob(long steamId)`

Retourne le job actuel.

Exemple :

```csharp
string job = RpCharacter.GetJob( steamId );
```

### `SetJob(long steamId, string job)`

Modifie le job du personnage.

Si la valeur est vide, le job devient `"Citoyen"`.

Exemple :

```csharp
RpCharacter.SetJob( steamId, "Médecin" );
```

---

## Argent

### `GetMoney(long steamId)`

Retourne l’argent actuel.

Exemple :

```csharp
long money = RpCharacter.GetMoney( steamId );
```

### `SetMoney(long steamId, long money)`

Définit directement l’argent.

La valeur minimale est `0`.

Exemple :

```csharp
RpCharacter.SetMoney( steamId, 5000 );
```

### `AddMoney(long steamId, long amount)`

Ajoute ou retire de l’argent.

Si le résultat passe sous zéro, il est clamp à `0`.

Exemple :

```csharp
RpCharacter.AddMoney( steamId, 250 );
RpCharacter.AddMoney( steamId, -100 );
```

---

## Nettoyage des noms

### `CleanNamePart(string value, int maxLen)`

Nettoie un prénom ou un nom.

Caractères autorisés :

* lettres
* espace
* apostrophe
* tiret

Le système :

* supprime les caractères invalides
* évite les doubles espaces
* coupe selon `maxLen`

Exemple :

```csharp
string clean = RpCharacter.CleanNamePart( "  Jean--Paul!! ", 16 );
```

---

# API UI / Client : `RpIdentity`

`RpIdentity` est prévu pour les menus, HUD et prompts.

## À quoi sert cette classe

Elle permet de :

* lire rapidement le prénom / nom / nom complet local
* afficher une preview avant validation réelle
* déclencher la sauvegarde serveur via commande console

---

## Méthodes utiles

### `ClearLocalPreview()`

Vide la preview locale.

Exemple :

```csharp
RpIdentity.ClearLocalPreview();
```

### `CleanNamePart(string value, int maxLen)`

Passe directement par `RpCharacter.CleanNamePart`.

Exemple :

```csharp
var clean = RpIdentity.CleanNamePart( input, 16 );
```

### `HasIdentity()`

Retourne si le joueur local a une identité.

Exemple :

```csharp
if ( RpIdentity.HasIdentity() )
{
    // afficher HUD RP
}
```

### `GetFullName()`

Retourne le nom complet local.

Exemple :

```csharp
var name = RpIdentity.GetFullName();
```

### `TrySetIdentity(string firstName, string lastName)`

Définit localement la preview puis :

* si host : sauvegarde directe via `RpCharacter`
* sinon : envoie `rp_set_identity` au serveur

Exemple :

```csharp
RpIdentity.TrySetIdentity( "Wayzix", "Sabaku" );
```

---

## Propriétés utiles

### `RpIdentity.FirstName`

Retourne le prénom local.

Priorité :

1. preview locale
2. personnage actif

### `RpIdentity.LastName`

Retourne le nom local.

### `RpIdentity.WyzRpName`

Retourne le nom complet local affichable dans le HUD/UI.

Comportement :

* utilise d’abord la preview locale
* sinon lit `RpCharacter.GetFullName`

---

# ConCmd disponibles : `RpCommands`

Toutes ces commandes sont utiles pour debug, admin tools, tests, ou intégration rapide.

---

## Initialisation

### `rp_init`

Initialise le système RP.

```txt
rp_init
```

---

## Lecture des infos joueur

### `rp_me`

Affiche toutes les infos du personnage actif :

* SteamId
* CharacterId
* Slot
* Name
* First
* Last
* Money
* Job

```txt
rp_me
```

### `rp_name_get`

Affiche le nom RP.

```txt
rp_name_get
```

### `rp_job_get`

Affiche le job.

```txt
rp_job_get
```

### `rp_money_get`

Affiche l’argent.

```txt
rp_money_get
```

### `rp_has_identity`

Indique si le joueur a une identité.

```txt
rp_has_identity
```

---

## Modification de l’identité

### `rp_set_identity <firstName> <lastName>`

Définit le prénom et le nom du joueur.

```txt
rp_set_identity Wayzix Sabaku
```

---

## Modification du job

### `rp_set_job <job>`

Définit le job.

```txt
rp_set_job Citoyen
rp_set_job Policier
rp_set_job Medecin
```

Note :
si le job contient des espaces, selon le contexte de console il faudra parfois gérer ça côté appel.

---

## Gestion de l’argent

### `rp_set_money <amount>`

Définit l’argent.

```txt
rp_set_money 5000
```

### `rp_add_money <amount>`

Ajoute de l’argent.

```txt
rp_add_money 250
```

### `rp_take_money <amount>`

Retire de l’argent.

```txt
rp_take_money 100
```

---

## Sauvegarde / rechargement

### `rp_save`

Sauvegarde le personnage actif.

```txt
rp_save
```

### `rp_reload`

Vide le cache puis recharge le personnage actif.

```txt
rp_reload
```

---

## Slots de personnages

### `rp_slot_create <slot>`

Crée ou charge un personnage sur un slot.

```txt
rp_slot_create 1
rp_slot_create 2
```

### `rp_slot_info <slot>`

Affiche les infos du slot demandé.

```txt
rp_slot_info 2
```

### `rp_slot_use <slot>`

Charge le slot demandé comme slot actif.

```txt
rp_slot_use 2
```

### `rp_slot_copy_active_to <slot>`

Copie le personnage actif dans un autre slot.

```txt
rp_slot_copy_active_to 3
```

---

## Outils preview / nettoyage

### `rp_identity_clear_preview`

Vide la preview locale du nom.

```txt
rp_identity_clear_preview
```

### `rp_name_preview`

Affiche la preview locale actuelle.

```txt
rp_name_preview
```

### `rp_name_clean <raw> <maxLen>`

Teste le nettoyage d’un texte de nom.

```txt
rp_name_clean "Jean@@@ Dupont" 16
```

---

## Debug rapide

### `rp_debug_set_demo`

Applique un profil démo :

* FirstName = Wayzix
* LastName = Sabaku
* WyzRpName = Wayzix Sabaku
* Money = 2500
* Job = Citoyen

```txt
rp_debug_set_demo
```

### `rp_debug_reset`

Reset le profil actif :

* nom vide
* argent 0
* job Citoyen

```txt
rp_debug_reset
```

---

# Comment brancher son propre système dessus

## Exemple : afficher le nom dans une UI

```csharp
var rpName = RpCharacter.GetFullName( steamId );
```

## Exemple : vérifier si le joueur a créé son identité

```csharp
if ( !RpCharacter.HasIdentity( steamId ) )
{
    // ouvrir menu de création d'identité
}
```

## Exemple : donner un job

```csharp
RpCharacter.SetJob( steamId, "Police" );
```

## Exemple : payer un joueur

```csharp
RpCharacter.AddMoney( steamId, 500 );
```

## Exemple : acheter un item

```csharp
var money = RpCharacter.GetMoney( steamId );

if ( money >= 250 )
{
    RpCharacter.AddMoney( steamId, -250 );
    // give item
}
```

## Exemple : utiliser les slots

```csharp
RpCharacter.SetActiveSlot( steamId, 2 );
var active = RpCharacter.GetOrCreateActive( steamId );
```

---

# Comment construire un nouveau module proprement

## Exemple de logique recommandée

### Un système téléphone

* lit le nom avec `RpCharacter.GetFullName`
* lit l’argent avec `RpCharacter.GetMoney`

### Un système métier

* utilise `GetJob` / `SetJob`
* ajoute des permissions selon le job

### Un inventaire

* stocke ses propres données ailleurs
* se base sur `CharacterId` pour relier un inventaire à un personnage

### Une banque

* utilise `CharacterId` ou `SteamId + Slot`
* ne modifie pas directement autre chose que `Money` si compte principal

---

# HUD fourni

Le HUD `DarkRpHud` affiche :

* avatar Steam
* HP
* Armor
* Stamina
* nom RP
* argent

## Comportement

Le HUD est caché si :

* le joueur n’a pas encore d’identité
* le mode cinématique est activé

Condition utilisée :

```razor
RpIdentity.HasIdentity() && !CinematicModeSystem.CinematicEnabled
```

## Valeurs affichées

Le nom vient de :

```csharp
RpCharacter.GetFullName( steamId )
```

L’argent vient de :

```csharp
RpCharacter.GetMoney( steamId )
```

---

# Prompt d’identité fourni

Le composant `RpIdentityPrompt` sert à faire remplir :

* prénom
* nom

Puis à appeler :

```csharp
RpIdentity.TrySetIdentity( firstName, lastName );
```

## Important UI

Tu respectes déjà une bonne base :

* root en `pointer-events: none`
* overlay/fenêtre en `pointer-events: all`

Ça évite qu’un panel caché bloque les clics des autres menus.

---

# Bonnes pratiques pour les devs qui vont utiliser ton système

## 1. Toujours appeler `EnsureInitialized()`

Avant d’utiliser le système au boot d’un module important.

## 2. Toujours passer par `RpCharacter`

Ne modifie pas les fichiers à la main.

## 3. Utiliser `CharacterId` pour les futurs systèmes

Pour :

* inventaire
* appartement
* banque
* progression
* licence
* téléphone

`SteamId` identifie le joueur.
`CharacterId` identifie le personnage.

## 4. Ne pas reconstruire le nom RP à la main

Toujours utiliser :

```csharp
RpCharacter.GetFullName( steamId )
```

## 5. Utiliser `CleanNamePart`

Si tu fais un autre menu de création d’identité.

---

# Limitations actuelles

## 1. Pas de vraie base SQL

Pour l’instant le système stocke en fichiers texte dans `FileSystem.Data`.

## 2. Slot actif non persisté après restart

Le slot actif reste en mémoire seulement.

## 3. Pas de sync réseau dédiée

Le système repose sur :

* cache local
* commandes console
* accès direct partagé

## 4. Pas encore de permissions admin

Les commandes serveur existent, mais il n’y a pas encore de couche admin/owner/modérateur.

## 5. Pas encore de suppression de slot

On peut créer, charger, copier, mais pas supprimer proprement un personnage via API publique.

---

# Points à corriger / améliorer dans ton code

J’te le dis franchement pour que ce soit carré :

## 1. Tu as collé deux fois le même SCSS de `RpIdentityPrompt`

Dans ton message, le bloc `.razor.scss` est dupliqué.
Faut garder une seule version.

## 2. `rp_slot_use` ne force pas vraiment le slot actif proprement

Là tu fais :

```csharp
var data = RpCharacter.LoadBySteamIdAndSlot( steamId, slot );
RpCharacter.Save( data );
```

Ça marche indirectement parce que `Save` réécrit `ActiveSlots[data.SteamId] = data.Slot`,
mais le plus propre serait directement :

```csharp
RpCharacter.SetActiveSlot( steamId, slot );
var data = RpCharacter.GetOrCreateActive( steamId );
```

## 3. Ton système est appelé “API”, mais c’est une API interne

Donc dans le README faut bien écrire :

* **Internal API**
* **Gameplay API**
* **Character API**
  et pas faire croire que c’est une REST API.

## 4. Si plus tard tu veux un système vraiment solide

Le mieux sera :

* SQLite
* slot actif persisté
* sync réseau propre
* layer admin
* événements/hooks quand job/money/identity changent

---

# TL;DR pour les devs

## Lire les données

```csharp
RpCharacter.GetFullName( steamId );
RpCharacter.GetMoney( steamId );
RpCharacter.GetJob( steamId );
RpCharacter.HasIdentity( steamId );
```

## Modifier les données

```csharp
RpCharacter.TrySetIdentity( steamId, "Wayzix", "Sabaku" );
RpCharacter.SetJob( steamId, "Citoyen" );
RpCharacter.SetMoney( steamId, 1000 );
RpCharacter.AddMoney( steamId, 250 );
```

## Gérer les slots

```csharp
RpCharacter.CreateCharacter( steamId, 1 );
RpCharacter.SetActiveSlot( steamId, 2 );
RpCharacter.LoadBySteamIdAndSlot( steamId, 2 );
```

## Sauvegarder

```csharp
var data = RpCharacter.GetOrCreateActive( steamId );
RpCharacter.Save( data );
```

## Côté UI

```csharp
RpIdentity.HasIdentity();
RpIdentity.WyzRpName;
RpIdentity.TrySetIdentity( firstName, lastName );
```

---

Si tu veux, je peux maintenant te refaire ça en **README GitHub complet en markdown propre**, directement prêt à coller dans ton repo.
