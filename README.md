# Tutorial de Wordpress con Kubernetes

Para ver el tutorial introductorio y que instalar, checa este video:
- [Tutorial básico de Kubernetes y contenedores](/res/tutorial-k8s-wordpress.md)
- [Video tutorial](https://web.microsoftstream.com/video/380105ad-3a67-4134-8acd-608bbb5bacc3)

## Creacion de la base de datos en Mysql de Azure
![Base de Datos.](/img/1.png)

Seleccionamos los Servidores de Azure Database for MySQL

 - Creamos un solo servidor y lo configutamos de acuerdo a nuestras necesidades. 
 - Configuramos el nombre y contraseña del administrador.

![Configuracion base de datos.](/img/2.png)

-Ya configurado creamos el recurso

una vez creado el recurso de las bases de datos, nos vamos a Seguridad de la conexion.

- Permitimos el acceso a servicios de Azure.

- Agregamos la IP del cliente actual.

- Deshabilitamos conexion SSL y guardamos la configuracion.

![Configuracion base de datos.](/img/3.png)

## Para conectarte a la base de datos de MySQL

Nos vamos a la terminal de Azure llamada Cloud Shell que se encuentra ubicada aqui...

![Cloud Shell.](/img/0.png)

Puede que te pida crear una cuenta de almacenamiento, te recomiendo que solo pongas tu cuenta de Azure y los de mas lo dejes por "default".

![Cloud Shell.](/img/0-1.png)

1. Para hacer la conexion a nuestra Base de datos ejecutamos el siguiente comando:

```
mysql --host=SERVER_DATABASE --user=USUARIO_BASE_DE_DATOS -p 
```
**Nota**: ¿De donde salen estos datos?. Los datos los obtenemos de la base de datos creada.

![Configuracion base de datos.](/img/4.png)

**Nota**: Después de este comando te pedirá la contraseña que pusiste al crear el recurso Azure Database foy MySQL.

![Configuracion base de datos.](/img/19.png)

3. Creamos el nombre de la Base de Datos con el siguiente comando:

```
CREATE DATABASE (NOMBRE_DE_LA_BASE_DE_DATOS); 
```
![Configuracion base de datos.](/img/5.png)

4. Y salimos del CloudShell

```
exit; 
```

## Para conectarte a la base de datos de MySQL

Voy a usar las siguientes "variables" en las cuales tu vas a remplazar con lo que corresponde en tu implementación.

- GRUPO_DE_RECURSOS = El grupo de recursos donde pondremos nuestras soluciones de Azure.

- NOMBRE_RECURSO_CONTENEDOR = Como nombraremos a nuestro recurso Azure Container Registry (ACR).

- ACR_LOGIN_SERVER = El link de login de nuestro contenedor en Azure (lo obtendremos más tarde). 

En la palabra en mayúsculas iría el nombre que le quieras poner a tu grupo de recursos.

1. Creas un grupo de recursos de Azure.

```
az group create --name GRUPO_DE_RECURSOS --location eastus
```

![Creacion del recurso ACR.](/img/6.png)

2. Creas el recurso Azure Container Registry (ACR) en plan básico

```
az acr create --resource-group NOMBRE_GRUPO_RECURSO --name NOMBRE_ACR --sku Basic
```

![Creacion del recurso ACR.](/img/7.png)

3. Creación del recurso de Azure Kubernetes Service (AKS)

```
az aks create --resource-group NOMBRE_GRUPO_RECURSO --name NOMBRE_AKS --node-count 2 --generate-ssh-keys
```
![Creacion del recurso ACR.](/img/8.png)


## Imagen de Wordpress Con Docker

1. Visitamos la pagina oficial y bajamos la carpeta de Docker

![Wordpresss.](/img/10.png)

2. Ya que descargamos el archivo, se descomprime y creamos una carpeta donde contenga todo nuestro recurso de wordpress, una vez hecho esto renombramos la carpeta por public y abrimos la carpeta en Visual Studio Code y quedaria algo asi:

![Carpeta de Wordpress.](/img/11.png)

3. Al abrir nuestro archivo en Visual Studio Code modificamos los siguientes archivos:

4. el archivo con el nombre: wp-config-sample, por el nombre:
```
wp-config.php
```
 - Y copiamos este codigo y lo insertamos en el archivo:

```PHP
//Using environment variables for DB connection information

// ** MySQL settings - You can get this info from your web host ** //
/** The name of the database for WordPress */

$connectstr_dbhost = getenv('DATABASE_HOST');
$connectstr_dbusername = getenv('DATABASE_USERNAME');
$connectstr_dbpassword = getenv('DATABASE_PASSWORD');

/** MySQL database name */
define('DB_NAME', 'flexibleserverdb');

/** MySQL database username */
define('DB_USER', $connectstr_dbusername);

/** MySQL database password */
define('DB_PASSWORD',$connectstr_dbpassword);

/** MySQL hostname */
define('DB_HOST', $connectstr_dbhost);

/** Database Charset to use in creating database tables. */
define('DB_CHARSET', 'utf8');

/** The Database Collate type. Don't change this if in doubt. */
define('DB_COLLATE', '');


/** SSL*/
define('MYSQL_CLIENT_FLAGS', MYSQLI_CLIENT_SSL);
```

- Quedando como resultado este archivo:

![wp-config.](/img/13.png)

- Una vez hecho esto modificamos los siguientes parametros dentro del mismo archivo:

![wp-config.](/img/12.png)

- Los datos los obtenemos del servidor 

![wp-config.](/img/27.png)

5. Una vez hecho esto creamos un Arhivo de docker.

- Para hacerlo damos click en nuevo archivo y lo nombramos como:

```
Dockerfile
```
![Doockerfile](/img/14.png)

- Hecho esto copiamos todo este codigo a este archivo dentro del archivo de Docker:

```Docker
FROM php:7.2-apache
COPY public/ /var/www/html/
RUN docker-php-ext-install mysqli
RUN docker-php-ext-enable mysqli
```
![Comando de Docker.](/img/15.png)

6. Creamos un archivo con extension .ymal que lo colocamos el nombre que nosotros queramos en este caso es: mywordpress.ymal

![Comando de Docker.](/img/16.png)

- Le insertamos el siguientes codigo:

**Archivo mywordpress.yaml**

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-blog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wordpress-blog
  template:
    metadata:
      labels:
        app: wordpress-blog
    spec:
      containers:
      - name: wordpress-blog
        image: ACR_LOGIN_SERVER/NOMBRE_SOLUCION:VERSION_SOLUCION
        ports:
        - containerPort: 80
        env:
        - name: DATABASE_HOST
          value: "SERVER_BASE_DE_DATOS"
        - name: DATABASE_USERNAME
          value: "USUARIO_BASE_DE_DATOS"
        - name: DATABASE_PASSWORD
          value: "PASSWORD_BASE_DE_DATOS"
        - name: DATABASE_NAME
          value: "NOMBRE_BASE_DE_DATOS"
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: "app"
                    operator: In
                    values:
                    - wordpress-blog
              topologyKey: "kubernetes.io/hostname"
---
apiVersion: v1
kind: Service
metadata:
  name: php-svc
spec:
  type: LoadBalancer
  ports:
    - port: 80
  selector:
    app: wordpress-blog
```
**Nota**: remplaza lo que esta en mayúsculas en los valores de `value` e `image` con lo que corresponda en tu caso.

![Archivo .ymal.](/img/17.png)

**IMPORTANTE**

Esto lo hacemos en la terminal de nuestra maquina y se tiene que tener el recurso de docker corriendo de lo contrario marca error.

1. Nos colocamos en el directorio donde tenemos nuestro recurso de docker

2. Para construir la imagen de docker y posteriormente subirla.

```
docker build --tag myblog:latest .
```
![Docker.](/img/18.png)

3. Login en el recurso ACR (se debe hacer desde Azure CLI para desktop)

```
az acr login --name NOMBRE_ACR
```
![ACR.](/img/22.png)

4. Creación de un tag de docker para subirlo después, para obtener el ACR_LOGIN_SERVER lo obtenemos del registro del contenedor

![ACR_LOGIN.](/img/23.png)

```
docker tag myblog:latest ACR_LOGIN_SERVER/wordpress-blog:latest
```
![docker tag.](/img/24.png)

5. Subir la imagen de docker al ACR en la nube

```
docker push ACR_LOGIN_SERVER/wordpress-blog:latest
```
![docker tag.](/img/26.png)

6. Regresamos a nuestros archivos de Visual Studio Code e insertamos el LOGIN_REGISTRY en el archivo WP_CONFIG en la siguiente linea:

![docker tag.](/img/25.png)

**Nota**: Cambia `wordpress-blog:latest` por el que pusiste en el comando `docker tag`.

7. Adjuntar el ACR al servicio de Kubernetes.

```
az aks get-credentials --resource-group GRUPO_DE_RECURSOS --name NOMBRE_RECURSO_K8S
```
![Get-credentials](/img/28.png)

8. Ejecutar todo lo que este en el archivo YAML y que wordpress comience a instalarlo.

```
kubectl apply -f mywordpress.yaml
```

![Get-credentials](/img/29.png)

9. Puedes corroborar que los nodos se encuentren funcionando con siguiente comando:

```
kubectl get nodes
```

```
kubectl get pods
```
![Get-credentials](/img/30.png)
