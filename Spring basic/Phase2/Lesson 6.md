# 📘 Phase 2, Lesson 6: Docker & PostgreSQL Setup

## 🎯 Learning Goal
- ✅ Understand why we use Docker for databases.
- ✅ Create a `docker-compose.yml` file to run PostgreSQL and pgAdmin.
- ✅ Connect to the database visually using pgAdmin.

---

## 💡 The Concept: Why Docker?
In the old days, developers had to install PostgreSQL directly on their laptops. This caused problems: "It works on my machine, but not on yours!" because everyone had different versions or settings.

**Docker** solves this. It packages the database into a "container" (like a lightweight virtual machine). You just run one command, and you get a perfectly configured database, identical for every developer in the world.

---

## 🛠️ Step-by-Step Build

### Step 1: Install Docker
If you haven't already, download and install **Docker Desktop** from [docker.com](https://www.docker.com/products/docker-desktop/). Make sure it is running (you should see the Docker whale icon in your system tray/menu bar).

### Step 2: Create `docker-compose.yml`
In the **root folder** of your project (the same folder that contains your `pom.xml`), create a new file named exactly `docker-compose.yml`.

Paste this code into it:

```yaml
version: '3.8'
services:
  # 1. The actual PostgreSQL Database
  postgres:
    image: postgres:16
    container_name: my-postgres-db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: admin123
      POSTGRES_DB: product_db
    ports:
      - "5432:5432" # Maps port 5432 on your laptop to port 5432 in the container
    volumes:
      - postgres_data:/var/lib/postgresql/data # Saves data even if container restarts

  # 2. pgAdmin (A visual tool to look inside the database)
  pgadmin:
    image: dpage/pgadmin4
    container_name: my-pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80" # Access pgAdmin at http://localhost:5050
    depends_on:
      - postgres # Ensures postgres starts first

volumes:
  postgres_data:
```

### Step 3: Start the Database
Open your terminal, navigate to your project's root folder, and run:
```bash
docker-compose up -d
```
*(The `-d` means "detached mode", so it runs in the background).*

### Step 4: Verify it's Running
Run this command:
```bash
docker ps
```
You should see two containers listed: `my-postgres-db` and `my-pgadmin`.

### Step 5: Open pgAdmin
1. Open your web browser.
2. Go to: **http://localhost:5050**
3. Login with:
   - **Email:** `admin@admin.com`
   - **Password:** `admin123`
4. Click **"Add New Server"** (the plug icon with a plus sign).
5. In the **Connection** tab, fill in:
   - **Host name/address:** `postgres` (This is the name of the service in docker-compose)
   - **Port:** `5432`
   - **Username:** `admin`
   - **Password:** `admin123`
6. Click **Save**. Expand the server tree on the left. You will see your `product_db` database!

---

## 🚨 Common Errors
| Error | Cause | Fix |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop is not running. | Open Docker Desktop and wait for the green "Engine running" indicator. |
| `Port 5432 is already allocated` | You already have PostgreSQL installed on your laptop. | Change the first port in docker-compose to `"5433:5432"`, and update your app config later. |

---

## 🛠️ Exercise
1. Stop the containers by running: `docker-compose down`
2. Start them again with: `docker-compose up -d`
3. Log into pgAdmin and verify the database is still there (thanks to the `volumes` configuration!).

---

## 🧠 Quiz
1. What is the main benefit of using Docker for a database instead of installing it directly on your laptop?
2. What URL do you use to access pgAdmin in your browser?
3. In the `docker-compose.yml`, what does `depends_on: - postgres` do?

---

## 🛑 STOP
Reply with your exercise confirmation and quiz answers before moving to Lesson 7.