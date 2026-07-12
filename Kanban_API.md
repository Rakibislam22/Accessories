# API Documentation

This document provides a complete guide for using the backend API of the
**Kanban** and **Image Annotation** project.

## Base URL

``` text
http://127.0.0.1:8000/api/
```

------------------------------------------------------------------------

# Authentication

This API uses **JWT (JSON Web Token)** authentication.

All protected endpoints require an **Access Token** in the
`Authorization` header. Only **Register**, **Login**, and **Refresh
Token** endpoints are public.

### Header Format

``` text
Authorization: Bearer <your_access_token>
```

------------------------------------------------------------------------

# API Endpoints

## 1. User Registration

Create a new user account.

-   **Endpoint:** `POST /api/auth/register/`
-   **Method:** `POST`
-   **Authentication:** Public

### Request Body

``` json
{
  "username": "newuser",
  "first_name": "New",
  "last_name": "User",
  "email": "newuser@example.com",
  "password": "a-strong-password",
  "password2": "a-strong-password"
}
```

### Success Response (201 Created)

``` json
{
  "username": "newuser",
  "email": "newuser@example.com",
  "first_name": "New",
  "last_name": "User"
}
```

### Error Response (400 Bad Request)

``` json
{
  "username": [
    "A user with that username already exists."
  ],
  "password": [
    "Password fields didn't match."
  ]
}
```

------------------------------------------------------------------------

## 2. User Login

Authenticate a user and receive **Access** and **Refresh** JWT tokens.

-   **Endpoint:** `POST /api/auth/login/`
-   **Method:** `POST`
-   **Authentication:** Public

### Request Body

``` json
{
  "username": "newuser",
  "password": "a-strong-password"
}
```

### Success Response (200 OK)

``` json
{
  "refresh": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

> **Note:** Decoding the Access Token provides user information such as
> `username`, `email`, `first_name`, and `last_name`, which can be used
> by the frontend to display the logged-in user's information.

### Error Response (401 Unauthorized)

``` json
{
  "detail": "No active account found with the given credentials"
}
```

------------------------------------------------------------------------

## 3. Refresh Access Token

Generate a new Access Token using a valid Refresh Token.

-   **Endpoint:** `POST /api/auth/login/refresh/`
-   **Method:** `POST`
-   **Authentication:** Public

### Request Body

``` json
{
  "refresh": "<your_refresh_token>"
}
```

### Success Response (200 OK)

``` json
{
  "access": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

------------------------------------------------------------------------

## 4. Get User Profile

Retrieve the authenticated user's profile.

-   **Endpoint:** `GET /api/auth/profile/`
-   **Method:** `GET`
-   **Authentication:** Required

### Headers

``` text
Authorization: Bearer <your_access_token>
```

### Success Response (200 OK)

``` json
{
  "id": 1,
  "username": "newuser",
  "email": "newuser@example.com",
  "first_name": "New",
  "last_name": "User"
}
```

### Error Response (401 Unauthorized)

``` json
{
  "detail": "Authentication credentials were not provided."
}
```

------------------------------------------------------------------------

# Other Protected Endpoints

All other APIs (such as **Tasks** and **Annotations**) require a valid
Access Token in the request header.

## Example: Create a New Task

-   **Endpoint:** `POST /api/tasks/`
-   **Method:** `POST`
-   **Authentication:** Required

### Headers

``` text
Authorization: Bearer <your_access_token>
```

### Request Body

``` json
{
  "title": "My new task",
  "description": "Details about the task."
}
```
