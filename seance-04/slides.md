---
theme: default
title: Web 2 - Séance 04 - Hachage et Sécurité
---

# Web 2 — Séance 04
## Hachage et Sécurité

---

# ❌ NE PAS : Stocker les mots de passe en clair

```ts
// DANGER ! Cela expose les données sensibles
interface User {
  id: number;
  email: string;
  password: string; // ❌ JAMAIS en clair !
}

const users = [
  { id: 1, email: "john@gmail.com", password: "secret123" },
  { id: 2, email: "jane@gmail.com", password: "password456" },
];
```

Si la BDD est piratée, tous les mots de passe sont compromis. <br>
Et si l'utilisateur réutilise ce mot de passe ailleurs... catastrophe !

À la place, on stocke un **hash** du mot de passe, pas le mot de passe lui-même.
```ts
interface User {
  id: number;
  email: string;
  passwordHash: string; // ✅ Stocker le hash du mot de passe
}
```

---

# Hachage vs Chiffrement

- **Chiffrement (symétrique)** : Un texte **chiffré** avec une clé secrète peut être **déchiffré** avec la même clé.
- **Hachage** : Un texte **haché** avec un algorithme ne peut pas être **déchiffré**. C'est unidirectionnel.
  - On peut cependant comparer un texte avec son hash pour vérifier si c'est le même.

---

# Propriétés d'un bon algorithme de hachage

hash = H(entrée)

- **Taille fixe** (ex: 256 bits)
- **Unicité** : unique pour chaque entrée (collision rare)
- **Irréversibilité** : on ne peut pas retrouver l'entrée à partir du hash
- **Déterminisme** : même entrée → même hash
- **Chaotique** : petit changement dans l'entrée → hash complètement différent
- **Coûteux en calcul** : volontairement lent, pour ralentir les attaques par force brute

---

# Hachage avec Salt

- Sans Salt : deux utilisateurs avec le même mot de passe auront le même hash
- Avec Salt : on ajoute une valeur aléatoire au mot de passe avant de le hacher
  - Stockée dans la BDD avec le hash, pour pouvoir vérifier le mot de passe plus tard
  - Modifie le hash même si deux utilisateurs ont le même mot de passe
- Protège contre les attaques par dictionnaire et rainbow tables
  - Attaque par dictionnaire : tester tous les mots de passe courants
  - Rainbow table : table pré-calculée de mots de passe et leurs hash

---

# bcrypt : Qu'est-ce que c'est ?

Library de hachage avec salt intégré.

```ts
// Installation
npm install bcrypt
npm install --save-dev @types/bcrypt

// Imports
import bcrypt from "bcrypt";
```

BCrypt fait tout automatiquement :
- Génère un salt aléatoire
- Ajoute le salt au mot de passe
- Hache plusieurs fois (le "cost")
  - Coût = nombre de tours de hachage
  - Augmente le temps de calcul pour ralentir les attaques par force brute
  - Chaque +1 double le temps : cost=10 &rightarrow; ~100ms, cost=12 &rightarrow; ~400ms
- Retourne salt + hash en un seul string

---

# Utilisation de bcrypt

```ts
import bcrypt from "bcrypt";

// Hacher un mot de passe
const password = "MySecretPassword123!";
const saltRounds = 10; // coût de hachage
bcrypt.hash(password, saltRounds); // retourne une Promise<string> avec le hash

// Vérifier un mot de passe
const tentativePassword = "MySecretPassword123!";
const storedHash = "$2b$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcg7b3XeKeU6xBJxvxaXUtSQm1S";
bcrypt.compare(tentativePassword, storedHash); // retourne une Promise<boolean> : true si correspond, false sinon
```

---

# Fonction asynchrone
##

Les fonctions de bcrypt prennent du temps de calcul (100ms+). Si on les exécute de manière synchrone, elles bloquent l'exécution et ralentissent le serveur. À la place, on utilise des fonctions **asynchrones**.

JavaScript/TypeScript est **single-threaded** : une seule tâche peut s'exécuter à la fois. Si une fonction prend du temps, elle bloque tout le serveur.

Une fonction asynchrone permet de libérer le thread principal pendant le calcul. Le reste du code peut continuer à s'exécuter pendant que la fonction asynchrone s'exécute en arrière-plan.

---

# Fonction asynchrone : Promises

Une fonction asynchrone retourne une **Promise**.

- Une Promise est un objet qui représente une valeur qui sera disponible dans le futur
- Les Promises peuvent être dans 3 états :
  1. **Pending** : en attente, pas encore résolue
  2. **Fulfilled** : résolue avec succès, valeur disponible
  3. **Rejected** : rejetée avec une erreur, valeur non disponible
- Le résultat d'une Promise est récupéré via des **callbacks**
  - **then()** : pour gérer le succès
  - **catch()** : pour gérer l'erreur

---

# Fonction asynchrone : Promises (exemple)

```ts
// Bcrypt.hash() retourne une Promise
const hashPromise: Promise<string> = bcrypt.hash("MySecretPassword123!", 10);

// Ici, la Promise est en état "pending" (en attente)

hashPromise.then((hash) => {
  console.log("Hash généré :", hash); // Ici, la Promise est "fulfilled" (résolue)
}).catch((error) => {
  console.error("Erreur lors du hachage :", error); // Ici, la Promise est "rejected" (rejetée)
});
```

Une fonction qui fait appel à une Promise ne peut pas retourner directement la valeur. <br>
À la place, elle pourrait retourner une Promise elle-même, ou accepter une fonction callback.

```ts
function hashPassword(plainPassword: string, callback: (hash: string | null) => void) {
  bcrypt.hash(plainPassword, 10).then((hash) => {
    callback(hash);
  }).catch((error) => {
    callback(null);
  });
}
```

---

# Fonction asynchrone : async/await

Autre syntaxe pour gérer les Promises qui est plus lisible et évite les callbacks imbriqués : `async/await`

- Une fonction déclarée avec `async` :
  - Retourne automatiquement une Promise
  - Peut utiliser `await` dans son corps
- Le mot-clé `await` :
  - Peut être utilisé uniquement dans une fonction `async`
  - Permet d'attendre la résolution d'une fonction asynchrone (Promise) avant de continuer l'exécution du code

```ts
async function hashPassword(plainPassword: string): Promise<string> {
    try {
        const hash: string = await bcrypt.hash(plainPassword, 10);
        return hash;
    } catch (error) {
        throw new Error("Erreur lors du hachage");
    }
}
```

---

# bcrypt : Hash et Salt

```ts
import bcrypt from "bcrypt";

// Créer un hash
async function hashPassword(plainPassword: string): Promise<string> {
  const saltRounds = 10;
  const hash = await bcrypt.hash(plainPassword, saltRounds);
  return hash;
}

// Exemple (await doit être dans une fonction async)
const password = "MySecretPassword123!";
const hash = await hashPassword(password);
console.log(hash);
// $2b$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcg7b3XeKeU6xBJxvxaXUtSQm1S
```

---

# bcrypt : Comparaison

```ts
import bcrypt from "bcrypt";

// Comparer le mot de passe saisi avec le hash stocké
async function verifyPassword(plainPassword: string, storedHash: string): Promise<boolean> {
  const isMatch = await bcrypt.compare(plainPassword, storedHash);
  return isMatch;
}

// Exemple (await doit être dans une fonction async)
const userPassword = "MySecretPassword123!";
const storedHash = "$2b$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcg7b3XeKeU6xBJxvxaXUtSQm1S";

const isCorrect = await verifyPassword(userPassword, storedHash);
console.log(isCorrect); // true

const isWrong = await verifyPassword("WrongPassword", storedHash);
console.log(isWrong); // false
```

---

# Route d'Inscription avec bcrypt

```ts
authController.post("/register", async (req: Request, res: Response) => {
    const body: unknown = req.body;
    if (!isCredentialsDTO(body)) return res.sendStatus(400); // Bad Request

    const { email, password } = body;

    // Vérifier email pas déjà utilisé
    const existingUser = UsersService.getByEmail(email);
    if (existingUser) return res.sendStatus(409); // Conflict

    // Hacher le mot de passe
    const passwordHash = await bcrypt.hash(password, 10);

    // Sauvegarder l'utilisateur
    const user = UsersService.create({ email, passwordHash, role: "user" });

    // Créer un token
    const token = generateToken({ id: user.id, email: user.email, role: user.role });

    res.status(201).json({ token });
});
```

---

# Route de Connexion avec bcrypt

```ts
authController.post("/login", async (req: Request, res: Response) => {
    const body: unknown = req.body;
    if (!isCredentialsDTO(body)) return res.sendStatus(400); // Bad Request

    const { email, password } = body;

    const user = UsersService.getByEmail(email);
    if (!user) return res.sendStatus(401); // Unauthorized

    const isPasswordValid = await bcrypt.compare(password, user.passwordHash);
    if (!isPasswordValid) return res.sendStatus(401); // Unauthorized

    // Créer un token
    const token = generateToken({ id: user.id, email: user.email, role: user.role });

    res.json({ token });
});
```

---

# Flux Complet : Register

```http
### 1. Register (créer un compte)
# @name register
POST /auth/register
Content-Type: application/json

{
  "email": "john@gmail.com",
  "password": "MySecurePassword123!"
}

### 

# Serveur hache le mot de passe avec bcrypt
# Stocke : user{ email, passwordHash: "$2b$10$..." }
# Retourne : { token: "eyJh..." }

# 2. Client stocke le token
```

---

# Flux Complet : Login

```http
### 3. Login (se connecter)
# @name login
POST /auth/login
Content-Type: application/json

{
  "email": "john@gmail.com",
  "password": "MySecurePassword123!"
}

###

# Serveur compare password avec bcrypt.compare()
# Génère nouveau token
# Retourne : { token: "eyJh..." }

# 4. Client stocke le token
```

---

# Flux Complet : Accès protégé

```http
### 5. Accès route protégée
GET /recipes
Authorization: {{login.response.body.token}}

# Middleware vérifie le token
# Retourne : [...recipes]

### 6. Suppression protégée
DELETE /recipes/5
Authorization: {{login.response.body.token}}

# Middleware vérifie le token
# Route vérifie autorisation (est-ce l'auteur ?)
# Supprime
```

---

# Récapitulatif Séance 04

- **Hachage vs Chiffrement** — Unidirectionnel vs bidirectionnel
- **Propriétés d'un bon algorithme de hachage** — Taille fixe, unicité, irréversibilité, déterminisme, chaotique, calculatoire
- **Salt** — Valeur aléatoire pour protéger contre les attaques par dictionnaire et rainbow tables
- **bcrypt** — Library de hachage avec salt intégré
- **Fonctions asynchrones** — Promises et async/await pour ne pas bloquer le serveur
- **Route d'inscription** — Hacher le mot de passe avec bcrypt avant de stocker
- **Route de connexion** — Comparer le mot de passe avec bcrypt.compare() pour vérifier l'identité

**Prochaine séance** : Séance 05 — Query Selector (frontend "old school", avant de passer à React).

---

# Exercice filé S04

1. Ajoutez le hachage des mots de passe avec bcrypt dans votre backend
2. Testez entièrement votre backend
3. Vérifiez que les mots de passe ne sont plus stockés en clair dans la BDD