# Ecommerce Microservices

This project contains:

- 4 Spring Boot based microservices (User, Cart, Order, Product)
- Gateway with Spring Cloud
- PostgreSQL database
- ElasticSearch to index Products
- Web Server with React

## Run Application locally

The entire application can be run locally with docker-compose by following these steps:

1. At the root of the project, rename **.sample-env** to **.env**
2. Create [Stripe](https://stripe.com/) and [Auth0](auth0.com) accounts, and fill the following environment variables in **.env**
   | Environment variable | Example Value |
   | -------------------------- | ----------------------------|
   | AUTH0_ISSUER | https://**dev-id**.us.auth0.com/|
   | AUTH0_USERINFO_URL | https://**dev-id**.us.auth0.com/userinfo |
   | STRIPE_API_KEY | Token starting with pk_test... |

   The remaining variables don't need to be changed for the app to run.

3. At the root of the project, run the following command
   `docker compose up --build `
4. When all the containers are running, open the browser and connect to http://localhost:8888
5. Register as a regular user OR login as admin using `admin@example.com` and `password` as credentials.

## Application architecture

![api gateway flow](https://github.com/amol9372/ecommerce-spring-boot-backend-apis/assets/20081129/a432fac4-ce61-4cca-a64c-aa459d525c2a)

## Postman collection (Public)

```

https://www.postman.com/galactic-eclipse-361945/workspace/public-9372/collection/1877749-6742d038-c937-4aac-838e-7e30ff85865d?action=share&creator=1877749

```

## Server Events (ADMIN user only)

In Postman use the below url to recieve server side events

```curl

ws://localhost:8091/ws/events

```

## Features

- Authentication with Jwt

- Add/remove products

- Create categories with hierarchy

- Manage address

- Feature template

- Shopping cart

- Checkout page

- Server Events page

## Tech stack

- Java 17

- Spring boot 3

- Elasticsearch 8

- Spring data JPA

- Postgres 14

- Spring cloud gateway

- Docker (docker-compose)

- React

## ER Diagram

<img  width="1676"  alt="Pasted Graphic"  src="https://github.com/amol9372/ecommerce-spring-boot-backend-apis/assets/20081129/94d43c0d-2d2e-40be-a44d-dec762b3ffb2">

## API Root Endpoint

### Spring cloud Gateway

`https://localhost:8091`

### Microservice Endpoints

| Service | Base URL                     |
| ------- | ---------------------------- |
| User    | `https://localhost:8080/api` |
| Product | `https://localhost:8081/api` |
| Cart    | `https://localhost:8082/api` |
| Order   | `https://localhost:8083/api` |

### Swagger endpoints

| Type       | Endpoint      |
| ---------- | ------------- |
| Api docs   | `/api/docs`   |
| Swagger UI | `/swagger-ui` |
