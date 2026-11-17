# Frontend Implementation Artifact (Developer 2)

> **Status:** 🟡 Awaiting User Approval · Created: 2026‑11‑17

---

## Overview

This guide is the **implementation plan for the React/Vite frontend** of the Swim Academy Phase 1 project. It follows the decisions in the Master Project Plan (`phase1-backend-heavy-plan.md`) **without deviation**. All API paths, request/response structures, and business rules are defined in the shared `API_CONTRACT.md` (generated separately). The frontend will be built as a **static SPA** served by Nginx.

---

## 1. Folder & Component Structure (owned by Developer 2)

```
frontend/
├─ public/                     # Vite static assets (favicon, index.html)
├─ src/
│   ├─ api/                   # Axios instance + TanStack Query hooks
│   │   ├─ axiosInstance.ts
│   │   └─ hooks/            # useCoaches, usePrograms, useEnroll, useInquiry, etc.
│   ├─ components/            # Reusable UI pieces
│   │   ├─ Navbar.tsx
│   │   ├─ Footer.tsx
│   │   ├─ Card/               # ProgramCard, CoachCard, InfoCard
│   │   ├─ MapIframe.tsx       # Free‑iframe Google Map component
│   │   ├─ LoadingSpinner.tsx
│   │   └─ ErrorBoundary.tsx
│   ├─ pages/                  # One component per public page
│   │   ├─ Home.tsx
│   │   ├─ About.tsx
│   │   ├─ Coaches.tsx
│   │   ├─ Programs.tsx
│   │   ├─ Enroll.tsx
│   │   └─ Contact.tsx
│   ├─ schemas/                # Zod validation schemas (enrollment, inquiry)
│   │   ├─ enrollmentSchema.ts
│   │   └─ inquirySchema.ts
│   ├─ types/                  # TypeScript interfaces mirroring DTOs from API contract
│   │   ├─ Coach.ts
│   │   ├─ Program.ts
│   │   ├─ Batch.ts
│   │   ├─ Enrollment.ts
│   │   └─ Inquiry.ts
│   ├─ App.tsx                 # React Router v7 + Layout wrapper
│   └─ main.tsx                # Vite entry point
├─ vite.config.ts
├─ tsconfig.json
├─ package.json
├─ Dockerfile                 # Multi‑stage: builder + nginx static serve
├─ src/tests/                 # Vitest / React Testing Library tests
└─ .env.example               # REACT_APP_API_BASE_URL, REACT_APP_MOCK
```

*Ownership matrix:* All files under `frontend/` are owned by **Developer 2**. Shared files (`docker-compose.yml`, `nginx/nginx.conf`, `README.md`, `API_CONTRACT.md`) require joint review.

---

## 2. Project Setup (Vite + TypeScript)

```bash
# From the repo root
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install
# Additional dependencies
npm add axios @tanstack/react-query zod react-hook-form @hookform/resolvers tailwindcss postcss autoprefixer
# Initialize Tailwind
npx tailwindcss init -p
```

Configure `tailwind.config.js` to use the **brand colors** once you decide them (see Open Question 1). Example placeholder:
```js
module.exports = {
  content: ["./src/**/*.{js,ts,tsx,jsx}", "./index.html"],
  theme: {
    extend: {
      colors: {
        primary: "#0066cc",
        accent: "#ff6600",
      },
    },
  },
  plugins: [],
};
```

---

## 3. Routing & Layout

* Use **React Router v7** (`BrowserRouter`).
* `App.tsx` defines a `<Layout>` component that renders `<Navbar/>`, `<Outlet/>`, and `<Footer/>`.
* Routes (public):
  ```tsx
  <Routes>
    <Route path="/" element={<Home/>} />
    <Route path="/about" element={<About/>} />
    <Route path="/coaches" element={<Coaches/>} />
    <Route path="/programs" element={<Programs/>} />
    <Route path="/enroll" element={<Enroll/>} />
    <Route path="/contact" element={<Contact/>} />
    <Route path="*" element={<NotFound/>} />
  </Routes>
  ```
* All URLs are **relative**; the base URL is configured via `REACT_APP_API_BASE_URL` (default `http://localhost/api`).

---

## 4. API Integration (using shared contract)

### 4.1 Axios Instance
```ts
// src/api/axiosInstance.ts
import axios from "axios";

const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || "http://localhost/api",
  timeout: 8000,
});

export default api;
```

### 4.2 TanStack Query Hooks (examples)
```ts
// src/api/hooks/usePrograms.ts
import { useQuery } from "@tanstack/react-query";
import api from "../axiosInstance";
import { Program } from "../../types/Program";

export const usePrograms = (filters: { ageGroup?: string; skillLevel?: string }) =>
  useQuery<Program[]>({
    queryKey: ["programs", filters],
    queryFn: async () => {
      const resp = await api.get<Program[]>("/public/programs", { params: filters });
      return resp.data;
    },
    staleTime: 5 * 60 * 1000, // 5 min – catalog changes rarely
  });
```
* Similar hooks for coaches, enrollment submission, inquiry submission, and optional status lookup.

---

## 5. UI Pages & Components

### 5.1 Home (`Home.tsx`)
* Hero banner with academy name & tagline (coming from branding question). CTA button links to `/enroll`.
* Featured programs section – uses `usePrograms({ size: 3 })` and renders `ProgramCard` components.
* Facility highlights – data from `useAcademyInfo` (key `facility_highlights`).
* Quick inquiry modal – re‑uses `Contact` form component.

### 5.2 About (`About.tsx`)
* Static content – pulls values from `academy-info` keys: `pool_depth`, `pool_temp`, `safety_standards`.
* Simple layout with Tailwind utility classes.

### 5.3 Coaches (`Coaches.tsx`)
* Grid of `CoachCard` – displays `photo_url`, name, certifications, specialties, experience years.
* Data via `useCoaches` query.

### 5.4 Programs (`Programs.tsx`)
* Filter bar – age‑group tabs + skill‑level pills; updates query params passed to `usePrograms`.
* Each `ProgramCard` shows program details and **informational batch schedule** (no enrollment button). Batch schedule extracted from the program detail endpoint (`GET /public/programs/{id}`) which includes an array of related batches.
* “Apply for this program” button navigates to `/enroll?programId=<id>`.

### 5.5 Enroll (`Enroll.tsx`)
* **Single‑step form** (applicant type, personal details, program selection). Implemented with `react-hook-form` + Zod resolver.
* Form fields (mirroring `EnrollmentRequest`):
  - `applicantType` (radio: Parent/Child vs Adult)
  - `studentName` (required)
  - `studentDob` (optional, date picker, validated if provided)
  - `guardianName` (required when applicantType = Parent/Child)
  - `phone` (required, regex `^[0-9]{10,12}$`)
  - `email` (optional, email format)
  - `emergencyContact` (optional)
  - `medicalNotes` (optional, textarea)
  - `swimmingBackground` (optional, textarea)
  - `programId` (select – pre‑filled from URL query param if present)
* On submit, call `useSubmitEnrollment` mutation (POST `/public/enrollments`).
* Success UI shows **Reference ID** and message “We will contact you soon – pay at the pool”.
* Errors from the server (duplicate, age‑group mismatch) are displayed inline using the RFC 7807 `detail` field.

### 5.6 Contact (`Contact.tsx`)
* Displays address, phone, WhatsApp link, pool hours – data from `useAcademyInfo`.
* **MapIframe** component renders the free‑iframe URL stored in `academy_info` key `map_iframe_url`.
* Inquiry form (name, email, phone, message) – validated by Zod; POST `/public/inquiries`. Shows toast on success.

---

## 6. Reusable Components
| Component | Purpose |
|-----------|---------|
| `Navbar` | Site navigation, brand logo, responsive hamburger menu.
| `Footer` | Contact info, social links, copyright.
| `Card` (ProgramCard, CoachCard) | Consistent styling for list items.
| `MapIframe` | Simple wrapper that injects the iframe HTML safely.
| `LoadingSpinner` | Central loading indicator used by all TanStack Query hooks.
| `ErrorBoundary` | Catches rendering errors, displays fallback UI.
| `Toast` (via `react-hot-toast` or similar) | For success / error notifications.

All components use **Tailwind utility classes** and respect the brand color variables defined in `tailwind.config.js`.

---

## 7. Validation (Zod + React‑Hook‑Form)
* **Enrollment schema** (`src/schemas/enrollmentSchema.ts`):
  ```ts
  import { z } from "zod";
  export const enrollmentSchema = z.object({
    applicantType: z.enum(["PARENT_CHILD", "ADULT_SELF"]),
    studentName: z.string().min(1),
    studentDob: z.string().optional().refine(val => !val || !isNaN(Date.parse(val)), { message: "Invalid date" }),
    guardianName: z.string().optional().refine((v, ctx) => ctx.parent.applicantType === "PARENT_CHILD" ? !!v : true, { message: "Guardian required for child" }),
    phone: z.string().regex(/^\d{10,12}$/),
    email: z.string().email().optional(),
    emergencyContact: z.string().optional(),
    medicalNotes: z.string().optional(),
    swimmingBackground: z.string().optional(),
    programId: z.coerce.number().positive(),
  });
  ```
* **Inquiry schema** similar, with required `name`, `phone`, `message`.
* Errors from Zod are displayed beneath the corresponding input field.

---

## 8. Mock Data Strategy (for early UI work)
1. **Generate mock JSON** from the Flyway seed data (copy `backend/src/main/resources/db/migration/V2__seed_programs.sql` etc. into a JSON shape). Store in `frontend/src/mock/api-mocks.json`:
   ```json
   {
     "coaches": [ … ],
     "programs": [ … ],
     "academyInfo": { "address": "…", "map_iframe_url": "<iframe …>" }
   }
   ```
2. **Environment flag** – `VITE_MOCK=true` in `.env` toggles mock mode.
3. **Axios interceptor** (in `axiosInstance.ts`):
   ```ts
   if (import.meta.env.VITE_MOCK === "true") {
     api.interceptors.request.use(cfg => {
       const url = cfg.url?.replace(/^\/api\/v1\/public\//, "");
       // simple router that returns data from the mock JSON based on the URL pattern
       // e.g. "coaches" → mock.coaches, "programs" → mock.programs, etc.
       return Promise.reject({ mockResponse: mockData[url] });
     });
   }
   ```
4. Hook implementations check `error.mockResponse` and return it as a resolved promise when mock mode is on.
5. When the backend is ready, set `VITE_MOCK=false` and the same hooks will perform real HTTP calls.

---

## 9. Testing Strategy (Frontend)
### 9.1 Unit Tests (Vitest)
* Component snapshot tests for `ProgramCard`, `CoachCard`, `MapIframe`.
* Zod schema validation unit tests.
* Hook unit tests using `@testing-library/react`’s `renderHook` with mock Axios responses.

### 9.2 Integration Tests (React Testing Library)
* Render the full page flow:
  1. Load `/programs`, select a program, click **Apply**, navigate to `/enroll`.
  2. Fill the enrollment form (valid data) and submit – mock the POST response to return a reference ID.
  3. Verify that the success screen shows the reference ID.
* Test error handling by mocking a `409 duplicate-enrollment` response and checking the inline error message.

Run all tests with:
```bash
cd frontend
npm run test
```

---

## 10. Docker & Deployment (frontend side)
### 10.1 Dockerfile (multi‑stage)
```dockerfile
# ---------- Builder ----------
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json .
RUN npm ci --silent
COPY . .
ARG VITE_API_BASE_URL
ARG VITE_MOCK
ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}
ENV VITE_MOCK=${VITE_MOCK}
RUN npm run build

# ---------- Runtime (served by nginx) ----------
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
# nginx.conf will be provided by the shared repo (nginx/nginx.conf)
```
### 10.2 Environment variables (passed at build time)
* `VITE_API_BASE_URL` – e.g., `http://localhost/api` for local dev, `https://swimacademy.in/api` for production.
* `VITE_MOCK` – `true` during early UI work, `false` for live integration.

### 10.3 docker‑compose integration (excerpt)
```yaml
services:
  frontend:
    build:
      context: ./frontend
      args:
        VITE_API_BASE_URL: http://backend:8080/api
        VITE_MOCK: "false"
    depends_on:
      - backend
    ports:
      - "80:80"   # Nginx in the shared nginx service will proxy static files
```
(Actual static serving is done by the shared `nginx` container; the `frontend` service simply builds the assets.)

---

## 11. Milestones & Implementation Sequence (for Developer 2)
1. **M1 – Repo bootstrap & Vite init** (week 1)
2. **M2 – Tailwind config & brand colors** (week 1‑2) – awaiting branding decision.
3. **M3 – Axios instance, API contract typing, mock JSON** (week 2)
4. **M4 – Layout, Navbar, Footer, routing** (week 2‑3)
5. **M5 – Page skeletons (Home, About, Coaches, Programs)** (week 3)
6. **M6 – Reusable Card components, MapIframe** (week 3‑4)
7. **M7 – Enroll form with Zod + react‑hook‑form** (week 4)
8. **M8 – Contact page + inquiry form** (week 4‑5)
9. **M9 – TanStack Query hooks for all public endpoints** (parallel with M5‑M8)
10. **M10 – Mock‑mode integration & local UI work** (week 5)
11. **M11 – Component & hook unit tests** (week 5‑6)
12. **M12 – End‑to‑end page flow tests (React Testing Library)** (week 6)
13. **M13 – Dockerfile, build CI, integration with shared Nginx** (week 6)
14. **M14 – Final QA, documentation (README, usage notes)** (week 6)

---

## 12. Ownership & Coordination
* All files under `frontend/` belong to **Developer 2**.
* **Shared files** (`docker-compose.yml`, `nginx/nginx.conf`, `README.md`, `API_CONTRACT.md`) require a PR that both developers approve.
* **API contract changes** – edit `API_CONTRACT.md`, open a PR, both devs merge, then:
  - Dev 1 updates DTOs/controllers.
  - Dev 2 updates TypeScript interfaces, Zod schemas, and mock JSON.
* **Mock data updates** – when seed data changes, run the helper script `npm run generate-mocks` (to be added) which reads `backend/src/main/resources/db/migration/V2‑V5.sql` and outputs fresh JSON.

---

## 13. Next Steps
* **Approve** this Frontend Implementation Artifact.
* Once approved, the artifact will be committed as `FRONTEND_IMPLEMENTATION.md`.
* Then the **API Contract** artifact will be generated.

---

*This document is stored as an artifact and will be updated automatically if you request changes.*