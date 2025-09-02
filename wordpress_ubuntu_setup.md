# WordPress Setup on Ubuntu with Redis and MariaDB - Instructor Guide

## Prerequisites
- Fresh Ubuntu 20.04 or 22.04 server
- Root or sudo access
- Basic familiarity with command line

## Step 1: Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Required Packages

```bash
sudo apt install -y curl wget unzip software-properties-common apt-transport-https ca-certificates
```

## Step 3: Install and Configure MariaDB

### Install MariaDB Server
```bash
sudo apt install -y mariadb-server mariadb-client
```

### Secure MariaDB Installation
```bash
sudo mysql_secure_installation
```

**Follow these prompts:**
- Enter current password for root: *Press Enter (no password initially)*
- Set root password? **Y**
- Enter new root password: *Choose a strong password*
- Remove anonymous users? **Y**
- Disallow root login remotely? **Y**
- Remove test database? **Y**
- Reload privilege tables? **Y**

### Create WordPress Database and User
```bash
sudo mysql -u root -p
```

In the MariaDB shell, execute:
```sql
CREATE DATABASE wordpress_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'secure_password_here';
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Step 4: Install PHP and Required Extensions

### Add PHP Repository
```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
```

### Install PHP 8.2 and Extensions
```bash
sudo apt install -y php8.2 php8.2-fpm php8.2-mysql php8.2-curl php8.2-gd php8.2-xml php8.2-mbstring php8.2-zip php8.2-intl php8.2-bcmath php8.2-redis php8.2-imagick
```

### Configure PHP-FPM
```bash
sudo systemctl enable php8.2-fpm
sudo systemctl start php8.2-fpm
```

## Step 5: Install and Configure Nginx

### Install Nginx
```bash
sudo apt install -y nginx
```

### Create Nginx Configuration for WordPress (Multiple Domains)
```bash
sudo nano /etc/nginx/sites-available/wordpress
```

Add this configuration:
```nginx
server {
    listen 80;
    server_name csslosa.com www.csslosa.com csslosa.ng www.csslosa.ng comlagalumni.org www.comlagalumni.org comlagalumni.ng www.comlagalumni.ng;
    root /var/www/wordpress;
    index index.php index.html index.htm;

    # Redirect HTTP to HTTPS (will be uncommented after SSL setup)
    # return 301 https://$server_name$request_uri;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;

    # WordPress specific rules
    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_index index.php;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
    }

    # Cache static files
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|pdf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Deny access to sensitive files
    location ~ /\. {
        deny all;
    }

    location ~* /(?:uploads|files)/.*\.php$ {
        deny all;
    }

    # Let's Encrypt challenge location
    location ~ /.well-known {
        allow all;
    }
}
```

### Enable the Site
```bash
sudo ln -s /etc/nginx/sites-available/wordpress /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

## Step 6: Install and Configure Redis

### Install Redis Server
```bash
sudo apt install -y redis-server
```

### Configure Redis
```bash
sudo nano /etc/redis/redis.conf
```

Make these changes:
```
# Find and modify these lines:
maxmemory 256mb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000
```

### Restart Redis
```bash
sudo systemctl restart redis-server
sudo systemctl enable redis-server
```

### Test Redis Connection
```bash
redis-cli ping
```
*Should return: PONG*

## Step 7: Download and Install WordPress

### Create WordPress Directory
```bash
sudo mkdir -p /var/www/wordpress
cd /tmp
```

### Download WordPress
```bash
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo cp -r wordpress/* /var/www/wordpress/
```

### Set Proper Permissions
```bash
sudo chown -R www-data:www-data /var/www/wordpress/
sudo find /var/www/wordpress/ -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress/ -type f -exec chmod 644 {} \;
```

### Configure WordPress
```bash
cd /var/www/wordpress/
sudo cp wp-config-sample.php wp-config.php
sudo nano wp-config.php
```

Update database configuration:
```php
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'secure_password_here');
define('DB_HOST', 'localhost');
define('DB_CHARSET', 'utf8mb4');
define('DB_COLLATE', 'utf8mb4_unicode_ci');
```

Add Redis configuration before "/* That's all, stop editing! */" line:
```php
// Redis Configuration
define('WP_REDIS_HOST', '127.0.0.1');
define('WP_REDIS_PORT', 6379);
define('WP_REDIS_TIMEOUT', 1);
define('WP_REDIS_READ_TIMEOUT', 1);
define('WP_REDIS_DATABASE', 0);

// Security keys (generate from https://api.wordpress.org/secret-key/1.1/salt/)
define('AUTH_KEY',         'your-unique-phrase-here');
define('SECURE_AUTH_KEY',  'your-unique-phrase-here');
define('LOGGED_IN_KEY',    'your-unique-phrase-here');
define('NONCE_KEY',        'your-unique-phrase-here');
define('AUTH_SALT',        'your-unique-phrase-here');
define('SECURE_AUTH_SALT', 'your-unique-phrase-here');
define('LOGGED_IN_SALT',   'your-unique-phrase-here');
define('NONCE_SALT',       'your-unique-phrase-here');
```

## Step 8: Final WordPress File Permissions

### Ensure Proper Ownership and Permissions
```bash
sudo chown -R www-data:www-data /var/www/wordpress/
sudo find /var/www/wordpress/ -type d -exec chmod 755 {} \;
sudo find /var/www/wordpress/ -type f -exec chmod 644 {} \;
sudo chmod 600 /var/www/wordpress/wp-config.php
```

## Step 9: Install and Configure Let's Encrypt SSL Certificates

### Install Certbot
```bash
sudo apt install -y snapd
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
```

### Obtain SSL Certificates for All Domains
```bash
sudo certbot --nginx -d csslosa.com -d www.csslosa.com -d csslosa.ng -d www.csslosa.ng -d comlagalumni.org -d www.comlagalumni.org -d comlagalumni.ng -d www.comlagalumni.ng
```

**Follow the prompts:**
- Enter your email address for important account notifications
- Agree to the Terms of Service (A)
- Choose whether to share your email with EFF (Y/N)
- Certbot will automatically configure SSL and update your Nginx configuration

### Set Up Automatic SSL Renewal
```bash
sudo crontab -e
```

Add this line to renew certificates automatically:
```
0 12 * * * /usr/bin/certbot renew --quiet
```

### Verify SSL Configuration
After Certbot completes, it will automatically update your Nginx configuration. Verify the changes:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Update WordPress Configuration for HTTPS
```bash
sudo nano /var/www/wordpress/wp-config.php
```

Add these lines before "/* That's all, stop editing! */" line:
```php
// Force SSL for WordPress
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
    $_SERVER['HTTPS'] = 'on';
}

define('FORCE_SSL_ADMIN', true);

// Set proper WordPress URLs (replace with your primary domain)
define('WP_HOME','https://csslosa.com');
define('WP_SITEURL','https://csslosa.com');
```

## Step 10: Configure WordPress for Multiple Domains

### Install WordPress Multisite Domain Mapping (Optional)
If you want different content for different domains, you can set up WordPress Multisite. However, for the same content across all domains, the current setup will work.

### Add Domain Redirects (Optional)
If you want to redirect all domains to a primary domain (e.g., csslosa.com), add this to your Nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/wordpress-redirects
```

```nginx
# Redirect secondary domains to primary domain
server {
    listen 80;
    listen 443 ssl http2;
    server_name csslosa.ng www.csslosa.ng comlagalumni.org www.comlagalumni.org comlagalumni.ng www.comlagalumni.ng;
    
    ssl_certificate /etc/letsencrypt/live/csslosa.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/csslosa.com/privkey.pem;
    
    return 301 https://csslosa.com$request_uri;
}
```

Enable the redirect configuration:
```bash
sudo ln -s /etc/nginx/sites-available/wordpress-redirects /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## Step 11: Configure Firewall (UFW)

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
```

## Step 12: Final System Configuration

### Restart All Services
```bash
sudo systemctl restart nginx
sudo systemctl restart php8.2-fpm
sudo systemctl restart mariadb
sudo systemctl restart redis-server
```

### Verify Services Status
```bash
sudo systemctl status nginx
sudo systemctl status php8.2-fpm
sudo systemctl status mariadb
sudo systemctl status redis-server
```

## Step 13: Complete WordPress Setup

1. Navigate to **https://csslosa.com** (or any of your configured domains) in a web browser
2. **SUCCESS INDICATOR:** If you see the WordPress installation screen, your setup is working correctly! This confirms:
   - ✅ Nginx is serving the site properly
   - ✅ PHP 8.2 is processing requests
   - ✅ MariaDB database connection is established
   - ✅ All domains are accessible
   - ✅ SSL certificates are active (if configured)

3. Follow the WordPress installation wizard:
   - **Choose language**
   - **Enter site information:**
     - Site Title (can be changed later)
     - Username (**avoid "admin"** for security - use something unique)
     - Strong password (WordPress will suggest one)
     - Your email address
   - **Click "Install WordPress"**

4. **After successful installation**, you'll see a success message with login details

## Step 14: Post-Installation Verification

### Verify System Services
After WordPress installation, verify all services are running correctly:

```bash
# Check all services status
sudo systemctl status nginx php8.2-fpm mariadb redis-server

# Verify WordPress can connect to database
mysql -u wp_user -p wordpress_db -e "SELECT 'Database connection successful';"

# Test Redis connection
redis-cli ping
# Should return: PONG

# Check Redis memory usage
redis-cli info memory

# Test all configured domains
for domain in csslosa.com csslosa.ng comlagalumni.org comlagalumni.ng; do
    echo "Testing $domain:"
    curl -I "https://$domain" 2>/dev/null | head -1
done
```

### Performance Monitoring Commands
```bash
# Monitor system resources
htop

# Check PHP-FPM processes
sudo ps aux | grep php-fpm

# Monitor Nginx connections
sudo ss -tuln | grep :80
sudo ss -tuln | grep :443

# Check disk usage
df -h /var/www/wordpress/
```

## Step 15: Performance Optimization (Optional)

### Optimize PHP-FPM
```bash
sudo nano /etc/php/8.2/fpm/pool.d/www.conf
```

Adjust these values based on your server resources:
```
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests = 500
```

### Optimize MariaDB
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Add under [mysqld] section:
```
innodb_buffer_pool_size = 256M
innodb_log_file_size = 64M
query_cache_type = 1
query_cache_size = 32M
```

### Restart Services
```bash
sudo systemctl restart php8.2-fpm
sudo systemctl restart mariadb
```

## Step 16: Final System Verification

### Complete System Health Check
```bash
# Verify all services are running
sudo systemctl status nginx php8.2-fpm mariadb redis-server

# Test database connectivity
mysql -u wp_user -p wordpress_db -e "SHOW TABLES;" 2>/dev/null && echo "Database: ✅ Connected"

# Test Redis connectivity  
redis-cli ping > /dev/null && echo "Redis: ✅ Connected"

# Check SSL certificates status
sudo certbot certificates

# Verify domain resolution
for domain in csslosa.com csslosa.ng comlagalumni.org comlagalumni.ng; do
    echo -n "$domain: "
    dig +short $domain | head -1
done

# Check Nginx configuration syntax
sudo nginx -t

# Test PHP processing
echo "<?php phpinfo(); ?>" | sudo tee /var/www/wordpress/info.php > /dev/null
echo "PHP test file created at: https://csslosa.com/info.php"
echo "⚠️  Remove this file after testing: sudo rm /var/www/wordpress/info.php"
```

### Performance Baseline Commands
```bash
# Check current resource usage
echo "=== System Resources ==="
free -h
df -h /var/www/wordpress/
echo ""

echo "=== Redis Memory Usage ==="
redis-cli info memory | grep used_memory_human

echo ""
echo "=== Active Connections ==="
sudo ss -tuln | grep -E ':(80|443|3306|6379)'

echo ""
echo "=== PHP-FPM Process Count ==="
sudo ps aux | grep php-fpm | grep -v grep | wc -l
```

## Installation Complete! 🎉

### What You've Successfully Set Up:
- ✅ **Ubuntu Server** with latest security updates
- ✅ **MariaDB** database with WordPress-optimized configuration  
- ✅ **PHP 8.2** with all required extensions and Redis support
- ✅ **Nginx** web server with security headers and caching
- ✅ **Redis** server for object caching and performance
- ✅ **Let's Encrypt SSL** certificates for all 4 domains with auto-renewal
- ✅ **WordPress** core files with secure permissions
- ✅ **Multi-domain support** for csslosa.com, csslosa.ng, comlagalumni.org, comlagalumni.ng
- ✅ **Firewall protection** with UFW
- ✅ **System optimization** for production use

### Success Indicators:
1. **WordPress installation screen appears** when visiting any configured domain
2. **SSL certificates are active** (green padlock in browser)
3. **All system services are running** without errors
4. **Database connectivity confirmed** 
5. **Redis caching operational**

### Next Steps (WordPress Admin Level):
Once WordPress installation is completed through the web interface:
- Access wp-admin with your created credentials
- Install and configure Redis Object Cache plugin for optimal performance
- Install security plugins as needed
- Configure WordPress settings according to your requirements

### Maintenance Commands to Remember:
```bash
# Check system status
sudo systemctl status nginx php8.2-fpm mariadb redis-server

# Monitor performance
htop
redis-cli info memory

# SSL renewal (automatic, but manual check)
sudo certbot certificates

# Update system packages
sudo apt update && sudo apt upgrade -y
```

The server infrastructure is now production-ready and optimized for high-performance WordPress hosting across multiple domains.

## Important DNS Configuration Requirements

### Before Starting Installation
**CRITICAL:** Ensure all domains point to your server's IP address:

**A Records needed:**
- `csslosa.com` → Your Server IP
- `www.csslosa.com` → Your Server IP  
- `csslosa.ng` → Your Server IP
- `www.csslosa.ng` → Your Server IP
- `comlagalumni.org` → Your Server IP
- `www.comlagalumni.org` → Your Server IP
- `comlagalumni.ng` → Your Server IP
- `www.comlagalumni.ng` → Your Server IP

**Verify DNS propagation before proceeding:**
```bash
# Test each domain
nslookup csslosa.com
nslookup csslosa.ng  
nslookup comlagalumni.org
nslookup comlagalumni.ng
```

## Troubleshooting Tips

### Check Error Logs
- Nginx: `sudo tail -f /var/log/nginx/error.log`
- PHP: `sudo tail -f /var/log/php8.2-fpm.log`
- MariaDB: `sudo tail -f /var/log/mysql/error.log`
- SSL/Certbot: `sudo tail -f /var/log/letsencrypt/letsencrypt.log`

### Test Database Connection
```bash
mysql -u wp_user -p wordpress_db
```

### Test Redis Connection
```bash
redis-cli
> ping
> set test "hello"
> get test
> exit
```

### Verify PHP Extensions
```bash
php -m | grep -E "(redis|mysql|curl|gd)"
```

### Test SSL Certificates
```bash
sudo certbot certificates
```

### Check Domain Resolution
```bash
# Test if domains resolve to your server
dig +short csslosa.com
dig +short csslosa.ng
dig +short comlagalumni.org  
dig +short comlagalumni.ng
```

### SSL Certificate Troubleshooting
If SSL certificate generation fails:
```bash
# Check if domains are accessible
curl -I http://csslosa.com
curl -I http://csslosa.ng
curl -I http://comlagalumni.org
curl -I http://comlagalumni.ng

# Generate certificates one domain at a time if bulk fails
sudo certbot --nginx -d csslosa.com -d www.csslosa.com
sudo certbot --nginx -d csslosa.ng -d www.csslosa.ng
sudo certbot --nginx -d comlagalumni.org -d www.comlagalumni.org
sudo certbot --nginx -d comlagalumni.ng -d www.comlagalumni.ng
```

## Security Considerations

1. **Change default SSH port** (if applicable)
2. **SSL/TLS certificates configured** with Let's Encrypt auto-renewal
3. **Configure automatic security updates**
4. **Install security plugins** (Wordfence, iThemes Security)
5. **Regular backups** setup
6. **Keep all software updated**
7. **Configure Content Security Policy** headers
8. **Set up fail2ban** for brute force protection

### Additional Security Headers
Add to your Nginx configuration:
```bash
sudo nano /etc/nginx/sites-available/wordpress
```

Add these headers in the server block:
```nginx
# Additional Security Headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "camera=(), microphone=(), geolocation=()" always;
```

## Performance Monitoring

- Monitor Redis usage: `redis-cli info memory`
- Monitor database performance: `sudo mysql -e "SHOW PROCESSLIST;"`
- Monitor system resources: `htop` or `top`
- Check SSL certificate expiry: `sudo certbot certificates`

## WordPress Configuration Notes

### Multiple Domain Handling
The setup allows all domains to serve the same WordPress installation. WordPress will automatically handle requests from any of the configured domains.

### Primary Domain Setting
In `wp-config.php`, we set `csslosa.com` as the primary domain. You can change this to any of your domains if needed.

### SEO Considerations
- Consider using canonical URLs to avoid duplicate content issues
- Set up 301 redirects if you want all domains to redirect to one primary domain
- Use the WordPress SEO plugin to manage canonical URLs properly

---

**Important Notes:** 
- Replace domain names with your actual domains if different
- Ensure all domains have proper DNS A records pointing to your server
- SSL certificate generation requires domains to be accessible via HTTP first
- Test each domain individually after setup to ensure proper SSL configuration
