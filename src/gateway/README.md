# Gateway

API Gateway (.NET + YARP). A single project (no domain): the only public entry point for the web app.

- Routes requests to the backend services
- Validates the Keycloak JWT and forwards it unchanged (services validate it again)
- CORS, rate limiting, health checks
- No business logic
