# Project Deployment Guide

This project consists of a Laravel backend and React frontend for user management (CRUD operations).

## Server Information

- **Application Server**: `192.168.109.146` (user: pnc, password: 1234567)
- **Database Server**: `192.168.108.234` (user: pnc, password: 1234567)
- **Database Name**: `myproject_db`
- **Database User**: `dba`
- **Database Password**: `abc123`

## Database Schema

The `users` table has the following structure:
- `id` (int, auto_increment, primary key)
- `name` (varchar(100), not null)
- `email` (varchar(150), not null, unique)
- `password` (varchar(255), not null)
- `phone` (varchar(20), nullable)
- `address` (text, nullable)
- `role` (enum('admin','user'), default 'user')
- `created_at` (timestamp)
- `updated_at` (timestamp)

---

## Step-by-Step Deployment Instructions

### Step 1: Prepare the Application Server

SSH into your application server:

```bash
ssh pnc@192.168.109.146
# Enter password: 1234567
```

### Step 2: Install Required Software

#### Install PHP and Extensions

```bash
sudo apt update
sudo apt install -y php php-fpm php-cli php-common php-mysql php-zip php-gd php-mbstring php-curl php-xml php-bcmath
```

#### Install Composer

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```

#### Install Node.js and npm

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

#### Install Nginx

```bash
sudo apt install -y nginx
```

### Step 3: Upload Project Files

On your local machine, upload the project to the application server:

```bash
# From your local machine
scp -r C:\Users\NITA.CHROUN\Desktop\deploy\ pnc@192.168.109.146:/var/www/
```

Or use rsync for better performance:

```bash
rsync -avz C:\Users\NITA.CHROUN\Desktop\deploy/ pnc@192.168.109.146:/var/www/deploy/
```

### Step 4: Set Up Backend (Laravel)

SSH into the application server and navigate to the backend directory:

```bash
ssh pnc@192.168.109.146
cd /var/www/deploy/backend
```

#### Install Dependencies

```bash
composer install --no-dev --optimize-autoloader
```

#### Configure Environment

```bash
cp .env.example .env
nano .env
```

Update the following values in `.env`:

```env
APP_NAME="MyProject"
APP_ENV=production
APP_KEY=base64:YOUR_GENERATED_KEY_HERE
APP_DEBUG=false
APP_URL=http://192.168.109.146

DB_CONNECTION=mysql
DB_HOST=192.168.108.234
DB_PORT=3306
DB_DATABASE=myproject_db
DB_USERNAME=dba
DB_PASSWORD=abc123
```

Generate application key:

```bash
php artisan key:generate
```

#### Set Permissions

```bash
sudo chown -R www-data:www-data /var/www/deploy/backend
sudo chmod -R 755 /var/www/deploy/backend
sudo chmod -R 775 /var/www/deploy/backend/storage
sudo chmod -R 775 /var/www/deploy/backend/bootstrap/cache
```

#### Clear and Cache Configuration

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

### Step 5: Set Up Frontend (React)

Navigate to the frontend directory:

```bash
cd /var/www/deploy/frontend
```

#### Install Dependencies

```bash
npm install
```

#### Build for Production

```bash
npm run build
```

### Step 6: Configure Nginx

Create Nginx configuration file for the backend:

```bash
sudo nano /etc/nginx/sites-available/backend
```

Add the following configuration:

```nginx
server {
    listen 80;
    server_name 192.168.109.146;
    root /var/www/deploy/backend/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    index index.php index.html;

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Create a symlink to enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/backend /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
```

Test Nginx configuration:

```bash
sudo nginx -t
```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

### Step 7: Configure Nginx for Frontend (Optional - Separate Server)

If you want to serve the frontend separately, create another configuration:

```bash
sudo nano /etc/nginx/sites-available/frontend
```

Add the following configuration:

```nginx
server {
    listen 8080;
    server_name 192.168.109.146;
    root /var/www/deploy/frontend/dist;

    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable the frontend site:

```bash
sudo ln -s /etc/nginx/sites-available/frontend /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Step 8: Configure CORS (If Frontend and Backend on Different Ports)

Edit the Laravel CORS configuration:

```bash
cd /var/www/deploy/backend
nano config/cors.php
```

Update the paths and allowed origins:

```php
'paths' => ['api/*', 'sanctum/csrf-cookie'],

'allowed_methods' => ['*'],

'allowed_origins' => ['http://192.168.109.146:8080', 'http://192.168.109.146'],

'allowed_origins_patterns' => [],

'allowed_headers' => ['*'],

'exposed_headers' => [],

'max_age' => 0,

'supports_credentials' => false,
```

### Step 9: Update Frontend API URL

Update the API URL in the frontend code:

```bash
cd /var/www/deploy/frontend/src
nano UserCrud.jsx
```

Change the API_URL to your server IP:

```javascript
const API_URL = 'http://192.168.109.146/api/users'
```

Rebuild the frontend:

```bash
cd /var/www/deploy/frontend
npm run build
```

### Step 10: Test the Application

1. **Test Backend API**:
   ```bash
   curl http://192.168.109.146/api/users
   ```

2. **Test Frontend**:
   Open your browser and navigate to `http://192.168.109.146` (or `http://192.168.109.146:8080` if using separate frontend server)

3. **Test CRUD Operations**:
   - Create a new user
   - Read/View users
   - Update a user
   - Delete a user

---

## API Endpoints

The backend provides the following API endpoints for user management:

- `GET /api/users` - Get all users
- `POST /api/users` - Create a new user
- `GET /api/users/{id}` - Get a specific user
- `PUT /api/users/{id}` - Update a user
- `DELETE /api/users/{id}` - Delete a user

---

## Troubleshooting

### Database Connection Issues

If you encounter database connection errors, check:

1. Database server is accessible:
   ```bash
   mysql -h 192.168.108.234 -u dba -pabc123 -e "SHOW DATABASES;"
   ```

2. Firewall settings on database server:
   ```bash
   sudo ufw allow from 192.168.109.146 to any port 3306
   ```

### Permission Issues

If you encounter permission errors, run:

```bash
sudo chown -R www-data:www-data /var/www/deploy/backend
sudo chmod -R 755 /var/www/deploy/backend
sudo chmod -R 775 /var/www/deploy/backend/storage
```

### Nginx 502 Bad Gateway

Check if PHP-FPM is running:

```bash
sudo systemctl status php8.2-fpm
sudo systemctl start php8.2-fpm
```

### Frontend Not Loading

1. Check if the build was successful:
   ```bash
   ls -la /var/www/deploy/frontend/dist
   ```

2. Check Nginx error logs:
   ```bash
   sudo tail -f /var/log/nginx/error.log
   ```

---

## Maintenance Commands

### View Laravel Logs

```bash
tail -f /var/www/deploy/backend/storage/logs/laravel.log
```

### Clear Laravel Cache

```bash
cd /var/www/deploy/backend
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

### Restart Services

```bash
sudo systemctl restart nginx
sudo systemctl restart php8.2-fpm
```

---

## Security Notes

1. **Change default passwords**: Update the default passwords for both servers and database.
2. **Enable HTTPS**: Configure SSL certificates for production use.
3. **Firewall**: Configure firewall rules to restrict access.
4. **Environment variables**: Never commit `.env` files to version control.
5. **Regular updates**: Keep PHP, Node.js, and system packages updated.

---

## Quick Reference

### SSH Commands

```bash
# Application Server
ssh pnc@192.168.109.146

# Database Server
ssh pnc@192.168.108.234
```

### Database Access

```bash
mysql -h 192.168.108.234 -u dba -pabc123 myproject_db
```

### Project Locations

- Backend: `/var/www/deploy/backend`
- Frontend: `/var/www/deploy/frontend`

### Service Management

```bash
sudo systemctl status nginx
sudo systemctl restart nginx
sudo systemctl status php8.2-fpm
sudo systemctl restart php8.2-fpm
```
