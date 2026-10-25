# EasyColoc

EasyColoc est une application web de gestion de colocations. Elle permet de créer un espace partagé, d'y inviter des membres et d'enregistrer des dépenses communes avec une répartition à parts égales et un solde par membre.

> Ce README décrit l'implémentation présente dans le dépôt. Certaines fonctions restent incomplètes ; elles sont signalées dans les sections « Limites connues » et « Sécurité et fiabilité ».

## Fonctionnalités présentes

- Inscription, connexion, déconnexion, vérification d'adresse e-mail, réinitialisation et modification du mot de passe (Laravel Breeze).
- Modification du profil et suppression du compte.
- Création et consultation des colocations auxquelles l'utilisateur appartient.
- Création d'une colocation avec son créateur comme propriétaire et membre initial.
- Invitation par e-mail, acceptation ou refus, et expiration des invitations.
- Enregistrement d'une dépense par un membre actif et mise à jour des soldes des membres actifs selon un partage égal.
- Départ d'un membre avec redistribution de son solde entre les membres restants ; le score de réputation évolue également lors du départ.
- Désactivation d'une colocation par son propriétaire.

La création d'une colocation est refusée à un utilisateur déjà membre d'une colocation active. Cette règle est vérifiée dans le contrôleur lors de l'affichage du formulaire et de la soumission.

## Technologies

| Domaine | Technologie |
| --- | --- |
| Backend | PHP `^8.2`, Laravel `^12.0` |
| Rendu web | Blade, composants Laravel Breeze |
| Base de données par défaut | SQLite |
| Frontend | Vite, Tailwind CSS, Alpine.js, Axios |
| Tests | Pest `^3.8`, PHPUnit |
| Gestion des dépendances PHP | Composer |
| Gestion des dépendances JavaScript | npm |

## Prérequis

- PHP 8.2 ou supérieur, avec les extensions requises par Laravel et le pilote PDO de la base choisie.
- Composer.
- Node.js et npm compatibles avec Vite 7.
- SQLite pour la configuration par défaut, ou une base prise en charge par Laravel correctement configurée.

## Installation locale

Depuis la racine du dépôt, dans PowerShell :

```powershell
composer install
if (!(Test-Path .env)) { Copy-Item .env.example .env }
php -r "file_exists('database/database.sqlite') || touch('database/database.sqlite');"
php artisan key:generate
php artisan migrate
npm install
npm run build
```

La création du fichier SQLite est nécessaire pour le chemin par défaut configuré par Laravel. Pour une autre base de données, renseignez les variables `DB_*` de `.env` à la place.

La commande de développement configurée dans Composer lance le serveur web, le worker de file d'attente, les journaux et Vite :

```powershell
composer run dev
```

L'application est servie par défaut par `php artisan serve` à l'adresse `http://localhost:8000`. Pour que le planificateur traite les invitations expirées, lancez aussi, dans un autre terminal :

```powershell
php artisan schedule:work
```

### E-mails

Le fichier `.env.example` configure `MAIL_MAILER=log` : les e-mails sont journalisés par défaut, ils ne sont pas transmis à une boîte réelle. Pour envoyer des invitations, configurez un transport de courrier dans `.env` (par exemple `MAIL_MAILER=smtp` et les paramètres SMTP correspondants). Ne publiez jamais les identifiants de courrier ni le fichier `.env`.

## Configuration

La configuration locale est chargée depuis `.env`, créé à partir de `.env.example`. Les paramètres utiles comprennent :

| Variable | Usage |
| --- | --- |
| `APP_NAME`, `APP_ENV`, `APP_DEBUG`, `APP_URL` | Nom, environnement, débogage et URL de l'application |
| `DB_CONNECTION`, `DB_DATABASE` et variables `DB_*` | Connexion à la base de données |
| `MAIL_MAILER` et variables `MAIL_*` | Transport et expéditeur des e-mails |
| `SESSION_DRIVER`, `CACHE_STORE`, `QUEUE_CONNECTION` | Stockage des sessions, du cache et des tâches |

Conservez `APP_DEBUG=false` dans un environnement accessible au public et utilisez une clé d'application générée localement. Les valeurs d'exemple ne doivent pas être traitées comme une configuration de production.

## Architecture et organisation

L'application suit l'organisation MVC de Laravel :

- `routes/web.php` définit les pages et actions métier accessibles sur le web ; `routes/auth.php` contient les routes d'authentification.
- `app/Http/Controllers/` orchestre les requêtes pour les colocations, dépenses, adhésions, invitations, profils et comptes.
- `app/Models/` contient les modèles Eloquent `User`, `Colocation`, `MemberShip`, `Invitation` et `Expence`.
- `app/Http/Requests/` porte notamment la validation de la mise à jour du profil.
- `resources/views/` contient les pages Blade, composants d'interface et e-mail d'invitation.
- `database/migrations/`, `database/factories/` et `database/seeders/` définissent le schéma, les fabriques et les données initiales.
- `app/Console/Commands/ExpireInvitations.php` marque comme expirées les invitations en attente dont la date limite est passée.
- `routes/console.php` planifie cette commande toutes les deux minutes.
- `resources/js/` et `resources/css/` sont les entrées compilées par Vite.
- `public/` contient le point d'entrée HTTP et les fichiers publics.
- `tests/Feature/` et `tests/Unit/` contiennent les tests automatisés.

### Modèle de données

| Entité | Rôle |
| --- | --- |
| `users` | Comptes, rôle, réputation et indicateur de bannissement |
| `colocations` | Nom, statut (`active` / `inactive`) et propriétaire |
| `member_ships` | Lien entre utilisateur et colocation, rôle, dates d'adhésion/départ et solde |
| `invitations` | Adresse invitée, jeton, statut, expéditeur et date d'expiration |
| `expences` | Dépense, payeur, montant et date associés à une colocation |

Le nom `expences` est celui utilisé dans le code et dans le schéma. Les migrations créent aussi les tables Laravel pour les sessions, le cache, les tâches et la réinitialisation des mots de passe.

### Parcours métier

1. Un utilisateur connecté crée une colocation ; il en devient propriétaire et premier membre.
2. Le propriétaire peut envoyer une invitation par e-mail. L'invitation reçoit un jeton et une échéance de deux minutes.
3. Un utilisateur connecté disposant du lien peut consulter l'invitation et l'accepter ou la refuser. L'acceptation est refusée si le compte appartient déjà à une colocation active.
4. Un membre actif enregistre une dépense. Le montant est partagé entre les membres actifs ; le solde du payeur augmente de sa part avancée et celui des autres membres diminue de leur part.
5. Lorsqu'un membre quitte une colocation, son solde est redistribué entre les membres restants et son adhésion est clôturée.
6. Le propriétaire peut désactiver sa colocation ; les adhésions actives sont alors marquées comme terminées.

## Routes principales

Les routes métier ci-dessous sont protégées par le middleware `auth`, sauf indication contraire.

| Méthode | Chemin | Fonction |
| --- | --- | --- |
| `GET` | `/` | Page d'accueil |
| `GET` | `/dashboard` | Tableau de bord (authentification et adresse vérifiée requises) |
| `GET` | `/colocations` | Liste des colocations visibles par le compte |
| `GET` | `/colocations/create` | Formulaire de création |
| `POST` | `/colocations` | Création d'une colocation |
| `GET` | `/colocations/{colocation}` | Détail d'une colocation |
| `PUT` / `PATCH` | `/colocations/{colocation}` | Mise à jour du nom (contrôle propriétaire présent) |
| `DELETE` | `/colocations/{colocation}` | Désactivation (contrôle propriétaire présent) |
| `POST` | `/colocations/{colocation}/invitations` | Envoi d'une invitation |
| `GET` | `/invitations/{token}` | Consultation d'une invitation |
| `POST` | `/invitations/{token}/accept` | Acceptation d'une invitation |
| `POST` | `/invitations/{token}/refuse` | Refus d'une invitation |
| `POST` | `/colocations/{colocation}/expences` | Enregistrement d'une dépense |
| `POST` | `/colocations/{colocation}/quit` | Départ du membre connecté |
| `GET`, `PATCH`, `DELETE` | `/profile` | Consultation, mise à jour et suppression du profil |

Laravel Breeze ajoute les routes d'inscription, connexion, déconnexion, vérification d'adresse e-mail et réinitialisation de mot de passe. Les routes d'édition et de suppression générées pour certaines ressources ne signifient pas que ces opérations métier sont toutes implémentées.

## Tests et vérifications

Commandes prévues par le projet :

```powershell
composer test
php artisan test
npm run build
```

`composer test` efface le cache de configuration avant d'exécuter les tests. La configuration de test utilise SQLite en mémoire, une file d'attente synchrone et un transport e-mail en mémoire.

La suite existante comprend les tests d'authentification, d'inscription, de vérification d'adresse e-mail, de mot de passe et de profil, ainsi que des tests d'exemple. Elle ne couvre pas encore les principaux parcours de colocation, invitation, dépense et calcul des soldes.

## Limites connues et priorités d'amélioration

Les constats suivants proviennent de la lecture du code ; ils ne constituent pas un audit de sécurité exhaustif ni une analyse des dépendances contre une base de CVE.

### Priorité élevée

- **Contrôle d'accès aux détails d'une colocation :** la route de détail exige une connexion, mais `ColocationController::show` ne vérifie pas que le compte est propriétaire ou membre. Les détails chargés incluent notamment les membres, invitations et dépenses. Ajouter une autorisation fondée sur la colocation avant d'exposer ces données.

### Priorité moyenne

- **Destinataire des invitations :** l'acceptation et le refus exigent une session, mais ne comparent pas l'adresse du compte connecté à l'adresse invitée. Le refus ne vérifie pas non plus l'échéance. Vérifier le destinataire et l'état/échéance avant toute transition.
- **Couverture métier :** aucun test dédié n'a été trouvé pour la création d'une colocation, l'accès aux données d'une colocation, les invitations, les dépenses, la redistribution ou les montants arrondis. Ajouter des tests de fonctionnalités et de cas limites avant une mise en production.
- **Calcul monétaire :** les soldes sont persistés en décimal mais le contrôleur calcule et manipule les répartitions avec des nombres flottants. Une stratégie en unités monétaires mineures et des règles explicites d'arrondi aideraient à éviter les écarts.
- **Route du tableau de bord déclarée deux fois :** `/dashboard` est enregistré deux fois dans `routes/web.php`. Garder une définition unique afin d'éviter les divergences entre les données préparées par la route et celles calculées dans la vue.

### Priorité faible / fonctionnelle

- La durée d'expiration est codée à deux minutes dans le contrôleur. Vérifier si cette durée convient à l'usage prévu.
- Le contrôleur des adhésions ne fournit pas d'implémentation pour la route de liste des membres.
- La mise à jour et la suppression des dépenses ne sont pas implémentées ; la fonctionnalité exposée est leur création et la mise à jour des soldes.
- Le mécanisme de réputation existe au départ d'un membre, mais aucun parcours d'administration ou de gestion des comptes bannis n'a été vérifié.

## Déploiement et contribution

Aucun pipeline CI/CD, conteneur ou procédure de déploiement spécifique au projet n'a été identifié dans les fichiers examinés. Avant tout déploiement, définir une configuration adaptée à l'environnement, désactiver le débogage public, configurer une base persistante et un transport de courrier, exécuter les migrations, construire les ressources (`npm run build`) et faire fonctionner le planificateur Laravel.

Aucun fichier `LICENSE` dédié au projet n'a été repéré. La licence mentionnée dans `composer.json` ne suffit pas à confirmer les droits de redistribution de l'application ; clarifiez ce point avant publication. Pour contribuer, ouvrez une issue décrivant le changement, puis une pull request accompagnée de tests couvrant le comportement ajouté.
