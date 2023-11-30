# Guide on how to setup Wordpress with Docker Compose.

Here are the key steps that we are going to cover:

* Configuring Docker Compose file
* Configuring Nginx
* Generating certificates for HTTP to HTTPS redirection
* Configure basic authentication for wp-admin page

<ins>***Ensure that you have installed Docker and Compose of the latest versions***<ins/>


## 1. Configuring Docker Compose file

To create a compose file we use **`vim docker-compose.yml`** command and there the following lines:
```
version: '3'
services:

  wordpress:
    image: wordpress:php8.2-fpm
    container_name: php-fpm
    volumes:
      - ./wp-files:/var/www/html
    networks:
      - app-net
    environment:
      WORDPRESS_DB_HOST: <db hostname>:3306
      WORDPRESS_DB_NAME: <db_name>
      WORDPRESS_DB_USER: <user_name>
      WORDPRESS_DB_PASSWORD: <password>
    depends_on:
      - mysql

  nginx:
    image: nginx
    container_name: nginx
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./wp-files:/var/www/html
      - ./certs:/etc/ssl/certs
      - ./.htpasswd:/etc/nginx/.htpasswd
    networks:
      - app-net
    depends_on:
      - wordpress
      - mysql

  mysql:
    image: mysql
    volumes:
      - ./data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: <root_password>
      MYSQL_DATABASE: db_name
      MYSQL_USER: <user_name>
      MYSQL_PASSWORD: <password>
    networks:
      - app-net
     
networks:
  app-net:
    driver: bridge
```
Here you can see three main images that we use: <ins>***nginx, wordpress and mysql***<ins/>.

Here in wordpress we specified the bind mount under *<ins>volumes<ins/>* section, then the environmental variables which will construct the ***wp-config*** file, the *<ins>networks<ins/>* section indicates which network this service belongs to and the *<ins>depends_on<ins/>* section which indicates after what *service* the **wordpress** service have to start.

## 1.2 .env and .gitignore files

It is crucial to have ***.env*** file where we can specify values of variables defined in docker-compose file under the services.
This is done for security. We just create .env file, add there variables and their values. After this, we add that .env file to ***.gitignore*** file. This will ignore the specified files and directories, i.e will not push them to our repository.

## 2. Configuring nginx

```
events {
    worker_connections 1024;   
}

http {
    server {

        listen 80;
        server_name mamajan.com www.mamajan.com;
	    return 301 <https:$server_name$request_uri>;

    }

    server {
	
        listen 443    ssl;
	    server_name mamajan.com www.mamajan.com;

	    ssl_certificate       /etc/ssl/certs/mamajan.com.pem;
        ssl_certificate_key   /etc/ssl/certs/mamajan.com-key.pem;

        root /var/www/html;
	    index index.php index.html index.htm;

        location / {
            try_files $uri $uri/ /index.php$is_args$args;
        }

        location ~ \.php$ {
	        include fastcgi_params;
	        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
	        fastcgi_pass wordpress:9000;
	    }

	    location /wp-admin {
	        auth_basic "Admin Page";
            auth_basic_user_file /etc/nginx/.htpasswd;
	        try_files $uri $uri/ =404;
        }

    }
    
}
```

- Configuring nginx is a crucial step. Nginx acts as a ***reverse proxy*** that forwards requests from the internet to the appropriate Docker container running WordPress.
- We use this configuration to provide SSL/TLS Termination. This means that Nginx decrypts the incoming HTTPS traffic and forwards it to the WordPress container using HTTP.

[!] Do not forget to add the domain name to ***/etc/hosts*** file like this:

* `IP-Address  mamajan.com`

## 3. Generating Certificates for website (SSL/TLS termination)

To generate local CA certificate we use this command:

* `mkcert -install`

The certificate will be saved into the local store, which we can locate with the command:

* `mkcert -CAROOT`

The output of this command will show the path to our certificate.

Next, we will generate a certificate for our website:

* `mkcert <website_name> <IP>`

The output of the aforementioned command will include the location of the newly created pem files.

## 4. Configuring basic authentication for **wp-admin** page

First of all we need to generate ***.htpasswd*** file.

To generate it we use :

* `htpasswd -c .htpasswd <username>`

This command will prompt us to write a password.

The ***.htpasswd*** file will store the username and encrypted password.

After that we add the following to the nginx configuration:

```
location /wp-admin {        
    auth_basic "Admin Page";
    auth_basic_user_file /etc/nginx/.htpasswd;	        
    try_files $uri $uri/ =404;     
}
```

These lines will tell the user to authenticate before accessing  to our account.

# Final Step

Now we have to run `docker-compose up -d` command to create our services and to access our website we need to open browser and type our **<ins>domain name<ins/>**.