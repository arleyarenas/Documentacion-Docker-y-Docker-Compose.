# Docker explicado: de cero a Docker Compose

Nov 13, 2026 · @Foxy

## 1. Qué es Docker y qué problema resuelve

Docker empaqueta una aplicación junto con todo lo que necesita para funcionar (librerías, configuración, versión exacta del programa) en un **contenedor** que se ejecuta igual en cualquier máquina que tenga Docker. Resuelve el clásico «en mi ordenador funciona»: si funciona en el contenedor, funciona en el portátil, en el servidor y en la nube.

Un contenedor parece una máquina virtual pequeña, pero no lo es. Una máquina virtual emula un ordenador entero y arranca su propio sistema operativo. Un contenedor es un grupo de procesos aislados que **comparten el kernel** del sistema anfitrión: Linux le da su propio sistema de ficheros, su red y su lista de procesos, pero no hay un segundo sistema operativo debajo.

&#91;embedded content: Máquina virtual frente a contenedor · qué lleva cada uno\]

En la práctica lo usaste a la vez: Docker funcionaba **dentro** de una máquina virtual Ubuntu (VirtualBox, red host-only). Las dos tecnologías no compiten, se combinan.

|  | Máquina virtual | Contenedor |
| --- | --- | --- |
| Qué aísla | Un ordenador entero | Un grupo de procesos |
| Sistema operativo | Uno propio por VM | Comparte el kernel del anfitrión |
| Arranque | Minutos | Segundos o menos |
| Tamaño típico | Varios GB | Decenas o cientos de MB |
| Aislamiento | Muy fuerte | Fuerte, pero comparte kernel |
| Uso típico | Otro SO, laboratorios completos | Desplegar aplicaciones y servicios |

## 2. Conceptos clave: imagen, contenedor, capas y registro

Todo Docker gira alrededor de cuatro piezas. Si entiendes cómo se relacionan, el resto son comandos.

- **Imagen**: una plantilla de solo lectura con un sistema de ficheros mínimo, el programa y su configuración. Por ejemplo, `httpd:2.4.65` es Apache 2.4.65 listo para arrancar. No se ejecuta: es el molde.
- **Contenedor**: una imagen en marcha. Añade una capa fina de escritura encima de la imagen y ejecuta su proceso principal. De una misma imagen puedes lanzar muchos contenedores a la vez.
- **Capas**: una imagen está hecha de capas apiladas, una por cada instrucción del Dockerfile. Las capas se comparten entre imágenes y se guardan en caché, por eso descargar o reconstruir suele ser rápido.
- **Registro**: el almacén de imágenes. El público por defecto es **Docker Hub**, de donde salieron `httpd`, `mysql`, `php`, `phpmyadmin` y `redis` en la práctica.

Cada imagen se identifica como `nombre:etiqueta`. La etiqueta (tag) es la versión: `httpd:2.4.65-alpine` es Apache 2.4.65 sobre Alpine Linux, una base de unos pocos MB. Si no pones etiqueta, Docker usa `latest`, que puede cambiar sin avisar.

&#91;embedded content: Ciclo de vida · Dockerfile, imagen, contenedor y registro\]

La analogía de la cocina ayuda a no mezclar los conceptos:

| Docker | En la cocina | Ejemplo de la práctica |
| --- | --- | --- |
| Dockerfile | La receta escrita | `dockerfile-basic/Dockerfile` |
| `docker build` | Cocinar siguiendo la receta | `docker build -t apache-custom:v1.0 .` |
| Imagen | El plato preparado y congelado | `apache-custom:v1.0` |
| `docker run` | Sacar una ración y servirla | `docker run -d -p 8081:80 ...` |
| Contenedor | La ración servida | `apache-dockerfile` |
| Registro | El supermercado de platos preparados | Docker Hub |

Una idea importante: **la imagen no cambia nunca**. Lo que escribe un contenedor va a su capa propia y se pierde al borrarlo, salvo que lo guardes en un volumen (apartado 5).

## 3. Cómo funciona por dentro

Cuando escribes `docker ...` no es ese comando el que hace el trabajo. Docker es una arquitectura cliente-servidor: el **cliente** (`docker`, en tu terminal) envía la orden a través de una API al **daemon** (`dockerd`), un servicio que corre en segundo plano y gestiona imágenes, contenedores, volúmenes y redes.

&#91;embedded content: Arquitectura de Docker · cliente, daemon y registro\]

Esto es lo que pasa cuando ejecutas `docker run -d -p 8080:80 --name primer-apache httpd:2.4.65`:

1. El cliente envía la orden al daemon.
2. El daemon busca la imagen `httpd:2.4.65` en local. Si no está, la descarga de Docker Hub (un `docker pull` automático).
3. Crea el contenedor: le añade una capa de escritura, le asigna una IP en la red `bridge` y prepara la regla que redirige el puerto 8080 del host al 80 del contenedor.
4. Arranca el proceso principal de la imagen (en Apache, `httpd-foreground`).
5. Como usaste `-d`, te devuelve el ID del contenedor y te deja la terminal libre.

Por debajo, el daemon usa funciones del kernel de Linux: los **namespaces** aíslan lo que ve cada contenedor (procesos, red, ficheros) y los **cgroups** limitan lo que puede gastar (CPU, memoria). Es lo que hacía el bloque `deploy.resources.limits` del compose de la práctica. Por eso Docker funciona de forma nativa en Linux; en Windows y Mac, Docker Desktop monta una pequeña VM Linux por debajo.

Un detalle importante: **un contenedor vive mientras vive su proceso principal**. Si Apache se para por un error de configuración, el contenedor pasa a `Exited`. Es justo lo que pasó con el error `No MPM loaded` en la Parte 3.

## 4. Comandos esenciales

Con una docena de comandos haces el 90 % del trabajo diario. Están agrupados por lo que gestionan, con ejemplos sacados de la práctica.

### Contenedores

| Comando | Para qué sirve | Ejemplo |
| --- | --- | --- |
| `docker run` | Crea y arranca un contenedor a partir de una imagen | `docker run -d -p 8080:80 --name primer-apache httpd:2.4.65` |
| `docker ps` | Lista los contenedores en marcha | `docker ps` |
| `docker ps -a` | Lista también los parados (`Exited`) | Para ver si uno se ha caído al arrancar |
| `docker logs` | Muestra lo que ha escrito el proceso principal | `docker logs apache-configured` |
| `docker exec -it` | Ejecuta un comando dentro de un contenedor en marcha | `docker exec -it primer-apache /bin/bash` |
| `docker stop` / `start` | Para o vuelve a arrancar un contenedor | `docker stop apache-custom` |
| `docker rm` | Borra un contenedor parado | `docker rm apache-custom` |
| `docker rm -f` | Lo para y lo borra de golpe | `docker rm -f apache-ssl` |
| `docker inspect` | Muestra toda su configuración en JSON (IP, volúmenes, puertos) | `docker inspect lamp-mysql` |

### Imágenes

| Comando | Para qué sirve | Ejemplo |
| --- | --- | --- |
| `docker pull` | Descarga una imagen del registro | `docker pull httpd:2.4.65` |
| `docker images` | Lista las imágenes locales y su tamaño | Comparar `apache-custom` con `apache-multistage` |
| `docker build` | Construye una imagen desde un Dockerfile | `docker build -t apache-exercise-aas:v1.0 .` |
| `docker rmi` | Borra una imagen | `docker rmi apache-custom:v1.0` |
| `docker system prune` | Limpia contenedores parados, redes sin uso e imágenes huérfanas | Útil cuando el disco se llena |

### Las opciones de `docker run` que más usarás

- `-d`: en segundo plano (*detached*). Sin él, la terminal se queda enganchada al contenedor.
- `-p HOST:CONTENEDOR`: publica un puerto. `-p 8084:80` = lo que llegue al 8084 del host va al 80 del contenedor.
- `-v ORIGEN:DESTINO`: monta una carpeta o un volumen (apartado 5).
- `--name`: le pone nombre. Si no, Docker inventa uno como `happy_turing`.
- `-e VAR=valor`: pasa una variable de entorno, como `MYSQL_ROOT_PASSWORD`.
- `--rm`: borra el contenedor automáticamente al pararse. Ideal para pruebas.

### La secuencia para reconstruir

Cada vez que cambies algo que va **dentro** de la imagen (Dockerfile, ficheros copiados con `COPY`), la imagen vieja no se entera. Hay que reconstruir y volver a crear el contenedor:

```bash
docker rm -f apache-configured
docker build -t apache-configured:v1.2 .
docker run -d -p 8084:80 --name apache-configured apache-configured:v1.2
docker ps
```

El `rm -f` del principio evita el error `The container name ... is already in use`, que aparece porque un contenedor parado sigue ocupando su nombre.

## 5. Puertos y volúmenes: conectar el contenedor con el exterior

Un contenedor está aislado: tiene su propia red y su propio sistema de ficheros. Para que sirva de algo hay que abrirle dos tipos de puertas: **puertos** para que llegue el tráfico y **volúmenes** para que los datos sobrevivan.

&#91;embedded content: Puertos, bind mounts y volúmenes · cómo entra y sale lo que importa\]

### Puertos (`-p`)

`-p 8084:80` significa: «todo lo que llegue al puerto 8084 del host, mándalo al puerto 80 del contenedor». Siempre se lee **host a la izquierda, contenedor a la derecha**. Así puedes tener cinco Apaches escuchando en su puerto 80 interno, cada uno publicado en un puerto distinto del host (8080, 8081, 8082…), como en la práctica.

Dos cosas que aprendiste por las malas:

- Un puerto del host solo puede tenerlo **un** contenedor. Si otro ya lo usa, el `docker run` falla con `port is already allocated`. Fue lo que pasó cuando el viejo `apache-ssl` ocupaba el 8086 y el 8443.
- `EXPOSE 80` en el Dockerfile **no publica nada**: solo documenta qué puerto usa la imagen. Lo que abre el puerto es el `-p`.

### Tres formas de guardar datos

| Tipo | Cómo se escribe | Dónde viven los datos | Cuándo usarlo |
| --- | --- | --- | --- |
| Bind mount | `-v $(pwd)/html:/usr/local/apache2/htdocs` | En una carpeta tuya del host | Desarrollo: editas y ves el cambio al momento (Reto 1) |
| Volumen con nombre | `-v mysql-data:/var/lib/mysql` | En `/var/lib/docker/volumes/`, gestionado por Docker | Datos de bases de datos y todo lo que deba persistir (MySQL, Redis) |
| tmpfs | `--tmpfs /tmp` | En memoria RAM, se borra al parar | Datos temporales o sensibles |

La diferencia práctica entre las dos primeras: el bind mount depende de una ruta concreta de tu máquina, mientras que el volumen con nombre lo maneja Docker y funciona igual en cualquier sitio. Por eso en el compose la web va con bind mount (`./apache/html`) y MySQL y Redis con volumen (`mysql-data`, `redis-data`).

Para gestionarlos: `docker volume ls`, `docker volume inspect mysql-data` y `docker volume rm`. Cuidado con `docker compose down -v`: el `-v` **borra los volúmenes**, y con ellos los datos de la base de datos.

## 6. Dockerfile: crear tus propias imágenes

Un Dockerfile es un fichero de texto (sin extensión, llamado `Dockerfile`) con los pasos para construir una imagen. Docker lo lee de arriba abajo y cada instrucción crea una capa nueva.

### Las instrucciones

| Instrucción | Qué hace | Ejemplo de la práctica |
| --- | --- | --- |
| `FROM` | Imagen base de la que partes. Siempre la primera | `FROM httpd:2.4.65-alpine` |
| `LABEL` | Metadatos: autor, descripción, versión | `LABEL version="1.0"` |
| `RUN` | Ejecuta un comando durante la construcción | `RUN pecl install redis && docker-php-ext-enable redis` |
| `COPY` | Copia ficheros del host a la imagen | `COPY static-site/ /usr/local/apache2/htdocs/` |
| `WORKDIR` | Directorio de trabajo para lo que venga después | `WORKDIR /app` |
| `ENV` | Variable de entorno dentro de la imagen | `ENV TZ=Europe/Madrid` |
| `EXPOSE` | Documenta el puerto que usa (no lo publica) | `EXPOSE 80 443` |
| `USER` | Usuario con el que se ejecuta el proceso | `USER daemon` |
| `CMD` | Comando por defecto al arrancar el contenedor | `CMD ["httpd-foreground"]` |
| `ENTRYPOINT` | Comando fijo; `CMD` pasa a ser sus argumentos | Lo usan las imágenes de MySQL y PHP |

Un Dockerfile completo, como el del Reto 2:

```dockerfile
FROM httpd:2.4.65-alpine

LABEL maintainer="Arley Arenas Saavedra"
LABEL description="Lloc web estàtic del Repte 2"
LABEL version="1.0"

COPY static-site/ /usr/local/apache2/htdocs/

EXPOSE 80
CMD ["httpd-foreground"]
```

Y se construye con `docker build -t apache-exercise-aas:v1.0 .`. El punto final es el **contexto de construcción**: la carpeta que Docker puede ver. Un `COPY` solo puede coger ficheros de dentro de esa carpeta, por eso `static-site` tenía que estar al lado del Dockerfile.

### Capas y caché

&#91;embedded content: Capas de una imagen y caché de construcción\]

Cuando reconstruyes, Docker compara cada paso con la vez anterior. Si un paso y sus ficheros no han cambiado, reutiliza la capa guardada (en la salida verás `CACHED`). En cuanto un paso cambia, ese y **todos los siguientes** se rehacen. En el Reto 5 lo viste: al cambiar solo el `index.php`, el `pecl install redis` salió como `CACHED` y el build tardó segundos.

### Multi-stage: imágenes pequeñas

Un build multi-stage usa varias etapas con `FROM` y solo se queda con la última. La primera trae las herramientas para compilar o generar ficheros; la segunda copia el resultado con `COPY --from=`:

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
RUN mkdir -p dist && echo '<h1>Generat</h1>' > dist/index.html

FROM httpd:2.4.65-alpine
COPY --from=builder /app/dist/ /usr/local/apache2/htdocs/
```

Node.js no llega a la imagen final. Por eso `apache-multistage` pesaba casi lo mismo que un Apache Alpine pelado, mucho menos que si llevara Node dentro.

### Buenas prácticas

1. **Versión concreta, nunca `latest`**: `httpd:2.4.65-alpine`. Así el build de hoy y el de dentro de un año dan lo mismo.
2. **Imágenes base pequeñas**: las variantes `-alpine` o `-slim` ocupan una fracción y tienen menos superficie de ataque.
3. **Junta comandos en un solo `RUN`** con `&&` y limpia en el mismo paso (`rm -rf /tmp/pear`). Cada `RUN` es una capa, y lo que borras en una capa posterior sigue ocupando sitio en la anterior.
4. **Ordena por frecuencia de cambio**: dependencias arriba, tu código abajo, para aprovechar la caché.
5. **Usa `.dockerignore`** para que `.git`, logs o secretos no entren en el contexto.
6. **No ejecutes como root** si puedes evitarlo (`USER`). Ojo: un usuario sin privilegios no puede abrir puertos por debajo del 1024, como el 80.
7. **Nada de contraseñas en el Dockerfile**: quedan grabadas en las capas. Van en variables de entorno o en un `.env`.

## 7. Redes: cómo se hablan los contenedores

Cada contenedor recibe una IP privada dentro de una red virtual de Docker. Lo importante es esto: **en una red creada por ti (o por Compose), los contenedores se encuentran por su nombre**, gracias a un DNS interno de Docker. Por eso el `index.php` conectaba a MySQL con `$host = 'mysql'` y a Redis con `connect('redis', 6379)`, sin saber ninguna IP.

### Tipos de red

| Driver | Qué hace | Cuándo se usa |
| --- | --- | --- |
| `bridge` (por defecto) | Red privada en el host; los contenedores salen a Internet por NAT | Casi siempre. La `lamp-network` de la práctica era un bridge |
| `host` | El contenedor usa directamente la red del host, sin aislamiento | Cuando necesitas el máximo rendimiento de red |
| `none` | Sin red | Tareas aisladas que no deben comunicarse |
| `overlay` | Une contenedores de varias máquinas | Clústeres (Docker Swarm) |

Ojo con un matiz: en la red `bridge` **por defecto** (la que usa `docker run` si no dices nada) no hay resolución por nombre. Para que funcione, crea una red propia y conecta ahí los contenedores:

```bash
docker network create mi-red
docker run -d --name db --network mi-red mysql:8.0.35
docker run -d --name web --network mi-red -p 8080:80 php:8.2-apache
# desde "web", el host "db" ya resuelve
```

Compose hace esto solo: crea una red para el proyecto y registra cada servicio con su nombre.

### Dentro de la red frente a fuera

- **Entre contenedores** se usa el nombre del servicio y el puerto **interno**: `mysql:3306`, `redis:6379`. No hace falta publicar ningún puerto con `-p` para esto.
- **Desde fuera** (tu navegador, otro ordenador) se usa la IP del host y el puerto **publicado**: `192.168.56.102:8090`.

De aquí sale una buena práctica de seguridad: si solo Apache necesita hablar con MySQL, no publiques el 3306. Sin `ports:`, la base de datos sigue accesible para Apache pero invisible desde fuera.

Para investigar: `docker network ls`, `docker network inspect lamp-network` (muestra qué contenedores hay y sus IPs) y, desde dentro de un contenedor, `getent hosts mysql` para comprobar que el nombre resuelve. Es exactamente la comprobación que hiciste en la Parte 5.

## 8. Docker Compose: varios contenedores como una sola aplicación

Con `docker run` lanzas contenedores de uno en uno y tienes que acordarte de todas las opciones. **Docker Compose** describe la aplicación entera en un fichero `docker-compose.yml`: qué servicios hay, cómo se construyen, qué puertos, volúmenes y redes usan y en qué orden arrancan. Después, un solo comando lo levanta todo.

&#91;embedded content: El stack de la Parte 5 · 4 servicios, 1 red, 2 volúmenes\]

### Anatomía del fichero

Una versión resumida del compose del Reto 5:

```yaml
services:
  apache:
    build: ./apache            # construye desde apache/Dockerfile
    container_name: lamp-apache
    ports:
      - "${APACHE_PORT}:80"
    volumes:
      - ./apache/html:/var/www/html
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_started
    networks: [lamp-network]

  mysql:
    image: mysql:8.0.35
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
    volumes:
      - mysql-data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      retries: 3
      start_period: 300s
    networks: [lamp-network]

  redis:
    image: redis:7-alpine
    command: ["redis-server", "--appendonly", "yes"]
    volumes:
      - redis-data:/data
    networks: [lamp-network]

networks:
  lamp-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

Las claves más importantes:

- **`services`**: cada servicio acaba siendo un contenedor. Su nombre (`mysql`, `redis`) es también su nombre en la red.
- **`image` o `build`**: usar una imagen ya hecha o construir una desde un Dockerfile.
- **`depends_on`**: el orden de arranque. Con `condition: service_healthy`, Apache espera a que el healthcheck de MySQL diga que está listo, no solo a que su contenedor exista.
- **`healthcheck`**: un comando que Docker repite para saber si el servicio funciona de verdad. `start_period` es el margen inicial; en tu VM, MySQL tardó unos 4 minutos en inicializarse la primera vez, así que 30 s no bastaban.
- **`volumes` y `networks`** de abajo del todo: declaran los volúmenes y redes con nombre que usan los servicios.

### Variables y el fichero `.env`

Compose lee automáticamente un fichero `.env` en la misma carpeta y sustituye cada `${VARIABLE}`. Así las contraseñas y los puertos no están escritos en el YAML:

```bash
APACHE_PORT=8090
PHPMYADMIN_PORT=8091
MYSQL_ROOT_PASSWORD=rootpassword
MYSQL_DATABASE=testdb
```

El `.env` va siempre en el `.gitignore`: contiene credenciales. Y recuerda que `MYSQL_ROOT_PASSWORD` solo se aplica la **primera vez**, cuando el volumen está vacío. Si luego cambias la variable, MySQL sigue con la contraseña antigua.

### Comandos de Compose

| Comando | Qué hace |
| --- | --- |
| `docker compose up -d` | Crea y arranca todo en segundo plano |
| `docker compose up -d --build` | Igual, pero reconstruye las imágenes (necesario si cambias un Dockerfile) |
| `docker compose ps` | Estado de los servicios, incluido `healthy` |
| `docker compose logs -f mysql` | Logs de un servicio, en directo |
| `docker compose exec apache bash` | Entra en un servicio |
| `docker compose config` | Valida el YAML y muestra el resultado con las variables sustituidas |
| `docker compose down` | Para y borra contenedores y red (los volúmenes se quedan) |
| `docker compose down -v` | Lo mismo y además **borra los volúmenes** (y los datos) |

### Errores típicos de YAML

El YAML se rompe por detalles que no se ven bien. En el Reto 5 aparecieron todos:

- **Indentación**: se usan espacios, nunca tabuladores. Los servicios van a 2 espacios y sus claves a 4.
- **Claves repetidas**: dos `volumes:` o dos `ports:` en el mismo servicio no están permitidos.
- **Guion sin espacio**: `-redis-data:/data` no es un elemento de lista; tiene que ser `- redis-data:/data`.
- **Nombres que no coinciden**: `redis_data` en un sitio y `redis-data` en otro son dos volúmenes distintos.

Antes de levantar nada, `docker compose config` te dice la línea exacta del error.

## 9. Cuando algo falla: cómo diagnosticar

Casi todos los problemas se resuelven con el mismo método, en este orden:

1. **`docker ps -a`**: ¿el contenedor existe? ¿Está `Up` o `Exited`? ¿Qué puertos tiene publicados?
2. **`docker logs <nombre>`**: si se ha caído, el motivo casi siempre está en las últimas líneas.
3. **`curl -I http://...`**: prueba el servicio sin navegador (sin caché ni redirecciones guardadas) y te enseña el código HTTP y las cabeceras.
4. **Prueba desde dentro y desde fuera**: si `curl` funciona dentro de la VM y no desde Windows, el problema es de red, no de Docker.
5. **`docker exec -it <nombre> sh`**: entra y mira si los ficheros están donde crees (`ls -la`, `cat`).

Estos son los problemas que te salieron en la práctica, como referencia para la próxima vez:

| Síntoma | Causa | Solución |
| --- | --- | --- |
| El contenedor sale `Exited` nada más arrancar; el log dice `AH00534: No MPM loaded` | La configuración de Apache no cargaba ningún módulo MPM | Añadir `LoadModule mpm_event_module modules/mod_mpm_event.so` y reconstruir |
| `The container name ... is already in use` | El contenedor anterior, aunque esté parado, sigue ocupando el nombre | `docker rm -f <nombre>` antes del `run` |
| Un `run` falla o responde un contenedor que no es el tuyo | Otro contenedor ocupa el puerto del host | `docker ps` para ver quién lo tiene; borrarlo o usar otro puerto |
| El `.htaccess` no hace nada | El fichero se llamaba `htaccess`, sin punto (o `.htaccess.txt`) | `ls -la` y renombrarlo a `.htaccess` |
| `/old-pages` no redirige | La regla `^old-page$` exige la URL exacta | Usar la URL que define la regla |
| El navegador de Windows no conecta, pero `ping` responde | Docker está en la VM y `hosts` apuntaba a `127.0.0.1` | Poner en `hosts` la IP host-only de la VM |
| `dependency failed to start` en Compose | MySQL tardó más en arrancar que el margen del healthcheck | Esperar y repetir `up -d`, o subir `start_period` |
| Cambias un fichero y no se ve el cambio | El fichero va dentro de la imagen (`COPY`), no en un volumen | Reconstruir: `rm -f`, `build`, `run` (o `compose up -d --build`) |
| La línea con `\` no se ejecuta en Windows | La `\` solo continúa líneas en bash | En PowerShell, usar `` ` `` o escribirlo todo en una línea |

Un truco extra: un error **500** de Apache casi siempre es una errata en `.htaccess` o en la configuración. El detalle exacto está en `logs/error.log`.

## 10. Cómo aplicarlo y por dónde seguir

Docker es hoy la forma estándar de desplegar servicios. En un puesto de sistemas lo usarás para cosas como estas:

- **Montar entornos en minutos**: un servidor web, una base de datos o un panel de monitorización (Grafana, Prometheus) sin instalar nada en el sistema.
- **Que desarrollo y producción sean iguales**: la misma imagen pasa por el portátil, las pruebas y el servidor.
- **Varios servicios en un solo servidor** sin conflictos de versiones: dos webs con PHP 7 y PHP 8 conviviendo, cada una en su contenedor.
- **Laboratorios desechables**: probar algo y borrarlo con `docker rm -f` sin ensuciar la máquina.

### Un ejemplo cercano: el proyecto ReBoot

Lo que has hecho en la Parte 5 es casi la base de un proyecto real como ReBoot (web e inventario de ordenadores donados) en el servidor de la HP. Un `docker-compose.yml` con la web (Apache + PHP), la base de datos del inventario (MySQL con volumen) y phpMyAdmin para administrarla, más un `.env` con las credenciales. Para moverlo a otro servidor bastaría con copiar la carpeta y ejecutar `docker compose up -d`.

### Siguientes pasos para aprender

1. **Publicar tus propias imágenes** en Docker Hub con `docker login`, `docker tag` y `docker push`.
2. **Un proxy inverso** (Nginx Proxy Manager o Traefik) delante de varios servicios, con HTTPS de Let's Encrypt. Es la evolución natural de los VirtualHosts de la Parte 4.
3. **Portainer**: una interfaz web para gestionar contenedores, útil en un servidor doméstico.
4. **Automatizar builds** con GitHub Actions: cada `git push` construye y publica la imagen.
5. **Kubernetes**, cuando haya que repartir contenedores entre muchas máquinas. Para empezar, Compose cubre la gran mayoría de casos.

### Chuleta final

```bash
# Contenedores
docker run -d -p HOST:CONT --name NOMBRE -v ORIGEN:DESTINO IMAGEN:TAG
docker ps -a                     # todos, también los parados
docker logs -f NOMBRE            # qué está pasando
docker exec -it NOMBRE sh        # entrar
docker rm -f NOMBRE              # parar y borrar

# Imágenes
docker build -t NOMBRE:TAG .     # el punto es el contexto
docker images
docker rmi NOMBRE:TAG

# Volúmenes y redes
docker volume ls
docker network inspect RED

# Compose
docker compose config            # validar el YAML
docker compose up -d --build
docker compose ps
docker compose logs -f SERVICIO
docker compose down              # sin -v, los datos se quedan

# Limpieza
docker system prune
```

La idea que lo resume todo: **el Dockerfile describe la imagen, la imagen es inmutable, el contenedor es desechable y los datos importantes van en volúmenes**. Si tienes eso claro, todo lo demás son opciones.
