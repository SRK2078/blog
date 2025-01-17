---
title: "Single Sign-On (SSO) Implementation Using Passport.js in TypeScript"
seoTitle: "SSO with Passport.js and TypeScript"
seoDescription: "Learn how to implement Single Sign-On (SSO) in Node.js using Passport.js and TypeScript for enhanced user experience and secure authentication"
datePublished: Fri Jan 17 2025 18:15:35 GMT+0000 (Coordinated Universal Time)
cuid: cm612yexd000609jxabrp9t46
slug: single-sign-on-sso-implementation-using-passportjs-in-typescript
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1737137497543/4c9f6a01-d26c-439c-a276-50251cdae746.png
tags: nodejs, typescript, passportjs, sso-authentication

---

Implementing Single Sign-On (SSO) in your application enhances user experience by allowing seamless access across multiple platforms with a single set of credentials. In Node.js applications, the Passport middleware simplifies this process by providing a flexible and modular authentication framework. This guide focuses on using Passport.js with TypeScript to maintain type safety and robust application architecture.

## **What Is Passport.js?**

Passport.js is an authentication middleware for Node.js that supports various strategies, including social logins (e.g., Google, GitHub) and traditional username-password mechanisms. It is modular, allowing developers to choose specific strategies as needed.

For more details, visit [Passport.js documentation](https://www.passportjs.org/docs).

---

## **Steps to Implement SSO in TypeScript**

### **Install Required Dependencies**

Begin by installing the necessary packages:

```bash
npm install passport passport-google-oauth20 passport-github2 express-session @types/passport @types/express-session
```

### **Initialize Passport**

Configure Passport and session handling in your application.

#### **Code Implementation:**

```typescript
import express, { Application, Request, Response, NextFunction } from "express";
import session from "express-session";
import passport from "passport";

const app: Application = express();

app.use(
  session({
    secret: "your_secret_key",
    resave: false,
    saveUninitialized: true,
  })
);

app.use(passport.initialize());
app.use(passport.session());
```

---

### **Configure Authentication Strategies**

#### **Google Strategy**

To authenticate users via Google, configure the Google strategy:

```typescript
import { Strategy as GoogleStrategy } from "passport-google-oauth20";

passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      callbackURL: "http://localhost:3000/auth/google/callback",
    },
    async (
      accessToken: string,
      refreshToken: string,
      profile: any,
      done: Function
    ) => {
      try {
        // Replace with your DB logic
        const user = await findOrCreateUser({ googleId: profile.id });
        return done(null, user);
      } catch (error) {
        return done(error);
      }
    }
  )
);
```

---

#### **GitHub Strategy**

Similarly, configure the GitHub strategy:

```typescript
import { Strategy as GitHubStrategy } from "passport-github2";

passport.use(
  new GitHubStrategy(
    {
      clientID: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      callbackURL: "http://localhost:3000/auth/github/callback",
    },
    async (
      accessToken: string,
      refreshToken: string,
      profile: any,
      done: Function
    ) => {
      try {
        // Replace with your DB logic
        const user = await findOrCreateUser({ githubId: profile.id });
        return done(null, user);
      } catch (error) {
        return done(error);
      }
    }
  )
);
```

### **Local Strategy**

For applications that require traditional username and password authentication, Passport.js provides the Local Strategy:

```typescript
import { Strategy as LocalStrategy } from "passport-local";

passport.use(
  new LocalStrategy((username, password, done) => {
    // Replace with your user retrieval and password verification logic
    const user = getUserByUsername(username);
    if (!user || !verifyPassword(user, password)) {
      return done(null, false, { message: "Invalid credentials" });
    }
    return done(null, user);
  })
);
```

By utilizing the Local Strategy, you can manage authentication within your application without relying on external providers. This approach is beneficial when you prefer to handle user credentials directly.

---

### **Serialization and Deserialization**

Handle session serialization and deserialization to maintain user sessions.

#### **Code Implementation:**

```typescript
passport.serializeUser((user: any, done: Function) => {
  done(null, user.id);
});

passport.deserializeUser(async (id: string, done: Function) => {
  try {
    const user = await findUserById(id); // Replace with your DB logic
    done(null, user);
  } catch (error) {
    done(error);
  }
});
```

---

### **Define Routes**

Set up routes for initiating authentication and handling callbacks.

#### **Google Authentication Routes:**

```typescript
app.get(
  "/auth/google",
  passport.authenticate("google", { scope: ["profile", "email"] })
);

app.get(
  "/auth/google/callback",
  passport.authenticate("google", { failureRedirect: "/" }),
  (req: Request, res: Response) => {
    res.redirect("/dashboard");
  }
);
```

---

#### **GitHub Authentication Routes:**

```typescript
app.get(
  "/auth/github",
  passport.authenticate("github", { scope: ["user:email"] })
);

app.get(
  "/auth/github/callback",
  passport.authenticate("github", { failureRedirect: "/" }),
  (req: Request, res: Response) => {
    res.redirect("/dashboard");
  }
);
```

### **Local Authentication Routes:**

```typescript
app.post(
  "/login",
  passport.authenticate("local", { failureRedirect: "/" }),
  (req: Request, res: Response) => {
    res.redirect("/dashboard");
  }
);
```

---

### **Handle User Logout**

Allow users to log out and securely end their sessions.

#### **Code Implementation:**

```typescript
app.get("/logout", (req: Request, res: Response, next: NextFunction) => {
  req.logout((err) => {
    if (err) {
      return next(err);
    }
    res.redirect("/");
  });
});
```

---

### **Handling User Authentication**

Protect routes that require authentication by checking if the user is authenticated, implementing a middleware function.

#### **Code Implementation:**

```typescript
function isAuthenticated(req: Request, res: Response, next: NextFunction) {
  if (req.isAuthenticated()) {
    return next();
  }
  res.redirect("/");
}
```

Now, you can use the `isAuthenticated` middleware to protect routes that require authentication:

```typescript
app.get("/dashboard", isAuthenticated, (req: Request, res: Response) => {
  res.send("Dashboard", { user: req.user });
});
```

---

### **Environment Variables**

Store sensitive credentials in an `.env` file:

```
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
```

Use a library like `dotenv` to load them:

```typescript
import dotenv from "dotenv";
dotenv.config();
```

---

### **Replace Placeholder Functions**

Replace the placeholder functions `findOrCreateUser` and `findUserById` with actual implementations for interacting with your database:

```typescript
async function findOrCreateUser(query: object): Promise<any> {
  // Example: Interact with database to find or create user
  return {};
}

async function findUserById(id: string): Promise<any> {
  // Example: Fetch user by ID from database
  return {};
}
```

---

## **Conclusion**

By leveraging Passport.js in TypeScript, you can create a robust SSO system with type safety, flexibility, and support for multiple authentication providers like Google and GitHub. For additional configurations, refer to the [Passport.js documentation](https://www.passportjs.org/docs/). 

*Implementing SSO not only enhances user convenience but also strengthens security by centralizing authentication processes.*
