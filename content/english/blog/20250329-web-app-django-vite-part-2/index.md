---
title: "Building a Modern Web App with Django, Vite & shadcn/ui (Part 2) – Authentication"
meta_title: ""
description: "this is meta description"
# date: 2025-03-31T09:00:00Z
# publishedDate: 2025-03-31T09:00:00Z
# thumbnail: "05-sidebar-menu.gif"
categories: ["Web Development"]
author: "Taufiq Ibrahim"
tags: ["react", "vite", "shadcn", "tailwind"]
draft: true
---

This is the 2nd article in a series on building a modern web application using the following technologies:
- Frontend service built using [React](https://react.dev/) + [Vite](https://vite.dev/)
- UI components using [shadcn/ui](https://ui.shadcn.com/)
- Backend service using [Django](https://www.djangoproject.com/)

The final code of this article available as a branch on https://github.com/tibrahimdev/web-app-django-vite-shadcn/tree/2-authentication

## The Goals
- A single protected page on `/dashboard` path showing a colapsible sidebar. Unauthenticated user shall be redirected to `/login` to perform login.
- Other path will return 404 page
- Create great UI with the help of `shadcn/ui` components

In the [last article](/blog/20250323-web-app-django-vite-part-1/), we successfully built the dashboard page for our web application. One crucial aspect that often gets overlooked in the early stages of development is authentication. However, laying the groundwork early ensures we can securely extend and refine it as our app evolves.

To keep our authentication system flexible and maintainable, we will start by building an agnostic authentication adapter. This will serve as a central mechanism for managing authentication state on the frontend, allowing us to adapt it to different authentication strategies over time.

Our approach will be incremental:
1. Develop auth adapter – Initially, we’ll structure it to handle authentication logic without being tied to any backend.
2. Implement a simple hardcoded user/password system – Before integrating with Django, we’ll mock authentication to test our setup and extend accordingly.
3. Connect to Django’s authentication system – Finally, we’ll replace the hardcoded logic with a real API-backed authentication flow.

By following this approach, we ensure our authentication system remains modular and adaptable, making future improvements seamless. Let’s get started! 🚀

---

## Initializing Auth Adapter
### Creating Auth Adapter Interface

Let's start by working on the Auth Adapter. Create a new directory `src/auth` and create a file `src/auth/adapters/AuthAdapter.ts`. We start by defining `AuthAdapter`, which outlines the required authentication methods:

```ts
// src/auth/adapters/AuthAdapter.ts
export type UserPasswordLoginCredentials = {email: string, password: string};
export type LoginCredentials = UserPasswordLoginCredentials;

export type SimpleLoginResponse = {token: string}
export type LoginResponse = SimpleLoginResponse;

export interface AuthAdapter {
    login: (credentials: LoginCredentials) => Promise<LoginResponse>
    logout: () => void;
    getUser: () => Promise<any>;
    refreshToken?: () => Promise<string>;
}
```

Above code defines a type-safe authentication adapter for handling login operations in a structured and extensible way.
- `UserPasswordLoginCredentials`: Represents a basic email-password login structure.
- `LoginCredentials`: Currently set as `UserPasswordLoginCredentials`, but can be extended in the future (e.g., OTP-based login, SSO, etc.).
- `SimpleLoginResponse`: Represents the minimal response format, containing only a token.
- `LoginResponse`: Currently, it only supports `SimpleLoginResponse` type, but this can be expanded to include additional information (e.g., user details, expiration, error messages, etc).
- We can later extend the interface for user fetching, token refresh, and error handling. For, now, we will focus on `login` method.

### Creating User + Password Auth Adapter
To keep things simple at the start, we create a dummy UserPasswordAdapter that returns default values. This allows us to test our implementation without requiring a real backend.

Create a new file `src/auth/adapters/UserPasswordAuthAdapter.ts`:
```ts
// src/auth/adapters/UserPasswordAuthAdapter.ts
import { AuthAdapter, LoginCredentials, LoginResponse } from "./AuthAdapter";

export class UserPasswordAuthAdapter implements AuthAdapter {
  async login(credentials: LoginCredentials): Promise<LoginResponse> {
    return { token: "dummytoken" };
  };

  logout() { };

  async getUser(): Promise<any> { };

  async refreshToken(): Promise<any> { };
}
```
The `UserPasswordAuthAdapter` implements the `AuthAdapter` interface and provides the following methods:

- `login(credentials: LoginCredentials)` – This method simulates a successful login by returning a dummy token (`"dummytoken"`). In a real implementation, this would verify the user’s credentials against a backend.
- `logout()` – A placeholder method for handling logout logic.
- `getUser()` – Currently empty, but in a real scenario, it would fetch user details from the authentication provider.
- `refreshToken()` – A placeholder for future token refresh functionality.

This adapter allows us to test the authentication mechanism without relying on a backend. As we progress, we will replace this with a real authentication system.

### Implement the Authentication Context & AuthProvider
We will use [React Context](https://react.dev/learn/passing-data-deeply-with-context) to provide authentication state to the entire app. This allows us to manage authentication state globally and keep our authentication logic centralized.

We will implement this section in a new file `src/auth/AuthProvider.tsx`. Let's create the Context.
```tsx
// src/auth/AuthProvider.tsx
import { createContext } from "react";
import { LoginResponse } from "./adapters/AuthAdapter";

interface AuthContextType {
    user: any;
    login: (credentials: any) => Promise<LoginResponse>;
    logout: () => void;  
}

const AuthContext = createContext<AuthContextType | null>(null);
```
To explain:
- `user: any` – Holds authenticated user information.
- `login(credentials: any): Promise<LoginResponse>` – Handles user login, delegating authentication to the adapter.
- `logout(): void` – Logs the user out.
- Since the authentication mechanism may change, we initialize the context as null, ensuring it is always provided within an `AuthProvider`.

To use the context effectively, we wrap our application in an `AuthProvider` component. The code should be updated as follows:

```tsx
// src/auth/AuthProvider.tsx
import { createContext, ReactNode } from "react";
import { AuthAdapter, LoginResponse } from "./adapters/AuthAdapter";

interface AuthContextType {
  user: any;
  login: (credentials: any) => Promise<LoginResponse>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextType | null>(null);

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error("useAuth must be used within an AuthProvider");
  return context;
}

export const AuthProvider: React.FC<{ adapter: AuthAdapter, children: ReactNode }> = ({ adapter, children }) => {
  console.log("AuthProvider used", adapter)
  const login = async (credentials: any): Promise<LoginResponse> => {
    const response = await adapter.login(credentials)
    return response
  }

  return <AuthContext.Provider value={{ user: null, login, logout: adapter.logout }} >
    {children}
  </AuthContext.Provider>
}

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) throw new Error("useAuth must be used within an AuthProvider");
  return context;
}
```
TODO: explain...

### Wrap Entire App Inside Auth Provider 
Next, we will wrap our entire app inside the `AuthProvider` so that authentication state is available across all routes. The best place to do this is in `main.tsx`, ensuring all pages and components inside `RouterProvider` can access authentication.

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import './index.css'
import { RouterProvider } from 'react-router'
import { router } from './router'
import { AuthProvider } from './auth/AuthProvider'                                  // Added
import { UserPasswordAuthAdapter } from './auth/adapters/UserPasswordAuthAdapter'   // Added

const authAdapter = new UserPasswordAuthAdapter();                                  // Added

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <AuthProvider adapter={authAdapter}>    {/*Added*/}
      <RouterProvider router={router} />
    </AuthProvider>                         {/*Added*/}
  </StrictMode>,
)
```

Now, let's head to http://localhost:5173/dashboard. Everything will look the same, except that we're now seeing **AuthProvider used** is printed on the browser **Console**. It means that the `AuthProvider` is already called.

{{< image src="01-console.png" webp="false" width="822px" >}}

Now, you're asking, why only print to console? The answer is because we haven't define which route is protected (meaning requires user to be logged in) versus public routes, which can be accessed by non logged in users.

---

## Configure Public vs Protected Routes
Let's go back to `src/router.tsx`.

Now that we have a working authentication system, we need to ensure that only authenticated users can access certain parts of our application. This is where **Protected Routes** concept comes in.

In React Router, we can use a wrapper component to check authentication status before rendering a route. If the user is not logged in, they will be redirected to the login page.

```tsx
// src/router.tsx
import { createBrowserRouter, Navigate, Outlet } from "react-router";
import { useAuth } from "./auth/AuthProvider";

const ProtectedRoute = () => {
  const { user } = useAuth();
  return user ? <Outlet /> : <Navigate to="/login" replace />;
}

export const router = createBrowserRouter([
  {
    path: "dashboard",
    element: <ProtectedRoute />,
    children: [
      {
        path: "",
        lazy: async () => ({
          Component: (await import("@/pages/dashboard")).default
        }),
      }
    ]
  }
]);
```

- `useAuth` provides the current authentication state.
- Inside the `ProtectedRoute`, we define if `user` exists, then render the current page (`Outlet`). Otherwise, redirect user to `/login`.
- Inside the `createBrowserRouter`:
    - We wrap `/dashboard` inside `ProtectedRoute`, so only logged-in users can access it.
    - Outlet ensures that child routes, such as `/dashboard/stats`, also follow the same protection.
    - If the user is not authenticated, they will be redirected to the login page.

Let's check it on http://localhost:5173/dashboard.

{{< image src="02-login-404.png" webp="false" width="822px" >}}

**Why 404?**


We haven't created the Login page. Let's create one.

## Creating Login Page

Create new file `src/pages/auth/login/index.tsx`.

```tsx
// src/pages/auth/login/index.tsx
export default function Page() {
  return (
    <div>
      <h1>Login</h1>
    </div>
  )
}
```

If we try again opening http://localhost:5173 right now, it will still be the same 404.

Why? Because we haven't define the router.

So yeah, let's add that to the `src/router.tsx`.

```tsx
// src/router.tsx

// existing code...

export const router = createBrowserRouter([
  {
    path: "/",
    children: [
      {
        path: "login",
        lazy: async () => ({
          Component: (await import("@/pages/auth/login")).default
        }),
      }
    ]
  },
  {
    path: "dashboard",
    element: <ProtectedRoute />,
    children: [
      {
        path: "",
        lazy: async () => ({
          Component: (await import("@/pages/dashboard")).default
        }),
      }
    ]
  }
]);

```


---
