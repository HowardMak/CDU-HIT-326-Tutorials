# Windows Environment Setup

## Task List Summary

- [ ] **Install PHP** - Choose XAMPP, WAMP, or standalone installation
- [ ] **Install MySQL** - Database server (included with XAMPP/WAMP or standalone)
- [ ] **Install Composer** - PHP dependency manager
- [ ] **Verify PHP Installation** - Check PHP version and extensions
- [ ] **Configure PHP Extensions** - Enable required extensions in php.ini
- [ ] **Verify MySQL Connection** - Test database connectivity
- [ ] **Create Laravel Project** - Install Laravel via Composer
- [ ] **Configure Database** - Update .env file and create database
- [ ] **Run Migrations** - Set up initial database tables
- [ ] **Start Development Server** - Verify Laravel installation works

## Prerequisites

- Windows 10 or later
- Administrator access to your computer
- Internet connection

## Step 1: Install PHP

### Option A: Using XAMPP (Recommended for Beginners)

1. Download XAMPP from [https://www.apachefriends.org/](https://www.apachefriends.org/)
2. Run the installer and select:
   - Apache
   - MySQL
   - PHP
   - phpMyAdmin
3. Install to `C:\xampp` (default location)
4. After installation, open XAMPP Control Panel
5. Start Apache and MySQL services

### Option B: Using WAMP

1. Download WAMP from [https://www.wampserver.com/](https://www.wampserver.com/)
2. Run the installer
3. Start WAMP server
4. Ensure both Apache and MySQL services are running (green icon)

### Option C: Standalone PHP Installation

1. Download PHP from [https://windows.php.net/download/](https://windows.php.net/download/)
2. Choose the Thread Safe version (ZIP file)
3. Extract to `C:\php`
4. Add PHP to your system PATH:
   - Right-click "This PC" → Properties → Advanced system settings
   - Click "Environment Variables"
   - Under "System variables", find "Path" and click "Edit"
   - Click "New" and add `C:\php`
   - Click "OK" to save

## Step 2: Install MySQL

### If using XAMPP/WAMP:

MySQL is included, but the `mysql` command may not be available in your terminal until you add it to PATH.

- XAMPP MySQL client is usually at `C:\xampp\mysql\bin\mysql.exe`
- WAMP MySQL client is usually under `C:\wamp64\bin\mysql\mysql8.x.x\bin\mysql.exe`

**Option A (recommended): Add MySQL to PATH**

1. Windows search: "Environment Variables" → "Edit the system environment variables"
2. Click "Environment Variables…"
3. Under "User variables", select "Path" → "Edit"
4. Click "New" and add the folder that contains `mysql.exe` (for example `C:\xampp\mysql\bin`)
5. Click OK to save, then close and reopen your terminal

**Option B: Run `mysql.exe` by full path**

Example (XAMPP):

```bash
"C:\xampp\mysql\bin\mysql.exe" --version
```

### Standalone MySQL Installation:

1. Download MySQL Installer from [https://dev.mysql.com/downloads/installer/](https://dev.mysql.com/downloads/installer/)
2. Choose "MySQL Installer for Windows"
3. Select "Developer Default" setup type
4. Complete the installation wizard
5. Set root password (remember this for later)
6. Configure MySQL to start as a Windows service
7. Ensure MySQL is on PATH so `mysql` works in a terminal (common folder: `C:\Program Files\MySQL\MySQL Server 8.0\bin`)
8. Verify the MySQL client works:
   ```bash
   mysql --version
   ```

## Step 3: Install Composer

1. Download Composer Windows Installer from [https://getcomposer.org/download/](https://getcomposer.org/download/)
2. Run the installer (`Composer-Setup.exe`)
3. The installer will:
   - Detect your PHP installation
   - Add Composer to your PATH
   - Install Composer globally
4. Verify installation by opening Command Prompt and running:
   ```bash
   composer --version
   ```

## Step 4: Verify PHP Installation

1. Open Command Prompt (cmd.exe)
2. Run:
   ```bash
   php -v
   ```
3. You should see PHP version information (PHP 8.1 or higher recommended)

## Step 5: Configure PHP Extensions

1. Locate your `php.ini` file:
   - XAMPP: `C:\xampp\php\php.ini`
   - WAMP: `C:\wamp64\bin\php\php8.x.x\php.ini`
   - Standalone: `C:\php\php.ini`
2. Open `php.ini` in a text editor
3. Uncomment (remove the `;`) the following extensions:
   ```
   extension=openssl
   extension=pdo_mysql
   extension=mbstring
   extension=fileinfo
   extension=zip
   ```
4. Save the file
5. Restart Apache (if using XAMPP/WAMP)

## Step 6: Verify MySQL Connection

1. Open Command Prompt
2. Test MySQL connection:
   ```bash
   mysql -u root -p
   ```
   If you see `'mysql' is not recognized...`, go back to Step 2 and add MySQL's `bin` folder to PATH (or run `mysql.exe` by full path).
3. Enter your MySQL root password
4. You should see the MySQL prompt
5. Type `exit` to leave MySQL

## Step 7: Install Laravel

1. Open Command Prompt
2. Navigate to your desired project directory:
   ```bash
   cd C:\Users\YourName\Documents
   ```
3. Create a new Laravel project:
   ```bash
   composer create-project laravel/laravel events-management
   ```
4. Wait for installation to complete (this may take a few minutes)

## Step 8: Configure Laravel Database

1. Navigate to your project directory:

   ```bash
   cd events-management
   ```
2. Copy the `.env.example` file to `.env`:

   ```bash
   copy .env.example .env
   ```
3. Generate application key:

   ```bash
   php artisan key:generate
   ```
4. Open `.env` file in a text editor
5. Update database configuration:

   ```env
   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=events_management
   DB_USERNAME=root
   DB_PASSWORD=your_mysql_password
   ```

## Step 9: Run Initial Migrations

1. In your project directory, run:
   ```bash
   php artisan migrate
   ```
2. If successful, you should see migration messages

## Step 10: Start Development Server

1. In your project directory, run:
   ```bash
   php artisan serve
   ```
2. Open your browser and navigate to: `http://localhost:8000`
3. You should see the Laravel welcome page

## Troubleshooting

### Composer not found

- Ensure Composer is in your PATH
- Restart Command Prompt after installation
- Try running: `C:\ProgramData\ComposerSetup\bin\composer.bat --version`

### PHP not found

- Verify PHP is in your system PATH
- Restart Command Prompt
- Check PHP installation: `where php`

### MySQL connection refused

- Ensure MySQL service is running
- Check MySQL is listening on port 3306
- Verify username and password in `.env`

### Permission denied errors

- Run Command Prompt as Administrator
- Check folder permissions
- Ensure Apache/MySQL services have proper permissions
- Introduction to Artisan commands

## Additional Resources

- [Laravel Installation Guide](https://laravel.com/docs/installation)
- [Composer Documentation](https://getcomposer.org/doc/)
- [MySQL Documentation](https://dev.mysql.com/doc/)
