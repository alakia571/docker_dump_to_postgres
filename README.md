# docker_dump_to_postgres

## how dump db from postgres container and resotre it out of docker in db 



To dump a PostgreSQL database from a Docker container and restore it to a PostgreSQL instance outside of Docker, you can follow these steps:

### Step 1: Dump the Database from the Docker Container

1. **Identify the Container and Database**: Make sure you know the name of the Docker container that is running PostgreSQL, the name of the database you want to dump, and the PostgreSQL user.

2. **Run the `pg_dump` Command**: Use the following command to dump the database from the Docker container:

   ```bash
   docker exec -t my_db_container pg_dump -U my_user -d my_database > my_database_dump.sql
   ```

   - `my_db_container`: The name or ID of your PostgreSQL Docker container.
   - `my_user`: The PostgreSQL user with access to the database.
   - `my_database`: The name of the database you want to dump.
   - `my_database_dump.sql`: The output file where the dump will be saved (you can name it whatever you like).

   Example:

   ```bash
   docker exec -t postgres_container pg_dump -U postgres -d mydb > mydb_dump.sql
   ```

   This command will create a `mydb_dump.sql` file in your current directory.

### Step 2: Copy the Dump File to Your Local Machine (if necessary)

If the dump file is created inside the Docker container, you might need to copy it to your local machine. However, in the previous example, the file is created directly on your local machine. If it's inside the container, use this command to copy it:

```bash
docker cp my_db_container:/path/to/my_database_dump.sql /local/path/to/save/
```

Replace `/path/to/my_database_dump.sql` with the actual path inside the container and `/local/path/to/save/` with the local directory where you want to save the dump file.

### Step 3: Restore the Dump to Your Local PostgreSQL Database

1. **Ensure PostgreSQL is Running Locally**: Make sure your local PostgreSQL service is running.

   ```bash
   sudo service postgresql start
   ```

   or

   ```bash
   systemctl start postgresql
   ```

2. **Create the Target Database**: If you don’t have a database already created, you can create a new one to restore the dump into:

   ```bash
   createdb -U your_local_user my_database
   ```

   Replace `your_local_user` with your local PostgreSQL username and `my_database` with the name of the database you want to restore into.

3. **Restore the Dump**: Use the `psql` command to restore the SQL dump into your local database:

   ```bash
   psql -U your_local_user -d my_database -f /path/to/my_database_dump.sql
   ```

   Example:

   ```bash
   psql -U postgres -d mydb -f mydb_dump.sql
   ```

   - `-U your_local_user`: The PostgreSQL user on your local machine.
   - `-d my_database`: The database you want to restore the dump into.
   - `-f /path/to/my_database_dump.sql`: The path to the dump file on your local machine.

### Example Workflow

1. **Dump the Database from Docker:**

   ```bash
   docker exec -t postgres_container pg_dump -U postgres -d mydb > mydb_dump.sql
   ```

2. **Create a New Database Locally:**

   ```bash
   createdb -U postgres mydb
   ```

3. **Restore the Dump Locally:**

   ```bash
   psql -U postgres -d mydb -f mydb_dump.sql
   ```

### Additional Notes:

- **Database Roles and Permissions**: If the dump includes roles or permissions, ensure that these roles exist in your local PostgreSQL instance before restoring, or modify the dump file accordingly.
  
- **File Paths**: Ensure you provide the correct paths if you are moving the dump file between directories or systems.

This process should successfully dump your PostgreSQL database from a Docker container and restore it to a PostgreSQL instance running on your local machine.




if psql: error: FATAL: Peer authentication failed for user "postgres":



The error `FATAL:  Peer authentication failed for user "postgres"` occurs because PostgreSQL is using "peer authentication" for the `postgres` user, which means it is trying to authenticate the user based on the operating system user name. If you're running the command as a user other than `postgres`, this can lead to the authentication failure.

### Ways to Fix the Peer Authentication Error:

#### Option 1: Use the `postgres` Linux User
If peer authentication is configured, you can switch to the `postgres` system user and then run the `psql` command:

1. Switch to the `postgres` user:

   ```bash
   sudo -i -u postgres
   ```

2. Run the `psql` command:

   ```bash
   psql -d mydb -f /path/to/mydb_dump.sql
   ```

   This should allow you to connect without the `Peer authentication failed` error.

#### Option 2: Modify PostgreSQL's `pg_hba.conf` File
If you want to use a different user or avoid switching to the `postgres` system user, you can modify the `pg_hba.conf` file to allow password authentication.

1. **Locate `pg_hba.conf`**: The `pg_hba.conf` file is typically located in the PostgreSQL data directory, which can vary depending on your installation. Common locations include:
   
   - `/etc/postgresql/<version>/main/pg_hba.conf` (Debian/Ubuntu)
   - `/var/lib/pgsql/data/pg_hba.conf` (CentOS/RHEL)
   - `/usr/local/pgsql/data/pg_hba.conf` (Custom builds)
   
   You can also determine its location by running:

   ```bash
   sudo -u postgres psql -c "SHOW hba_file;"
   ```

2. **Edit the `pg_hba.conf` File**:
   
   Open the `pg_hba.conf` file with a text editor, e.g., `nano`:

   ```bash
   sudo nano /etc/postgresql/<version>/main/pg_hba.conf
   ```
   
   Find the line that looks like this:

   ```plaintext
   local   all             postgres                                peer
   ```

   Change `peer` to `md5` (or `scram-sha-256` depending on your PostgreSQL version), so it looks like this:

   ```plaintext
   local   all             postgres                                md5
   ```

3. **Restart PostgreSQL**:

   After making changes to `pg_hba.conf`, restart the PostgreSQL service:

   ```bash
   sudo systemctl restart postgresql
   ```

4. **Use the `psql` Command with a Password**:
   
   Now, you can use the `psql` command with password authentication:

   ```bash
   psql -U postgres -d mydb -f /path/to/mydb_dump.sql
   ```

   You'll be prompted to enter the password for the `postgres` user.

#### Option 3: Use a Password with an Environment Variable
If you don't want to modify the `pg_hba.conf` file and are okay with using a password, you can use the `PGPASSWORD` environment variable:

```bash
export PGPASSWORD='your_password'
psql -U postgres -d mydb -f /path/to/mydb_dump.sql
```

This command sets the `PGPASSWORD` environment variable to your password for the duration of the session, allowing `psql` to connect without needing to modify the authentication method.

### Summary
- **Peer authentication** requires the operating system user to match the PostgreSQL user, which typically means running commands as the `postgres` system user.
- **Modifying `pg_hba.conf`** allows you to change the authentication method to password-based (`md5`), enabling you to log in with a password.
- **Using `PGPASSWORD`** allows you to pass the password inline without changing the authentication method.

Choose the method that best suits your needs.
