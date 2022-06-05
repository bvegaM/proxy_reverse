# Desarrollo de un proxy reverse utilizando Nginx

> Bryam Vega

Tomando como base la prática relacionada a una arquitectura orientada a servicios cuyo proyecto base se denomina ```kitchensink-angularjs``` y el cual contiene el archivo ```docker-compose.yml``` se realizo la siguiente modificación al mismo con el objetivo de generar dicha capa de api-layer, para eso utilizamos Nginx como nuestra capa de api. La configuración es la siguiente:

```yml
version: '3.6'

services:

  # se agrega el reverse proxy con nginx
  # se debe utilizar un archivo nginx.conf para poder crear
  # el reverse proxy
  reverseproxy:
    image: 'nginx:alpine'
    container_name: 'reverseproxy'
    hostname: 'reverseproxy'
    depends_on:
      - srvwildfly
    volumes:
      - ./reverseproxy/nginx.conf:/etc/nginx/nginx.conf
    ports:
      - 80:80
    networks:
        - demo_kcs_net

  srvdb:
    image: postgres
    container_name: srvdb
    hostname: srvdb
    environment:
      POSTGRES_USER: consultas
      POSTGRES_DB: consultas
      POSTGRES_PASSWORD: QueryConSql.pwd
    ports:
      - 4358:5432
    networks:
      - demo_kcs_net

  srvwildfly:
    image: myapp
    container_name: srvwildfly
    hostname: srvwildfly
    ports:
      - 8080:8080
      - 9990:9990
    depends_on:
      - srvdb
    networks:
      - demo_kcs_net
  
  pgadmin:
    image: dpage/pgadmin4
    environment:
      PGADMIN_DEFAULT_EMAIL: info@jtux.ec
      PGADMIN_DEFAULT_PASSWORD: clave
    ports:
      - 5050:80
    depends_on:
      - srvdb
    networks:
      - demo_kcs_net

networks:
  demo_kcs_net:
```

El archivo que se tiene que crear para poder balancer nuestro servidor de aplicaciones, es el siguiente. En la ruta del proyecto ```kitchensink-angularjs``` crear una carpeta llamada ```reverseproxy``` y dentro de ella crear el archivo ```nginx.conf``` el cual contiene la siguiente configuración:

```nginx
worker_processes 1;

events { worker_connections 1024; }

http {

    sendfile on;

    upstream wildfly {
        server srvwildfly:8080;
    }

    server {
        listen 80;
        location / {
            proxy_pass         http://wildfly;
            proxy_redirect     off;
            proxy_set_header   Host $host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;
        }
    }

}

```

Los resultados de esta prueba se encuentran en la carpeta ```img``` donde se encuentran capturas del uso de la capa de api layer. 

## Ejecutar el docker-compose

Para ejecutar el docker-compose es muy importante contar con el archivo de configuración del proxy reverse, el cual debe ser creado como se indico anteriormente **dentro del proyecto kitchensink-angularjs** caso contrario pueden existir problemas. Una vez hecho eso, construir la imagen java como se indico en clases y después escribir los siguientes comandos:

```bash
kitchensink-angularjs$ docker build -t myapp . #crea la imagen del proyecto java
kitchensink-angularjs$ docker-compuse up #levanta el resto de imagenes
```

## Pruebas

En caso de que se desee realizar las pruebas, simplemente acceder al sitio [kitchensink](http://localhost/kitchensink-angularjs/#/home) y dirigirse al inspeccionar de la página. Recargar la web y dirigirse a la petición principal, en ella encontrara el siguiente resultado:

```http
HTTP/1.1 200 OK
Server: nginx/1.21.6
Date: Tue, 31 May 2022 02:54:28 GMT
Content-Type: text/html
Content-Length: 2850
Connection: keep-alive
Last-Modified: Thu, 11 Mar 2021 06:44:42 GMT
Accept-Ranges: bytes
```








