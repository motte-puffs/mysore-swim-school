# Backend Implementation Artifact (Developer 1)

> **Status:** 🟡 Awaiting User Approval · Created: 2026‑11‑17

---

## Overview

This document is the **implementation guide for the Spring‑Boot backend** of the Swim Academy Phase 1 project. It follows **exactly** the decisions made in the Master Project Plan (`phase1-backend-heavy-plan.md`). All folder and package names, API paths, DTO fields, database columns, and business rules must remain unchanged.

---

## 1. Package & Folder Structure (owned by Developer 1)

```
backend/
├─ src/main/java/com/swimacademy/
│   ├─ config/               # Security, CORS, OpenAPI, Rate‑Limiting beans
│   ├─ controller/
│   │   ├─ public_/          # PublicCoachController, PublicProgramController,
│   │   │                     # PublicEnrollmentController, PublicInquiryController
│   │   └─ admin/            # AdminCoachController, AdminProgramController,
│   │                         # AdminBatchController, AdminEnrollmentController,
│   │                         # AdminInquiryController, AdminAcademyInfoController
│   ├─ dto/
│   │   ├─ request/          # EnrollmentRequest, ApproveRequest, RejectRequest,
│   │   │                     # InquiryRequest, CoachRequest, ProgramRequest, BatchRequest
│   │   └─ response/         # EnrollmentResponse, InquiryResponse, CoachResponse, …
│   ├─ entity/               # JPA entities (Coach, Program, Batch, Enrollment, Inquiry, AcademyInfo)
│   ├─ exception/            # GlobalExceptionHandler + custom exception classes
│   ├─ mapper/               # MapStruct mapper interfaces
│   ├─ repository/           # Spring Data JPA repositories
│   └─ service/              # Business logic (EnrollmentService, AdminService, …)
├─ src/main/resources/
│   ├─ application.yml       # Configurable properties (DB URL, admin credentials, etc.)
│   └─ db/migration/         # Flyway V1‑V5 scripts (schema + seed data)
├─ src/test/java/…          # Unit & integration tests (H2 + Testcontainers)
├─ pom.xml
└─ Dockerfile                # Multi‑stage build (builder + runtime)
```

---

## 2. Database Design (PostgreSQL 17/18)

* **ER diagram** – see Master Plan.
* **Key tables**: `coach`, `program`, `batch`, `enrollment`, `inquiry`, `academy_info`.
* **Constraints**:
  - `reference_id` UNIQUE (`SWM-YYYYMMDD-{UUID8}`).
  - `status` CHECK (`PENDING`,`CONFIRMED`,`REJECTED`).
  - Duplicate‑block index on `(student_name, program_id, status)` where `status <> 'REJECTED'`.
  - `fee_paise` BIGINT (store rupees as paise).
* **Flyway migrations** – V1 creates schema; V2‑V5 seed realistic placeholder data (programs, batches, coaches, academy info).

---

## 3. Spring Boot Project Setup

* **Java 21**, **Spring Boot 4.1.x** (parent version in `pom.xml`).
* **Dependencies** (managed by Maven):
  - `spring-boot-starter-web`
  - `spring-boot-starter-data-jpa`
  - `spring-boot-starter-security`
  - `spring-boot-starter-validation`
  - `springdoc-openapi-starter-webmvc-ui`
  - `mapstruct` + `mapstruct‑processor`
  - `flyway-core`
  - `bucket4j-core`
  - `postgresql` driver
  - `testcontainers` (postgres) for integration tests
* **Application.yml** includes placeholders:
  ```yaml
  spring:
    datasource:
      url: ${DB_URL:jdbc:postgresql://localhost:5432/swim_academy}
      username: ${DB_USER:swim_user}
      password: ${DB_PASS:swim_pass}
    jpa:
      hibernate:
        ddl-auto: validate
      show-sql: false
  server:
    port: 8080
  admin:
    username: ${ADMIN_USERNAME:admin}
    password: ${ADMIN_PASSWORD:admin123}
  ```

---

## 4. Core Implementation Details

### 4.1 Entities & Repositories
* Each table has a corresponding `@Entity` class with `@Id @GeneratedValue` and `@Column` mapping.
* Repositories extend `JpaRepository<…, Long>`.
* `EnrollmentRepository` includes a custom method to check duplicate submissions:
  ```java
  boolean existsByStudentNameAndProgramIdAndStatusNot(String name, Long programId, String excludedStatus);
  ```

### 4.2 DTOs & MapStruct Mappers
* Request DTOs use `javax.validation` annotations (`@NotBlank`, `@Pattern`, `@PastOrPresent`).
* Response DTOs expose only public fields (no internal timestamps unless needed).
* Mappers convert between entities and DTOs (e.g., `EnrollmentMapper`).

### 4.3 Controllers
* **Public controllers** – annotated `@RestController` + `@RequestMapping("/api/v1/public")`.
  - `PublicEnrollmentController` handles `POST /enrollments`.
  - `PublicProgramController` provides catalog endpoints.
* **Admin controllers** – annotated `@RestController` + `@RequestMapping("/api/v1/admin")` and secured with `@PreAuthorize("hasRole('ADMIN')")`.
  - Approve endpoint: `PUT /enrollments/{id}/approve` (body `{ "batchId": … }`).
  - Reject endpoint: `PUT /enrollments/{id}/reject` (body `{ "reason": "…" }`).

### 4.4 Validation & Exception Handling
* Bean validation (`@Valid`) on request bodies.
* `GlobalExceptionHandler` (`@RestControllerAdvice`) returns RFC 7807 problem‑detail JSON.
* Custom exceptions: `DuplicateEnrollmentException`, `BatchFullException`, `BatchProgramMismatchException`, `InvalidStatusTransitionException`, `AgeGroupMismatchException`.

### 4.5 Security & Rate Limiting
* **HTTP Basic Auth** for `/api/v1/admin/**`. Credentials from `application.yml` (environment variables).
* **CORS** – whitelist taken from `academy_info` key `allowed_origin` (seeded as `http://localhost`).
* **Bucket4j** in‑memory filter bean applied to the public filter chain with limits defined in Master Plan.

### 4.6 Enrollment Workflow
* **Submission** – service checks duplicate, validates age group, generates `referenceId` (`SWM-YYYYMMDD-{UUID8}`), saves with `batch_id = null`, `status = PENDING`.
* **Approval** – admin service loads enrollment, verifies batch belongs to same program, checks seat availability (`capacity - confirmedEnrollments`), calculates `package_start_date = today`, `package_end_date = today + program.packageDurationDays`, updates record (`status = CONFIRMED`).
* **Rejection** – sets `status = REJECTED` and stores `rejection_reason`.

---

## 5. Testing Strategy

### 5.1 Unit Tests (JUnit 5)
* Service layer tests – mock repositories with Mockito.
* Validation tests – ensure DTO constraints reject malformed data.
* Security tests – verify admin endpoints require auth.

### 5.2 Integration Tests (Testcontainers)
* Spin up a temporary PostgreSQL container.
* Run Flyway migrations automatically.
* Execute end‑to‑end scenarios:
  - Duplicate enrollment returns 409.
  - Age‑group mismatch returns 422.
  - Approve with full batch returns 409.
  - Approve with mismatched batch returns 422.
  - Successful approval updates dates correctly.

Run all tests with:
```bash
cd backend
mvn verify -Pintegration
```

---

## 6. Docker & Deployment

### 6.1 Dockerfile (multi‑stage)
```dockerfile
# ---------- Build Stage ----------
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

# ---------- Runtime Stage ----------
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 6.2 Environment Variables (to be placed in a `.env` file for Docker‑Compose)
```
DB_URL=jdbc:postgresql://postgres:5432/swim_academy
DB_USER=swim_user
DB_PASS=swim_pass
ADMIN_USERNAME=admin
ADMIN_PASSWORD=SuperSecret123
```

### 6.3 docker‑compose.yml (excerpt)
```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_DB: swim_academy
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build: ./backend
    environment:
      - DB_URL=${DB_URL}
      - DB_USER=${DB_USER}
      - DB_PASS=${DB_PASS}
      - ADMIN_USERNAME=${ADMIN_USERNAME}
      - ADMIN_PASSWORD=${ADMIN_PASSWORD}
    depends_on:
      - postgres
    ports:
      - "8080:8080"

volumes:
  pgdata:
```

---

## 7. Milestones & Implementation Sequence (for Developer 1)
1. **M1 – Repo bootstrap & Maven setup** (week 1)
2. **M2 – Flyway schema (V1) & entity definitions** (week 1‑2)
3. **M3 – DTOs, MapStruct mappers, basic validation** (week 2)
4. **M4 – Public controllers (catalog + enrollment POST)** (week 3)
5. **M5 – Admin security, CRUD controllers, approve/reject logic** (week 3‑4)
6. **M6 – Rate‑limiting filter (Bucket4j)** (week 4)
7. **M7 – Dockerfile, docker‑compose, env vars** (week 5)
8. **M8 – Unit tests for services** (parallel with M5‑M6)
9. **M9 – Integration tests with Testcontainers** (week 5‑6)
10. **M10 – Documentation (README, Swagger UI)** (week 6)

---

## 8. Ownership & Coordination
* All files under `backend/` are **owned** by Developer 1.
* The **API contract** (`API_CONTRACT.md`) lives at the repo root and must be edited via a PR that both developers review.
* Any change to request/response shapes, endpoint URLs, or status codes **must** be reflected in this contract before code changes.
* When the contract is updated, Developer 2 updates mock JSON and Axios hooks; Developer 1 updates DTOs/controllers.
* Shared files (`docker-compose.yml`, `nginx/nginx.conf`, `README.md`, `API_CONTRACT.md`) require joint review.

---

## 9. Next Steps
* **Approve** this Backend Implementation Artifact.
* Once approved, the artifact will be committed as `BACKEND_IMPLEMENTATION.md`.
* Then the **Frontend Implementation** and **API Contract** artifacts will be generated.

---

*This document is stored as an artifact and will be updated automatically if you request changes.*