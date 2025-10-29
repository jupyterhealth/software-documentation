---
title: Your First Local JupyterHealth Exchange
description: Get a working JupyterHealth Exchange running on your computer in 15 minutes
---

# Your First Local JupyterHealth Exchange

In this tutorial, we'll deploy JupyterHealth Exchange on your local computer. By the end, you'll have a working web application where you can explore organizations, studies, patients, and health observations.

## What We'll Build

We're going to set up a complete JupyterHealth Exchange that includes:

- A web interface running at http://localhost:8000
- Multiple example organizations (UC Berkeley, UCSF)
- Several research studies with different health data types
- Example patients with real blood pressure and heart rate data
- Different user types (administrators, researchers, patients)

This will take about 15 minutes. Let's get started!

## Before We Start

Make sure you have these installed on your computer:

- **Python 3.12** (recommended) - The project is configured for Python 3.12, though Django 5.2 also supports 3.10, 3.11, and 3.13. Check your version with `python --version` or `python3 --version`
- **PostgreSQL** - Version 12 or newer. Download from [postgresql.org](https://www.postgresql.org/download/)
- **Git** - Download from [git-scm.com](https://git-scm.com/downloads)
- **pip** - Should come with Python
- **A text editor** - Such as Notepad (Windows), TextEdit (Mac), VS Code, or Sublime Text

You'll also need to be comfortable running commands in a terminal.

**If you have multiple Python versions installed:**
The project's `Pipfile` specifies Python 3.12. If you have multiple Python versions, you may need to specify which one to use:

```bash
# Tell pipenv to use Python 3.12 specifically
PIPENV_PYTHON=3.12 pipenv install
# Or on Windows:
set PIPENV_PYTHON=3.12 && pipenv install
```

### Check if PostgreSQL is installed and running

After installing PostgreSQL, verify it's running:

**On Mac:**

```bash
brew services list | grep postgresql
```

If it says "started", you're good to go. If not, start it with:

```bash
brew services start postgresql
```

**On Windows:**
Check if the PostgreSQL service is running in Services (press Win+R, type `services.msc`, look for "postgresql").

**On Linux:**

```bash
sudo systemctl status postgresql
```

If it's not running, start it with:

```bash
sudo systemctl start postgresql
```

### Alternative: Using Docker for PostgreSQL

If you prefer not to install PostgreSQL system-wide, or if you want to avoid conflicts with an existing PostgreSQL installation, you can run PostgreSQL in a Docker container instead.

**Prerequisites for this option:**

- Docker Desktop installed ([docker.com](https://www.docker.com/products/docker-desktop))

**Start PostgreSQL in a container:**

```bash
docker run --name jhe-postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=jhe_dev \
  -e POSTGRES_USER=postgres \
  -p 5432:5432 \
  -d postgres:15
```

**What this does:**

- Creates a container named `jhe-postgres`
- Runs PostgreSQL 15
- Exposes it on localhost:5432 (same as a local installation)
- Creates the `jhe_dev` database automatically

**Create the jheuser:**

```bash
docker exec -it jhe-postgres psql -U postgres -d jhe_dev -c "CREATE USER jheuser WITH PASSWORD 'jhepassword';"
docker exec -it jhe-postgres psql -U postgres -d jhe_dev -c "GRANT ALL PRIVILEGES ON DATABASE jhe_dev TO jheuser;"
docker exec -it jhe-postgres psql -U postgres -d jhe_dev -c "GRANT ALL ON SCHEMA public TO jheuser;"
```

**To stop the container later:**

```bash
docker stop jhe-postgres
```

**To restart it:**

```bash
docker start jhe-postgres
```

**To remove it completely:**

```bash
docker stop jhe-postgres
docker rm jhe-postgres
```

If you use this Docker approach, your `.env` file should use `DB_HOST="localhost"` (which is already the default in `dot_env_example.txt`), and you can skip the rest of Step 2 below.

## Step 1: Set Up Your Python Environment

First, let's install pipenv, which will manage our Python dependencies.

**On Mac (with Homebrew):**

```bash
brew install pipenv
```

**On Linux/WSL:**

```bash
pip3 install --user pipenv
```

**On Windows:**

```bash
pip install --user pipenv
```

**If you get "externally-managed-environment" error** (common on macOS/Linux):

This is because of PEP 668 which protects your system Python from being corrupted. **Never use `--break-system-packages`** as it can damage your Python installation.

Instead, use one of these safe alternatives:

```bash
# Best option for Mac: Use Homebrew
brew install pipenv

# Best option for any platform: Use pipx (a tool designed for installing Python apps)
# First install pipx, then use it to install pipenv:
python3 -m pip install --user pipx
python3 -m pipx ensurepath
pipx install pipenv

# Alternative: Check if pipenv is already installed
which pipenv || echo "Not found"
```

Close and reopen your terminal after installing so pipenv is in your PATH.

Great! Now we have pipenv ready to manage our project dependencies.

## Step 2: Create Your Database

We need a PostgreSQL database for JupyterHealth Exchange to store all its data.

### Access PostgreSQL

We'll use the command line tool `psql` to create the database.

**First, try connecting with the default postgres user:**

```bash
psql -U postgres -d postgres
```

**If you get "role 'postgres' does not exist"** (common on Homebrew/Mac installs):

On Homebrew PostgreSQL, the default user is your username. Try this instead:

```bash
psql -d postgres
```

This will connect you as your current user. If this also fails, you might need to create the postgres role first:

```bash
createuser -s postgres  # Creates postgres superuser
psql -U postgres -d postgres
```

If this asks for a password, use the password you set during PostgreSQL installation. If you don't remember setting one, try leaving it blank (just press Enter).

If the command isn't found, you may need to add PostgreSQL to your PATH or use the full path (e.g., `C:\Program Files\PostgreSQL\15\bin\psql.exe -U postgres` on Windows).

### Create the Database

Once you're connected to PostgreSQL (you'll see a prompt like `postgres=#`), run these commands one at a time:

```sql
CREATE DATABASE jhe_dev;
CREATE USER jheuser WITH PASSWORD 'jhepassword';
    GRANT ALL PRIVILEGES ON DATABASE jhe_dev TO jheuser;
```

**For PostgreSQL 15 and newer**, you also need this command:

```sql
\c jhe_dev
GRANT ALL ON SCHEMA public TO jheuser;
```

Type `\q` and press Enter to exit psql.

**Alternative: Using command line directly**

If you prefer not to enter the interactive psql shell, you can run these commands directly:

```bash
psql -U postgres -c "CREATE DATABASE jhe_dev;"
psql -U postgres -c "CREATE USER jheuser WITH PASSWORD 'jhepassword';"
psql -U postgres -c "GRANT ALL PRIVILEGES ON DATABASE jhe_dev TO jheuser;"
psql -U postgres -d jhe_dev -c "GRANT ALL ON SCHEMA public TO jheuser;"
```

Perfect! We now have a database ready for our Exchange.

## Step 3: Get the JupyterHealth Exchange Code

Clone the repository and navigate into it:

```bash
git clone https://github.com/jupyterhealth/jupyterhealth-exchange.git
cd jupyterhealth-exchange
```

We're now in the jupyterhealth-exchange directory. Take a moment to look around - you'll see Django project files, a `core` directory with the main application code, and a `data` directory with example health data formats.

## Step 4: Configure Your Environment

JupyterHealth Exchange needs to know how to connect to your database and other settings. We'll create a configuration file:

```bash
cp dot_env_example.txt .env
```

Now open the `.env` file in your text editor:

- **Windows**: `notepad .env`
- **Mac**: `open -e .env`
- **Linux**: `nano .env` or `gedit .env`
- **Or** use VS Code: `code .env`

The example file already has sensible defaults for local development, but let's verify the database settings match what we created in Step 2:

```env
DB_NAME="jhe_dev"
DB_USER="jheuser"
DB_PASSWORD="jhepassword"
DB_HOST="localhost"
DB_PORT=5432
```

If you used different values when creating your database, update these lines to match.

### Generate a SECRET_KEY

> **⚠️ SECURITY WARNING:** The `dot_env_example.txt` file contains a `SECRET_KEY` value. **Do NOT use this example value!** Every installation must have its own unique SECRET_KEY.

Django uses the SECRET_KEY to protect session data, password reset tokens, and other security-critical features. If multiple installations share the same SECRET_KEY, attackers could potentially forge session tokens or compromise user data.

**Generate your own SECRET_KEY now:**

```bash
openssl rand -base64 32
```

This will output a random string like `xJ8kP2mN9qR5tY7wA3fD6hL4sG8bV1nC0zX5mK7jP9q=`

**Copy this value and update the SECRET_KEY line in your `.env` file:**

```env
SECRET_KEY="xJ8kP2mN9qR5tY7wA3fD6hL4sG8bV1nC0zX5mK7jP9q="
```

Replace the existing value with your newly generated key.

> **Note:** If you skip this step, JupyterHealth Exchange will auto-generate a random SECRET_KEY when it starts. However, this auto-generated key only works with a single server process. For consistency and to avoid issues when scaling up later, it's best to set your own SECRET_KEY now.

**About the other secrets in the file:**

Notice the file also contains OAuth keys (`OIDC_RSA_PRIVATE_KEY`, `PATIENT_AUTHORIZATION_CODE_CHALLENGE`, etc.). These are **demo keys for development only** - they're intentionally included in the example file to make this tutorial work smoothly with the seeded test data. In a real deployment, you would generate your own secure OAuth keys following the instructions in the [main README](https://github.com/jupyterhealth/jupyterhealth-exchange#getting-started).

Save and close the `.env` file. We're ready to start!

## Step 5: Install Dependencies

Let's install all the Python packages JupyterHealth Exchange needs:

```bash
pipenv install --deploy
```

The `--deploy` flag ensures you get the exact package versions from the lockfile, matching what the developers use.

**What to expect:**

- This will download and install Django, PostgreSQL drivers, OAuth libraries, and 20+ other packages
- You'll see lots of text scrolling by - this is normal!
- It might take 2-5 minutes depending on your internet connection
- The final message should say something like "Successfully installed..." with a list of packages

**If you see an error** about pipenv not being found, try closing and reopening your terminal, or install it globally:

```bash
pip install --user pipenv
```

**If you see a Python version mismatch error**, pipenv requires Python 3.12. Use the `PIPENV_PYTHON=3.12` approach mentioned in the prerequisites section.

**Important note about running commands:**

In the remaining steps, we'll run Python commands using `pipenv run python manage.py ...`. This automatically uses the correct Python environment without needing to activate a shell.

Alternatively, you can activate the Python environment once and then run commands directly:

```bash
pipenv shell
```

If you use `pipenv shell`, your command prompt will change to show `(jupyterhealth-exchange)` at the beginning, and you can then run commands like `python manage.py migrate` without the `pipenv run` prefix.

**For this tutorial, we'll use `pipenv run`** to be more explicit about what's happening.

## Step 6: Set Up the Database

Now we'll create all the database tables that JupyterHealth Exchange needs:

```bash
pipenv run python manage.py migrate
```

**What to expect:**

- You'll see output like "Running migrations..." followed by several lines showing "Applying core.0001_initial... OK"
- This creates tables for users, organizations, studies, patients, observations, and more
- Takes about 5-10 seconds
- The last line should say something like "Applying sessions.0001_initial... OK"

**If you see an error** about "connection refused" or "authentication failed", double-check:

1. PostgreSQL is running (see "Before We Start" section)
1. Your `.env` file has the correct database credentials
1. You ran the `GRANT ALL ON SCHEMA public TO jheuser;` command if using PostgreSQL 15+

## Step 7: Load Example Data

This is where it gets exciting! We'll populate the database with realistic example data:

```bash
pipenv run python manage.py seed
```

> **Note:** This tutorial follows the "quick start" approach from the repository README. The `seed` command automatically handles OAuth application setup, so we skip the manual OAuth configuration steps (8-12 in the README). The demo OAuth keys in your `.env` file are pre-configured to work with the seeded data.

**If you see a "duplicate key" error**, the database already has data from a previous run. You have two options:

1. **Flush and re-seed** (recommended for learning):

   ```bash
   pipenv run python manage.py seed --flush-db
   ```

   This will clear all existing data and start fresh.

1. **Create a new database** (if you want to preserve existing data):

   - Create a new database in PostgreSQL with a different name
   - Update `DB_NAME` in your `.env` file
   - Run `pipenv run python manage.py migrate` again
   - Then run the seed command

The `seed` command creates:

- Two university organizations (UC Berkeley and UCSF) with hierarchical sub-organizations
- Several research studies focusing on different health metrics
- Multiple user accounts (administrators, researchers, patients)
- Example patients enrolled in various studies
- Health observations including blood pressure and heart rate data
- All the necessary OAuth configuration

**What to expect:**

- You'll see output saying "Creating organizations...", "Creating studies...", etc.
- This takes about 10-20 seconds
- The final message should say something like "Database seeded successfully!"
- If you see any warnings (yellow text), that's usually okay - we're looking for success messages

When it finishes, we'll have a fully populated Exchange!

## Step 8: Start Your JupyterHealth Exchange

First, let's collect the static files (images, CSS, JavaScript) so they display properly:

```bash
pipenv run python manage.py collectstatic --noinput
```

This copies all static files to a single location where Django can serve them. You should see a message like "186 static files copied to '/path/to/staticfiles'."

Now let's start the web server:

```bash
pipenv run python manage.py runserver
```

You should see output like:

```
System check identified no issues (0 silenced).
October 24, 2025 - 15:30:00
Django version 5.2.0, using settings 'jhe.settings'
Starting development server at http://localhost:8000/
Quit the server with CONTROL-C.
```

**Important:** Keep this terminal window open! The server needs to keep running for the website to work. You'll use a web browser in the next step, but don't close this terminal.

🎉 **Congratulations!** Your JupyterHealth Exchange is now running!

## Step 9: Explore the Web Interface

Open your web browser and go to http://localhost:8000

You'll see the JupyterHealth Exchange login page.

### Log In as a Researcher

Let's log in as Mary, who is a research manager at UC Berkeley:

- **Email**: `mary@example.com`
- **Password**: `Jhe1234!`

After logging in, you'll see the Organizations page. Notice Mary has access to three organizations:

- University of California Berkeley
- College of Computing, Data Science and Society
- Berkeley Institute for Data Science (BIDS)

These are nested organizations - BIDS is part of the College, which is part of UC Berkeley. This hierarchical structure is typical of real research institutions.

### View Studies

Click on "Studies" in the navigation. You'll see research studies like:

- "BIDS Study on BP & HR" - collecting blood pressure and heart rate data
- "BIDS Study on BP" - collecting only blood pressure data

Click on one of the studies to see its details. You'll see which patients are enrolled and what types of data the study is collecting.

### View Patients

Click on "Patients" in the navigation. You'll see patients like Peter and Pamela from BIDS. Each patient has:

- Basic demographic information
- Study enrollments
- Consent status for each study

Click on a patient to see their details.

### View Observations

Click on a patient, then scroll down to see their health observations. You'll see real blood pressure and heart rate measurements with timestamps. This is actual health data in the Open mHealth format!

Notice how the data flows: Organizations contain Studies, Studies contain Patients, and Patients have Observations.

## Step 10: Explore as an Administrator

Let's see what a superuser can do. Log out (click your email in the top right, then "Logout").

Now log in as Sam, the system administrator:

- **Email**: `sam@example.com`
- **Password**: `Jhe1234!`

As a superuser, Sam can:

- View and edit all organizations
- Manage data sources (the devices and apps that generate health data)
- Access the Django admin panel at http://localhost:8000/admin

Try browsing to http://localhost:8000/admin to see Django's built-in administration interface. You can see all the database tables and manage everything from here.

## What You've Learned

🎉 **Well done!** You've successfully:

- ✅ Set up a complete JupyterHealth Exchange locally
- ✅ Populated it with realistic research data
- ✅ Explored the hierarchical organization structure
- ✅ Viewed research studies and enrolled patients
- ✅ Examined health observations in Open mHealth format
- ✅ Experienced different user roles (researcher and administrator)

You now understand the core data model:

```
Organizations (hierarchical)
    └── Studies (research projects)
        └── Patients (study participants)
            └── Observations (health data)
```

## Understanding the Example Data

The seed data created two complete research programs for you to explore:

**UC Berkeley Program:**

- BIDS runs two studies on cardiovascular health
- Patients Peter and Pamela are enrolled
- Data includes blood pressure and heart rate measurements

**UCSF Program:**

- Multiple cardiology labs (Moslehi Lab, Olgin Lab)
- Studies on temperature, oxygen saturation, and respiratory rate
- Different patients enrolled in different studies

You can explore all of this through the web interface!

## Next Steps

Now that you have a working JupyterHealth Exchange, you can:

1. **Experiment with the REST API** - JupyterHealth Exchange provides a full FHIR R5 API. Try the [API reference](../reference/exchange-apis.md) to learn how to fetch data programmatically.

1. **Upload your own data** - Create a new patient, generate an invitation link, and use the API to upload health observations.

1. **Learn about deployment** - When you're ready to deploy for real users, see our [Kubernetes deployment tutorial](exchange-on-kubernetes.md).

1. **Understand the architecture** - Read the [Exchange Overview](../explanation/exchange-overview.md) to learn how JupyterHealth Exchange works under the hood.

1. **Connect to JupyterHub** - JupyterHealth Exchange can work with JupyterHub to give researchers computational notebooks with access to health data.

## Troubleshooting

### "python: command not found" or "No module named django"

This usually means you forgot to use `pipenv run` or activate the pipenv environment. Make sure you're running commands with:

```bash
cd jupyterhealth-exchange
pipenv run python manage.py <command>
```

Or activate the environment first:

```bash
pipenv shell
# Then you can run: python manage.py <command>
```

If using `pipenv shell`, you should see `(jupyterhealth-exchange)` at the start of your command prompt.

### "Error: Your local changes to the following files would be overwritten"

If you cloned a branch other than main, switch to main:

```bash
git stash  # Save any local changes
git checkout main
```

### "FATAL: password authentication failed for user 'jheuser'"

Check your database settings in `.env` match the user and password you created in PostgreSQL. Common issues:

- The password in `.env` doesn't match what you used in `CREATE USER`
- The database name is wrong (should be `jhe_dev`)
- PostgreSQL isn't running (see "Before We Start" section)

### "permission denied for schema public"

This happens with PostgreSQL 15 and newer. Connect to your database and run:

```bash
psql -U postgres -d jhe_dev -c "GRANT ALL ON SCHEMA public TO jheuser;"
```

### "Browser shows blank screen after login"

This can happen on Windows. Check the [troubleshooting guide](https://github.com/jupyterhealth/jupyterhealth-exchange/blob/main/doc/troubleshooting-local-dev.md) for OIDC configuration fixes.

### "Port 8000 is already in use"

Another program is using port 8000. Either stop that program, or run Django on a different port:

```bash
pipenv run python manage.py runserver 8001
```

Then access the site at http://localhost:8001

## Stopping the Server

When you're done, stop the development server by pressing `Ctrl+C` in the terminal where `runserver` is running.

Your database and all the data will remain intact. Next time you want to start the server, just run:

```bash
cd jupyterhealth-exchange
pipenv run python manage.py runserver
```

Or if you prefer using the shell:

```bash
cd jupyterhealth-exchange
pipenv shell
python manage.py runserver
```

Happy exploring! 🎉
