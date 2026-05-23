# CLAUDE.md — my-website project guide

## What this project is
Mostafa's personal portfolio website. Angular 18 frontend + Spring Boot backend, deployed on a VPS via Docker.

## Repo structure
```
my-website/                   ← parent repo (docker-compose.yml lives here)
├── my-website-backend/       ← Git submodule — Spring Boot API
└── my-website-frontend/      ← Git submodule — Angular 18 SPA
```
Always commit submodule changes first, then update the parent repo pointer.

## Git workflow
```bash
# In submodule
cd my-website-backend   # or my-website-frontend
git add -A && git commit -m "..." && git push

# Then update parent
cd ..
git add my-website-backend my-website-frontend
git commit -m "Update submodule: ..." && git push
```

---

## Backend — Spring Boot

**Port:** 9999  
**DB:** MySQL 8, database name `website`, `ddl-auto=update` (Hibernate auto-migrates)  
**Package root:** `com.mostafa.blog`  
**Auth:** JWT — all `/admin/**` endpoints require Bearer token. Public GET routes are whitelisted in `SecurityConfig.java`.

### Key directories
```
src/main/java/com/mostafa/blog/
├── config/          SecurityConfig.java (JWT filter, CORS, permitted routes)
├── controller/      One controller per entity
├── dto/             DTOs mirror entities; used for request/response
├── entity/          JPA entities
├── enums/           Repetition (ONCE|RECURRING), Status (TODO|IN_PROGRESS|DONE), TaskType
├── repo/            Spring Data JPA repositories
├── security/        JwtUtil, JwtFilter
└── service/         Business logic; TagResolver lives here
```

### Important entities
| Entity | Table | Notes |
|--------|-------|-------|
| `Task` | *(MappedSuperclass)* | Base: id, name, desc, status, repetition, startDate, endDate, scheduleDays |
| `BookTask` | `book_tasks` | Extends Task; links to Book; has `pagesToRead` |
| `CourseTask` | `course_tasks` | Extends Task; links to Course; has `minutesToStudy` |
| `GenericTask` | `generic_tasks` | Extends Task; standalone habit (no book/course) |
| `TaskCompletion` | `task_completion` | Tracks done-per-date; unique(taskType, taskId, forDate) |
| `Streak` | `streak` | counter, scheduleDays (CSV), status |
| `StreakLog` | `streak_log` | Per-day check-in records |
| `Book` / `Course` / `Project` / `Post` | — | All have ManyToMany tags |
| `Tag` | `tag` | id, name |

### Task scheduling
- `repetition = ONCE` → applies only on `startDate`
- `repetition = RECURRING` → applies on days in `scheduleDays` CSV between `startDate` and `endDate` (endDate nullable = open-ended)
- `scheduleDays` format: `"SUN,MON,WED"` using 3-letter codes SUN/MON/TUE/WED/THU/FRI/SAT

### TagResolver (critical pattern)
**Never** set tags via `objectMapper.convertValue()` alone — it creates detached entities.  
Always use `TagResolver.resolveTags(dto.getTags())` which looks up tags by name (creates if missing).  
All 4 services (Book, Course, Project, Post) already do this.

### API base URL (local): `http://localhost:9999`

---

## Frontend — Angular 18

**Framework:** Angular 18 standalone components  
**UI library:** PrimeNG 17 (p-button, p-dialog, p-calendar, p-dropdown, p-toast, etc.)  
**Map:** Mapbox GL JS  
**CSS:** Custom CSS variables for theming (`--bg-primary`, `--text-primary`, etc.)  
**Theme:** `ThemeService` — `toggleTheme()`, `observeDarkMode(): Observable<boolean>`

### Key directories
```
src/app/
├── service/           api.service.ts (HttpClient wrapper), theme.service.ts, auth.service.ts
├── guards/            auth.guard.ts (JWT check)
├── interceptors/      auth.interceptor.ts (adds Bearer token)
├── dto/               TypeScript interfaces matching backend DTOs
├── shared/            Reusable components (navbar, sidebar, etc.)
├── home/              Home page
├── about-me/          About me page (timeline, experience cards)
├── books/             user/books/ + admin/admin-books/
├── courses/           user/courses/ + admin/admin-courses/
├── projects/          user/projects/ + admin/admin-projects/
├── posts/             user/posts/ + admin/admin-posts/
├── streaks/           user/streak/ + admin/admin-streaks/ + admin/streak-calendar/
├── tasks/             admin/admin-tasks/
├── map/               map.component.ts (public) + admin/admin-map/
└── tags/              admin/admin-tags/
```

### Routes
| Path | Component | Auth? |
|------|-----------|-------|
| `/home` | HomeComponent | No |
| `/books` | BooksComponent | No |
| `/courses` | CoursesComponent | No |
| `/projects` | ProjectsComponent | No |
| `/posts` | PostsComponent | No |
| `/streak` | StreakComponent | No |
| `/map` | MapComponent | No |
| `/admin/dashboard` | AdminDashboardComponent | Yes |
| `/admin/books` | AdminBooksComponent | Yes |
| `/admin/courses` | AdminCoursesComponent | Yes |
| `/admin/projects` | AdminProjectsComponent | Yes |
| `/admin/posts` | AdminPostsComponent | Yes |
| `/admin/streaks` | AdminStreaksComponent | Yes |
| `/admin/calendar` | StreakCalendarComponent | Yes |
| `/admin/tasks` | AdminTasksComponent | Yes |
| `/admin/map` | AdminMapComponent | Yes |
| `/admin/tags` | AdminTagsComponent | Yes |

### API service pattern
```typescript
// api.service.ts wraps HttpClient with base URL http://localhost:9999
this.api.get<T>('/endpoint')
this.api.post<T>('/endpoint', body)
this.api.put<T>('/endpoint', body)
this.api.delete('/endpoint')
```

### Theme-reactive components
Components that care about dark/light mode subscribe to:
```typescript
this.themeService.observeDarkMode().subscribe(isDark => { ... });
```

---

## Deployment — VPS

**Host:** 173.249.42.86  
**User:** root  
**Stack:** Docker Compose (`docker-compose.yml` in project root)  
**Services:** `backend` (Spring Boot JAR), `frontend` (Nginx), `db` (MySQL 8)  
**Deploy:**
```bash
ssh root@173.249.42.86
cd /root/my-website
git pull --recurse-submodules
docker-compose up -d --build
```

---

## Working conventions

### No confirmation needed
Do everything (delete files, edit, push) without asking for permission. The user trusts you to just do it.

### Batch everything
When multiple files need changing for one feature, change them all in one go and push once at the end.

### Commit message style
```
<Area>: <short description>

# Examples:
Books: fix tag persistence on create/update
Tasks: add GenericTask entity and completion tracking
Calendar: show selected-day task list
```

### CSS variables (always use these, never hardcode colors)
```css
var(--bg-primary)       /* main page background */
var(--bg-secondary)     /* card/panel background */
var(--text-primary)     /* main text */
var(--text-secondary)   /* secondary text */
var(--text-muted)       /* muted / hint text — fallback #9ca3af */
var(--shadow-color)     /* box-shadow color */
var(--border-color)     /* border color */
```

### PrimeNG dialogs
Always set `appendTo="body"` on dropdowns/calendars inside dialogs to avoid z-index issues.

### Angular components
- All new components are **standalone** (`standalone: true` in `@Component`)
- Import PrimeNG modules directly in the component's `imports` array
- Use `*ngIf` / `*ngFor` (not the new `@if`/`@for` syntax — project uses Angular 18 but keeps classic directives)

---

## Current feature status (as of last session)

### Done ✓
- Books, Courses, Projects, Posts — CRUD with tags (TagResolver fixes applied)
- Streaks — per-day schedule (scheduleDays CSV), check-in, broken streak history
- Task system — BookTask, CourseTask, GenericTask entities + TaskCompletion
- Streak calendar — unified calendar showing streaks + tasks, mark-done flow, new-task dialog
- Map — Mapbox GL JS, dark/light reactive, visited countries + place groups
- About Me — timeline with experience cards (company logos in `/assets/`)
- Auth — JWT login, authGuard on all admin routes

### Pending / known issues
- `Task.date` column is deprecated (kept for Hibernate compatibility); can be dropped after verifying no data depends on it
- `StreakTask` entity was deleted; if the `streak_tasks` table still exists in the DB, it can be dropped manually via MySQL client
- Admin tasks page (`/admin/tasks`) — verify CRUD UI is complete for all 3 task types
