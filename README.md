# Práctica 2.3: autenticación básica de /admin

Módulo: Despliegue de aplicaciones web (2º DAW)

Alumno: Javier Doñoro Garcia

Fecha: 18 / 11 / 2026

## Introducción

En esta práctica he protegido la zona /admin del sitio taskflow.local de la práctica anterior con autenticación HTTP básica. Ahora, para entrar en /admin hay que poner usuario y contraseña, y el resto del sitio sigue abierto como antes.

## Teoría

### Cómo funciona la autenticación básica

Cuando se pide una página protegida sin credenciales, Apache responde con un 401 Unauthorized y la cabecera WWW-Authenticate: Basic realm="...". El navegador saca entonces la ventana de usuario y contraseña, y vuelve a hacer la petición con la cabecera Authorization: Basic seguida del texto usuario:contraseña codificado en Base64. Apache lo decodifica, lo compara con el hash que hay en el fichero .htpasswd y, si coincide, devuelve la página con un 200. Si no coincide, vuelve a dar 401.

### Por qué no es segura sin HTTPS

Base64 no es cifrado, solo es una forma de codificar el texto y se puede deshacer con un comando (base64 -d). Por eso, si la conexión va por HTTP normal, cualquiera que capture el tráfico en la misma red puede ver la contraseña. Con HTTPS la cabecera va cifrada dentro de la conexión TLS, así que en un servidor real la autenticación básica solo se debería usar con HTTPS.

### Alternativas

- Digest: en vez de mandar la contraseña manda un hash calculado con un valor aleatorio que da el servidor. Es mejor que Basic sin HTTPS, pero usa MD5 y hoy está bastante anticuada.
- Formulario con sesión: la propia aplicación tiene una página de login, y al entrar el servidor crea una sesión y le da al navegador una cookie. Permite cerrar sesión, bloquear tras varios intentos fallidos o añadir doble factor.
- JWT: después del login el servidor da un token firmado que el cliente manda en cada petición con Authorization: Bearer. Se usa mucho en APIs y aplicaciones SPA, y también necesita HTTPS.

### Política de la contraseña

Para el usuario admin he usado una contraseña de más de 12 caracteres con mayúsculas, minúsculas, números y símbolos, que no es ninguna palabra del diccionario ni tiene que ver con admin o con TaskFlow. Lo he hecho así porque la autenticación básica no limita los intentos, y una contraseña corta se podría sacar por fuerza bruta. Además la guardé con bcrypt (opción -B), que es más lento de romper que el MD5 que usa htpasswd por defecto.

La contraseña se debería cambiar si la conoce alguien que ya no debería, si hay sospecha de que se ha filtrado o cada cierto tiempo, por ejemplo cada 90 días. Para cambiarla basta con volver a ejecutar htpasswd sin -c sobre el mismo usuario:

```bash
sudo htpasswd -B /etc/apache2/.htpasswd admin
```

El fichero .htpasswd lo he dejado en /etc/apache2, fuera del DocumentRoot, para que nunca se pueda descargar desde la web.

## Paso 1. Crear la carpeta /admin

Creé la carpeta admin dentro del DocumentRoot con un index.html sencillo y volví a poner www-data como propietario:

```bash
sudo mkdir -p /var/www/taskflow/admin
echo "<h1>Zona de administración de TaskFlow</h1>" | sudo tee /var/www/taskflow/admin/index.html
sudo chown -R www-data:www-data /var/www/taskflow
ls -l /var/www/taskflow/admin
```

La primera vez la línea del echo se me partió en dos al pegarla y dio un error, así que la repetí entera y ya funcionó.

Captura 1: ls -l de la carpeta admin, con el index.html y propietario www-data.

![Captura 1](img/captura1.png)

## Paso 2. Crear el fichero de contraseñas

Creé el fichero con el usuario admin:

```bash
sudo htpasswd -c -B /etc/apache2/.htpasswd admin
```

La opción -c crea el fichero, y solo se usa la primera vez, porque si se vuelve a usar con el fichero ya creado lo sobrescribe y se pierden los usuarios que hubiera. La opción -B hace que la contraseña se guarde con bcrypt. El comando pide la contraseña dos veces y no se ve al escribirla.

Después cambié el propietario y los permisos para que solo root y Apache puedan leerlo:

```bash
sudo chown root:www-data /etc/apache2/.htpasswd
sudo chmod 640 /etc/apache2/.htpasswd
ls -l /etc/apache2/.htpasswd
```

Al principio se me olvidó el chmod y el fichero quedaba con -rw-r--r--, que deja leerlo a cualquier usuario. Lo ejecuté después y ya quedó con -rw-r-----.

Captura 2: el htpasswd y los permisos antes y después del chmod.

![Captura 2a](img/captura2a.png)

![Captura 2b](img/captura2b.png)

```bash
sudo cat /etc/apache2/.htpasswd
```

Captura 3: contenido del fichero. La contraseña no está en claro, se ve el hash que empieza por $2y$, que es el de bcrypt.

![Captura 3](img/captura3.png)

## Paso 3. Configurar la autenticación en taskflow.conf

Reescribí taskflow.conf con lo que ya tenía de la práctica 2.2 y añadí un bloque Directory nuevo para la carpeta admin:

```bash
sudo tee /etc/apache2/sites-available/taskflow.conf <<'FIN'
<VirtualHost *:80>
    ServerName   taskflow.local
    ServerAlias  www.taskflow.local
    DocumentRoot /var/www/taskflow

    <Directory /var/www/taskflow>
        Options -Indexes +FollowSymLinks
        AllowOverride None
        Require all granted
        FallbackResource /index.html
    </Directory>

    <Directory /var/www/taskflow/admin>
        AuthType Basic
        AuthName "Zona de administracion TaskFlow"
        AuthUserFile /etc/apache2/.htpasswd
        Require valid-user
    </Directory>

    ErrorLog  ${APACHE_LOG_DIR}/taskflow_error.log
    CustomLog ${APACHE_LOG_DIR}/taskflow_access.log combined
</VirtualHost>
FIN
```

AuthType Basic indica que es autenticación básica, AuthName es el texto que aparece al pedir la contraseña, AuthUserFile es la ruta del fichero de usuarios y Require valid-user hace que solo entre un usuario que esté en ese fichero con su contraseña correcta. Este Require sustituye al Require all granted de la carpeta padre solo para /admin.

He usado Directory, que va por la carpeta del disco, y no Location, que va por la URL. Así da igual cómo se escriba la dirección: cualquier petición que acabe leyendo un fichero de esa carpeta pide contraseña. Además, con AllowOverride None un .htaccess no puede quitar la protección.

Luego comprobé la sintaxis y recargué Apache:

```bash
cat /etc/apache2/sites-available/taskflow.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Captura 4: el fichero taskflow.conf con el bloque de /admin, el configtest con Syntax OK y la recarga. El aviso AH00558 sale porque no hay un ServerName global, pero no afecta a la práctica.

![Captura 4](img/captura4.png)

## Paso 4. Comprobar que funciona

Primero pedí /admin/ sin credenciales y luego con una contraseña incorrecta:

```bash
curl -I http://taskflow.local/admin/
curl -I -u admin:malamente http://taskflow.local/admin/
```

En los dos casos devuelve HTTP/1.1 401 Unauthorized con la cabecera WWW-Authenticate: Basic realm="Zona de administracion TaskFlow".

Captura 5: las dos peticiones con 401.

![Captura 5](img/captura5.png)

Después la pedí con el usuario admin. Al poner solo -u admin, curl pide la contraseña por teclado y así no se queda escrita en el historial:

```bash
curl -u admin http://taskflow.local/admin/
```

Con la contraseña correcta devuelve el contenido de la página de administración, o sea que la petición da 200.

Captura 6: acceso con la contraseña correcta.

![Captura 6](img/captura6.png)

Por último comprobé que el resto del sitio sigue accesible sin contraseña:

```bash
curl -I http://taskflow.local/
```

Captura 7: la raíz del sitio devuelve 200 OK sin pedir nada.

![Captura 7](img/captura7.png)

## Paso 5. Añadir y borrar usuarios

Para probar la gestión de usuarios añadí un usuario profe. Esta vez sin -c, para no borrar el fichero:

```bash
sudo htpasswd -B /etc/apache2/.htpasswd profe
sudo cat /etc/apache2/.htpasswd
```

En el fichero aparecen los dos usuarios, admin y profe, cada uno con su hash. Después lo borré con -D:

```bash
sudo htpasswd -D /etc/apache2/.htpasswd profe
sudo cat /etc/apache2/.htpasswd
```

y ya solo queda admin. No hace falta recargar Apache al añadir o borrar usuarios, porque lee el fichero en cada petición.

Por último miré el log de errores del sitio:

```bash
sudo tail -n 5 /var/log/apache2/taskflow_error.log
```

Ahí aparece el error AH01617 de auth_basic con Password Mismatch para "/admin/", que corresponde a la prueba que hice con la contraseña incorrecta.

Captura 8: el fichero con los dos usuarios, el borrado de profe, el fichero con solo admin y el log con el Password Mismatch.

![Captura 8](img/captura8.png)

Resumen de comandos de htpasswd:

| Comando | Para qué sirve |
|---|---|
| sudo htpasswd -c -B /etc/apache2/.htpasswd admin | Crear el fichero con el primer usuario (solo la primera vez) |
| sudo htpasswd -B /etc/apache2/.htpasswd usuario | Añadir un usuario o cambiarle la contraseña |
| sudo htpasswd -v /etc/apache2/.htpasswd usuario | Comprobar la contraseña de un usuario |
| sudo htpasswd -D /etc/apache2/.htpasswd usuario | Borrar un usuario |

## Conclusión

La carpeta /admin de taskflow.local ahora pide usuario y contraseña: sin credenciales o con una contraseña mala devuelve 401, y con la contraseña correcta devuelve la página. El resto del sitio sigue abierto. La contraseña está guardada con bcrypt en un fichero fuera del DocumentRoot y con permisos 640. Lo que queda pendiente para un servidor real es poner HTTPS, porque sin él la contraseña viaja solo en Base64 y se podría leer.
