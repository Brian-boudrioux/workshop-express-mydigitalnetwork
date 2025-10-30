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

Exemple de modèle de base :

```js
// models/AuthModel.js
import db from "../config/db.js";

export const createUser = async (email, passwordHash) => {
  const [result] = await db.query(
    "INSERT INTO users (email, password) VALUES (?, ?)",
    [email, passwordHash]
  );
  return result.insertId;
};

export const findUserByEmail = async (email) => {
  const [[user]] = await db.query("SELECT * FROM users WHERE email = ?", [email]);
  return user;
};
```

---

### 🧑‍💻 3. Contrôleur d’authentification – `AuthController.js`

Ce fichier contient deux fonctions principales :
- **`register()`** : inscription d’un utilisateur avec hash du mot de passe via Argon2.  
- **`login()`** : connexion avec vérification du mot de passe et génération d’un token JWT contenant l’`id` utilisateur.

---

#### ➕ a. Inscription d’un utilisateur

```js
// controllers/AuthController.js
import argon2 from "argon2";
import jwt from "jsonwebtoken";
import { createUser, findUserByEmail } from "../models/AuthModel.js";

export const register = async (req, res) => {
  try {
    const { email, password } = req.body;

    // Vérifier si l’utilisateur existe déjà
    const existingUser = await findUserByEmail(email);
    if (existingUser) {
      return res.status(400).json({ message: "Utilisateur déjà existant" });
    }

    // Hasher le mot de passe avec Argon2
    const hashedPassword = await argon2.hash(password);

    // Créer l’utilisateur en BDD
    const userId = await createUser(email, hashedPassword);

    res.status(201).json({
      message: "Utilisateur créé avec succès",
      userId,
    });
  } catch (error) {
    console.error(error);
    res.status(500).json({ message: "Erreur lors de l'inscription" });
  }
};
```

---

#### 🗝️ b. Connexion utilisateur

```js
export const login = async (req, res) => {
  try {
    const { email, password } = req.body;

    // Étape 1 : vérifier si l’utilisateur existe
    const user = await findUserByEmail(email);
    if (!user) {
      return res.status(404).json({ message: "Utilisateur introuvable" });
    }

    // Étape 2 : vérifier que le mot de passe correspond
    const isPasswordValid = await argon2.verify(user.password, password);
    if (!isPasswordValid) {
      return res.status(401).json({ message: "Mot de passe incorrect" });
    }

    // Étape 3 : générer le token JWT contenant l'id de l'utilisateur
    const token = jwt.sign(
      { id: user.id, email: user.email },
      process.env.JWT_SECRET,
      { expiresIn: "1h" }
    );

    // Étape 4 : renvoyer le token dans les headers + message JSON
    res
      .header("Authorization", `Bearer ${token}`)
      .json({ message: "Connexion réussie", token });
  } catch (error) {
    console.error(error);
    res.status(500).json({ message: "Erreur lors de la connexion" });
  }
};
```

---

### 🔒 4. Middleware de vérification du token – `verifyToken.js`

Ce middleware protège les routes privées en vérifiant la présence et la validité du token JWT.

```js
// middlewares/isAuth.js
import jwt from "jsonwebtoken";

export const isAuth = (req, res, next) => {
  const authHeader = req.headers.authorization;
  const token = authHeader && authHeader.split(" ")[1];

  if (!token) {
    return res.status(403).json({ message: "Aucun token fourni" });
  }

  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) {
      return res.status(401).json({ message: "Token invalide ou expiré" });
    }
    req.user = decoded; // contient l'id et l'email
    next();
  });
};
```

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

## 🧪 Étape 4 – Tests avec Jest & Supertest  

### Objectif  
Apprendre à tester ton API avec Jest pour la logique et Supertest pour les routes.

### Étapes  
1. Installer :
   ```bash
   npm install --save-dev jest supertest
   ```
2. Créer un fichier `tests/user.test.js` :
   ```js
   import request from "supertest";
   import app from "../index.js";

   describe("GET /api/users", () => {
     it("should return users", async () => {
       const res = await request(app).get("/api/users");
       expect(res.statusCode).toBe(200);
     });
   });
   ```
3. Ajouter dans `package.json` :
   ```json
   "scripts": {
     "test": "jest --watchAll"
   }
   ```

### 💡 Mini-défi  
- Quelle différence entre un test unitaire et un test d’intégration ?  
- Pourquoi tester ton API est-il crucial avant le déploiement ?  

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
