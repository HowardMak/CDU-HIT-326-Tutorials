# macOS Environment Setup

## Task List Summary

- [ ] **Install Homebrew** - Package manager for macOS
- [ ] **Install PHP 8.1+** - Verify PHP and required extensions
- [ ] **Install MySQL** - Database server and start service
- [ ] **Install Composer** - PHP dependency manager
- [ ] **Verify MySQL Connection** - Test database connectivity
- [ ] **Create Laravel Project** - Install Laravel via Composer
- [ ] **Configure Database** - Update .env file and create database
- [ ] **Run Migrations** - Set up initial database tables
- [ ] **Start Development Server** - Verify Laravel installation works

## Prerequisites

- macOS 10.15 (Catalina) or later
- Administrator access to your computer
- Internet connection
- Xcode Command Line Tools (will be installed automatically)

## Step 1: Install Homebrew

Homebrew is a package manager for macOS that makes installing development tools easier.

1. Open Terminal (Applications → Utilities → Terminal)
2. Install Homebrew by running:
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
3. Follow the on-screen instructions
4. If prompted, enter your administrator password
5. After installation, you may need to add Homebrew to your PATH. The installer will provide instructions if needed.

## Step 2: Install PHP

1. Open Terminal
2. Install PHP using Homebrew:
   ```bash
   brew install php
   ```
3. Verify installation:
   ```bash
   php -v
   ```
4. You should see PHP version 8.1 or higher

### Enable Required PHP Extensions

Laravel requires several PHP extensions. Most are included, but verify:

```bash
php -m | grep -E "openssl|pdo_mysql|mbstring|tokenizer|xml|ctype|json|bcmath|fileinfo"
```

All extensions should be listed. If any are missing, they're usually enabled by default in Homebrew PHP.

## Step 3: Install MySQL

1. Open Terminal
2. Install MySQL using Homebrew:

   ```bash
   brew install mysql
   ```
3. Start MySQL service:

   ```bash
   brew services start mysql
   ```
4. Secure MySQL installation (optional but recommended):

   ```bash
   mysql_secure_installation
   ```

   - Follow the prompts
   - Set a root password (remember this for later)
   - Answer "yes" to remove anonymous users, disallow root login remotely, and remove test database

### Alternative: Using MySQL via XAMPP

If you prefer XAMPP:

1. Download from [https://www.apachefriends.org/](https://www.apachefriends.org/)
2. Install the .dmg file
3. Start Apache and MySQL from XAMPP Control Panel

## Step 4: Install Composer

1. Open Terminal
2. Install Composer using Homebrew:
   ```bash
   brew install composer
   ```
3. Verify installation:
   ```bash
   composer --version
   ```

### Alternative: Manual Composer Installation

If Homebrew installation doesn't work:

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer
```

## Step 5: Verify MySQL Connection

1. Open Terminal
2. Test MySQL connection:
   ```bash
   mysql -u root -p
   ```
3. Enter your MySQL root password (or press Enter if no password set)
4. You should see the MySQL prompt
5. Type `exit` to leave MySQL

## Step 6: Install Laravel

1. Open Terminal
2. Navigate to your desired project directory:

   ```bash
   cd ~/Documents
   ```

   Or create a projects directory:

   ```bash
   mkdir -p ~/Projects
   cd ~/Projects
   ```
3. Create a new Laravel project:

   ```bash
   composer create-project laravel/laravel events-management
   ```
4. Wait for installation to complete (this may take a few minutes)

## Step 7: Configure Laravel Database

1. Navigate to your project directory:

   ```bash
   cd events-management
   ```
2. Copy the `.env.example` file to `.env`:

   ```bash
   cp .env.example .env
   ```
3. Generate application key:

   ```bash
   php artisan key:generate
   ```
4. Open `.env` file in a text editor:

   ```bash
   open -e .env
   ```

   Or use your preferred editor (VS Code, Sublime Text, etc.)
5. Update database configuration:

   ```env
   DB_CONNECTION=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_DATABASE=events_management
   DB_USERNAME=root
   DB_PASSWORD=your_mysql_password
   ```
6. Create the database in MySQL:

   ```bash
   mysql -u root -p
   ```

   Then in MySQL:

   ```sql
   CREATE DATABASE events_management;
   exit;
   ```

## Step 8: Run Initial Migrations

1. In your project directory, run:
   ```bash
   php artisan migrate
   ```
2. If successful, you should see migration messages

## Step 9: Start Development Server

1. In your project directory, run:
   ```bash
   php artisan serve
   ```
2. Open your browser and navigate to: `http://localhost:8000`
3. You should see the Laravel welcome page
4. To stop the server, press `Ctrl + C` in Terminal

## Troubleshooting

### Homebrew command not found

- Ensure Homebrew is properly installed
- Add to PATH if needed: `echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zshrc`
- Then run: `source ~/.zshrc`

### PHP version mismatch

- Check which PHP is being used: `which php`
- If using system PHP, use Homebrew PHP: `brew link php`
- Restart Terminal

### MySQL service won't start

- Check if MySQL is running: `brew services list`
- Start MySQL: `brew services start mysql`
- Check MySQL logs: `tail -f /usr/local/var/mysql/*.err` (or `/opt/homebrew/var/mysql/*.err` for Apple Silicon)

### Permission denied errors

- Use `sudo` only when necessary
- Check file permissions: `ls -la`
- Ensure you own the project directory

### Port 8000 already in use

- Use a different port: `php artisan serve --port=8001`
- Or find and kill the process using port 8000:
  ```bash
  lsof -ti:8000 | xargs kill
  ```

### Composer memory limit

If you encounter memory issues:

```bash
php -d memory_limit=-1 /usr/local/bin/composer install
```

## Useful Terminal Commands

- Check PHP version: `php -v`
- Check MySQL version: `mysql --version`
- Check Composer version: `composer --version`
- List running services: `brew services list`
- Stop MySQL: `brew services stop mysql`
- Restart MySQL: `brew services restart mysql`

## Additional Resources

- [Laravel Installation Guide](https://laravel.com/docs/installation)
- [Homebrew Documentation](https://docs.brew.sh/)
- [Composer Documentation](https://getcomposer.org/doc/)
- [MySQL Documentation](https://dev.mysql.com/doc/)
