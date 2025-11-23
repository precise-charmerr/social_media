# Database Setup Guide - PostgreSQL + Flyway

Learn how to set up PostgreSQL and Flyway migrations for your Spring Boot social media app.

## Overview

- **Database**: PostgreSQL (relational DB)
- **Migration Tool**: Flyway (manages schema versions)
- **Why?**: Migrations track DB changes over time; Flyway auto-applies them on app startup

---

## Step 1: Add PostgreSQL & Flyway Dependencies to `pom.xml`

Open `backend/pom.xml` and find the `<dependencies>` section. Add these two dependencies:

```xml
<!-- PostgreSQL JDBC Driver -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <version>42.7.1</version>
</dependency>

<!-- Flyway for Database Migrations -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>9.22.3</version>
</dependency>
```

**Why?**
- `postgresql`: lets Java connect to PostgreSQL
- `flyway-core`: automatically runs SQL migration files on startup

**Where to add?** Look for existing dependencies like `spring-boot-starter-web` and add these after them.

---

## Step 2: Create Migration Files Directory

Flyway looks for SQL files in a specific location. Create this folder structure:

```
backend/src/main/resources/db/migration/
```

Run this command:
```bash
mkdir -p backend/src/main/resources/db/migration
```

**Why?** Flyway scans this folder for `.sql` files named `V{version}__{description}.sql`

---

## Step 3: Create Initial Migrations

Create these SQL migration files in `backend/src/main/resources/db/migration/`:

### File 1: `V1__Create_users_table.sql`

```sql
CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    bio TEXT,
    profile_picture_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
```

**What it does:** Creates a `users` table with profile fields.

### File 2: `V2__Create_posts_table.sql`

```sql
CREATE TABLE IF NOT EXISTS posts (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    likes_count INT DEFAULT 0,
    comments_count INT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
CREATE INDEX idx_posts_created_at ON posts(created_at DESC);
```

**What it does:** Creates a `posts` table linked to `users`.

### File 3: `V3__Create_comments_table.sql`

```sql
CREATE TABLE IF NOT EXISTS comments (
    id BIGSERIAL PRIMARY KEY,
    post_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_comments_post_id ON comments(post_id);
CREATE INDEX idx_comments_user_id ON comments(user_id);
```

**What it does:** Creates a `comments` table linked to `posts` and `users`.

### File 4: `V4__Create_follows_table.sql`

```sql
CREATE TABLE IF NOT EXISTS follows (
    id BIGSERIAL PRIMARY KEY,
    follower_id BIGINT NOT NULL,
    following_id BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(follower_id, following_id),
    FOREIGN KEY (follower_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY (following_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_follows_follower_id ON follows(follower_id);
CREATE INDEX idx_follows_following_id ON follows(following_id);
```

**What it does:** Creates a `follows` table for user relationships.

---

## Step 4: Update `application.properties`

Edit `backend/src/main/resources/application.properties`:

```properties
# Server
server.port=8080

# PostgreSQL Configuration
spring.datasource.url=jdbc:postgresql://localhost:5432/social_media
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA Configuration
spring.jpa.database-platform=org.hibernate.dialect.PostgreSQLDialect
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true

# Flyway Configuration
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration
spring.flyway.baseline-on-migrate=true
```

**Key points:**
- `jdbc:postgresql://localhost:5432/social_media` — connects to local PostgreSQL, database `social_media`
- `ddl-auto=validate` — Flyway manages schema, Hibernate just validates
- `flyway.baseline-on-migrate=true` — allows Flyway to work with existing DBs

---

## Step 5: Set Up Local PostgreSQL (Docker)

Create `docker-compose.yml` in the `backend/` folder:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: social_media
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

**To run:**
```bash
cd backend
docker-compose up -d
```

**To stop:**
```bash
docker-compose down
```

**Why Docker?** Consistent local DB, no manual PostgreSQL install.

---

## Step 6: Create `application-dev.properties`

Create `backend/src/main/resources/application-dev.properties` for local development (overrides `application.properties`):

```properties
# Development Profile - Local PostgreSQL (Docker)
spring.datasource.url=jdbc:postgresql://localhost:5432/social_media
spring.datasource.username=postgres
spring.datasource.password=postgres
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
logging.level.root=INFO
logging.level.com.socialmedia=DEBUG
```

**To run with dev profile:**
```bash
mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"
```

---

## Step 7: Verify Setup

1. **Start PostgreSQL:**
   ```bash
   cd backend
   docker-compose up -d
   ```

2. **Check DB is running:**
   ```bash
   docker-compose logs postgres
   ```
   (Should see "database system is ready to accept connections")

3. **Build & run the app:**
   ```bash
   mvn clean package
   mvn spring-boot:run -Dspring-boot.run.arguments="--spring.profiles.active=dev"
   ```

4. **Check migrations ran:**
   - Look for logs: `Successfully completed 4 migrations`
   - Or connect to DB: `psql -h localhost -U postgres -d social_media` and run `\dt` (list tables)

---

## Understanding Flyway

| File | Purpose |
|------|---------|
| `V1__Create_users_table.sql` | Version 1: initial schema |
| `V2__Create_posts_table.sql` | Version 2: add posts |
| `V3__Create_comments_table.sql` | Version 3: add comments |
| `V4__Create_follows_table.sql` | Version 4: add follows |

**Key rules:**
- File names: `V{number}__{description}.sql`
- Numbers must be in order (1, 2, 3, 4)
- Flyway runs each once and tracks in `flyway_schema_history` table
- Never modify old migrations; create new ones

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| "Connection refused" | Check `docker-compose up -d` is running, wait 10s |
| "Migration failed" | Check SQL syntax in migration files, or delete DB and re-run |
| Tables not created | Check logs for "error" keyword; ensure `db/migration/` folder exists |
| "Can't find driver" | Add `postgresql` dependency to `pom.xml` and rebuild |

---

## Next Steps

Once DB is set up:
1. Create Java entity classes (JPA) matching the schema
2. Create repositories (Spring Data JPA)
3. Build REST controllers to insert/query data
4. Add authentication (Spring Security)

Good luck! 🚀
