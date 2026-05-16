# Guide de Gestion des Contributeurs

Ce document explique comment gérer la liste des contributeurs affichée dans la page **Crédits** de Better Mod Manager (BMM).

## 🚀 Fonctionnement du Système

La liste des contributeurs n'est plus codée en dur dans l'application. Elle est chargée dynamiquement à chaque démarrage :

1.  **Source Principale (GitHub) :** L'application tente de télécharger le fichier `contributors.json` depuis le dépôt distant. Cela permet de mettre à jour les crédits en temps réel sans que les utilisateurs aient besoin de télécharger une nouvelle version de BMM.
2.  **Fallback Local :** Si l'utilisateur est hors-ligne ou si GitHub est inaccessible, BMM utilise le fichier `frontend/assets/contributors.json` inclus dans l'installation.

## 📂 Structure du fichier `contributors.json`

Le fichier contient deux sections principales : les contributeurs et les messages du bandeau défilant (marquee).

```json
{
    "contributors": [
        {
            "id": "identifiant_unique",
            "username": "Nom d'utilisateur",
            "display_name": "Nom d'affichage (Optionnel)",
            "pfp": "Lien vers l'image ou chemin local",
            "role": "Le rôle affiché (Texte fixe)",
            "description": "Le message ou bio (Texte fixe)",
            "category": "staff | kofi | tester",
            "subcategory": "dev | community_support | early_access | ptb",
            "github": "Lien GitHub (Optionnel)",
            "website": "Lien site web (Optionnel)",
            "discord_id": "ID Discord (Optionnel)"
        }
    ],
    "messages": [
        "Texte du premier message défilant",
        "Texte du second message",
        "..."
    ]
}
```

## 🛠️ Comment ajouter un contributeur ou un message

### 1. Préparer l'image (pfp)
Vous avez deux options pour la photo de profil :
*   **Lien externe (Recommandé) :** Utilisez un lien direct (ex: `https://avatars.githubusercontent.com/u/12345`).
*   **Fichier local :** Placez l'image dans `frontend/assets/userpfp/` et utilisez le chemin `assets/userpfp/nom_image.png`.

### 2. Modifier le JSON
Ajoutez une nouvelle entrée dans le fichier `contributors.json`. 

> [!IMPORTANT]
> Contrairement au reste de l'application, les champs **role** et **description** ici ne sont PAS des clés de traduction. Écrivez directement le texte que vous voulez voir apparaître.

**Exemple pour un donateur Ko-fi :**
```json
{
    "id": "nouveau_donateur",
    "username": "PseudoDonateur",
    "display_name": "Super Donateur",
    "pfp": "https://link-to-pfp.com/image.png",
    "role": "Généreux Donateur Ko-fi",
    "description": "Merci énormément pour ton soutien au projet !",
    "category": "kofi"
}
```

### 3. Déploiement
*   **Pour une mise à jour immédiate :** Modifiez le fichier sur le dépôt GitHub lié à `CONTRIBUTORS_REMOTE_URL` (généralement dans le dossier `frontend/assets/` de la branche `main`).
*   **Pour une version stable :** Mettez également à jour le fichier dans le dossier `frontend/assets/` du code source pour la prochaine release.

## ⚠️ Notes Techniques
*   **Cache :** Le chargement distant utilise `cache: 'no-cache'` pour s'assurer que les nouveaux donateurs apparaissent immédiatement.
*   **Sécurité (CSP) :** L'application autorise désormais tous les liens `https://*` pour les images (PFPs).
*   **Rétrocompatibilité :** Si vous utilisez un point (ex: `contributor.role.dev`) dans le champ `role`, l'app tentera quand même de le traduire via les fichiers `fr.json`/`en.json`. Si vous voulez du texte fixe, évitez les points ou assurez-vous que ce n'est pas une clé existante.