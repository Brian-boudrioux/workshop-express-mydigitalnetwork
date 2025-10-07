# Workshop – Créer une API REST avec Express & MySQL  
## Projet fil rouge : Réseau social interne MyDigitalSchool  

### 🎯 Objectif général  
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

## 🔐 Étape 3 – Authentification avec JWT  

### Objectif  
Permettre aux utilisateurs de s’inscrire, se connecter et accéder à des routes protégées.

### Étapes  
1. Installer les dépendances :
   ```bash
   npm install bcrypt jsonwebtoken
   ```
2. Créer un modèle `AuthModel.js` avec création et recherche d’utilisateur.
3. Ajouter un `AuthController.js` :
   - `register()` → hash du mot de passe avec bcrypt
   - `login()` → vérification du mot de passe, génération d’un token JWT
4. Créer un middleware `verifyToken.js` :
   ```js
   import jwt from "jsonwebtoken";
   export const verifyToken = (req, res, next) => {
     const token = req.headers.authorization?.split(" ")[1];
     if (!token) return res.status(403).json({ message: "No token" });
     jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
       if (err) return res.status(401).json({ message: "Invalid token" });
       req.user = user;
       next();
     });
   };
   ```

### 💡 Mini-défi  
- Pourquoi stocker le mot de passe haché plutôt qu’en clair ?  
- Quelle différence entre un cookie de session et un JWT ?  

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
