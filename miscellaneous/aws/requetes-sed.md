# Requêtes SQL (examen SED)

Toutes les requêtes de la séance, dans l'ordre. Chaque exercice contient : l'énoncé, la requête, puis ce qu'elle fait.

Légende des schémas : ♦ = clé primaire, ● = attribut obligatoire (`NOT NULL`), ○ = attribut facultatif.

---

## Partie 1 : Création de tables (DDL)

### 1. Création d'une table simple

> **Énoncé** : Créez une table nommée «test» contenant un simple champ «id» de type INT.

```sql
CREATE TABLE test (
    id INT
);
```

* On crée la table `test` avec `CREATE TABLE`.
* `id INT` : une seule colonne de type entier.
* Pas de `NOT NULL` : une colonne accepte NULL par défaut (le `DESC` affiche `Null = YES`).
* `INT` et `INTEGER` sont équivalents.

### 2. Création d'une table simple avec champs obligatoires

> **Énoncé** : Créez la table «médecin» dont le schéma est le suivant :
>
> | | Champ | Type |
> |---|---|---|
> | ● | id | INT |
> | ● | nom | VARCHAR(255) |
> | ○ | no_téléphone | CHAR(10) |

```sql
CREATE TABLE médecin (
    id INT NOT NULL,
    nom VARCHAR(255) NOT NULL,
    no_téléphone CHAR(10)
);
```

* `id` et `nom` sont obligatoires (●) : on ajoute `NOT NULL`.
* `no_téléphone` est facultatif (○) : pas de `NOT NULL`.
* `CHAR(10)` parce que le numéro a toujours la même longueur (10 chiffres).
* `VARCHAR(255)` parce que le nom a une longueur variable.

### 3. Création d'une table simple avec clé primaire

> **Énoncé** : Créez la table «utilisateur» selon le schéma suivant :
>
> | | Champ | Type |
> |---|---|---|
> | ♦ ● | nom_utilisateur | VARCHAR(255) |
> | ● | courriel | VARCHAR(255) |
> | ● | mot_de_passe | VARCHAR(255) |
> | ● | date_inscription | DATE |

```sql
CREATE TABLE utilisateur (
    nom_utilisateur VARCHAR(255) PRIMARY KEY,
    courriel VARCHAR(255) NOT NULL,
    mot_de_passe VARCHAR(255) NOT NULL,
    date_inscription DATE NOT NULL
);
```

* Ordre d'une colonne : **nom, type, contraintes** (`PRIMARY KEY` vient après le type).
* `PRIMARY KEY` implique déjà `NOT NULL` : pas besoin de l'écrire.
* Les trois autres colonnes sont obligatoires : `NOT NULL`.
* `DATE` pour la date d'inscription.

### 4. Création d'une table avec clé primaire artificielle auto-incrémentale

> **Énoncé** : Créez la table «client» selon le schéma suivant :
>
> | | Champ | Type |
> |---|---|---|
> | ♦ ● | id | INT |
> | ● | nom | VARCHAR(255) |
> | ● | prénom | VARCHAR(255) |
> | ○ | compagnie | VARCHAR(255) |
> | ○ | no_téléphone | CHAR(10) |

```sql
CREATE TABLE client (
    id INT PRIMARY KEY AUTO_INCREMENT,
    nom VARCHAR(255) NOT NULL,
    prénom VARCHAR(255) NOT NULL,
    compagnie VARCHAR(255),
    no_téléphone CHAR(10)
);
```

* `id` est une clé **artificielle** : identifiant sans sens métier, généré par le système.
* `AUTO_INCREMENT` : MySQL attribue 1, 2, 3... automatiquement (colonne numérique seulement).
* `nom` et `prénom` sont obligatoires (●).
* `compagnie` et `no_téléphone` sont facultatifs (○).

### 5. Création d'une clé étrangère

> **Énoncé** (résumé) : Créez les tables `race` et `chien` selon le schéma. Chaque chien a une race (obligatoire); une race peut avoir plusieurs chiens.
>
> | race | | | chien | | |
> |---|---|---|---|---|---|
> | ♦ | id | INT | ♦ | id | INT |
> | ● | nom | VARCHAR(255) | ● | nom | VARCHAR(255) |
> | | | | ● | race_id | INT |

```sql
CREATE TABLE race (
    id INT PRIMARY KEY,
    nom VARCHAR(255) NOT NULL
);

CREATE TABLE chien (
    id INT PRIMARY KEY,
    nom VARCHAR(255) NOT NULL,
    race_id INT NOT NULL,
    FOREIGN KEY (race_id) REFERENCES race(id)
);
```

* On crée d'abord la table **parente** (`race`), puis la table **enfant** (`chien`).
* `race_id` est la clé étrangère : elle va du côté « plusieurs » (un chien a une race, une race a plusieurs chiens).
* `NOT NULL` sur `race_id` : la relation est obligatoire (barre `┼` dans le schéma).
* `FOREIGN KEY (race_id) REFERENCES race(id)` : chaque `race_id` doit exister dans `race`.
* Les types doivent être identiques des deux côtés (`INT` et `INT`).
* Dans le `DESC`, `race_id` affiche `Key = MUL`.

### 6. Deux clés étrangères, dont une facultative (préparation de l'exercice d'insertion)

> **Énoncé** : Créer les tables du schéma suivant, qui sert à l'exercice « Insertion dans des tables liées » :
>
> | chien | | | maître | | | race | | |
> |---|---|---|---|---|---|---|---|---|
> | ♦ | id | INT | ♦ | id | INT | ♦ | id | INT |
> | ● | nom | VARCHAR(255) | ● | nom | VARCHAR(255) | ● | nom | VARCHAR(255) |
> | ● | race_id | INT | ● | prénom | VARCHAR(255) | | | |
> | ○ | maître_id | INT | | | | | | |
>
> Relations : un chien a exactement une race (`┼`); un chien a zéro ou un maître (`o|`).

```sql
CREATE TABLE race (
    id INT PRIMARY KEY,
    nom VARCHAR(255) NOT NULL
);

CREATE TABLE maître (
    id INT PRIMARY KEY,
    nom VARCHAR(255) NOT NULL,
    prénom VARCHAR(255) NOT NULL
);

CREATE TABLE chien (
    id INT PRIMARY KEY,
    nom VARCHAR(255) NOT NULL,
    race_id INT NOT NULL,
    maître_id INT,
    FOREIGN KEY (race_id) REFERENCES race(id),
    FOREIGN KEY (maître_id) REFERENCES maître(id)
);
```

* On crée les deux tables parentes (`race`, `maître`) avant `chien`.
* `race_id INT NOT NULL` : un chien a **exactement une** race.
* `maître_id INT` sans `NOT NULL` : un chien a **zéro ou un** maître (symbole `o|`).
* Deux clés étrangères = deux lignes `FOREIGN KEY ... REFERENCES ...`.
* Le nom de table doit être écrit pareil partout (`maître` avec accent, dans la FK aussi).

### 7. Relation plusieurs à plusieurs (table de jonction)

> **Énoncé** : Faites le script d'implantation des tables dont voici le schéma logique représentant les relations entre les entités d'une banque. Nommez vos clés étrangères avec le format «table_champ».
>
> | compte | | | titulaire | | | type | | |
> |---|---|---|---|---|---|---|---|---|
> | ♦ | numéro | CHAR(7) | ♦ | nas | CHAR(9) | ♦ | id | INT |
> | ● | solde | INT | ● | nom | VARCHAR(255) | ● | description | VARCHAR(255) |
> | | | | ● | prénom | VARCHAR(255) | | | |
>
> Relations : **compte ↔ titulaire** est plusieurs-à-plusieurs; **compte → type** est plusieurs-à-un (un compte a exactement un type).

```sql
CREATE TABLE type (
    id INT PRIMARY KEY,
    description VARCHAR(255) NOT NULL
);

CREATE TABLE compte (
    numéro CHAR(7) PRIMARY KEY,
    solde INT NOT NULL,
    type_id INT NOT NULL,
    FOREIGN KEY (type_id) REFERENCES type(id)
);

CREATE TABLE titulaire (
    nas CHAR(9) PRIMARY KEY,
    nom VARCHAR(255) NOT NULL,
    prénom VARCHAR(255) NOT NULL
);

CREATE TABLE compte_titulaire (
    compte_numéro CHAR(7) NOT NULL,
    titulaire_nas CHAR(9) NOT NULL,
    PRIMARY KEY (compte_numéro, titulaire_nas),
    FOREIGN KEY (compte_numéro) REFERENCES compte(numéro),
    FOREIGN KEY (titulaire_nas) REFERENCES titulaire(nas)
);
```

* **compte ↔ type** est 1:N : la clé étrangère `type_id` va dans `compte` (côté « plusieurs »), en `NOT NULL`.
* **compte ↔ titulaire** est N:N (compte conjoint, titulaire avec plusieurs comptes) : impossible directement, on crée une **table de jonction** `compte_titulaire`.
* La table de jonction contient une colonne par clé primaire des deux tables reliées.
* Deux `FOREIGN KEY` : une vers chaque table parente.
* `PRIMARY KEY (compte_numéro, titulaire_nas)` : clé primaire **composée**. C'est la **paire** qui doit être unique (empêche d'enregistrer deux fois le même lien), pas chaque colonne seule.
* On crée les trois tables parentes avant la table de jonction.
* Le `é` de `numéro` doit être identique partout (colonne, référence, clé composée).

---

## Partie 2 : Insertion de données

### 8. Insertion dans des tables liées

> **Énoncé** : Dans le schéma chien / maître / race, insérez les données suivantes :
>
> **Races** : 1. Chihuahua, 2. Doberman, 3. Cocker, 4. Caniche
>
> **Maîtres** : 1. Madelgarde Thibault, 2. Xaviel Magnan, 3. Cérélise Ranger
>
> **Chiens** :
> 1. Givré, un cocker appartenant à Xaviel
> 2. Soupotomates, un doberman appartenant à Madelgarde
> 3. Wouf, un caniche appartenant à Cérélise
> 4. Capitaine, un chihuahua sans propriétaire

```sql
INSERT INTO race (id, nom) VALUES
    (1, 'Chihuahua'),
    (2, 'Doberman'),
    (3, 'Cocker'),
    (4, 'Caniche');

INSERT INTO maître (id, nom, prénom) VALUES
    (1, 'Thibault', 'Madelgarde'),
    (2, 'Magnan', 'Xaviel'),
    (3, 'Ranger', 'Cérélise');

INSERT INTO chien (id, nom, race_id, maître_id) VALUES
    (1, 'Givré', 3, 2),
    (2, 'Soupotomates', 2, 1),
    (3, 'Wouf', 4, 3),
    (4, 'Capitaine', 1, NULL);
```

* On insère d'abord les tables **parentes** (`race`, `maître`), puis l'enfant (`chien`).
* On fournit les `id` nous-mêmes, car il n'y a pas d'`AUTO_INCREMENT`.
* Une paire de parenthèses par ligne à insérer.
* Le texte est entre apostrophes (`'Chihuahua'`).
* Dans `chien`, `race_id` et `maître_id` reprennent les `id` des tables parentes (Givré est un cocker, donc `race_id = 3`).
* Capitaine n'a pas de propriétaire : `maître_id = NULL` (permis, colonne facultative).

---

## Partie 3 : Requêtes SELECT

Schéma «entreprises» : `région`, `entreprise`, `scian`, `employé`, `emploi`.

### 9. Jointure simple 1 (entreprise et domaine SCIAN)

> **Énoncé** : La table scian contient la description des codes du «Système de classification des industries de l'Amérique du Nord». Chaque entreprise au Canada possède un de ces codes qui indique dans quel champ elle œuvre.
>
> Donnez la requête qui permet d'obtenir le nom et la description du domaine d'action de l'entreprise dont le numéro du Registre des Entreprises du Québec est 6079850087.

```sql
SELECT entreprise.nom, scian.description
FROM entreprise
JOIN scian ON entreprise.scian_code = scian.code
WHERE entreprise.req = 6079850087;
```

* Le SCIAN est un dictionnaire de codes d'industries : `scian.code` → `scian.description`.
* On sélectionne `nom` (table `entreprise`) et `description` (table `scian`).
* Les deux colonnes sont dans deux tables : il faut une **jointure**.
* `ON entreprise.scian_code = scian.code` : clé étrangère = clé primaire.
* `WHERE entreprise.req = 6079850087` : on garde l'entreprise demandée. Pas d'apostrophes, `req` est un `BIGINT`.

### 10. Jointure simple 2 (jointure, alias, tri et limite)

> **Énoncé** : Donnez la requête qui permet d'obtenir le nom des 10 premières entreprises répertoriées selon l'ordre alphabétique de nom, avec le nom de la région où elles sont situées.
>
> Les deux colonnes portaient la même entête «nom». Nommez-les plutôt «Nom d'entreprise» et «Région administrative».

```sql
SELECT entreprise.nom AS `Nom d'entreprise`,
       région.nom AS `Région administrative`
FROM entreprise
JOIN région ON entreprise.région_code = région.code
ORDER BY entreprise.nom
LIMIT 10;
```

* On sélectionne le nom de l'entreprise et celui de la région : deux colonnes `nom`, donc les **préfixes** sont obligatoires.
* `AS` renomme les colonnes dans le résultat (backticks à cause des espaces et de l'apostrophe).
* Jointure : `entreprise.région_code = région.code`.
* `ORDER BY entreprise.nom` : tri alphabétique **avant** de limiter.
* `LIMIT 10` : on garde les 10 premières lignes triées. Sans `ORDER BY`, ce serait 10 lignes au hasard.

### 11. Jointure sur la clé d'un employé (deux emplois)

> **Énoncé** : L'employé dont le NAS est 200332220 a deux emplois. Donnez la requête qui permet d'obtenir son prénom et son nom, avec la date de début de chacun de ses emplois. Nommez les colonnes «Prénom», «Nom» et «Date de début».

```sql
SELECT employé.prénom AS `Prénom`,
       employé.nom AS `Nom`,
       emploi.début AS `Date de début`
FROM employé
JOIN emploi ON emploi.employé_nas = employé.nas
WHERE employé.nas = '200332220';
```

* Prénom et nom sont dans `employé`, la date de début dans `emploi` : jointure.
* `emploi.employé_nas = employé.nas` : clé étrangère = clé primaire.
* `WHERE employé.nas = '200332220'` : apostrophes, car `nas` est un `CHAR(9)` (du texte).
* Le résultat a deux lignes (deux emplois) avec le même prénom et nom.
* Alias avec backticks, surtout pour `Date de début` (espaces).

### 12. Éléments sans correspondance (régions sans entreprise)

> **Énoncé** : Cette base de données semble loin d'être complète. En fait, plusieurs régions en sont absentes. Donnez la requête qui permet d'obtenir, en ordre alphabétique de nom, la liste des régions pour lesquelles on n'a aucune donnée d'entreprises.

```sql
SELECT région.nom
FROM région
LEFT JOIN entreprise ON entreprise.région_code = région.code
WHERE entreprise.req IS NULL
ORDER BY région.nom;
```

* On veut les régions **sans** entreprise : on doit garder **toutes** les régions, donc `LEFT JOIN` avec `région` à gauche.
* Pour une région sans entreprise, les colonnes de `entreprise` valent `NULL`.
* `WHERE entreprise.req IS NULL` ne garde que ces régions (`req`, clé primaire, n'est jamais `NULL` s'il y a une entreprise).
* On écrit `IS NULL`, jamais `= NULL`.
* `ORDER BY région.nom` : ordre alphabétique demandé.
* Un `INNER JOIN` ne marcherait pas : il ferait disparaître les régions sans entreprise.

Alternative avec sous-requête :

```sql
SELECT nom
FROM région
WHERE code NOT IN (SELECT région_code FROM entreprise WHERE région_code IS NOT NULL)
ORDER BY nom;
```

* La sous-requête liste les codes de région utilisés.
* `NOT IN` garde les régions dont le code n'y est pas.
* Le `IS NOT NULL` est essentiel : une seule valeur `NULL` dans la sous-requête fait que `NOT IN` ne retourne plus rien.

### 13. Sous-requête (embauches le jour de naissance d'un employé)

> **Énoncé** : Donnez la requête qui permet d'obtenir, en ordre croissant, les REQ de toutes les entreprises ayant embauché quelqu'un le jour de la naissance de l'employé dont le NAS est 324508400.

```sql
SELECT DISTINCT emploi.entreprise_req
FROM emploi
WHERE emploi.début = (
    SELECT naissance
    FROM employé
    WHERE nas = '324508400'
)
ORDER BY emploi.entreprise_req;
```

* Étape 1 (sous-requête) : trouver la date de naissance de l'employé `324508400`. Elle retourne une seule valeur, car `nas` est une clé primaire.
* Étape 2 : chercher dans `emploi` les emplois dont le `début` est égal à cette date.
* On affiche `entreprise_req` : pas besoin de joindre `entreprise`, `emploi` contient déjà le REQ.
* `DISTINCT` : une entreprise qui a embauché plusieurs personnes ce jour-là ne doit apparaître qu'une fois.
* `ORDER BY` : croissant par défaut.

### 14. Deux jointures en chaîne

> **Énoncé** : Donnez la requête qui permet d'obtenir le nom des deux entreprises qui embauchent l'employé «Eldéard Normandin» ainsi que sa date d'embauche, en nommant les colonnes «Employeur» et «Date d'embauche».

```sql
SELECT entreprise.nom AS `Employeur`,
       emploi.début AS `Date d'embauche`
FROM employé
JOIN emploi ON emploi.employé_nas = employé.nas
JOIN entreprise ON entreprise.req = emploi.entreprise_req
WHERE employé.prénom = 'Eldéard'
  AND employé.nom = 'Normandin';
```

* On part de `employé` (on connaît le nom de la personne), on passe par `emploi` (qui lie employé et entreprise), puis on arrive à `entreprise`.
* Jointure 1 : `emploi.employé_nas = employé.nas`.
* Jointure 2 : `emploi.entreprise_req = entreprise.req`.
* Deux colonnes `nom` différentes (employé et entreprise) : les préfixes sont obligatoires.
* `WHERE` avec `AND` : le prénom **et** le nom doivent correspondre.
* Le résultat a deux lignes, une par employeur.

---

## Partie 4 : Service de livraison de repas

Tables utilisées : `Utilisateur` et `Role_utilisateur` (lien : `Role_utilisateur.utilisateur_code = Utilisateur.code`).

### 15. Jointure avec filtre (clients seulement)

> **Énoncé** : Rédigez une requête permettant d'afficher les noms, prénoms, courriels et numéros de téléphone de tous les utilisateurs qui sont des clients du service de livraison.

```sql
SELECT DISTINCT Utilisateur.nom,
       Utilisateur.prénom,
       Utilisateur.courriel,
       Utilisateur.téléphone
FROM Utilisateur
JOIN Role_utilisateur ON Role_utilisateur.utilisateur_code = Utilisateur.code
WHERE Role_utilisateur.role = 'client';
```

* Les coordonnées sont dans `Utilisateur`, le rôle dans `Role_utilisateur` : jointure.
* `WHERE role = 'client'` : le type `SET(client, livreur, gérant)` n'accepte que ces trois valeurs.
* `DISTINCT` : un utilisateur peut avoir plusieurs lignes avec le même rôle (à cause de l'`horodatage`).
* Attention aux majuscules des noms de tables (`Utilisateur`, pas `utilisateur`).

### 16. LEFT JOIN et tri sur plusieurs colonnes

> **Énoncé** : Rédigez une requête permettant d'afficher les noms, prénoms, courriels, numéros de téléphone et rôles de tous les utilisateurs du service de livraison en les triant par rôle et code.

```sql
SELECT Utilisateur.nom,
       Utilisateur.prénom,
       Utilisateur.courriel,
       Utilisateur.téléphone,
       Role_utilisateur.role
FROM Utilisateur
LEFT JOIN Role_utilisateur ON Role_utilisateur.utilisateur_code = Utilisateur.code
ORDER BY Role_utilisateur.role, Utilisateur.code;
```

* « **Tous** les utilisateurs » : certains n'ont aucun rôle, donc `LEFT JOIN` (un `INNER JOIN` les ferait disparaître).
* Sans rôle, `role` vaut `NULL`.
* `ORDER BY role, code` : tri par rôle, puis, à rôle égal, par code.
* MySQL place les `NULL` **en premier** en ordre croissant : l'utilisateur sans rôle (Braso) sort en tête.
* Un `SET` se trie selon l'ordre de sa définition (client, livreur, gérant), pas alphabétiquement.
* Un utilisateur avec deux rôles apparaît sur deux lignes.

### 17. Compter avec GROUP BY

> **Énoncé** : Compter le nombre de rôles des utilisateurs d'un service de livraison de repas (jointure entre 2 tables, utilisation de fonction et d'alias).
>
> Rédigez une requête permettant d'afficher les noms, prénoms, courriels, numéros de téléphone et nombre de rôles de tous les utilisateurs du service de livraison. Le nombre de rôles doit être affiché dans une colonne « nombre de rôles » et la liste doit être triée par code d'utilisateurs.

```sql
SELECT Utilisateur.nom,
       Utilisateur.prénom,
       Utilisateur.courriel,
       Utilisateur.téléphone,
       COUNT(Role_utilisateur.role) AS `nombre de rôles`
FROM Utilisateur
LEFT JOIN Role_utilisateur ON Role_utilisateur.utilisateur_code = Utilisateur.code
GROUP BY Utilisateur.code
ORDER BY Utilisateur.code;
```

* On veut **une ligne par utilisateur** avec son nombre de rôles : `GROUP BY Utilisateur.code`.
* `COUNT(Role_utilisateur.role)` compte les rôles de chaque groupe.
* `LEFT JOIN` : tous les utilisateurs doivent apparaître, même sans rôle.
* `COUNT(colonne)` ignore les `NULL` : un utilisateur sans rôle obtient **0**. Avec `COUNT(*)`, il obtiendrait 1.
* Alias `nombre de rôles` entre backticks (espaces et accent).
* `ORDER BY Utilisateur.code` : tri demandé.
* Si les rôles peuvent se répéter, utiliser `COUNT(DISTINCT Role_utilisateur.role)`.

### 18. Détecter l'absence de correspondance (utilisateurs sans rôle)

> **Énoncé** : Afficher les utilisateurs d'un service de livraison (jointure entre 2 tables, détection d'un champ null). Rédigez une requête permettant d'afficher les noms, prénoms, courriels, numéros de téléphone de tous les utilisateurs du service de livraison qui n'ont pas de rôle.

```sql
SELECT Utilisateur.nom,
       Utilisateur.prénom,
       Utilisateur.courriel,
       Utilisateur.téléphone
FROM Utilisateur
LEFT JOIN Role_utilisateur ON Role_utilisateur.utilisateur_code = Utilisateur.code
WHERE Role_utilisateur.utilisateur_code IS NULL;
```

* `LEFT JOIN` garde tous les utilisateurs; ceux sans rôle ont des `NULL` du côté de `Role_utilisateur`.
* `WHERE Role_utilisateur.utilisateur_code IS NULL` ne garde que ces utilisateurs-là (cette colonne est `NOT NULL`, donc un `NULL` veut dire « aucune correspondance »).
* On ne sélectionne que les coordonnées, pas la colonne `role`.
* Résultat attendu : Braso Juan.

Alternative avec sous-requête :

```sql
SELECT nom, prénom, courriel, téléphone
FROM Utilisateur
WHERE code NOT IN (SELECT utilisateur_code FROM Role_utilisateur);
```

---

## Partie 5 : Examen formatif, Section 2 (modifier une base de données existante)

**Contexte commun à toute la section.** Le service de location de vélos en libre-service gagne en popularité. La flotte n'est plus suffisante, les usagers réclament des vélos à assistance électrique, et le service veut se conformer au standard General Bikeshare Feed Specification (GBFS). L'analyste du projet a établi les modifications au schéma, et il ne reste qu'à les implémenter.

**Attention : les données existantes doivent être conservées.** On ne supprime donc jamais une table pour la recréer : on modifie ce qui existe avec `ALTER TABLE` (la **structure**) et `UPDATE` (les **données**).

**Schéma final (après toutes les modifications)**

| Table | Colonnes |
|---|---|
| **Stations** | ♦ `id` INT, ● `nom` VARCHAR(255) UNIQUE, ○ `longitude` FLOAT, ○ `latitude` FLOAT, ● `nb_ancrages` INT, ... |
| **Velos** | ♦ `code` CHAR(6), ● `id_station` INT (FK), ● `date_pmes` TIMESTAMP, ● `est_defectueux` TINYINT(1) DEFAULT 0, ● `id_type` INT (FK) |
| **Types** | ♦ `id` INT, ● `categorie` ENUM('vélo', 'vélo-cargo'), ● `modele` VARCHAR(255), ● `propulsion` ENUM('humain', 'electrique'), ○ `autonomie` INT |
| **Locations** | ♦ `id` INT, ● `id_usager` INT (FK), ● `code_velo` CHAR(6) (FK), ● `id_station_depart` INT (FK), ○ `id_station_arrivee` INT (FK), ● `date_depart` TIMESTAMP, ○ `date_arrivee` TIMESTAMP |

**Les 6 vérifications à faire pour chaque colonne (CREATE ou ALTER)** : nom exact, type avec sa longueur, `NOT NULL` si « obligatoire », `PRIMARY KEY` / `AUTO_INCREMENT` si c'est l'id, `DEFAULT` si l'énoncé donne une valeur par défaut, `UNIQUE` si « unique ».

### 19. Ajouter une table (Section 2.1, 2 points)

> **Énoncé** : Il faut d'abord ajouter une nouvelle table `Types` qui permettra de gérer différents types et modèles de vélos :
>
> * `id` : identifiant entier auto-incrémenté (clé primaire)
> * `categorie` : une chaîne de caractères obligatoire de type ENUM avec les valeurs prédéfinies 'vélo' et 'vélo-cargo'
> * `modele` : une chaîne de caractères obligatoire de longueur 255 permettant de décrire le modèle
> * `propulsion` : une chaîne de caractères obligatoire de type ENUM avec les valeurs prédéfinies 'humain' et 'electrique'
> * `autonomie` : un entier représentant le nombre de kilomètres d'autonomie de la batterie pour les modèles à propulsion électrique

**Étapes sur papier**

* **Quelle commande ?** `CREATE TABLE Types` : c'est une **nouvelle** table, donc pas de `ALTER`. Les données existantes (Velos, Stations...) ne sont pas touchées.
* **Quelles colonnes, dans l'ordre de l'énoncé ?**

| Colonne | Type | Contraintes | Pourquoi |
|---|---|---|---|
| id | INT | clé primaire, AUTO_INCREMENT | « identifiant entier auto-incrémenté » |
| categorie | ENUM('vélo', 'vélo-cargo') | NOT NULL | « obligatoire », valeurs prédéfinies |
| modele | VARCHAR(255) | NOT NULL | « obligatoire, longueur 255 » |
| propulsion | ENUM('humain', 'electrique') | NOT NULL | « obligatoire », valeurs prédéfinies |
| autonomie | INT | aucune | seulement pour les modèles électriques, donc facultatif |

* **Y a-t-il une clé étrangère ?** Non. Le lien entre `Velos` et `Types` est une autre question (2.3).
* **À vérifier avant de lancer :** noms de colonnes identiques à l'énoncé (`modele`, sans accent), valeurs des ENUM exactes (accents et trait d'union : `'vélo'`, `'vélo-cargo'`, `'electrique'` sans accent), virgules entre les colonnes, parenthèses fermées, point-virgule final.

```sql
CREATE TABLE Types (
  id INT PRIMARY KEY AUTO_INCREMENT,
  categorie ENUM('vélo', 'vélo-cargo') NOT NULL,
  modele VARCHAR(255) NOT NULL,
  propulsion ENUM('humain', 'electrique') NOT NULL,
  autonomie INT
);
```

**Explication**

* `ENUM(...)` limite la colonne à une liste de valeurs permises, écrites entre apostrophes. Toute autre valeur est refusée.
* `id INT PRIMARY KEY AUTO_INCREMENT` : la base attribue elle-même 1, 2, 3... à chaque nouvelle ligne.
* `autonomie` n'a pas de `NOT NULL` : l'énoncé ne dit pas « obligatoire », car un vélo à propulsion humaine n'a pas de batterie.
* `'electrique'` s'écrit sans accent, exactement comme dans l'énoncé.

### 20. Ajouter un champ obligatoire et le remplir (Section 2.2, 2 points)

> **Énoncé** : Tous nos vélos actuels ont été achetés au même moment. Afin de documenter cette information, nous allons effectuer les opérations suivantes :
>
> * Ajouter un champ obligatoire `date_pmes` (première mise en service) de type TIMESTAMP à la table `Velos`.
> * Inscrire la date du 15 avril 2024 pour tous les vélos de la flotte actuelle.

**Étapes sur papier**

* **Quelle commande ?** La table `Velos` existe déjà et contient des vélos : on la modifie avec `ALTER TABLE ... ADD COLUMN`, puis on remplit avec `UPDATE`.
* **Le piège :** le champ doit être obligatoire (`NOT NULL`), mais les vélos existants n'ont encore aucune date. Un `NOT NULL` immédiat serait refusé, ou remplirait n'importe quoi.
* **Donc, en 3 étapes :**
  1. Ajouter la colonne **facultative** (`NULL`).
  2. Remplir toutes les lignes avec `UPDATE`.
  3. Rendre la colonne obligatoire avec `MODIFY`.
* **« Pour tous les vélos »** : `UPDATE` **sans** `WHERE`.

```sql
ALTER TABLE Velos ADD COLUMN date_pmes TIMESTAMP NULL;

UPDATE Velos SET date_pmes = '2024-04-15';

ALTER TABLE Velos MODIFY date_pmes TIMESTAMP NOT NULL;
```

**Explication**

Voici l'état de la table à chaque étape :

| Étape | code | date_pmes |
|---|---|---|
| Départ | V1, V2 | *la colonne n'existe pas* |
| Après `ADD COLUMN ... NULL` | V1, V2 | NULL, NULL |
| Après `UPDATE` | V1, V2 | 2024-04-15, 2024-04-15 |
| Après `MODIFY ... NOT NULL` | V1, V2 | inchangé, mais plus aucun NULL permis |

* On ajoute d'abord la colonne **facultative**, car les vélos existants n'ont pas encore de valeur.
* Le `UPDATE` n'a pas de `WHERE` parce que l'énoncé dit « **tous** » les vélos. Sans `WHERE`, toutes les lignes sont modifiées.
* Le `MODIFY ... NOT NULL` vient **à la fin** : si on le mettait avant de remplir, la commande échouerait (des lignes seraient vides).
* `MODIFY` redéfinit toute la colonne : on y réécrit le type **et** `NOT NULL`.
* **Alternative en une seule commande :** `ALTER TABLE Velos ADD COLUMN date_pmes TIMESTAMP NOT NULL DEFAULT '2024-04-15';` Elle fonctionne, mais le défaut resterait aussi pour les vélos futurs. L'énoncé demande d'**inscrire** la date, pas de définir un défaut : la méthode en 3 étapes est la plus fidèle.

**`MODIFY` ou `UPDATE` ?**

| | MODIFY | UPDATE |
|---|---|---|
| Change | la **définition** de la colonne | les **valeurs** dans les lignes |
| Commande | `ALTER TABLE ... MODIFY` | `UPDATE ... SET ... WHERE` |
| Exemple | rendre `date_pmes` obligatoire | mettre `'2024-04-15'` dans `date_pmes` |

### 21. Ajouter une clé étrangère et lui associer des données (Section 2.3, 2 points)

> **Énoncé** : Tous nos vélos actuels sont du même modèle. Afin de documenter cette information, nous allons effectuer les opérations suivantes :
>
> * Ajouter aux données le type de vélo pour notre flotte actuelle : vélo de modèle Iconic A12344 v2023 à propulsion humaine.
> * Associer ce type de vélo à tous nos vélos actuels en créant une clé étrangère `id_type` avec sa contrainte d'intégrité à la table `Velos` (nommez-la `fk_Velos_Types`).

**Étapes sur papier**

* **Idée :** au lieu de répéter « Iconic A12344 » sur chaque vélo, on met le modèle **une seule fois** dans `Types`, et chaque vélo pointe vers lui avec un numéro (`id_type`).
* **Étapes dans l'ordre :**
  1. `INSERT` : créer la fiche du type dans `Types` (le type doit exister avant qu'on puisse y pointer).
  2. `ALTER TABLE Velos ADD COLUMN id_type ... NULL` : ajouter la colonne **facultative** (les vélos existants n'ont pas encore de type).
  3. `UPDATE` : remplir `id_type` pour **tous** les vélos (pas de `WHERE`).
  4. `MODIFY ... NOT NULL` : rendre la colonne obligatoire (dans le schéma, `id_type` est `NOT NULL`).
  5. `ADD CONSTRAINT fk_Velos_Types FOREIGN KEY ...` : verrouiller le lien, avec le nom exigé.
* **À vérifier :** `id_type` est un `INT`, comme `Types.id`. La clé étrangère est dans la table enfant (`Velos`), du côté « plusieurs ».

```sql
INSERT INTO Types (categorie, modele, propulsion)
VALUES ('vélo', 'Iconic A12344 v2023', 'humain');

ALTER TABLE Velos ADD COLUMN id_type INT NULL;

UPDATE Velos
SET id_type = (SELECT id FROM Types WHERE modele = 'Iconic A12344 v2023');

ALTER TABLE Velos MODIFY id_type INT NOT NULL;

ALTER TABLE Velos
ADD CONSTRAINT fk_Velos_Types FOREIGN KEY (id_type) REFERENCES Types(id);
```

**Explication**

L'état des tables à chaque étape :

| Étape | Types | Velos.id_type |
|---|---|---|
| Départ | *(vide)* | *la colonne n'existe pas* |
| 1. `INSERT` | `id = 1`, vélo, Iconic A12344 v2023, humain | *n'existe pas* |
| 2. `ADD COLUMN ... NULL` | inchangé | NULL, NULL |
| 3. `UPDATE` | inchangé | 1, 1 |
| 4. `MODIFY ... NOT NULL` | inchangé | 1, 1 (plus de NULL permis) |
| 5. `ADD CONSTRAINT` | inchangé | 1, 1 (lien verrouillé) |

* **Pourquoi le `SELECT` dans le `UPDATE` ?** Il faut mettre dans `id_type` **l'id** du type Iconic, mais on ne le connaît pas avec certitude : c'est la base qui l'a attribué avec `AUTO_INCREMENT`. Le sous-select va le chercher par le nom du modèle (`SELECT id FROM Types WHERE modele = '...'` donne 1). On peut aussi écrire `SET id_type = 1` si on est sûr de l'id, mais le `SELECT` est la version sans risque.
* **Pourquoi l'étape 5 (la contrainte) ?** Après l'étape 3, `id_type` contient bien des `1`, mais ce ne sont que **des nombres** : rien n'empêche d'y écrire 99, et la base ne sait pas que ça pointe vers `Types`. La contrainte crée le lien officiel, et la base l'impose :
  * un `id_type` doit exister dans `Types.id`, sinon la valeur est refusée;
  * on ne peut pas supprimer un type encore utilisé par un vélo.
* **Pourquoi cet ordre ?** On ne peut pas pointer vers quelque chose qui n'existe pas (1), ni rendre obligatoire une colonne vide (2 à 4), ni verrouiller un lien avant que les valeurs soient bonnes (5).
* Les étapes 1 à 4 mettent les **données** au bon endroit, et l'étape 5 **verrouille** le lien.
* Même principe qu'à la question 20 : **ajouter facultatif, remplir, puis rendre obligatoire**.

### 22. Ajouter un champ avec valeur par défaut (Section 2.4, 1 point)

> **Énoncé** : Les bornes permettent de signaler un vélo défectueux, mais notre modèle de données ne le permet pas encore. Ajoutez un champ obligatoire `est_defectueux` de type TINYINT(1) qui a comme valeur par défaut faux (0) à la table `Velos`.

**Étapes sur papier**

* **Quelle commande ?** La table `Velos` existe déjà : `ALTER TABLE ... ADD COLUMN`.
* **Colonne :** `est_defectueux`, type `TINYINT(1)`, `NOT NULL` (« obligatoire »), `DEFAULT 0` (« valeur par défaut faux (0) »).
* **Faut-il remplir les vélos existants ?** Non : le `DEFAULT` leur donne automatiquement 0.

```sql
ALTER TABLE Velos ADD COLUMN est_defectueux TINYINT(1) NOT NULL DEFAULT 0;
```

**Explication**

* `TINYINT(1)` sert de booléen : 0 = faux, 1 = vrai.
* `NOT NULL` parce que l'énoncé dit « champ obligatoire ».
* `DEFAULT 0` parce que l'énoncé donne la valeur par défaut « faux (0) ».
* **Une seule requête suffit.** Contrairement à `date_pmes` (question 20), ici l'énoncé donne un `DEFAULT` : les vélos existants reçoivent 0, donc pas besoin d'`UPDATE` ni de `MODIFY`.
* **Règle pour une colonne obligatoire sur une table qui contient déjà des lignes :**
  * avec un `DEFAULT` donné par l'énoncé : une seule requête;
  * sans valeur par défaut : colonne facultative, `UPDATE`, puis `NOT NULL`.

### 23. Modifier des enregistrements (Section 2.5, 2 points)

> **Énoncé** : Afin de planifier l'entretien de la flotte, nous souhaitons identifier les vélos qui ont été le plus utilisés. Les vélos dont la durée totale d'utilisation cumulée dépasse 5000 minutes doivent être considérés comme défectueux. Modifiez les enregistrements dans la table `Velos` pour mettre `est_defectueux = 1` pour ces vélos.

**Étapes sur papier**

* **Quelle commande ?** On modifie des **valeurs** existantes : `UPDATE ... SET est_defectueux = 1`.
* **Pour quelles lignes ?** Seulement certains vélos : `WHERE`. Sans `WHERE`, tous seraient défectueux.
* **Comment trouver les vélos à plus de 5000 minutes ?** Avec une requête à l'intérieur, construite en 3 temps :
  1. la durée de **chaque** location, en minutes;
  2. le **total par vélo** (`SUM` avec `GROUP BY`);
  3. ne garder que les totaux **supérieurs à 5000** (`HAVING`).
* **Comment relier les deux ?** `WHERE code IN (...)` : on ne modifie que les vélos dont le code est dans la liste trouvée.
* **Comme la prof l'exige :** écrire d'abord le `SELECT` (étapes 1 à 3) pour vérifier qu'on trouve les bons vélos, puis l'entourer avec l'`UPDATE`.

```sql
UPDATE Velos
SET est_defectueux = 1
WHERE code IN (
  SELECT code_velo
  FROM Locations
  GROUP BY code_velo
  HAVING SUM(TIMESTAMPDIFF(MINUTE, date_depart, date_arrivee)) > 5000
);
```

**Explication**

Ce sont **deux requêtes emboîtées**, construites de l'intérieur vers l'extérieur. Exemple avec des données :

| Location | code_velo | durée (min) |
|---|---|---|
| 1 | V1 | 3000 |
| 2 | V1 | 2500 |
| 3 | V2 | 1000 |
| 4 | V2 | 800 |

* **Étape 1, la durée de chaque location :** `TIMESTAMPDIFF(MINUTE, date_depart, date_arrivee)` calcule les minutes entre le départ et l'arrivée. C'est la colonne « durée » ci-dessus.
* **Étape 2, le total par vélo :**

```sql
SELECT code_velo, SUM(TIMESTAMPDIFF(MINUTE, date_depart, date_arrivee)) AS total
FROM Locations
GROUP BY code_velo;
```

| code_velo | total |
|---|---|
| V1 | 5500 |
| V2 | 1800 |

  `GROUP BY` regroupe les lignes de chaque vélo, et `SUM` additionne leurs durées (donc la durée **cumulée**).
* **Étape 3, garder ceux qui dépassent 5000 :** on ajoute `HAVING ... > 5000`. On utilise `HAVING` parce qu'on filtre sur un **total** : `WHERE` ne peut pas filtrer une somme. Le `>` est strict, car l'énoncé dit « dépasse ». Résultat : seulement **V1**.
* **Étape 4, modifier ces vélos :** `UPDATE Velos SET est_defectueux = 1 WHERE code IN (...)`. `IN (...)` veut dire « dont le code est dans cette liste ». Seul V1 est modifié.
* Dans la requête finale, le `SELECT` ne renvoie que `code_velo` (pas `total`), car `IN` compare avec **une seule** colonne.
* Une location encore en cours (`date_arrivee` vide) donne `NULL`, et `SUM` l'ignore.
* **Pourquoi pas de `WHERE` à la question 20 mais un `WHERE` ici ?** À la question 20, l'énoncé dit « **tous** » les vélos. Ici, il ne vise que ceux qui dépassent 5000 minutes. La règle : « tous » = pas de `WHERE`; « certains », « le vélo X », « ceux qui... » = `WHERE` obligatoire.

---

## Partie 6 : Examen formatif, Section 3 (interrogation)

### 24. Afficher les informations d'une station avec des comptages (Section 3.1, 2 points)

> **Énoncé** : Afficher le nombre de vélos disponibles à une station. Affichez pour la station `@station` :
>
> * Nom de la station
> * Longitude et latitude
> * Nombre total de vélos
> * Nombre de vélos à propulsion électrique

**Étapes sur papier (les 6 questions)**

* **Colonnes demandées :** le nom, la longitude et la latitude viennent de `Stations`. Les deux nombres sont des comptages.
* **Tables :** `Stations` (la station), `Velos` (les vélos qui y sont) et `Types` (pour savoir si un vélo est électrique).
* **Jointures :** `Velos.id_station` pointe vers `Stations.id`, et `Velos.id_type` pointe vers `Types.id`.
* **`LEFT JOIN` :** la station doit s'afficher même si elle n'a aucun vélo (les comptes seront alors 0). Un `JOIN` simple la ferait disparaître.
* **Filtre :** `WHERE s.nom = @station` garde une seule station.
* **Regroupement :** `GROUP BY` regroupe les vélos de la station en une seule ligne. On y met toutes les colonnes non comptées.

```sql
SELECT s.nom AS `Nom`, s.longitude AS `Longitude`, s.latitude AS `Latitude`,
       COUNT(v.code) AS `Nombre total de vélos`,
       COUNT(CASE WHEN t.propulsion = 'electrique' THEN 1 END) AS `Nombre de vélos à propulsion électrique`
FROM Stations s
LEFT JOIN Velos v ON v.id_station = s.id
LEFT JOIN Types t ON t.id = v.id_type
WHERE s.nom = @station
GROUP BY s.id, s.nom, s.longitude, s.latitude;
```

**Explication**

* `COUNT(v.code)` compte les vélos de la station. Il ignore les `NULL`, donc il donne 0 si la station est vide (avec `COUNT(*)`, il donnerait 1).
* `COUNT(CASE WHEN t.propulsion = 'electrique' THEN 1 END)` : le `CASE` donne 1 pour un vélo électrique et `NULL` pour les autres. Le `COUNT` ne compte donc que les électriques.
* Les alias servent à nommer les colonnes. **Ils doivent être écrits exactement comme dans la sortie attendue** (majuscule, accents, espaces) : à la première tentative, Progression a refusé `nom`, `longitude`, `latitude` parce que la sortie attendue écrit `Nom`, `Longitude`, `Latitude`.
* **À vérifier :** les deux derniers alias ci-dessus sont ceux de l'énoncé, mais ils doivent être comparés aux en-têtes exacts de la sortie attendue dans Progression. Si `@station` est un **numéro** plutôt qu'un nom, remplacer par `WHERE s.id = @station`.

### 25. Les 10 vélos les plus utilisés (Section 3.2, 2 points)

> **Énoncé** : Afin de planifier l'entretien, nous souhaitons trouver les vélos qui ont le plus roulé.
>
> * Affichez le code, la date de première mise en service, le modèle et la durée totale d'utilisation (en minutes) des 10 vélos les plus utilisés.
> * Nommer les colonnes : Code, Mise en service, Modèle, Minutes.

**Étapes sur papier (les 6 questions)**

* **Colonnes demandées :** le code et la date de mise en service (`date_pmes`) viennent de `Velos`, le modèle de `Types`, et les minutes sont un total calculé. Les alias sont exactement ceux de l'énoncé : `Code`, `Mise en service`, `Modèle`, `Minutes`.
* **Tables :** `Velos`, `Locations` (les durées) et `Types` (le modèle).
* **Jointures :** `Locations.code_velo` pointe vers `Velos.code`, et `Velos.id_type` pointe vers `Types.id`. Un `JOIN` simple suffit : on cherche les vélos **les plus utilisés**, donc ceux qui ont des locations.
* **Filtre :** aucun `WHERE`, car tous les vélos sont considérés.
* **Regroupement :** `GROUP BY` regroupe les locations de chaque vélo en une ligne. On y met les colonnes affichées qui ne sont pas des totaux (`code`, `date_pmes`, `modele`).
* **Tri et limite :** `ORDER BY Minutes DESC` met les plus utilisés en premier, puis `LIMIT 10` garde les 10 premiers. Le tri doit venir **avant** le `LIMIT`.

```sql
SELECT v.code AS `Code`,
       v.date_pmes AS `Mise en service`,
       t.modele AS `Modèle`,
       SUM(TIMESTAMPDIFF(MINUTE, l.date_depart, l.date_arrivee)) AS `Minutes`
FROM Velos v
JOIN Locations l ON l.code_velo = v.code
JOIN Types t ON t.id = v.id_type
GROUP BY v.code, v.date_pmes, t.modele
ORDER BY `Minutes` DESC
LIMIT 10;
```

**Explication**

* `SUM(TIMESTAMPDIFF(...))` : `TIMESTAMPDIFF` donne la durée d'une location en minutes, et `SUM` additionne celles de chaque vélo. C'est le même calcul qu'à la question 23.
* `ORDER BY Minutes DESC` : `DESC` pour avoir les **plus grands** totaux d'abord.
* `LIMIT 10` : seulement les 10 premières lignes. S'il était placé avant le tri, on garderait 10 vélos au hasard.
* Alias entre backticks, avec les accents et les espaces exigés par l'énoncé.

---

## Rappels rapides

* **Ordre des clauses** : `SELECT ... FROM ... JOIN ... ON ... WHERE ... GROUP BY ... ORDER BY ... LIMIT ...`
* **Modèle de jointure** : `FROM table_a JOIN table_b ON table_a.fk = table_b.pk`
* **`JOIN` = `INNER JOIN`** : garde seulement les lignes avec correspondance des deux côtés.
* **`LEFT JOIN`** : garde toute la table de gauche (après `FROM`), `NULL` à droite sans correspondance.
* **« Tous les X, même sans Y »** : `LEFT JOIN` avec X à gauche.
* **« X sans Y »** : `LEFT JOIN ... WHERE colonne_droite IS NULL`.
* **Texte et dates** : entre apostrophes. **Nombres** : sans apostrophes.

### Rappels du formatif (Sections 2 et 3)

* **Données existantes à conserver** : jamais de `DROP` ni de recréation, seulement `ALTER TABLE` (structure) et `UPDATE` (valeurs).
* **Colonne obligatoire sur une table qui contient déjà des lignes** : avec un `DEFAULT` donné, une seule requête; sans valeur par défaut, colonne `NULL`, puis `UPDATE`, puis `MODIFY ... NOT NULL`.
* **`MODIFY` change la définition de la colonne; `UPDATE` change les valeurs.** `MODIFY` redéfinit toute la colonne : réécrire le type et `NOT NULL`.
* **`UPDATE`** : « tous » = pas de `WHERE`; « certains » = `WHERE` obligatoire.
* **Clé étrangère** : le type de la colonne est le même que celui de la clé référencée, la table parente existe déjà, et la contrainte se nomme comme l'énoncé le demande (`ADD CONSTRAINT nom FOREIGN KEY (col) REFERENCES parent(id)`).
* **`WHERE` filtre des lignes; `HAVING` filtre des totaux** (`SUM`, `COUNT`) après un `GROUP BY`.
* **Durée entre deux dates** : `TIMESTAMPDIFF(MINUTE, date_debut, date_fin)`.
* **Compter une partie des lignes** : `COUNT(CASE WHEN condition THEN 1 END)`.
* **Alias** : écrits exactement comme dans la sortie attendue (majuscules, accents, espaces), entre backticks.
* **Top N** : `ORDER BY ... DESC` puis `LIMIT N`, dans cet ordre.
