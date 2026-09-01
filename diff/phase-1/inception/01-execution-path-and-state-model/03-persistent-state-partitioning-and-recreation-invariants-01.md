# 영속 상태 분할과 재생성 불변 조건

## `feat(compose): 세 서비스 토폴로지 구성`

diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
new file mode 100644
index 0000000..52cf8b7
--- /dev/null
+++ b/srcs/docker-compose.yml
@@ -0,0 +1,37 @@
+services:
+  nginx:
+    build:
+      context: ./requirements/nginx
+    image: inception-nginx:local
+    container_name: inception-nginx
+    restart: unless-stopped
+    ports:
+      - "443:443"
+    networks:
+      - inception
+
+  mariadb:
+    build:
+      context: ./requirements/mariadb
+    image: inception-mariadb:local
+    container_name: inception-mariadb
+    restart: unless-stopped
+    networks:
+      - inception
+
+  wordpress:
+    build:
+      context: ./requirements/wordpress
+    image: inception-wordpress:local
+    container_name: inception-wordpress
+    restart: unless-stopped
+    networks:
+      - inception
+
+networks:
+  inception:
+    driver: bridge
+
+volumes:
+  mariadb_data:
+  wordpress_data:


## `feat(compose): 준비 상태에 따라 영속 서비스 연결`

diff --git a/.env.example b/.env.example
index 2deceb8..072c9d8 100644
--- a/.env.example
+++ b/.env.example
@@ -2,9 +2,13 @@ DOMAIN_NAME=localhost
 
 MYSQL_DATABASE=wordpress
 MYSQL_USER=wpuser
+MYSQL_PASSWORD=change-me-db-password
+MYSQL_ROOT_PASSWORD=change-me-root-password
 
 WORDPRESS_TITLE=Inception
 WORDPRESS_ADMIN_USER=admin
+WORDPRESS_ADMIN_PASSWORD=change-me-admin-password
 WORDPRESS_ADMIN_EMAIL=admin@example.com
 WORDPRESS_USER=author
+WORDPRESS_USER_PASSWORD=change-me-author-password
 WORDPRESS_USER_EMAIL=author@example.com
diff --git a/srcs/docker-compose.yml b/srcs/docker-compose.yml
index d5fca56..699d7ea 100644
--- a/srcs/docker-compose.yml
+++ b/srcs/docker-compose.yml
@@ -9,8 +9,19 @@ services:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
     ports:
       - "443:443"
+    volumes:
+      - wordpress_data:/var/www/html:ro
+    depends_on:
+      wordpress:
+        condition: service_healthy
     networks:
       - inception
+    healthcheck:
+      test: ["CMD-SHELL", "curl -kfsS https://127.0.0.1/healthz >/dev/null"]
+      interval: 10s
+      timeout: 5s
+      retries: 6
+      start_period: 20s
 
   mariadb:
     build:
@@ -21,8 +32,18 @@ services:
     environment:
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
+      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:?set MYSQL_ROOT_PASSWORD}
+      MYSQL_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD}
+    volumes:
+      - mariadb_data:/var/lib/mysql
     networks:
       - inception
+    healthcheck:
+      test: ["CMD-SHELL", "mysqladmin ping -h127.0.0.1 -uroot -p\"$${MYSQL_ROOT_PASSWORD}\" --silent"]
+      interval: 10s
+      timeout: 5s
+      retries: 8
+      start_period: 30s
 
   wordpress:
     build:
@@ -32,15 +53,30 @@ services:
     restart: unless-stopped
     environment:
       DOMAIN_NAME: ${DOMAIN_NAME:?set DOMAIN_NAME}
+      WORDPRESS_DB_HOST: mariadb
       MYSQL_DATABASE: ${MYSQL_DATABASE:?set MYSQL_DATABASE}
       MYSQL_USER: ${MYSQL_USER:?set MYSQL_USER}
+      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD:?set MYSQL_PASSWORD}
       WORDPRESS_TITLE: ${WORDPRESS_TITLE:?set WORDPRESS_TITLE}
       WORDPRESS_ADMIN_USER: ${WORDPRESS_ADMIN_USER:?set WORDPRESS_ADMIN_USER}
+      WORDPRESS_ADMIN_PASSWORD: ${WORDPRESS_ADMIN_PASSWORD:?set WORDPRESS_ADMIN_PASSWORD}
       WORDPRESS_ADMIN_EMAIL: ${WORDPRESS_ADMIN_EMAIL:?set WORDPRESS_ADMIN_EMAIL}
       WORDPRESS_USER: ${WORDPRESS_USER:?set WORDPRESS_USER}
+      WORDPRESS_USER_PASSWORD: ${WORDPRESS_USER_PASSWORD:?set WORDPRESS_USER_PASSWORD}
       WORDPRESS_USER_EMAIL: ${WORDPRESS_USER_EMAIL:?set WORDPRESS_USER_EMAIL}
+    volumes:
+      - wordpress_data:/var/www/html
+    depends_on:
+      mariadb:
+        condition: service_healthy
     networks:
       - inception
+    healthcheck:
+      test: ["CMD-SHELL", "REQUEST_METHOD=GET SCRIPT_NAME=/ping SCRIPT_FILENAME=/ping cgi-fcgi -bind -connect 127.0.0.1:9000 | grep -q pong"]
+      interval: 10s
+      timeout: 5s
+      retries: 8
+      start_period: 40s
 
 networks:
   inception:


