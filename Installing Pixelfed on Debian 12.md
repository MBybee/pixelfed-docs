# Installing Pixelfed on Debian 12 with Postgres 18

Several things are taken as "understood" for this guide. 

* You're running this guide as root. Services will all be rootless.
* The test site "test.com" should be replaced with your site name.
* You have already registered a site and pointed it to your host.
* You are using Nginx, UFW and Fail2Ban or their equiv, and are familiar with their config.
* You have read the official guide. This one doesn't give many "whys", just mostly shorthand steps.

### Prep


1. Prereq installation 

   I'm using nginx to proxy my site, but it's up to you. Regardless, next up you need to configure your site and your certbot. These instructions below are for nginx.
   You'll also need PHP 8.3, if that's not in your apt repo, there's guides: https://linuxcapable.com/how-to-install-php-8-3-on-debian-linux/

   1. Install nginx and certbot

      ```bash
      apt install nginx certbot python3-certbot-nginx
      systemctl stop nginx # Because it just instantly starts up
      ```

   2. Set up your basic nginx config - remember to only have 80 in here initially

      ```bash
      vi /etc/nginx/sites-enabled/default
      cat /etc/nginx/sites-enabled/default |grep -v "#" |grep -v "^$"
      # Cloudflare sends the real client IP in this header
      real_ip_header CF-Connecting-IP;
      limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m; # Rate limiting
      server {
          listen 80; listen [::]:80;
          server_name _ test.com www.test.com;
          root /var/www/pixelfed; # Just a symlink to /data/pixelfed/public to be tidy
          index index.php;
          client_max_body_size 10m;
          add_header X-Robots-Tag "noai, noimageai";
          add_header X-Frame-Options "SAMEORIGIN";
          add_header X-Content-Type-Options "nosniff";
          location = /favicon.ico { access_log off; log_not_found off; } 
          
          # Block AI scrapers
          if ($http_user_agent ~* "(AdsBot-Google|Amazonbot|anthropic-ai|Applebot|Applebot-Extended|AwarioRssBot|AwarioSmartBot|Bytespider|CCBot|ChatGPT-User|ClaudeBot|Claude-Web|cohere-ai|DataForSeoBot|Diffbot|FacebookBot|FriendlyCrawler|Google-Extended|GoogleOther|GPTBot|img2dataset|ImagesiftBot|magpie-crawler|Meltwater|omgili|omgilibot|peer39_crawler|peer39_crawler/1.0|PerplexityBot|PiplBot|scoop.it|Seekr|YouBot)")
          {
              return 403;
          }
          location ~ /\.(?!well-known).* {
              deny all; # Block unusual stuff like keys from being directly accessible
          }
          location /login {
          		limit_req zone=login burst=10 nodelay; # Rate limit bad logins
          		try_files $uri $uri/ /index.php?$query_string;
          		access_log on;
      		}
          location / {
          	try_files $uri $uri/ /index.php?$query_string;
          }
          location ~ \.php$ {
      			#include fastcgi_params;
      			include fastcgi.conf;
      			fastcgi_pass unix:/run/php/pixelfed.sock;
      			fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
      			fastcgi_param PATH_INFO $fastcgi_path_info;
      			fastcgi_param HTTPS on;
         }
         location ~* \.(jpg|jpeg|gif|png|css|js|ico|webp|avif|svg)$ {
         		expires 30d;
         		access_log off;
         }
      }
      
      # Test the config
      nginx -test
      # Restart nginx
      systemctl start nginx
      # Allow http/https through the firewall for certbot
      ufw allow http && ufw allow https
      ```
   
      Obviously, if you *want* AI crawlers, you'd not keep the scraper blocking there.
   
   3. Get your certificate
   
      ```
      certbot --nginx -d test.com
      ```
      You probably will want to clean up the mess it made of your nginx files, but that's up to you.
   
   
      4. Now we go through lock it back down. At this point, there's no services or user whatsoever on your instance. Lock down your UFW again if you like. In my case, I only permit access from my subnet while I work on the host.
   
            ```bash
            ufw status numbered
            Status: active
                 To                         Action      From
                 --                         ------      ----
            [ 1] 22                         ALLOW IN    73.26.0.0/16              
            [ 2] 443                        ALLOW IN    Anywhere                  
            [ 3] 443 (v6)                   ALLOW IN    Anywhere (v6) 
            
            # Remove the global rules for HTTPS
            ufw delete 3
            ufw delete 2 
            # Replace it with a subnet rule
            ufw allow from 73.26.0.0/16 to any port 443
            ufw status
            Status: active
            To                         Action      From
            --                         ------      ----
            22                         ALLOW       73.26.0.0/16              
            443                        ALLOW       73.26.0.0/16              
            
            ```
   


   6. Now let's verify your site is up, and that we can connect. It won't go anywhere yet.

      ```bash
      # This should just show a valid ssl connection to the server, since that's all you're testing
      curl https://test.com
      ```

7. Time to install your database. In my case, I'm a postgres fan, so that's what I'm installing here. It also supports mariadb.

   ```bash
   apt install -y postgresql-common
   /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
   apt update && apt upgrade
   apt install postgresql-18 postgresql-18-pgvector
   # Creating new PostgreSQL cluster 18/main ...
   # /usr/lib/postgresql/18/bin/initdb -D /var/lib/postgresql/18/main --auth-local peer --auth-host scram-sha-256 --no-instructions
   # The files belonging to this database system will be owned by user "postgres".
   # This user must also own the server process.
         
   ```

   Now that postgres18 is installed and running as non-root, we add the pgvector package and set up the pixelfed user

   ```bash
   # Create the password you're going to use for pixelfed on postgres
   echo "POSTGRES_PIXELFED_PASSWORD="`openssl rand -base64 32` >> /data/.db_env
   chmod 640 /data/.db_env
   sudo -u postgres psql
   postgres=# create extension vector;
   postgres=# create user pixelfed createdb;
   postgres=# alter user pixelfed with password '(your password)';
   postgres=# SELECT * FROM pg_extension WHERE extname = 'vector';
     oid  | extname | extowner | extnamespace | extrelocatable | extversion | extconfig | extcondition 
   -------+---------+----------+--------------+----------------+------------+-----------+--------------
    16388 | vector  |       10 |         2200 | t              | 0.8.2      |           | 
   (1 row)
   postgres=# \du
                                List of roles
    Role name |                         Attributes                         
   -----------+------------------------------------------------------------
    pixelfed  | Create DB
    postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS
   ```

8. Install the giant pile of dependencies

   ```bash
   apt install libheif-dev php8.3 php8.3-fpm jpegoptim optipng pngquant php8.3-gd git composer ffmpeg redis php8.3-bcmath php8.3-curl php8.3-redis php8.3-xml php8.3-zip php8.3-pgsql php8.3-intl php8.3-imagick php8.3-mbstring php8.3-fileinfo
   
   # Set a default password for redis - just in case
   echo "REDIS_PIXELFED_PASSWORD="`openssl rand -base64 32` >> /data/.db_env
   # Confirm Redis is also running non-root, and has the port and socket expected for pixelfed
   grep -E "daemon|sock|port|requirepass" /etc/redis/redis.conf |grep -v "#"
   daemonize yes
   port 0
   unixsocket /run/redis/redis.sock
   unixsocketperm 770
   daemonize yes
   requirepass (your password)
   
   # Check your php-fpm modules against the required set: https://pixelfed.github.io/docs-next/running-pixelfed/prerequisites.html
   php8.3-fpm -m 
   ```

9. Check your ports, make sure everything is set up and nothing is exposed

   ```bash
   netstat -an|grep LIST|head -20
   tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN     
   tcp        0      0 127.0.0.1:5432          0.0.0.0:*               LISTEN     
   tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN     
   tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN
   ```
   
This shows I've got 80, 443, and 22 listening (that's HTTP, HTTPS, and SSH) so those will be controlled by UFW. 
   
Everything else is running only on localhost. The prep is done, the rest of this will be specific to pixelfed.


### Pixelfed Install


1. Create the Pixelfed userid

   ```bash
   useradd -rU -s /bin/bash pixelfed # Create the user
   usermod -aG redis pixelfed # Add it to the redis group for the socket
   usermod -aG pixelfed www-data # Add nginx to the pixelfed group for the socket
   ```

2. Configure the php setting - this will need to match your nginx sizing as well, so make sure both are updated.

   Set like this for higher security, but minimal image optimizing, as it will disable jpegoptim: 

   ​	*disable_functions = exec,passthru,shell_exec,system,proc_open,popen*

   ```bash
   grep -v ";" /etc/php/8.3/fpm/php.ini |grep -v "^$" |grep -E "post_max|max_file|file_|max_exec|disable_func"
   disable_functions = passthru,system,proc_open,popen
   max_execution_time = 600
   post_max_size = 15M
   file_uploads = On
   upload_max_filesize = 15M
   max_file_uploads = 20
   ```
   And now configure the PHP socket:

   ```bash
   cat /etc/php/8.3/fpm/pool.d/pixelfed.conf 
   [pixelfed]
   user = pixelfed
   group = pixelfed
   listen = /run/php/pixelfed.sock
   listen.owner = pixelfed
   listen.group = pixelfed
   listen.mode = 0660
   pm = dynamic
   pm.max_children = 5
   pm.start_servers = 2
   pm.min_spare_servers = 1
   pm.max_spare_servers = 3
   php_flag[display_errors] = off
   php_admin_value[open_basedir] = /data/pixelfed:/tmp
   
   # And restart the services
   systemctl restart php8.3-fpm
   systemctl status php8.3-fpm
   
   ls -l /run/php/ 
   total 4
   lrwxrwxrwx 1 root     root     30 Apr  1 11:46 php-fpm.sock -> /etc/alternatives/php-fpm.sock
   -rw-r--r-- 1 root     root      6 Apr  2 16:03 php8.3-fpm.pid
   srw-rw---- 1 www-data www-data  0 Apr  2 16:03 php8.3-fpm.sock
   srw-rw---- 1 pixelfed pixelfed  0 Apr  2 16:03 pixelfed.sock
   
   ```

3. Now fetch the actual app from github and set the permissions

   ```bash
   mkdir /data
   chown pixelfed:pixelfed /data
   cd /data
   git clone -b dev https://github.com/pixelfed/pixelfed.git pixelfed
   cd /data/pixelfed
   chown -R pixelfed:pixelfed /data/pixelfed/*
   find /data/pixelfed/ -type d -exec chmod 755 {} \;
   find /data/pixelfed/ -type f -exec chmod 644 {} \;
   ln -s /data/pixelfed/public/ /var/www/pixelfed
   ```

4. Next up, composer as the pixelfed user

   ```bash
   su - pixelfed
   cd /data/pixelfed
   # The --no-dev may not be stable for pixelfed yet
   composer install --no-dev --no-ansi --no-interaction --optimize-autoloader
   Installing dependencies from lock file (including require-dev)
   Verifying lock file contents can be installed on current platform.
   Package operations: 194 installs, 0 updates, 0 removals
     - Downloading php-http/discovery (1.20.0)
   <...>
     - Installing php-http/discovery (1.20.0): Extracting archive
   <...>
   Generating optimized autoload files
   Class App\Rules\WebFinger located in ./app/Rules/Webfinger.php does not comply with psr-4 autoloading standard. Skipping.
   > Illuminate\Foundation\ComposerScripts::postAutoloadDump
   > @php artisan package:discover --ansi
      INFO  Discovering packages.  
     buzz/laravel-h-captcha ................................................ DONE
   
   116 packages you are using are looking for funding.
   Use the `composer fund` command to find out more!
   
   # You should probably look into funding some of them!
   # Ok, so this is done now.
   ```

5. Now we configure the install
   ```bash
   sudo chmod -R 770 storage bootstrap/cache
   sudo chown -R pixelfed:pixelfed storage bootstrap/cache
   sudo su - pixelfed
   pixelfed@www:/data/pixelfed$ chmod 640 .env
   pixelfed@www:/data/pixelfed$ cat .env
   APP_NAME="Your Site"
   APP_ENV="production"
   APP_KEY=
   APP_DEBUG="false"
   
   # Instance Configuration
   OPEN_REGISTRATION="false"
   ENFORCE_EMAIL_VERIFICATION="true"
   PF_MAX_USERS="100"
   PF_ENFORCE_MAX_USERS="true"
   OAUTH_ENABLED="true"
   ENABLE_CONFIG_CACHE="true"
   INSTANCE_DISCOVER_PUBLIC="true"
   INSTANCE_PUBLIC_LOCAL_TIMELINE="true"
   INSTANCE_CONTACT_EMAIL="info@test.com"
   APP_TIMEZONE="UTC"
   ACCOUNT_DELETION="true"
   ACCOUNT_DELETE_AFTER="false"
   
   # Media Configuration
   PF_OPTIMIZE_IMAGES="true"
   PF_OPTIMIZE_VIDEOS="true"
   IMAGE_QUALITY="80"
   MAX_PHOTO_SIZE="15000"
   MAX_CAPTION_LENGTH="1000"
   MAX_BIO_LENGTH="250"
   MAX_NAME_LENGTH="30"
   MAX_ALBUM_LENGTH="4"
   MAX_AVATAR_SIZE="2000"
   CUSTOM_EMOJI="true"
   PF_IMPORT_FROM_INSTAGRAM="true"
   PF_IMPORT_IG_MAX_POSTS="1000"
   PF_IMPORT_IG_ALLOW_VIDEO_POSTS="true"
   PF_IMPORT_IG_PERM_MIN_ACCOUNT_AGE="3"
   MEDIA_EXIF_DATABASE="false"
   IMAGE_DRIVER="imagick"
   ATOM_FEEDS="true"
   
   # Instance URL Configuration
   APP_URL="https://test.com"
   APP_DOMAIN="test.com"
   ADMIN_DOMAIN="test.com"
   SESSION_DOMAIN="test.com"
   #TRUST_PROXIES="*"
   TRUST_PROXIES="null"
   
   # Database Configuration
   DB_CONNECTION="pgsql"
   DB_HOST="127.0.0.1"
   DB_PORT="5432"
   DB_DATABASE="pixelfed"
   DB_USERNAME="pixelfed"
   DB_PASSWORD="(your password)"
   
   # Redis Configuration
   REDIS_CLIENT="predis"
   REDIS_SCHEME="unix"
   # Modern Laravel syntax (phpredis + predis both support this)
   REDIS_PATH="/run/redis/redis.sock"
   # Fallback
   REDIS_HOST="/run/redis/redis.sock"
   REDIS_PASSWORD="(your password)"
   REDIS_PORT="0"
   
   # Laravel Configuration
   SESSION_DRIVER="database"
   CACHE_DRIVER="redis"
   QUEUE_DRIVER="redis"
   BROADCAST_DRIVER="log"
   LOG_CHANNEL="stack"
   HORIZON_PREFIX="horizon-"
   
   # ActivityPub Configuration
   ACTIVITY_PUB="false"
   AP_REMOTE_FOLLOW="false"
   AP_INBOX="false"
   AP_OUTBOX="false"
   AP_SHAREDINBOX="false"
   
   # Experimental Configuration
   EXP_EMC="true"
   
   ## Mail Configuration (Post-Installer) - no mail configured yet
   MAIL_DRIVER=log
   MAIL_MAILER=log
   MAIL_HOST=smtp.mailtrap.io
   MAIL_PORT=2525
   MAIL_USERNAME=null
   MAIL_PASSWORD=null
   MAIL_ENCRYPTION=null
   MAIL_FROM_ADDRESS="info@test.com"
   MAIL_FROM_NAME="YourSite"
   MAIL_AUTO_TLS=true  # STARTTLS (false to bypass)
   
   ## S3 Configuration (Post-Installer)
   PF_ENABLE_CLOUD=false
   FILESYSTEM_CLOUD=s3
   #AWS_ACCESS_KEY_ID=
   #AWS_SECRET_ACCESS_KEY=
   #AWS_DEFAULT_REGION=
   #AWS_BUCKET=<BucketName>
   #AWS_URL=
   #AWS_ENDPOINT=
   #AWS_USE_PATH_STYLE_ENDPOINT=false
   ## Optional (WHEN_SUPPORTED (default) / WHEN_REQUIRED) https://docs.aws.amazon.com/sdkref/latest/guide/feature-dataintegrity.html
   # AWS_REQUEST_CHECKSUM_CALCULATION=WHEN_SUPPORTED
   # AWS_RESPONSE_CHECKSUM_VALIDATION=WHEN_SUPPORTED
   
   ```

6. First run

   ```bash
   pixelfed@www:/data/pixelfed$ php artisan key:generate
   php artisan storage:link
   php artisan migrate --force
     INFO  Running migrations.  
     2014_10_12_000000_create_users_table ................................................... 13.28ms DONE
     <... huge list ...>
     2026_03_15_032554_fix_passport_keys ..................................................... 0.87ms DONE
   
   php artisan import:cities
   php artisan instance:actor
   php artisan passport:keys
   php artisan route:cache
   php artisan view:cache
   
   # To clear these caches/routes later, if required:
   # php artisan route:clear
   # php artisan view:clear
   # php artisan config:clear
   ```

7. Create the service to run the queue (choosing the lighter option, though Horizon has more features, it uses way more RAM)

   ```bash
   root@www:/data/pixelfed# cat /etc/systemd/system/pixelfed.service 
   [Unit]
   Description=Pixelfed Queue Worker
   After=network.target redis-server.service postgresql.service nginx.service
   Requires=redis-server.service
   Requires=postgresql.service
   Requires=nginx.service
   
   [Service]
   Type=simple
   User=pixelfed
   WorkingDirectory=/data/pixelfed
   ExecStart=/usr/bin/php /data/pixelfed/artisan queue:work --queue=high,default,email,feed,inbox,follow,pushnotify,mmo,move,shared,story,groups,intbg,low,adelete,delete --sleep=3 --tries=3 --timeout=90
   Restart=on-failure
   RestartSec=5
   
   [Install]
   WantedBy=multi-user.target
   
   root@www:/data/pixelfed# systemctl daemon-reload
   root@www:/data/pixelfed# systemctl enable --now pixelfed.service
   	Created symlink /etc/systemd/system/multi-user.target.wants/pixelfed.service → /etc/systemd/system/pixelfed.service.
   
   root@www:/data/pixelfed# systemctl status pixelfed.service
   	● pixelfed.service - Pixelfed Queue Worker
        	Loaded: loaded (/etc/systemd/system/pixelfed.service; enabled; preset: enabled)
        	Active: active (running) since Fri 2026-04-03 10:51:04 MDT; 18s ago
      	Main PID: 1462 (php)
        	CGroup: /system.slice/pixelfed.service
                	└─1462 /usr/bin/php /data/pixelfed/artisan queue:work --sleep=3 --tries=3 --timeout=90
   	Apr 03 10:51:04 www.werebooks.com systemd[1]: Started pixelfed.service - Pixelfed Queue Worker.
   ```

8. Cron job schedule (as pixelfed user):

   ```bash
   pixelfed@www:~$ crontab -l
   */2 * * * * /usr/bin/php /data/pixelfed/artisan schedule:run >> /dev/null 2>&1
   ```

9. Create the admin account

   ```bash
   pixelfed@www:/data/pixelfed$ php artisan user:create
   ```

   Supply all the relevant info, make the user an admin, and manually verify the email address.

10. Go log in with this user, check the settings and descriptions and make sure everything is working properly. 

11. You're pretty well done here, just set up your email, storage, and decide on federation and such. You can switch the user to nologin, and give the server a nice offline backup.
    ```bash
    root@www:~# usermod -s /usr/sbin/nologin pixelfed
    # Logging in now just provide a shell:
    su -s /bin/bash pixelfed
    ```

---

### Post-config improvements

1. Adding a Fail2Ban config for pixelfed

   Pixelfed currently doesn't have really verbose login failure logging. There is a common pattern for failed logins, however, and we'll filter based on that. You get a "POST /login -> 302 -> GET /login -> 200" vs the successful login which is "POST /login -> 302 -> GET /i/web -> 200".

   What we have to do is look for that pattern.

   ```bash
   cat /etc/fail2ban/filter.d/pixelfed-auth.conf 
   [Definition]
   failregex = ^<HOST> .* "POST /login HTTP/.*" 302 .* "https://test\.com/login"
   ignoreregex =
   
   grep -v "#" /etc/fail2ban/jail.local |grep -v "^$" |head -65
   ...
   [pixelfed-auth]
   enabled  = true
   filter   = pixelfed-auth
   logpath  = /var/log/nginx/access.log
   maxretry = 5
   findtime = 60
   bantime  = 3600
   
   # If you want to test:
   fail2ban-regex /var/log/nginx/access.log /etc/fail2ban/filter.d/pixelfed-auth.conf
   # Then activate
   systemctl reload fail2ban
   
   # Test it from A DIFFERENT IP (like a phone that's not on your wifi)
   fail2ban-client status pixelfed-auth
   Status for the jail: pixelfed-auth
   |- Filter
   |  |- Currently failed:	0
   |  |- Total failed:	5
   |  `- File list:	/var/log/nginx/access.log
   `- Actions
      |- Currently banned:	1
      |- Total banned:	1
      `- Banned IP list:	172.59.186.0
   
   ```

   

2. If you want to get better image metadata and quality (protect privacy, keep key fields)

   1. Make sure you're allowing php to execute things (disable_functions = passthru,system,proc_open,popen)

   2. Copy in the jpegoptim, because it lives outside of the "safe zone" for php, and install exiftool

      ```bash
      cp /usr/bin/jpegoptim /data/pixelfed/vendor/bin/
      chown pixelfed:pixelfed /data/pixelfed/vendor/bin/jpegoptim
      apt install exiftool
      ```

   3. Enable metadata pass-thru, since by default it removes everything
      Set /data/pixelfed/.env:

      ```bash
      IMAGE_STRIP="false"
      ```
      
      Flush the cache and restart all the things.

      ```bash
      cd /data/pixelfed && php artisan config:clear && php artisan cache:clear && php artisan view:clear && php artisan route:clear && php artisan config:cache && php artisan route:cache && php artisan view:cache
      (exit to root and run)
      systemctl restart pixelfed && systemctl restart php8.3-fpm
      ```
   
   4. Confirm you can read the EXIF by uploading something from a mobile and checking the data

      ```bash
      exiftool storage/app/public/m/_v2/(num)/*/KfNNBL54Y3un/1D3p4oJprxWhbgtJIQYj2RpY7koxS4Kdz7QM1umO.jpg |grep -i model
      ```

      You'll see the phone model typically, like "Motorola"
   
   
   5. And now it's time to do some php. First, make sure to back stuff up
   
      ```bash
      cd /data/pixelfed
      cp ./config/image-optimizer.php /data/image-optimizer.php.bak
      cp ./app/Jobs/ImageOptimizePipeline/ImageResize.php /data/ImageResize.php.bak
      ```
   
   6. Now let's edit the config/image-optimizer.php - insert these options so you can make granular changes
   
      ```php
               Jpegoptim::class => [
                   '-m' . (int) env('IMAGE_QUALITY', 80),
                   '--strip-exif',  // this strips out EXIF data to protect privacy (might have phone/geo info)
                   //'--preserve', // this should keep ICC color profiles
                   //'--strip-xmp', // this would remove the copyright data
                   //'--strip-iptc', // this would remove more copyright data
                   //'--strip-com', // this can strip comments
                   '--all-progressive',  // this will make sure the resulting image is a progressive one
               ],
      ```
   
   
   
   7. Next up is the real money. The prior didn't really change anything, more just a note for future reference. Now we edit app/Jobs/ImageOptimizePipeline/ImageResize.php which does the actual work. The "PATCH" piece goes in this section at the very end, just add the part between the comments.
   
      ```php
      try {
                   $img = new Image;
                   $img->resizeImage($media);
                   // Patch to attempt to strip all metadata BUT copyright and ICC
                   $path = storage_path('app/'.$media->media_path);
                   $cmd = "exiftool -overwrite_original "
                       . "-all= "
                       . "-tagsfromfile @ -ICC_Profile -Copyright -Rights "
                       . escapeshellarg($path);
                   shell_exec($cmd);
                   // End patch
           } catch (\Exception $e) {
               if (config('app.dev_log')) {
                   Log::error('Image resize failed: '.$e->getMessage());
               }
           }
      ```
   
   8. Now flush the cache and restart all the things yet again.
   
      ```bash
      cd /data/pixelfed && php artisan config:clear && php artisan cache:clear && php artisan view:clear && php artisan route:clear && php artisan config:cache && php artisan route:cache && php artisan view:cache
      # exit to root and run
      systemctl restart pixelfed && systemctl restart php8.3-fpm
      ```
   
   9. Upload the same file to a new post, and verify the EXIF has been cleaned up
   
      ```bash
      exiftool storage/app/public/m/_v2/(num)/*/KfNNBLi7Y3In/1D3p4oJprxWhbgtJzQYj2RpY7koxS4Kdz7bM1umO.jpg |grep -i model
      ```
   
      Repeat with a pro image, if you like, and you'll see that tags added in tools like Photoshop will stick around.
   


3. Want an age verification screen that doesn't seem so... intrustive? (YMMV, this depends on your legal jurisdiction, etc.)

   
   1. Back up the age verification file
   
      ```bash
      cp /data/pixelfed/resources/views/auth/curated-register/partials/step-1.blade.php /data/step-1.blade.php.bak
      ```
   
   2. Edit the file, replace it with something... cleaner. Adjust this example to suit.
   
      ```html
      @php
      $id = str_random(14);
      @endphp
      <h1 class="text-center">Before you continue.</h1>
      @if(config_cache('app.rules') && strlen(config_cache('app.rules')) > 5)
      <p class="lead text-center">Let's go over a few basic guidelines established by the server's administrators.</p>
      
      @include('auth.curated-register.partials.server-rules')
      @else
      <p class="lead text-center mt-4"><span class="opacity-5">The admins have not specified any community rules, however we suggest you review the</span> <a href="/site/terms" target="_blank" class="text-white font-weight-bold">Terms of Use</a> <span class="opacity-5">and</span> <a href="/site/privacy" target="_blank" class="text-white font-weight-bold">Privacy Policy</a>.</p>
      @endif
      
      <div class="action-btns">
          <form method="post" id="{{$id}}" class="flex-grow-1">
              @csrf
              <input type="hidden" name="step" value="1">
              <button type="button" class="btn btn-primary rounded-pill font-weight-bold btn-block flex-grow-1" onclick="onSubmit()">Accept</button>
          </form>
      
          <a class="btn btn-outline-muted rounded-pill" href="/">Go back</a>
      </div>
      
      <div class="small-links">
          <a href="/login">Login</a>
          <span>·</span>
          <a href="/auth/sign_up/resend-confirmation">Re-send confirmation</a>
          <span>·</span>
          <a href="{{route('help.curated-onboarding')}}" target="_blank">Help</a>
      </div>
      
      @push('scripts')
      <style>
          .swal-footer {
              display: flex;
              justify-content: center;
          }
      </style>
      <script>
          function onSubmit() {
              @if ($errors->any())
              document.getElementById('{{$id}}').submit();
              return;
              @endif
              swal({
                  text: "Please select the region you are located in",
                  icon: "info",
                  buttons: {
                      cancel: false,
                      usa: {
                          text: "United States",
                          className: "swal-button--cancel",
                          value: "usa"
                      },
                      canada: {
                          text: "Canada",
                          className: "swal-button--cancel",
                          value: "canada"
                      },
                      mexico: {
                          text: "México",
                          className: "swal-button--cancel",
                          value: "mexico"
                      }
                  },
                  dangerMode: false,
              }).then((region) => {
                  handleRegion(region);
              })
          }
      
          function handleRegion(region) {
              if(!region) {
                  return;
              }
              let minAge = 16;
              if(['usa', 'canada', 'mexico'].includes(region)) {
                  minAge = 13;
              }
      
              const checkboxContainer = document.createElement('div');
              checkboxContainer.innerHTML = `
              <label style="display:flex;align-items:center;gap:10px;cursor:pointer;font-size:14px;justify-content:center;" class="swal-text">
              <input type="checkbox" id="ageConfirmCheckbox" style="width:18px;height:18px;cursor:pointer;">
              I confirm that I am at least ${minAge} years old
              </label>
              `;
      
              swal({
                  title: "Age Confirmation",
                  text: "Please confirm that you meet our age requirement.",
                  content: checkboxContainer,
                  buttons: {
                      cancel: false,
                      confirm: {
                          text: 'Confirm',
                          className: "swal-button--cancel",
                      }
                  }
              }).then(() => {
                  const checked = document.getElementById('ageConfirmCheckbox').checked;
                  if (!checked) {
                      swal({
                          title: "Age Confirmation Required",
                          text: `You must confirm you are at least ${minAge} years old to join.`,
                          icon: "error",
                          buttons: { cancel: "I understand" }
                      }).then(() => {
                          window.location.href = '/'
                      });
                      return;
                  }
                  document.getElementById('{{$id}}').submit();
              });
          }
      
          function calculateAge(dob) {
              const diff_ms = Date.now() - dob.getTime();
              console.log(diff_ms);
              const age_dt = new Date(diff_ms);
              return Math.abs(age_dt.getUTCFullYear() - 1970);
          }
      
          function getToday() {
              var today = new Date();
              var dd = today.getDate();
              var mm = today.getMonth() + 1;
              var yyyy = today.getFullYear();
      
              if (dd < 10) {
                 dd = '0' + dd;
              }
              if (mm < 10) {
                 mm = '0' + mm;
              }
      
              yyyy = yyyy - 10;
              return yyyy + '-' + mm + '-' + dd;
          }
      
          function isValidDate(d) {
              return d instanceof Date && !isNaN(d);
          }
      </script>
      @endpush
      ```
   
   3. Clear everything yet again, and force reload the site.
   
      
   
4. Some quick commands you'll be using a LOT

   ```bash
   # Clear everything
   cd /data/pixelfed && php artisan config:clear && php artisan cache:clear && php artisan view:clear && php artisan route:clear && php artisan config:cache && php artisan route:cache && php artisan view:cache
   
   # Restart the services
   systemctl restart pixelfed && systemctl restart php8.3-fpm
   
   # Checking on storage usage
   du -h /data/pixelfed/storage/app/public/m/_v2|tail
   
   # Checking the logs - unfortunately, they're by day
   tail -100 /data/pixelfed/storage/logs/laravel-2026-04-06.log
   
   # Logging in to check a specific weird setting
   php artisan tinker
   >> config('mail.default')
   
   # Testing mail
   php artisan tinker
   >> Mail::raw('test', fn($m) => $m->to('info@test.com'));
   ```

5.  Easy fix to annoying problem - case sensitive hashtags. I just tweaked the DB to make that field case-insensitive. You'll want to do this early enough that you don't have a ton of hashtags to merge or delete.

   ```sql
   sudo -u postgres psql pixelfed
   CREATE EXTENSION IF NOT EXISTS citext;
   
   pixelfed=# \d+ hashtags
                                                                         Table "public.hashtags"
       Column    |              Type              | Collation | Nullable |               Default                | Storage  | Compression | Stats target | Description 
   --------------+--------------------------------+-----------+----------+--------------------------------------+----------+-------------+--------------+-------------
    id           | bigint                         |           | not null | nextval('hashtags_id_seq'::regclass) | plain    |             |              | 
    name         | character varying(191)         |           | not null |                                      | extended |             |              | 
    slug         | character varying(191)         |           | not null |                                      | extended |             |              | 
    is_nsfw      | boolean                        |           | not null | false                                | plain    |             |              | 
    is_banned    | boolean                        |           | not null | false                                | plain    |             |              | 
    created_at   | timestamp(0) without time zone |           |          |                                      | plain    |             |              | 
    updated_at   | timestamp(0) without time zone |           |          |                                      | plain    |             |              | 
    cached_count | integer                        |           |          |                                      | plain    |             |              | 
    can_trend    | boolean                        |           |          |                                      | plain    |             |              | 
    can_search   | boolean                        |           |          |                                      | plain    |             |              | 
   Indexes:
       "hashtags_pkey" PRIMARY KEY, btree (id)
       "hashtags_can_search_index" btree (can_search)
       "hashtags_can_trend_index" btree (can_trend)
       "hashtags_is_banned_index" btree (is_banned)
       "hashtags_is_nsfw_index" btree (is_nsfw)
       "hashtags_name_unique" UNIQUE CONSTRAINT, btree (name)
       "hashtags_slug_unique" UNIQUE CONSTRAINT, btree (slug)
   Not-null constraints:
       "hashtags_id_not_null" NOT NULL "id"
       "hashtags_name_not_null" NOT NULL "name"
       "hashtags_slug_not_null" NOT NULL "slug"
       "hashtags_is_nsfw_not_null" NOT NULL "is_nsfw"
       "hashtags_is_banned_not_null" NOT NULL "is_banned"
   Access method: heap
   
   ALTER TABLE hashtags ALTER COLUMN name TYPE citext;
   
   ```

   

6. Users got stuck in the email confirmation phase? Make sure to check in the curated_registers table.

   ```sql
   sudo -u postgres psql pixelfed
   select * from curated_registers;
   delete from curated_registers where username = 'username';
   ```

   

7. Mailgun not working? This is NOT the best method, but it's what I was forced to do because right now pixelfed is a mess of older/newer style variables

   ```yaml
   ## Mail Configuration (Post-Installer)
   MAIL_FROM_ADDRESS="info@email.mg.test.com"
   MAIL_FROM_NAME="YourSite"
   # This is supposed to be the correct set, but needed both
   MAIL_DRIVER=mailgun
   MAILGUN_DOMAIN=mg.test.com
   MAILGUN_SECRET=(key)
   MAILGUN_ENDPOINT=api.mailgun.net
   # This is supposed to be the older set, but was still needed
   MAIL_MAILER=mailgun 
   MAIL_HOST=smtp.mailgun.org
   MAIL_PORT=587
   MAIL_USERNAME=email.mg.test.com
   MAIL_PASSWORD=(key)
   MAIL_ENCRYPTION=tls
   MAIL_AUTO_TLS=true  # STARTTLS (false to bypass)
   
   ```

   After you do all that, clear the cache/config, restart the app AND the php fpm. Send a test email from artisan to confirm.

8. The end. For now :D

