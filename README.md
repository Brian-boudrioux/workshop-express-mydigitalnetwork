Dans ce workshop, tu vas apprendre à construire pas à pas une **API REST complète** avec **Express** et **MySQL** pour un **réseau social interne** destiné aux étudiants et intervenants de *MyDigitalSchool*.  

Le projet servira de fil conducteur pour introduire progressivement :  
- la structure **MVC**,  
- la **gestion de l’authentification** (JWT),  
- les **tests automatisés** (Jest & Supertest),  
- et une **messagerie instantanée** (Socket.io).  

---

## 🧩 Étape 1 – Initialisation du projet & connexion MySQL  

### Objectif  
Mettre en place un projet Node.js fonctionnel, installer Express et connecter une base de données MySQL.

### Étapes  
1. **Initialiser le projet**
   ```bash
   mkdir mds-social-api
   cd mds-social-api
   npm init -y
   npm install express mysql2 dotenv
   ```
2. **Créer un premier serveur**
   Crée un fichier `index.js` :
   ```js
   import express from "express";
   const app = express();
   app.use(express.json());

   app.get("/", (req, res) => {
     res.send("Bienvenue sur l’API MyDigitalSchool!");
   });

   app.listen(3000, () => console.log("Server running on http://localhost:3000"));
   ```
3. **Configurer la connexion MySQL**
   - Crée un fichier `.env` :
     ```
     DB_HOST=localhost
     DB_USER=root
     DB_PASSWORD=
     DB_NAME=mds_social
     ```
   - Puis `config/db.js` :
     ```js
     import mysql from "mysql2/promise";
     import dotenv from "dotenv";
     dotenv.config();

     export const db = await mysql.createConnection({
       host: process.env.DB_HOST,
       user: process.env.DB_USER,
       password: process.env.DB_PASSWORD,
       database: process.env.DB_NAME,
     });
     console.log("✅ MySQL connected");
     ```
   - Importer la connexion dans `index.js` pour tester.

### 🚀 Test  
Lance le serveur :
```bash
node index.js
```
Ouvre http://localhost:3000 → le message d’accueil doit s’afficher.

### 💡 Mini-défi  
- Pourquoi Express est-il souvent choisi pour construire des API REST ?  
- Peux-tu expliquer la différence entre `mysql` et `mysql2/promise` ?  

---

## 🧱 Étape 2 – Structurer le projet avec une architecture MVC  

### Objectif  
Organiser ton code de manière claire avec les dossiers :  
`models`, `controllers`, `routes`, et `config`.

### Étapes  
1. **Créer la structure de base**
   ```
   mds-social-api/
   ├── config/
   ├── controllers/
   ├── models/
   ├── routes/
   ├── index.js
   └── .env
   ```
2. **Créer une première ressource : User**
   - `models/UserModel.js`
     ```js
     import { db } from "../config/db.js";

     export const getAllUsers = async () => {
       const [rows] = await db.query("SELECT * FROM users");
       return rows;
     };
     ```
   - `controllers/UserController.js`
     ```js
     import { getAllUsers } from "../models/UserModel.js";

     export const fetchUsers = async (req, res) => {
       const users = await getAllUsers();
       res.json(users);
     };
     ```
   - `routes/UserRoutes.js`
     ```js
     import express from "express";
     import { fetchUsers } from "../controllers/UserController.js";
     const router = express.Router();

     router.get("/", fetchUsers);
     export default router;
     ```
3. **Brancher les routes dans `index.js`**
   ```js
   import userRoutes from "./routes/UserRoutes.js";
   app.use("/api/users", userRoutes);
   ```

### 💡 Mini-défi  
- Qu’apporte la séparation MVC à ton projet ?  
- Peux-tu expliquer le rôle exact de chaque couche ?  

---

## 🧩 Étape 2bis – Implémenter un CRUD complet  

### Objectif  
Avant d’ajouter l’authentification, il est important de rendre l’API pleinement fonctionnelle avec un **CRUD complet** pour les utilisateurs et les posts.  
Cette étape te permettra de pratiquer la logique MVC et la manipulation des données avec MySQL.

### Guidelines  

1. **Créer les modèles (`UserModel`, `PostModel`)**  
   - Définir les fonctions pour **créer, lire, mettre à jour et supprimer** des enregistrements.  
   - Pour chaque modèle, prévoir des fonctions : `create`, `getById`, `getAll`, `update`, `delete`.  
   - Vérifier les types et la validation de données avant de les envoyer à MySQL.

2. **Créer les contrôleurs (`UserController`, `PostController`)**  
   - Appeler les fonctions du modèle et gérer les réponses HTTP.  
   - Prévoir des statuts corrects : 200 (OK), 201 (Created), 404 (Not Found), 500 (Erreur serveur).  
   - Ajouter des messages clairs pour chaque réponse.

3. **Définir les routes REST (`UserRoutes`, `PostRoutes`)**  
   - Routes principales :  
     ```
     POST /api/users      # créer un utilisateur
     GET /api/users/:id   # récupérer un utilisateur
     PUT /api/users/:id   # mettre à jour
     DELETE /api/users/:id# supprimer
     GET /api/users       # liste tous les utilisateurs
     ```
   - Idem pour les posts : `POST /api/posts`, `GET /api/posts/:id`, etc.  
   - Utiliser des **noms clairs et cohérents** pour les endpoints.

4. **Tester le CRUD**  
   - Utiliser Postman ou Insomnia pour tester toutes les routes.  
   - Vérifier que les données sont correctement enregistrées, récupérées, mises à jour et supprimées.  

### 💡 Mini-défi  
- Ajoute un champ `created_at` pour chaque table (`users` et `posts`) et fais en sorte qu’il soit renvoyé par toutes les routes GET.  
- Réfléchis à : pourquoi est-il important de séparer clairement les routes POST, GET, PUT, DELETE dans une API REST ?  

---

## 🔐 Étape 3 – Authentification avec JWT et Argon2

### 🎯 Objectif  
Permettre aux utilisateurs de **s’inscrire**, **se connecter** et **accéder à des routes protégées** en toute sécurité.

---

### ⚙️ 1. Installation des dépendances

Installe les bibliothèques nécessaires :

```bash
npm install argon2 jsonwebtoken
```

👉 `argon2` permet de **hasher et vérifier les mots de passe** de manière sécurisée.  
👉 `jsonwebtoken` permet de **générer et vérifier les tokens d’accès** pour les routes protégées.

---

### 🧱 2. Modèle utilisateur – `AuthModel.js`

Le modèle doit permettre de :
- Créer un utilisateur avec un mot de passe hashé.
- Trouver un utilisateur par son email.

Exemple d’approche :

```js
// Pseudo-exemple
export const createUser = async (email, passwordHash) => {
  // Insertion SQL : INSERT INTO users (email, password) VALUES (?, ?)
};

export const findUserByEmail = async (email) => {
  // Requête SQL : SELECT * FROM users WHERE email = ?
};
```

---

### 🧑‍💻 3. Contrôleur d’authentification – `AuthController.js`

Ce fichier contient deux fonctions principales :
- **`register()`** : inscription d’un utilisateur avec hash du mot de passe via Argon2.  
- **`login()`** : connexion avec vérification du mot de passe et génération d’un token JWT contenant l’`id` utilisateur.

---

#### ➕ a. Inscription d’un utilisateur

1. **Récupérer les données** envoyées dans le corp de la requette (email, mot de passe).
2. **Vérifier si l’utilisateur existe déjà**.
3. **Hasher le mot de passe** avec Argon2 :

```js
const hashedPassword = await argon2.hash(password);
```

4. **Enregistrer le nouvel utilisateur** avec le mot de passe hashé.
5. **Retourner un message de succès**.

---

#### 🗝️ b. Connexion utilisateur

1. **Chercher l’utilisateur** en base à partir de son email.  
   Exemple :

   ```js
   const user = await findUserByEmail(email);
   if (!user) {
     // utilisateur introuvable
   }
   ```

2. **Comparer le mot de passe envoyé** avec celui hashé en base :  

   ```js
   const isPasswordValid = await argon2.verify(user.password, password);
   if (!isPasswordValid) {
     // mot de passe incorrect
   }
   ```

3. **Générer un token JWT** contenant l’identifiant de l’utilisateur :

   ```js
   const token = jwt.sign(
     { id: user.id, email: user.email },
     process.env.JWT_SECRET,
     { expiresIn: "1h" }
   );
   ```

4. **Renvoyer le token** dans la réponse (par exemple dans les headers) :

   ```js
   res.header("Authorization", `Bearer ${token}`);
   ```

---

### 🔒 4. Middleware de vérification du token – `isAuth.js`

Le but est de **protéger certaines routes** en vérifiant le token JWT.

Étapes principales :

1. **Récupérer le token** dans les headers :

   ```js
   const token = req.headers.authorization?.split(" ")[1];
   ```

2. **Vérifier la validité du token** :

   ```js
   jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
     if (err) {
       // Token invalide ou expiré
     }
   });
   ```

3. **Stocker les infos du token** (id, email) dans `req.user` pour y accéder dans les routes protégées.


---

### 🚀 5. Exemple d’utilisation dans une route protégée

```js
// routes/UserRoutes.js
import express from "express";
import { isAuth } from "../middlewares/isAuth.js";

const router = express.Router();

router.get("/profile", isAuth, async (req, res) => {
  res.json({
    message: "Accès autorisé à la route protégée",
    user: req.user,
  });
});

export default router;
```

---

### 🧪 6. Exemple de test via Postman ou cURL

#### 🔸 Inscription

```bash
POST /register
Content-Type: application/json

{
  "email": "test@example.com",
  "password": "MonSuperMotDePasse"
}
```

#### 🔸 Connexion

```bash
POST /login
Content-Type: application/json

{
  "email": "test@example.com",
  "password": "MonSuperMotDePasse"
}
```

**Réponse :**

```json
{
  "message": "Connexion réussie",
  "token": "eyJhbGciOiJIUzI1..."
}
```

#### 🔸 Route protégée

```bash
GET /profile
Authorization: Bearer <votre_token>
```

---

### ✅ 7. Résumé des bonnes pratiques

- Utiliser **Argon2** pour le hash des mots de passe (plus sûr que bcrypt).  
- Ne jamais stocker les mots de passe en clair.  
- Définir une variable d’environnement `JWT_SECRET` forte et unique.  
- Renvoyer le token via les **headers** pour plus de sécurité.  
- Ajouter une **expiration courte** du token (`1h` par exemple).  

---

### 💡 8. Pour aller plus loin

- Implémente un **système de refresh token** pour renouveler l’accès sans reconnecter.  
- Ajoute une **gestion des rôles utilisateurs (admin, user, etc.)**.  
- Protège certaines routes selon le rôle stocké dans le token.  

---

## 🧪 Étape 6 – Tests d’intégration avec Jest et Supertest

### 🎯 Objectif  
Apprendre à **tester une API Express** connectée à une **base de données** avec **Jest** et **Supertest**.

---

### ⚙️ 1. Installation des dépendances

```bash
npm install --save-dev jest supertest
```

Ajoute également `cross-env` si tu veux définir des variables d’environnement spécifiques aux tests :

```bash
npm install --save-dev cross-env
```

---

### 🧱 2. Configuration du script de test

Dans ton `package.json`, ajoute un script dédié :

```json
"scripts": {
  "test": "cross-env NODE_ENV=test jest --runInBand"
}
```

> `--runInBand` permet d’éviter les conflits de connexions simultanées à la base de données pendant les tests.

---

### 🧩 3. Base de données de test

Crée une base spécifique pour les tests, par exemple `myapp_test`.

Exemple de configuration MySQL dans `config/db.js` :

```js
import mysql from "mysql2/promise";

const db = await mysql.createConnection({
  host: process.env.DB_HOST,
  port: process.env.DB_PORT,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.NODE_ENV === "test" ? "myapp_test" : process.env.DB_NAME
});

export default db;
```

💡 L’idée est d’utiliser une **BDD séparée** pour éviter d’écraser les données de production pendant les tests.

---

### 🧪 4. Exemple de test d’intégration – `users.test.js`

Ce test illustre :
- la création d’un serveur de test avec Supertest,
- l’utilisation d’une **base temporaire de test**,
- la **fermeture propre** de la connexion à la fin avec `afterAll()`.

```js
import request from "supertest";
import app from "../app.js";       // ton application Express
import db from "../config/db.js";  // ta connexion MySQL

describe("🧪 Tests d'intégration de l'API Users", () => {

  beforeAll(async () => {
    // Nettoyer la base avant de commencer les tests
    await db.query("DELETE FROM users");
  });

  test("GET /users doit retourner un tableau d'utilisateurs", async () => {
    const res = await request(app).get("/users");
    expect(res.statusCode).toBe(200);
    expect(Array.isArray(res.body)).toBe(true);
  });

  afterAll(async () => {
    // Fermer proprement la connexion à la base après tous les tests
    await db.end();
  });
});
```

---

### 🧠 5. Mise en pratique : écrire des tests d’intégration pour les routes `/posts`

C’est ton tour 🎯  
Tu vas maintenant écrire des **tests d’intégration** pour les routes liées aux **articles (posts)** de ton API.

#### Objectif
Tester les routes :
- `GET /posts` → Récupération de tous les articles  
- `GET /posts/:id` → Récupération d’un article précis  

---

##### 🔹 Étapes suggérées

##### 1. Préparer la base de test
- Vide la table `posts` au début de la suite de tests (`beforeAll()`).
- Insère manuellement quelques articles de test (2 ou 3).

##### 2. Écrire le test pour `GET /posts`
- Fais un appel avec Supertest à `GET /posts`.
- Vérifie :
  - que le statut de réponse est `200`,
  - que la réponse est un **tableau**,
  - et qu’il contient le bon **nombre d’articles** insérés.

🧩 **Guide :**
```js
expect(res.statusCode).toBe(200);
expect(Array.isArray(res.body)).toBe(true);
expect(res.body.length).toBe(3);
```

##### 3. Écrire le test pour `GET /posts/:id`
- Fais un appel à `GET /posts/1` (ou un ID existant).
- Vérifie :
  - que le statut de réponse est `200`,
  - que la réponse contient un **objet unique**,
  - et que les propriétés (`id`, `title`, `content`, etc.) sont présentes.

🧩 **Guide :**
```js
expect(res.statusCode).toBe(200);
expect(res.body).toHaveProperty("id");
expect(res.body).toHaveProperty("title");
```

##### 4. Nettoyer après les tests
- Ferme la connexion à la base de données avec `afterAll()` :
  ```js
  afterAll(async () => await db.end());
  ```

---

#### 💡 Conseils

- Utilise des **données cohérentes** entre les tests pour éviter les erreurs d’ID.  
- Pense à **vider la BDD** avant chaque suite de tests pour un état propre.  
- Exécute les tests avec :
  ```bash
  npm test
  ```
- Utilise des **describe()** séparés pour les différents groupes de routes (ex: `users`, `posts`, `comments`).

---

### ✅ Résultat attendu

À la fin de cette mise en pratique, tu sauras :

- Mettre en place une base de test dédiée.  
- Écrire des tests d’intégration pour plusieurs routes (`GET /posts` et `GET /posts/:id`).  
- Nettoyer ton environnement de test.  
- Structurer une suite de tests robuste pour ton API Node.js.  
  
---

## 💬 Étape 5 – Messagerie instantanée avec Socket.io  

### Objectif  
Permettre aux utilisateurs de discuter en temps réel.

### Étapes  
1. Installer :
   ```bash
   npm install socket.io
   ```
2. Adapter ton serveur dans `index.js` :
   ```js
   import { Server } from "socket.io";
   import http from "http";
   const server = http.createServer(app);
   const io = new Server(server, { cors: { origin: "*" } });

   io.on("connection", (socket) => {
     console.log("User connected:", socket.id);
     socket.on("message", (data) => io.emit("message", data));
   });

   server.listen(3000, () => console.log("🚀 Server running on port 3000"));
   ```
3. Créer un modèle `MessageModel.js` pour sauvegarder les messages en BDD.

### 💡 Mini-défi  
- Quelle différence entre HTTP et WebSocket ?  
- Comment garantir la sécurité d’un chat temps réel ?  

---

## 🧹 Étape 6 – Middleware global, gestion d’erreurs & déploiement  

### Objectif  
Rendre l’API robuste et prête à être déployée.

### Étapes  
1. Créer un middleware `errorHandler.js` :
   ```js
   export const errorHandler = (err, req, res, next) => {
     console.error(err.stack);
     res.status(500).json({ message: "Internal Server Error" });
   };
   ```
   Et l’utiliser dans `index.js` :
   ```js
   app.use(errorHandler);
   ```
2. Ajouter un script de production :
   ```json
   "scripts": {
     "start": "node index.js"
   }
   ```
3. Tester ton API avant le déploiement.

### 💡 Mini-défi  
- Qu’est-ce qu’un middleware global ?  
- Quelles sont les bonnes pratiques avant de déployer une API REST ?  

---

## 🧠 Conclusion  

Félicitations 🎉  
Tu as créé les bases d’un **réseau social interne MyDigitalSchool**, en comprenant :  
- la structure MVC,  
- l’authentification JWT,  
- les tests automatisés,  
- et la communication temps réel.

Prochaine étape : améliorer l’expérience front (React, Vue, etc.) et connecter cette API pour donner vie à ton mini-réseau social 🚀  

---

## 🔗 Ressources utiles  
- [Documentation Express](https://expressjs.com/fr/)  
- [MySQL2 Docs](https://www.npmjs.com/package/mysql2)  
- [JWT.io](https://jwt.io/)  
- [Socket.io](https://socket.io/docs/v4)  
- [Jest](https://jestjs.io/docs/getting-started)
