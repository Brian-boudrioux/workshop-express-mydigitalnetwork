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

## 🧪 Étape 4 – Tests d’intégration avec Jest et Supertest

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
    // réinitialiser les id en auto_increment pour s'assurer que les tests soit reproductible
    await db.query("ALTER TABLE posts AUTO_INCREMENT = 1");
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
- Réinitialiser les id en auto_increment pour s'assurer que les tests soit reproductible (`beforeAll()`).
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

### 🎯 Objectif
Mettre en place une **messagerie instantanée entre utilisateurs connectés**, en utilisant :
- le serveur Express déjà existant (`config/server.js`)
- l’authentification JWT déjà en place
- **Socket.io** pour gérer la communication temps réel.

---

### ⚙️ 1. Installation de Socket.io

Installe la dépendance :

```bash
npm install socket.io
```

---

### 🧩 2. Création du module Socket.io

➡️ Dans le dossier `config/`, crée un nouveau fichier :  
`config/socket.js`

Ce fichier va gérer toute la logique Socket.io, tout en **réutilisant le serveur Express déjà créé** dans ton projet.

**Exemple de structure :**

```js
// config/socket.js
import { Server } from "socket.io";
import jwt from "jsonwebtoken";
import dotenv from "dotenv";
dotenv.config();

let io;

export const initSocket = (server) => {
  io = new Server(server, {
    cors: {
      origin: "*", // à adapter pour la prod
      methods: ["GET", "POST"],
    },
  });

  // Middleware d’authentification JWT
  io.use((socket, next) => {
    const token = socket.handshake.auth?.token;
    if (!token) return next(new Error("Token manquant"));

    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      socket.user = decoded; // on garde l’utilisateur dans la session socket
      next();
    } catch (err) {
      console.error("JWT invalide :", err.message);
      next(new Error("Token invalide"));
    }
  });

  // Gestion des connexions utilisateurs
  io.on("connection", (socket) => {
    console.log(`✅ Utilisateur connecté : ${socket.user.email}`);

    // Réception d’un message
    socket.on("sendMessage", (data) => {
      const message = {
        sender: socket.user.email,
        content: data.content,
        timestamp: new Date(),
      };

      // Diffuse à tous les clients connectés
      io.emit("newMessage", message);
    });

    // Déconnexion
    socket.on("disconnect", () => {
      console.log(`❌ ${socket.user.email} s'est déconnecté`);
    });
  });

  console.log("💬 Socket.io initialisé");
  return io;
};

export const getIO = () => {
  if (!io) throw new Error("Socket.io non initialisé !");
  return io;
};
```

---

### 🧱 3. Adapter `index.js`

➡️ Ouvre ton fichier `index.js` et **remplace son contenu** par ce qui suit :

```js
import http from "http";
import app from "./config/server.js";
import { initSocket } from "./config/socket.js";

// Création du serveur HTTP à partir d’Express
const server = http.createServer(app);

// Initialisation de Socket.io avec ce serveur
initSocket(server);

// Démarrage du serveur
server.listen(3000, () => console.log("🚀 Server running on http://localhost:3000"));
```

💡 Ici, on garde le serveur Express inchangé, mais on le “surélève” pour qu’il supporte Socket.io.  
Le module `config/socket.js` prend ensuite le relais pour gérer toute la logique temps réel.

---

### 🧠 4. Côté client : connexion avec le token JWT (via Insomnia)

Tu peux tester ta messagerie **sans interface front-end**, directement depuis **Insomnia**, qui permet aussi de gérer les connexions **WebSocket**.

---

#### Etape 0 - Installation d'insomnia et du plugin socketio

1. Télécharge (Insomnia)[https://insomnia.rest/download]

2. Va dans préferences puis plugins et ajoute le plugin “insomnia-plugin-socketio”

#### 🧩 Étape 1 – Obtenir un token JWT

1. Lance ton API (`npm run dev`).
2. Dans **Postman**, envoie une requête `POST` vers :
   ```
   http://localhost:3000/api/login
   ```
3. Fournis un corps JSON valide :
   ```json
   {
     "email": "test@example.com",
     "password": "123456"
   }
   ```
4. Copie le **token JWT** reçu dans les headers `Authorization`

---

#### 🧩 Étape 2 – Connexion au serveur Socket.io via Insomnia

1. Ouvre un **nouvel onglet Socket.io** dans Postman.  
   Clique sur **“New → Socket.io”**.
2. Entre l’URL suivante :
   ```
   ws://localhost:3000
   ```
3. Clique sur **Headers** et ajoute :
   ```json
   {
     "token": "TON_JWT_ICI"
   }
   ```
   👉 Insomnia enverra automatiquement ce token dans le handshake WebSocket, comme ton serveur l’attend dans `socket.handshake.auth.token`.

4. Clique sur **Connect**.  
   Tu devrais voir dans ta console serveur :
   ```
   ✅ Utilisateur connecté : test@example.com
   ```

---

#### 🧩 Étape 3 – Envoyer un message

1. Une fois connecté, envoie un message au serveur en utilisant l’événement `sendMessage`.  
   Dans Insomnia :
   - Choisis le **type d’événement** : `sendMessage`
   - Dans le corps JSON, ajoute :
     ```json
     {
       "content": "Hello depuis Postman 👋"
     }
     ```

2. Tu devrais voir la réponse côté serveur :
   ```
   💬 Nouveau message reçu : Hello depuis Insomnia 👋
   ```

3. Tous les clients WebSocket connectés recevront un événement `newMessage` contenant le message complet :
   ```json
   {
     "sender": "test@example.com",
     "content": "Hello depuis Postman 👋",
     "timestamp": "2025-10-30T12:34:56.789Z"
   }
   ```

---

#### 🧩 Étape 4 – Tester plusieurs utilisateurs

1. Connecte-toi dans **deux onglets WebSocket Insomnia différents**, chacun avec un token JWT différent.  
2. Envoie un message depuis le premier compte :
   ```json
   {
     "content": "Salut 👋"
   }
   ```
3. Le second utilisateur devrait recevoir immédiatement ce message en temps réel.

---

✅ **Résultat attendu :**
- Le serveur affiche chaque connexion/déconnexion d’utilisateur.
- Les messages sont diffusés instantanément à tous les utilisateurs connectés avec un token valide.
- Les connexions sans token ou avec un token invalide sont refusées par Socket.io.

---

💡 **Astuce :**
Tu peux ouvrir la console “Messages” de Postman pour visualiser les événements entrants et sortants en temps réel.  
C’est très pratique pour tester les échanges WebSocket sans front-end.

---

### 🧩 5. Challenge – Améliorations du système de messagerie

#### 🎯 Objectif :
Faire évoluer le système de messagerie vers un modèle **entièrement privé et sécurisé**, où chaque message est adressé à un destinataire précis, sauvegardé en base de données et accessible uniquement par les utilisateurs concernés.

---

#### 💾 1. Sauvegarde des messages privés en base de données

**But :** conserver l’historique complet des échanges privés entre utilisateurs.

##### Étapes :
1. Crée une table `private_messages` :
   ```sql
   CREATE TABLE private_messages (
     id INT AUTO_INCREMENT PRIMARY KEY,
     sender_id INT NOT NULL,
     receiver_id INT NOT NULL,
     content TEXT NOT NULL,
     created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
     FOREIGN KEY (sender_id) REFERENCES users(id),
     FOREIGN KEY (receiver_id) REFERENCES users(id)
   );
   ```
2. Crée un fichier `models/MessageModel.js` :
   - Ajoute une méthode `createPrivateMessage(senderId, receiverId, content)` pour enregistrer un message.
   - Ajoute une méthode `getConversation(senderId, receiverId)` pour récupérer les messages entre deux utilisateurs.

3. Dans le handler `sendPrivateMessage` du fichier `config/socket.js` :
   - Récupère l’`id` de l’utilisateur connecté via `socket.user`.
   - Insère le message en BDD avant de le transmettre uniquement à la room du destinataire.

🧠 *Exemple d’approche* :
```js
const message = await messageModel.createPrivateMessage(socket.user.id, data.receiverId, data.content);
io.to(`user_${data.receiverId}`).emit("privateMessage", message);
```

---

#### 🔒 2. Rooms et communication privée

**But :** garantir que chaque utilisateur ne reçoive que les messages qui lui sont destinés.

##### Étapes :
1. Lorsqu’un utilisateur se connecte :
   - Authentifie le via son token JWT.
   - Fais-le rejoindre une room spécifique à son ID :
     ```js
     socket.join(`user_${socket.user.id}`);
     ```
2. Crée un événement `sendPrivateMessage` :
   - L’événement reçoit `{ receiverId, content }`.
   - Le serveur sauvegarde le message puis l’envoie uniquement à la room du destinataire.
3. Ajoute un événement `getConversation` pour récupérer les anciens messages entre deux utilisateurs :
   ```js
   socket.on("getConversation", async (receiverId) => {
     const messages = await messageModel.getConversation(socket.user.id, receiverId);
     socket.emit("conversationHistory", messages);
   });
   ```

💡 *Astuce :* Les rooms Socket.io permettent d’isoler les échanges de manière très simple et efficace.

---

#### 🧱 3. Chargement de l’historique au moment de la connexion

**But :** offrir à l’utilisateur la possibilité de consulter ses anciennes conversations dès sa connexion.

##### Étapes :
1. Lors de la connexion d’un utilisateur (`io.on("connection")`), récupère ses dernières conversations :
   ```js
   const messages = await messageModel.getRecentMessagesForUser(socket.user.id);
   socket.emit("previousMessages", messages);
   ```
2. Côté client (via Postman WebSocket ou interface web), écoute l’événement `previousMessages` pour afficher l’historique.

---

#### 🚫 4. Sécurité renforcée

**But :** sécuriser les échanges et prévenir tout accès non autorisé.

##### Étapes :
1. Vérifie le **token JWT** à chaque connexion Socket.io (dans ton middleware ou dès la phase d’authentification).
2. Déconnecte automatiquement les utilisateurs dont le token est invalide ou expiré.

---

#### ✅ Résultat attendu

À la fin de cette mise en pratique, ton système de messagerie :
- stocke les messages en BDD,  
- permet des conversations privées,  
- charge l’historique à la connexion,  
- et gère la sécurité JWT en continu.

---

#### 🧠 Bonus :
- Ajouter un système de **notifications** quand un nouveau message privé arrive.
- Permettre la **suppression** d’un message par l’expéditeur.
- Implémenter un indicateur “vu / non vu” côté client.
  
---

## 🧹 Étape 6 – Middleware global, sécurité, gestion d’erreurs & déploiement  

### 🎯 Objectif  
Rendre ton API **robuste**, **sécurisée** et **prête à être déployée** en production.

---

### 🧰 1. Ajouter des middlewares de sécurité

Avant toute chose, installe les bibliothèques nécessaires :

```bash
npm install helmet cors
```

#### 🪖 a. Protection des headers avec Helmet
Helmet aide à sécuriser ton API en configurant automatiquement plusieurs en-têtes HTTP :

```js
import helmet from "helmet";
app.use(helmet());
```

Helmet :
- empêche certaines failles XSS,
- masque les infos du serveur (`X-Powered-By`),
- renforce la politique de contenu.

> 💡 Tu peux personnaliser certaines options, par exemple :
> ```js
> app.use(helmet({
>   crossOriginResourcePolicy: false,
> }));
> ```

---

#### 🌍 b. Gestion du CORS (Cross-Origin Resource Sharing)
CORS permet à des applications front (par ex. React, Postman ou Socket.io) d’accéder à ton API.

Installe et configure :
```js
import cors from "cors";
app.use(cors({
  origin: ["http://localhost:3000", "http://127.0.0.1:3000"],
  credentials: true,
}));
```

> 💡 En production, restreins l’accès aux seuls domaines autorisés (ex: ton site déployé).

---

### 🧩 2. Gestion centralisée des erreurs

Crée un middleware global `middlewares/errorHandler.js` :

```js
// middlewares/errorHandler.js
export const errorHandler = (err, req, res, next) => {
  console.error("🔥 Error:", err.stack || err.message);
  
  const statusCode = err.status || 500;
  const message = err.message || "Internal Server Error";

  res.status(statusCode).json({
    success: false,
    message,
  });
};
```

Dans `config/server.js`, **place-le après toutes tes routes** :

```js
import { errorHandler } from "../middlewares/errorHandler.js";
app.use(errorHandler);
```

> 🧠 Ce middleware capture toutes les erreurs non gérées et renvoie une réponse JSON propre au client.

---

### ⚙️ 3. Variables d’environnement & configuration

Assure-toi que ton projet utilise un fichier `.env` pour les données sensibles :
```
PORT=3000
JWT_SECRET=superSecretKey
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=password
DB_NAME=mds_social
```

Et récupère ces valeurs via `process.env` dans ton code :
```js
import dotenv from "dotenv";
dotenv.config();
```

---

### 🚀 4. Préparation au déploiement

Ajoute un script dans ton `package.json` :

```json
"scripts": {
  "dev": "nodemon index.js",
  "start": "node index.js"
}
```

Avant le déploiement :
1. **Teste ton API localement** avec Postman et vérifie que tout fonctionne.
2. **Désactive** les logs inutiles (`console.log` massifs, par exemple).
3. **Active** un niveau de logs clair (par exemple via `winston`).
4. **Vérifie** que tes erreurs sont bien gérées par le middleware `errorHandler`.
5. **Teste la sécurité** avec un outil comme [OWASP ZAP](https://www.zaproxy.org/).

---

### 🧠 Mini-défi  

- Qu’est-ce qu’un middleware global et dans quel ordre doit-il être chargé ?  
- Pourquoi est-il important de restreindre le CORS en production ?  
- Quelles autres mesures de sécurité peux-tu ajouter avant un déploiement (indices : rate limiting, sanitization, logging...) ?

---

### ✅ Résultat attendu
Ton API :
- applique automatiquement des **headers de sécurité** avec Helmet,  
- autorise uniquement les **origines approuvées** avec CORS,  
- capture toutes les erreurs serveur dans un **middleware global**,  
- et peut être **déployée en production** sereinement.

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
