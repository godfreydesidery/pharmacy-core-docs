# Local development

You need JDK 21, Maven 3.9 and Node 22 with the Angular CLI. You don't need to install a database.

## 1. API

```bash
cd pharmacy-core-api
mvn -pl application/pharmacy-core -am spring-boot:run -Dspring-boot.run.profiles=local
```

- It starts an embedded PostgreSQL on port 54330, with its data in `application/pharmacy-core/target/local-postgres` and the production roles. It then migrates all schemas.
- It creates the root administrator and the demo tenant **Afya Pharmacy (AFYA)**, with branches HQ and MBZ.
- API: `http://localhost:8080/api/v1`
- Swagger UI: `http://localhost:8080/swagger-ui.html`
- Health: `http://localhost:8080/actuator/health`

| Who | Email | Password |
|---|---|---|
| Root administrator | `root@pharmacy.local` | `Root#2026` |
| Afya admin | `admin@afya.co.tz` | `Afya#2026` |
| Afya cashier (HQ) | `cashier@afya.co.tz` | `Cashier#2026` |

## 2. Web

```bash
cd pharmacy-core-web
npm install
npm start          # ng serve on http://localhost:4200, proxying /api to :8080
```

## 3. Common tasks

| Task | How |
|---|---|
| Reset the local database | Stop the API, then delete `pharmacy-core-api/application/pharmacy-core/target/local-postgres` |
| Use another DB port or data directory | `PHARMACY_LOCAL_DB_PORT=54331`, `PHARMACY_LOCAL_DATA=/path` |
| Run all tests | `mvn verify` in `pharmacy-core-api` |
| Fix formatting | `mvn spotless:apply` |
| Module diagrams | `mvn -pl architecture-tests -am verify`, then open `architecture-tests/target/spring-modulith-docs` |
| Run against Docker PostgreSQL | `docker compose up --build` in `pharmacy-core-api` |
