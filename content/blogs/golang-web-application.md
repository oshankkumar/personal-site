---
title: "Structuring SOLID Go Web Applications: A Practical Guide with Code Examples"
date: 2022-12-09T19:53:33+05:30
draft: false
author: "Oshank Kumar"
github_link: "https://github.com/oshankkumar/sockshop"
tags:
  - Web Application
  - Golang
  - SOLID Principles
image: /images/blogs/go_web_app.png
description: ""
toc: true
---

---


# Introduction

With over 8 years of experience in Go development, I've had the privilege of contributing to various open-source projects. Over time, I've refined my Go coding practices, striving to create code that's not just functional but also adheres to industry-standard design principles. In this blog post, I'll embark on a journey to explain how to develop a robust Go web application following the SOLID principles - a set of guidelines that emphasize clean, maintainable, and modular code.
To illustrate these principles in action, we'll take a real-world example: a microservice central to e-commerce applications - the user account management service. This microservice is responsible for providing RESTful APIs for tasks like user account creation, management, and login.
In this post, I'll demonstrate how to write this microservice step by step, adhering to SOLID principles, and create a codebase that's not only efficient but also highly maintainable. The entire source code for this service can be found in my GitHub repository at https://github.com/oshankkumar/sockshop.
Let's delve into the world of SOLID principles in Go, unraveling how they can elevate your web application development skills to the next level. In a follow-up post, I will delve deeper into SOLID principles and their application in this demo project.

---

## Package Layout
In the world of Go, clean code starts with a clean package layout.

### /api: Defining Contracts
Our /api package is designed to define contracts, such as request and response schemas. This package should not contain application logic or dependencies. Additionally It can also contain interface definitions. Which will just an abstraction to your business logic. It should not contain any concrete type which implements those interface.

```go
package api

import (
 "github.com/google/uuid"
)

type User struct {
 FirstName string    `json:"firstName"`
 LastName  string    `json:"lastName"`
 Username  string    `json:"username"`
 Password  string    `json:"-"`
 Email     string    `json:"email"`
 ID        uuid.UUID `json:"id"`
 Links     Links     `json:"_links"`
}

type UserService interface {
 Login(ctx context.Context, username, password string) (*User, error)
 Register(ctx context.Context, user User) (uuid.UUID, error)
 GetUser(ctx context.Context, id string) (*User, error)
}
```

In the above example, We've only defined the User type which will be used as a http request/response schema. In addition to that we've also define a UserService inerface which is a abstraction to the business logic. The actual implementation of this interface isn't defined here.

### /cmd: Application Entry Points
This will contain the main package(s) for this project. The directory name for each application in cmd should match the name of executable you want. You should not put a lot of code in main package. You should only initialize the app dependencies in here and inject it to higher level modules/components.

### /internal/domain: Defining Domain Entities
The /internal/domain package is where you define your application's domain-related information. It should contain application domain types and related interfaces, but no concrete implementations. This separation provides an abstraction of your application domain, independent of the actual implementation.
Here's an example from /internal/domain/users.go:

```go
// filename: domain/users.go

package domain

type User struct {
 ID         uuid.UUID `db:"id"`
 FirstName  string    `db:"first_name"`
 LastName   string    `db:"last_name"`
 Email      string    `db:"email"`
 Username   string    `db:"username"`
 Password   string    `db:"password"`
 Salt       string    `db:"salt"`
}

type UserStoreReader interface {
 GetUserByName(ctx context.Context, uname string) (User, error)
 GetUser(ctx context.Context, id string) (User, error)
 GetUsers(ctx context.Context) ([]User, error)
}

type UserStoreWriter interface {
 CreateUser(ctx context.Context, user *User) error
}

type UserStore interface {
 UserStoreReader
 UserStoreWriter
}
```

In this example, we've defined the User type and related interfaces. These interfaces provide a contract for how user data can be accessed and manipulated, but the actual implementation can vary, whether it's from a database or a third-party API.

### /internal: Application Logic
The /internal package is where you place the private application code. It defines concrete types with methods that contain the actual business logic. The /internal/app subpackage typically contains concrete types that implement interfaces defined in the /api package.
You can define packages like internal/db, internal/<api-client> ..etc. Which will define concrete types implementing interfaces defined in /domain package.
Here's an example from /internal/db/mysql/users.go:

```go
// filename: /internal/db/mysql/users.go

package mysql

func NewUserStore(db db.DBTx) *UserStore {
 return &UserStore{db: db}
}

type UserStore struct {
 db db.DBTx
}

func (u *UserStore) GetUserByName(ctx context.Context, uname string) (User, error) {...}
func (u *UserStore) GetUser(ctx context.Context, id string) (User, error) {...}
func (u *UserStore) GetUsers(ctx context.Context) ([]User, error) {...}                  
func (u *UserStore) CreateUser(ctx context.Context, user *User) error{...}
```

In this example, we've defined a concrete type UserStore that implements the UserStoreReader and UserStoreWriter interfaces from the /internal/domain package. This separation allows us to keep the application domain independent of the actual implementation, making it easy to switch between different data sources.

```go
// filename: /internal/app/users.go

package app

func NewUserService(s domain.UserStore, domain string) *UserService {
 return &UserService{userStore: s, domain: domain}
}

type UserService struct {
 userStore domain.UserStore
 domain    string
}

func (u *UserService) Login(ctx context.Context, username, password string) (*api.User, error) {...}

func (u *UserService) Register(ctx context.Context, user api.User) (uuid.UUID, error) {...}

func (u *UserService) GetUsers(ctx context.Context) ([]api.User, error){...}
```

In the above example we have defined a concreate type UserService which will implement the interface defined in the api package. This will contain the actual business logic. It will have the dependency of our application domain interface. It is not dependent on any specific implementation of our application domain.

### /internal/transport: Handling Communication
The /internal/transport package is responsible for the application's transport layer logic, such as HTTP or gRPC handlers. For example, you can define your application's HTTP handlers in the /transport/http subpackage.
Here's an example from /transport/http/users_handler.go:

```go
package http

import (
 "context"
 "encoding/json"
 "errors"
 "net/http"

 "github.com/gorilla/mux"

 "github.com/oshankkumar/sockshop/api"
 "github.com/oshankkumar/sockshop/domain"
)

type loginService interface {
 Login(ctx context.Context, username, password string) (*api.User, error)
}

func LoginHandler(l loginService) HandlerFunc {
 return func(w http.ResponseWriter, r *http.Request) *Error {
  username, pass, ok := r.BasicAuth()
  if !ok {
   return &Error{Code: http.StatusUnauthorized, Message: "user not authorised", Err: api.ErrUnauthorized}
  }

  user, err := l.Login(r.Context(), username, pass)

  switch {
  case errors.Is(err, api.ErrUnauthorized):
   return &Error{http.StatusUnauthorized, "user not authorised", err}
  case errors.Is(err, domain.ErrNotFound):
   return &Error{http.StatusNotFound, "user not found", err}
  case err != nil:
   return &Error{http.StatusInternalServerError, "user login failed", err}
  }

  RespondJSON(w, user, http.StatusOK)
  return nil
 }
}
```
Like in the above sample code, We have define http handler for user Login.
Since our login handler is just dependent on the Login method, so we have define a client specific interface. It follows Interface Segregation Principle. Interface segregation principle states many client-specific interfaces are better than one general-purpose interface. Clients should not be forced to implement a function they do no need.


## Conclusion
In this part of our journey into structuring SOLID Go web applications, we've explored the fundamental principles and package layout that set the stage for clean and maintainable code. In the next part of this series, we'll delve deeper into SOLID principles and demonstrate how they are applied in practice within this demo application.
For the complete application code and further exploration, you can find the project on GitHub at https://github.com/oshankkumar/sockshop.
Stay tuned for our next installment, where we'll dive into SOLID principles and their real-world implementation.
I'll write another post explaining SOLID principles and how it is being used in this demo application code.