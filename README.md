# Lab 4 - Registration, Login and JWT

| Information | Details | Additional Information |
|---|---|---|
| Student Name | Menda Lakshmanarao | |
| Student Registration Number | 252U1R5100 | Class & Section: 313 AI DE 2B |
| Study Level : UG/PG/Doctorate | UG | Year & Term: 2 & 2 |
| Subject Name | Backend Development Fundamentals | |
| Teacher / Faculty Name | Joel Prashanth D | |
| Name of the Assessment | LAB REPORT - 4 | |
| Date of Submission | 17-11-2026 | |

---

## 1. Objective

The objective of this lab is to implement user registration and login functionality using Node.js, Express.js and SQLite. Passwords are securely hashed using bcrypt before storing them in the database. Upon successful login, a JSON Web Token (JWT) is generated and returned to the client. This lab demonstrates fundamental concepts of authentication, password security and stateless token-based authorization.

---

## 2. Technologies Used

- **Node.js** - JavaScript runtime for server-side development
- **Express.js** - Web framework for building REST APIs
- **SQLite** - Lightweight relational database
- **bcrypt** - Password hashing library
- **jsonwebtoken** - Library for creating and verifying JWT tokens
- **Postman** - API testing tool

---

## 3. Project Structure

```text
lab-04/
├── app.js
├── setupDb.js
├── users.db
├── package.json
├── .gitignore
├── README.md
└── screenshots/
    ├── register_success.png
    ├── register_duplicate.png
    ├── login_success.png
    ├── login_unknown_user.png
    ├── login_wrong_password.png
    ├── database_users.png
    ├── password_hash_verification.png
    └── jwt_token_verification.png
```

---

## 4. Database Structure

The SQLite database `users.db` contains a single table named `user`.

| Column | Type | Constraints |
|---|---|---|
| id | INTEGER | PRIMARY KEY AUTOINCREMENT |
| username | TEXT | UNIQUE NOT NULL |
| name | TEXT | NOT NULL |
| password | TEXT | NOT NULL (stores bcrypt hash) |

The `username` column is unique to prevent duplicate registrations. The `password` column stores only bcrypt-hashed passwords, never plain text.

---

## 5. How to Run

```bash
cd lab-04
npm install
node setupDb.js
npm start
```

The server runs at:

```
http://localhost:3000
```

> **Note:** The JWT secret key is stored as a constant inside `app.js` for this lab. In Lab 8, the secret will be moved to a `.env` file using the `dotenv` package for better security.

---

## 6. API Endpoints

| Method | Endpoint | Description | Status |
|---|---|---|---|
| POST | /register/ | Register a new user | 201 |
| POST | /register/ | Duplicate username | 400 |
| POST | /login/ | Login successfully | 200 |
| POST | /login/ | Unknown username | 400 |
| POST | /login/ | Incorrect password | 400 |

### POST /register/

**Request Body:**

```json
{
  "username": "lakshman",
  "name": "Lakshman Rao",
  "password": "123456"
}
```

**Success Response (201):**

```json
{
  "message": "User Created Successfully"
}
```

**Duplicate Username Response (400):**

```json
{
  "message": "User already exists"
}
```

### POST /login/

**Request Body:**

```json
{
  "username": "lakshman",
  "password": "123456"
}
```

**Success Response (200):**

```json
{
  "jwt_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Unknown Username Response (400):**

```json
{
  "message": "Invalid User"
}
```

**Incorrect Password Response (400):**

```json
{
  "message": "Invalid Password"
}
```

---

## 7. Complete Source Code

### app.js

```javascript
const express = require("express");
const bcrypt = require("bcrypt");
const jwt = require("jsonwebtoken");
const { initializeDatabase } = require("./setupDb");

const app = express();
const PORT = 3000;
const JWT_SECRET = "my_secret_jwt_token_key";

app.use(express.json());

let db;

const startServer = async () => {
  db = await initializeDatabase();

  app.post("/register/", async (request, response) => {
    const { username, name, password } = request.body;

    if (!username || !name || !password) {
      return response
        .status(400)
        .json({ message: "All fields are required" });
    }

    const existingUser = await db.get(
      "SELECT * FROM user WHERE username = ?",
      [username]
    );

    if (existingUser) {
      return response
        .status(400)
        .json({ message: "User already exists" });
    }

    const hashedPassword = await bcrypt.hash(password, 10);

    await db.run(
      "INSERT INTO user (username, name, password) VALUES (?, ?, ?)",
      [username, name, hashedPassword]
    );

    response
      .status(201)
      .json({ message: "User Created Successfully" });
  });

  app.post("/login/", async (request, response) => {
    const { username, password } = request.body;

    if (!username || !password) {
      return response
        .status(400)
        .json({ message: "Username and password are required" });
    }

    const user = await db.get(
      "SELECT * FROM user WHERE username = ?",
      [username]
    );

    if (!user) {
      return response
        .status(400)
        .json({ message: "Invalid User" });
    }

    const isPasswordValid = await bcrypt.compare(password, user.password);

    if (!isPasswordValid) {
      return response
        .status(400)
        .json({ message: "Invalid Password" });
    }

    const payload = { username: user.username };
    const jwtToken = jwt.sign(payload, JWT_SECRET);

    response
      .status(200)
      .json({ jwt_token: jwtToken });
  });

  app.listen(PORT, () => {
    console.log(`Server running at http://localhost:${PORT}/`);
  });
};

startServer();
```

### setupDb.js

```javascript
const { open } = require("sqlite");
const sqlite3 = require("sqlite3");
const path = require("path");

const dbPath = path.join(__dirname, "users.db");

const initializeDatabase = async () => {
  const db = await open({
    filename: dbPath,
    driver: sqlite3.Database,
  });

  await db.exec(`
    CREATE TABLE IF NOT EXISTS user (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      username TEXT UNIQUE NOT NULL,
      name TEXT NOT NULL,
      password TEXT NOT NULL
    )
  `);

  console.log("Database initialized successfully");
  return db;
};

if (require.main === module) {
  initializeDatabase()
    .then(() => {
      console.log("Setup complete");
      process.exit(0);
    })
    .catch((error) => {
      console.error("Database setup failed:", error.message);
      process.exit(1);
    });
}

module.exports = { initializeDatabase };
```

### package.json

```json
{
  "name": "lab-04",
  "version": "1.0.0",
  "description": "Lab 4 - Registration, Login and JWT",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "setup": "node setupDb.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "sqlite": "^5.1.1",
    "sqlite3": "^5.1.7",
    "bcrypt": "^5.1.1",
    "jsonwebtoken": "^9.0.2"
  }
}
```

---

## 8. Postman Screenshots

### 1. Registration Success (HTTP 201)

![Registration Success](screenshots/register_success.png)

### 2. Registration Duplicate Username (HTTP 400)

![Registration Duplicate](screenshots/register_duplicate.png)

### 3. Login Success (HTTP 200)

![Login Success](screenshots/login_success.png)

### 4. Login Unknown User (HTTP 400)

![Login Unknown User](screenshots/login_unknown_user.png)

### 5. Login Wrong Password (HTTP 400)

![Login Wrong Password](screenshots/login_wrong_password.png)

### 6. Database Users View

![Database Users](screenshots/database_users.png)

### 7. Password Hash Verification

![Password Hash Verification](screenshots/password_hash_verification.png)

### 8. JWT Token Verification

![JWT Token Verification](screenshots/jwt_token_verification.png)

---

## 9. JWT Explanation

### What is JWT?

JSON Web Token (JWT) is an open standard for securely transmitting information between parties as a JSON object. It is compact, self-contained and commonly used for authentication and authorization in web applications.

### What is a Header?

The header contains metadata about the token. It specifies the token type (JWT) and the signing algorithm used (such as HS256).

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

### What is a Payload?

The payload contains the claims or data stored inside the token. In this lab, the payload contains only the username of the authenticated user.

```json
{
  "username": "lakshman",
  "iat": 1700000000
}
```

### What is a Signature?

The signature is created by combining the encoded header, encoded payload and a secret key using the specified algorithm. It ensures the token has not been tampered with.

```
HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)
```

### Why should passwords never be included in JWT payloads?

JWT payloads are only Base64-encoded, not encrypted. Anyone who obtains the token can decode the payload and read its contents. Including passwords would expose them to anyone who intercepts the token. Only non-sensitive identification data like usernames should be stored in the payload.

---

## 10. Viva Questions and Answers

**1. Why do we use POST for login?**

We use POST because login sends sensitive data like username and password. POST sends data in the request body, which is not visible in the URL. GET would expose credentials in the URL and browser history.

**2. What is the difference between hashing and encryption?**

Hashing is a one-way process that converts data into a fixed-length string. It cannot be reversed. Encryption is a two-way process where data can be encrypted and decrypted using a key. Passwords should always be hashed, not encrypted.

**3. What does `jwt.sign()` need?**

`jwt.sign()` requires two arguments: a payload object containing the data to include in the token, and a secret key string used to sign the token. An optional third argument can specify options like expiration time.

**4. What information should never be stored in the JWT payload?**

Passwords, password hashes, credit card numbers, personal identification numbers and any other sensitive information should never be stored in the JWT payload because it can be easily decoded by anyone.

**5. Why do we use bcrypt?**

Bcrypt is a password hashing algorithm specifically designed for securely hashing passwords. It includes a salt to protect against rainbow table attacks and has a configurable cost factor to make brute force attacks slower.

**6. What is the purpose of bcrypt.compare()?**

`bcrypt.compare()` compares a plain text password with a bcrypt hash. It hashes the input password with the same salt used in the stored hash and checks if they match. It returns true if the password is correct and false otherwise.

**7. What is a unique username?**

A unique username means no two users in the database can have the same username. The database enforces this using the UNIQUE constraint on the username column, preventing duplicate registrations.

**8. What is stateless authentication?**

Stateless authentication means the server does not store session information. Instead, the client sends a token (like JWT) with each request. The server verifies the token without maintaining session state, making the system scalable.

**9. Why do we use SQLite?**

SQLite is a lightweight, serverless, file-based database that requires no separate installation or configuration. It is ideal for development, learning and small applications. The entire database is stored in a single file.

**10. What is the purpose of a JWT signature?**

The JWT signature verifies that the token has not been tampered with. It is created using the header, payload and a secret key. When the server receives a token, it recalculates the signature to ensure the token is authentic and unmodified.

---

## 11. Problems Faced and Solutions

| Problem | Solution |
|---|---|
| **Incorrect database import path** | Used `require("./setupDb")` with the correct relative path and exported the `initializeDatabase` function properly. |
| **Missing `await` during database initialization** | Added `await` before all database operations (`db.get()`, `db.run()`, `db.exec()`) to ensure they complete before proceeding. |
| **bcrypt installation errors** | Ran `npm install bcrypt` with administrator privileges. Ensured Python and C++ build tools were available for native module compilation. |
| **Port already in use (EADDRINUSE)** | Stopped any existing Node.js processes using port 3000 before starting the server. Used `npx kill-port 3000` or `taskkill /F /PID <pid>` on Windows. |
| **Duplicate username errors** | Added a check using `SELECT * FROM user WHERE username = ?` before attempting to insert a new user. This provides a clear error message instead of a database constraint error. |
| **Invalid JWT secret or token generation issues** | Ensured the JWT secret is defined as a constant string before calling `jwt.sign()`. Verified that only the username is included in the payload. |
| **Incorrect SQL table or column names** | Double-checked the CREATE TABLE statement to ensure column names match exactly what is used in INSERT and SELECT queries. |

---

## 12. Conclusion

In this lab, a complete user registration and login system was implemented using Node.js, Express.js and SQLite. Passwords are securely hashed using bcrypt before being stored in the database. The login endpoint verifies credentials using `bcrypt.compare()` and generates a JWT token upon successful authentication. The JWT payload contains only the username, ensuring sensitive data is never exposed. All API endpoints return appropriate HTTP status codes and consistent JSON responses. This lab provided practical experience with password security, token-based authentication and RESTful API development.
