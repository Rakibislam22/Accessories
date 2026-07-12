# API Documentation

This document provides a comprehensive guide for using the backend API
of the **Kanban Board** and **Image Annotation** project.

## Base URL

``` text
http://127.0.0.1:8000/api/
```

## Authentication

This API uses JWT authentication.

All protected endpoints require the following header:

``` text
Authorization: Bearer <your_access_token>
```

Public endpoints: - Register - Login - Refresh Token

------------------------------------------------------------------------

# 1. Authentication API

**Base Path:** `/api/auth/`

## 1.1 User Registration

**Endpoint:** `POST /api/auth/register/`

``` json
{
  "username":"newuser",
  "first_name":"New",
  "last_name":"User",
  "email":"newuser@example.com",
  "password":"a-strong-password",
  "password2":"a-strong-password"
}
```

Response (201)

``` json
{
  "message":"User registered successfully."
}
```

## 1.2 User Login

**Endpoint:** `POST /api/auth/login/`

``` json
{
  "username":"newuser",
  "password":"a-strong-password"
}
```

Response (200)

``` json
{
  "refresh":"...",
  "access":"...",
  "user":{
    "id":1,
    "username":"newuser",
    "email":"newuser@example.com",
    "first_name":"New",
    "last_name":"User"
  }
}
```

## 1.3 Refresh Token

**Endpoint:** `POST /api/auth/login/refresh/`

``` json
{
  "refresh":"<your_refresh_token>"
}
```

Response (200)

``` json
{
  "access":"..."
}
```

## 1.4 Get User Profile

**Endpoint:** `GET /api/auth/profile/`

Response (200)

``` json
{
  "id":1,
  "username":"newuser",
  "email":"newuser@example.com",
  "first_name":"New",
  "last_name":"User"
}
```

------------------------------------------------------------------------

# 2. Kanban Board API

**Base Path:** `/api/tasks/`

## 2.1 List All Tasks

**Endpoint:** `GET /api/tasks/`

Returns:

``` json
[
  {
    "id":1,
    "title":"Design the new login page",
    "description":"Create a modern and user-friendly design.",
    "status":"To Do",
    "priority":"High",
    "due_date":"2026-08-15T18:00:00Z",
    "owner":1
  }
]
```

## 2.2 Create Task

**Endpoint:** `POST /api/tasks/`

``` json
{
  "title":"Setup frontend project",
  "description":"Use React with TypeScript.",
  "status":"To Do",
  "priority":"High",
  "due_date":"2026-08-12T10:00:00Z"
}
```

Returns the created task.

## 2.3 Retrieve / Update / Delete Task

Endpoint: `/api/tasks/<id>/`

  Operation   Method
  ----------- -------------------------
  Retrieve    GET
  Update      PUT / PATCH
  Delete      DELETE (204 No Content)

------------------------------------------------------------------------

# 3. Image Annotation API

**Base Path:** `/api/annotations/`

## 3.1 List All Annotations

**Endpoint:** `GET /api/annotations/`

``` json
[
  {
    "id":1,
    "image":"http://127.0.0.1:8000/media/images/example.jpg",
    "shapes":[
      {
        "type":"polygon",
        "points":[[10,20],[50,20],[30,60]]
      }
    ],
    "owner":1
  }
]
```

## 3.2 Create Annotation

**Endpoint:** `POST /api/annotations/`

Content-Type: `multipart/form-data`

  Field    Description
  -------- -------------
  image    Image file
  shapes   JSON string

Example:

``` json
"[{\"type\":\"polygon\",\"points\":[[10,20],[50,20],[30,60]]}]"
```

Returns the created annotation.

## 3.3 Retrieve / Update / Delete Annotation

Endpoint: `/api/annotations/<id>/`

  Operation   Method
  ----------- -------------------------
  Retrieve    GET
  Update      PUT / PATCH
  Delete      DELETE (204 No Content)
