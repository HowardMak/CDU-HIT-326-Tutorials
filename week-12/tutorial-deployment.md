# CI/CD and Deployment

## Task List Summary

- [ ] **Setup GitHub Actions for CI/CD** - Create workflow file for automated testing
- [ ] **Configure Production Environment** - Update .env with production settings
- [ ] **Optimize for Production** - Cache configuration, routes, views, and events
- [ ] **Deploy to Shared Hosting (cPanel)** - Upload files, configure web root, set permissions
- [ ] **Deploy to Laravel Forge** - Connect server, create site, connect repository, deploy
- [ ] **Setup Queue Workers** - Install Supervisor and configure queue workers
- [ ] **Setup Scheduled Tasks** - Configure cron job for Laravel scheduler
- [ ] **Configure SSL Certificate** - Install Let's Encrypt SSL with auto-renewal
- [ ] **Setup Database Backup** - Create backup script and schedule daily backups
- [ ] **Add Deployment to GitHub Actions** - Automate deployment with GitHub Actions
- [ ] **Verify Deployment** - Test application, check logs, verify workers and scheduled tasks
- [ ] **Complete Post-Deployment Checklist** - Verify all functionality and configurations
- [ ] **Monitor Application** - Set up monitoring for application health and logs

## Step 1: Setup GitHub Actions for CI/CD

Create `.github/workflows/ci.yml` in your project root:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  tests:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: events_management_test
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
    - uses: actions/checkout@v3

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.1'
        extensions: mbstring, pdo_mysql, xml, ctype, json, bcmath

    - name: Copy .env
      run: php -r "file_exists('.env') || copy('.env.example', '.env');"

    - name: Install Dependencies
      run: composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist

    - name: Generate key
      run: php artisan key:generate

    - name: Directory Permissions
      run: chmod -R 777 storage bootstrap/cache

    - name: Execute tests
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: events_management_test
        DB_USERNAME: root
        DB_PASSWORD: password
      run: php artisan test
```

## Step 2: Configure Production Environment

### Update .env for Production

Create production `.env` file:

```env
APP_NAME="Events Management"
APP_ENV=production
APP_KEY=base64:your-generated-key-here
APP_DEBUG=false
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=events_management
DB_USERNAME=your_production_username
DB_PASSWORD=your_secure_password

CACHE_DRIVER=file
SESSION_DRIVER=file
QUEUE_CONNECTION=sync

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailtrap.io
MAIL_PORT=2525
MAIL_USERNAME=your_username
MAIL_PASSWORD=your_password
MAIL_FROM_ADDRESS=noreply@yourdomain.com
```

## Step 3: Optimize for Production

Run these commands before deployment:

```bash
# Install dependencies without dev packages
composer install --no-dev --optimize-autoloader

# Cache configuration
php artisan config:cache

# Cache routes
php artisan route:cache

# Cache views
php artisan view:cache

# Cache events
php artisan event:cache
```

## Step 4: Deploy to Shared Hosting (cPanel)

### Upload Files

1. Compress your project (excluding `node_modules`, `.git`, etc.)
2. Upload via FTP or cPanel File Manager
3. Extract in `public_html` or subdirectory

### Configure Web Root

Create `.htaccess` in root directory:

```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

### Set Permissions

Via cPanel File Manager or SSH:

```bash
chmod -R 755 storage
chmod -R 755 bootstrap/cache
```

### Run Commands via SSH

```bash
cd /path/to/your/project
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan storage:link
```

## Step 5: Deploy to Laravel Forge

### Step 1: Connect Server

1. Sign up at Laravel Forge
2. Add a new server (DigitalOcean, AWS, etc.)
3. Wait for server provisioning

### Step 2: Create Site

1. Click "Create Site"
2. Enter domain name
3. Select PHP version (8.1+)
4. Choose database (MySQL)

### Step 3: Connect Repository

1. Go to site settings
2. Connect Git repository
3. Set branch (main)
4. Configure deployment script

### Step 4: Deployment Script

Forge provides a default script. Update if needed:

```bash
cd /home/forge/yourdomain.com
git pull origin main
composer install --no-interaction --prefer-dist --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
php artisan queue:restart
```

### Step 5: Deploy

Click "Deploy Now" or push to main branch for automatic deployment.

## Step 6: Setup Queue Workers

### Install Supervisor

```bash
sudo apt-get update
sudo apt-get install supervisor
```

### Create Supervisor Config

Create `/etc/supervisor/conf.d/laravel-worker.conf`:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/path/to/storage/logs/worker.log
stopwaitsecs=3600
```

### Start Supervisor

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

## Step 7: Setup Scheduled Tasks

Add to crontab:

```bash
crontab -e
```

Add this line:

```bash
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

## Step 8: Configure SSL Certificate

### Using Let's Encrypt

```bash
sudo apt-get install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

### Auto-Renewal

Test renewal:

```bash
sudo certbot renew --dry-run
```

## Step 9: Setup Database Backup

Create backup script `backup-database.sh`:

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/path/to/backups"
DB_NAME="events_management"
DB_USER="your_username"
DB_PASS="your_password"

mysqldump -u $DB_USER -p$DB_PASS $DB_NAME > $BACKUP_DIR/backup_$DATE.sql

# Keep only last 7 days of backups
find $BACKUP_DIR -name "backup_*.sql" -mtime +7 -delete
```

Make executable:

```bash
chmod +x backup-database.sh
```

Add to crontab for daily backups:

```bash
0 2 * * * /path/to/backup-database.sh
```

## Step 10: Add Deployment to GitHub Actions

Update `.github/workflows/ci.yml` to include deployment:

```yaml
  deploy:
    needs: [tests]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
    - uses: actions/checkout@v3

    - name: Deploy to Server
      uses: appleboy/scp-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        source: "."
        target: "/var/www/events-management"

    - name: Run Deployment Commands
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.SSH_KEY }}
        script: |
          cd /var/www/events-management
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          php artisan queue:restart
```

### Setup GitHub Secrets

In GitHub repository:

1. Go to Settings → Secrets and variables → Actions
2. Add secrets:
   - `HOST` - Your server IP/domain
   - `USERNAME` - SSH username
   - `SSH_KEY` - Private SSH key

## Step 11: Verify Deployment

1. **Check Application:**

   - Visit your domain
   - Verify pages load correctly
   - Test authentication
   - Test CRUD operations
2. **Check Logs:**

   ```bash
   tail -f storage/logs/laravel.log
   ```
3. **Check Queue Workers:**

   ```bash
   sudo supervisorctl status
   ```
4. **Check Scheduled Tasks:**

   - Verify cron job is running
   - Check scheduled commands execute

## Step 12: Post-Deployment Checklist

- [ ] Application loads correctly
- [ ] Database connection works
- [ ] Authentication works
- [ ] File uploads work (storage link created)
- [ ] Queue workers running
- [ ] Scheduled tasks configured
- [ ] SSL certificate installed
- [ ] Error logging working
- [ ] Backups configured
- [ ] Monitoring setup (optional)

## Step 13: Monitor Application

### Check Application Health

```bash
# Check if application is running
curl -I https://yourdomain.com

# Check database connection
php artisan tinker
# Then: DB::connection()->getPdo();
```

### Monitor Logs

```bash
# View recent errors
tail -n 100 storage/logs/laravel.log | grep ERROR

# Monitor live
tail -f storage/logs/laravel.log
```

## Common Deployment Issues

### Issue: 500 Error After Deployment

**Solution:**

```bash
# Clear all caches
php artisan config:clear
php artisan route:clear
php artisan view:clear
php artisan cache:clear

# Check permissions
chmod -R 775 storage bootstrap/cache

# Check .env file
# Verify APP_DEBUG=false
# Verify APP_KEY is set
```

### Issue: Assets Not Loading

**Solution:**

```bash
# Create storage link
php artisan storage:link

# Check public/storage exists
ls -la public/storage
```

### Issue: Database Connection Failed

**Solution:**

- Verify database credentials in `.env`
- Check database server is running
- Verify network connectivity
- Check firewall rules
