# PhoneTech Accessories — Site vitrine WordPress

Activité pratique du module **Développement Full Stack et Microservices**.
Objectif : découvrir le fonctionnement d'un CMS monolithique en réalisant un site vitrine d'accessoires téléphoniques avec WordPress.

**Réalisé par :** _Nom 1_ et _Nom 2_ (binôme)

---

## 1. Contenu du dépôt

| Fichier | Rôle |
|---|---|
| `docker-compose.yml` | Définition de l'environnement (WordPress + MariaDB) |
| `phonetech_db.sql` | Export de la base de données |
| `phonetech_files.zip` | Export des fichiers (`wp-content` : thèmes, extensions, images) |
| `phonetech_produits.csv` | Fichier d'import des 13 produits WooCommerce |
| `README.md` | Ce document |

---

## 2. Étapes de réalisation

### 2.1 Mise en place de l'environnement (Docker)

Le site tourne dans deux conteneurs Docker :

- `db` : MariaDB 11 (base `wordpress`)
- `wordpress` : image officielle WordPress, exposée sur le port **8080**

Le dossier `wp_content` est monté depuis l'hôte afin de pouvoir exporter facilement les fichiers du site.

```powershell
mkdir phonetech
cd phonetech
# création de docker-compose.yml
docker compose up -d
docker compose ps
```

Le site est alors accessible sur `http://localhost:8080`.

Un incident a été rencontré : l'activation de WooCommerce provoquait une erreur fatale, due à la limite de mémoire PHP. Elle a été résolue en ajoutant `WP_MEMORY_LIMIT` à 512M via la variable `WORDPRESS_CONFIG_EXTRA` dans `docker-compose.yml`.

### 2.2 Installation de WordPress

Installation via l'assistant web : titre du site **PhoneTech Accessories**, création d'un compte administrateur, langue française.

### 2.3 Extensions installées

| Extension | Usage |
|---|---|
| **Elementor** | Création et personnalisation de la page d'accueil |
| **WooCommerce** | Gestion du catalogue de produits |
| **Contact Form 7** | Formulaire de contact |
| **Yoast SEO** | Référencement naturel (expression clé, méta description) |

### 2.4 Thème

Le thème par défaut (Twenty Twenty-Five, thème à blocs) s'est révélé peu compatible avec les modèles de page d'Elementor (en-tête et menu absents). Le thème **Hello Elementor**, conçu pour Elementor, a donc été installé et activé. Il permet aussi de gérer le menu via **Apparence > Menus**.

### 2.5 Catalogue (WooCommerce)

- Configuration de la devise (dinar tunisien, TND) et passage de la boutique en ligne.
- Création de 5 catégories : Coques, Chargeurs, Écouteurs, Power banks, Supports.
- Import de **13 produits** via **Produits > Importer** à partir de `phonetech_produits.csv` (nom, description courte et longue, prix, catégorie, UGS).
- 4 produits marqués « mis en avant » : coque silicone souple, chargeur rapide 20W USB-C, écouteurs Bluetooth sans fil, power bank 10 000 mAh.

### 2.6 Pages du site

| Page | Contenu | Outil |
|---|---|---|
| **Accueil** | Bandeau de présentation, bouton vers le catalogue, produits phares (shortcode WooCommerce `[products visibility="featured" limit="4" columns="4"]`) | Elementor |
| **Catalogue** | Page « Boutique » générée par WooCommerce, avec les 13 produits | WooCommerce |
| **À propos** | Mission, présentation de l'entreprise, engagements | Éditeur WordPress |
| **Blog** | Liste des articles (page des articles définie dans **Réglages > Lecture**) | WordPress |
| **Contact** | Coordonnées et formulaire Contact Form 7 | Éditeur WordPress + Contact Form 7 |

Le menu principal regroupe : Accueil, Blog, À propos, Boutique, Contact.

### 2.7 Articles du blog

Trois articles ont été publiés :

1. Les tendances des accessoires mobiles en 2026
2. Conseils d'entretien pour votre smartphone
3. Les nouveautés technologiques à suivre

### 2.8 Référencement (Yoast SEO)

Pour chaque page principale (Accueil, À propos, Contact) et chaque article, une **expression clé principale** et une **méta description** ont été renseignées. Les permaliens ont été réglés sur « Titre de la publication » (`/%postname%/`).

### 2.9 Limites connues

- Le formulaire de contact s'affiche correctement, mais l'envoi d'e-mails ne fonctionne pas en environnement local : le conteneur ne dispose pas de serveur de messagerie. En production, il faudrait configurer un service SMTP (par exemple avec une extension dédiée).
- Les coordonnées et prix du site sont fictifs.

---

## 3. Export et restauration

### 3.1 Export

```powershell
# Base de données
docker exec phonetech-db-1 mariadb-dump -u wp -pwp_pass --result-file=/tmp/phonetech_db.sql wordpress
docker cp phonetech-db-1:/tmp/phonetech_db.sql .\phonetech_db.sql

# Fichiers
Compress-Archive -Path .\wp_content -DestinationPath .\phonetech_files.zip -Force
```

### 3.2 Restauration sur une autre machine

1. Placer `docker-compose.yml`, `phonetech_db.sql` et `phonetech_files.zip` dans un même dossier.
2. Extraire les fichiers du site :

```powershell
Expand-Archive -Path .\phonetech_files.zip -DestinationPath .
```

3. Démarrer les conteneurs :

```powershell
docker compose up -d
```

4. Importer la base de données (attendre quelques secondes que MariaDB soit prête) :

```powershell
docker cp .\phonetech_db.sql phonetech-db-1:/tmp/phonetech_db.sql
docker exec phonetech-db-1 sh -c "mariadb -u wp -pwp_pass wordpress < /tmp/phonetech_db.sql"
```

5. Ouvrir `http://localhost:8080`.

Le site est configuré avec l'adresse `http://localhost:8080`. Si on le restaure sous une autre URL, il faut mettre à jour les adresses du site (options `siteurl` et `home`, par exemple avec WP-CLI et `wp search-replace`).

---

## 4. Architecture de WordPress

WordPress est une application **PHP** qui s'exécute derrière un serveur web et s'appuie sur une base de données **MySQL/MariaDB**.

### 4.1 Vue d'ensemble

```
Navigateur
    │  HTTP
    ▼
Serveur web (Apache) + PHP        ← conteneur « wordpress »
    │
    ├── Cœur de WordPress (wp-admin, wp-includes)
    ├── wp-content/
    │     ├── themes/    (apparence : Hello Elementor)
    │     ├── plugins/   (Elementor, WooCommerce, Contact Form 7, Yoast SEO)
    │     └── uploads/   (médias téléversés)
    │
    ▼  SQL
Base de données MariaDB            ← conteneur « db »
```

### 4.2 Les composants

- **Cœur** : gère les requêtes, l'authentification, les articles, les pages, les utilisateurs et l'interface d'administration (`wp-admin`).
- **Thèmes** : définissent l'apparence et les modèles de page (hiérarchie des modèles).
- **Extensions (plugins)** : ajoutent des fonctionnalités sans modifier le cœur. Elles s'appuient sur le système de **hooks** (actions et filtres) fourni par WordPress.
- **Base de données** : stocke le contenu et la configuration, notamment :
  - `wp_posts` : articles, pages, produits, médias
  - `wp_postmeta` : métadonnées (prix, UGS, image mise en avant)
  - `wp_options` : réglages du site (URL, thème actif, extensions actives)
  - `wp_users` : comptes utilisateurs
  - `wp_terms` / `wp_term_taxonomy` : catégories et étiquettes
- **Dossier `wp-content/uploads`** : stocke les fichiers téléversés (images des produits et des articles).

### 4.3 Cycle d'une requête

1. Le visiteur demande une URL (par exemple `/boutique/`).
2. Le serveur web transmet la requête à `index.php`.
3. WordPress charge le cœur, les extensions actives et le thème.
4. Il interroge la base de données selon l'URL, choisit le modèle du thème adapté et génère la page HTML.
5. La page est renvoyée au navigateur.

---

## 5. Architecture monolithique et architecture microservices

### 5.1 Définitions

- **Monolithique** : l'application est un seul bloc, déployé comme une unité. L'interface, la logique métier et l'accès aux données sont dans le même code et le même processus.
- **Microservices** : l'application est découpée en petits services indépendants (catalogue, commandes, paiement, utilisateurs...), chacun avec sa propre base de données et son propre cycle de déploiement. Ils communiquent par API (REST, messages).

### 5.2 WordPress, un exemple de monolithe

Dans ce projet, le front-end, le back-office, la gestion des produits (WooCommerce), le formulaire de contact et le SEO tournent dans **la même application PHP**, avec **une seule base de données**. Les extensions s'ajoutent au même bloc : activer WooCommerce l'a d'ailleurs rendu plus lourd (erreur de mémoire rencontrée à l'activation).

### 5.3 Comparaison

| Critère | Monolithique (WordPress) | Microservices |
|---|---|---|
| Structure | Un seul bloc de code | Plusieurs services indépendants |
| Déploiement | Simple : un seul ensemble à déployer | Complexe : orchestration, plusieurs services |
| Mise en route | Rapide, adaptée à un petit projet comme un site vitrine | Plus longue, demande une infrastructure |
| Scalabilité | Globale : on duplique toute l'application | Fine : on ne duplique que le service sollicité |
| Base de données | Unique et partagée | Une base par service |
| Couplage | Fort : une extension défaillante peut bloquer tout le site | Faible : une panne reste isolée |
| Technologies | Une seule pile (PHP + MySQL) | Chaque service peut avoir sa propre technologie |
| Maintenance | Facile au début, devient difficile quand le code grossit | Plus lourde à l'échelle du système, mais chaque service reste simple |
| Équipes | Une équipe, un seul dépôt | Équipes autonomes par service |
| Tests | Plus simples (un seul ensemble) | Plus complexes (tests entre services) |
| Résilience | Point de défaillance unique | Meilleure tolérance aux pannes |
| Performances | Appels internes rapides | Latence liée aux appels réseau entre services |

### 5.4 Cas concret : boutique PhoneTech

- **En monolithe (ce projet)** : un seul WordPress gère le catalogue, les pages, le blog et le contact. C'est simple à installer et à exporter, suffisant pour un site vitrine.
- **En microservices** : on aurait un service Catalogue, un service Commandes, un service Paiement, un service Contenu (blog) et un service Notifications, avec une passerelle API devant. C'est plus adapté à une boutique à fort trafic ou à une équipe nombreuse, mais beaucoup plus complexe à mettre en place.

### 5.5 Une solution intermédiaire : WordPress « headless »

WordPress peut aussi servir uniquement de **back-office de contenu**, exposé par son **API REST**, tandis qu'une application séparée (React, Vue...) gère l'affichage. On sépare ainsi le front du back, ce qui rapproche l'architecture d'une approche découplée, sans aller jusqu'aux microservices.

### 5.6 Conclusion

Pour un site vitrine de petite taille, l'architecture monolithique de WordPress est un bon choix : rapide à mettre en place, facile à exporter et à maintenir. Les microservices deviennent pertinents lorsque l'application grandit, que la charge augmente ou que plusieurs équipes doivent travailler en parallèle.
