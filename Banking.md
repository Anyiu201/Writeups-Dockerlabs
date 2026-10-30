![[banking1.png]]
## Información general

- **Plataforma:** Dockerlbs
- **Sistema operativo:** Linux
- **Dificultad:** Fácil
- **Objetivo:** Practicar el hacking movil y analisis de APK vulnerable
- **IP del laboratorio:** 172.17.0.2

## Reconocimiento

###  Identificación del objetivo

Descargo la máquina desde la plataforma DockerLabs y descomprimo el archivo proporcionado para preparar su despliegue.

```bash
unzip banking.zip
```

A continuación, ejecuto el script de automatización incluido con el laboratorio:

```bash
sudo bash auto_deploy.sh banking.tar
```

Una vez desplegado el entorno, identifico la dirección IP asignada a la máquina objetivo:

```text
172.17.0.2
```


![[banking2.png|700]]
### Escaneo de puertos

Realizo un escaneo de puertos TCP con Nmap para identificar los servicios expuestos por la máquina objetivo.

```
nmap -p- --open -sS -sC -sV --min-rate=5000 -vvv -n -Pn -oN escaneo 172.17.0.2
```

- `-p-`: analiza todos los puertos TCP, del 1 al 65535.
- `--open`: muestra los puertos que Nmap identifica como abiertos.
- `-sS`: realiza un escaneo TCP SYN.
- `-sC`: ejecuta los scripts predeterminados de Nmap.
- `-sV`: intenta identificar las versiones de los servicios.
- `--min-rate=5000`: establece una tasa mínima de envío de 5000 paquetes por segundo, si las condiciones lo permiten.
- `-vvv`: aumenta el nivel de detalle de la salida.
- `-n`: evita la resolución DNS.
- `-Pn`: omite la detección inicial de disponibilidad del host.

El escaneo identifica el siguiente servicio:

![[banking3.png|700]]

| Puerto   | Servicio | Información identificada |
| -------- | -------- | ------------------------ |
| 8000/TCP | HTTP     | Servidor Web uvicorn     |

La presencia de un servicio HTTP indica que existe una superficie de ataque web que debe analizarse mediante la inspección de páginas, recursos publicados y posibles configuraciones incorrectas.
## Enumeración Web

### Servicio HTTP

Accedo al servicio web mediante el navegador:

```text
http://172.17.0.2:8000/
```

Durante la navegación identifico una página web que simula el portal de una institución bancaria. Observo dos elementos relevantes:

1. Una aplicación móvil destinada a descargar y administrar las cuentas bancarias.
2. Un mecanismo de autenticación para acceder al portal web.
    
La aplicación móvil se convierte en un punto de interés porque el análisis de su código puede revelar rutas, configuraciones o secretos que no deberían estar disponibles para un usuario no autorizado.

### Análisis estático de la APK con JADX

Descargo la aplicación móvil proporcionada por el laboratorio y procedo a inspeccionar su contenido mediante JADX.

JADX permite decompilar aplicaciones Android y examinar el código reconstruido a partir del archivo APK. Aunque el resultado no siempre reproduce exactamente el código fuente original, facilita la identificación de rutas, cadenas de texto, parámetros y configuraciones.

Durante la revisión encuentro una ruta que apunta a un recurso del servidor web.

![[banking4.png]]

A partir de la ruta identificada durante el análisis estático, accedo al recurso publicado en el servidor y compruebo que es posible recuperar un repositorio Git.

![[banking5.png|700]]

Ejecuto el siguiente comando:

```bash
git clone http://172.17.0.2:8000/backups/secret/.git/
```

El comando intenta clonar el repositorio accesible a través del servidor web.

La exposición de un directorio `.git` puede permitir recuperar información del proyecto, incluidos archivos de configuración, versiones anteriores de archivos y datos sensibles que no deberían estar publicados.

Una vez descargado el repositorio, reviso los archivos disponibles en busca de información sensible, como credenciales, claves criptográficas o tokens.

Durante la inspección encuentro un archivo llamado `Copy.py` que contiene una variable `SECRET_KEY` asociada al usuario `admin`.

![[banking6.png|700]]

Accedo al portal bancario y utilizo una cuenta de prueba creada durante el ejercicio.

Intercambio las solicitudes con Burp Suite para inspeccionar las peticiones HTTP y localizar el token de sesión utilizado por la aplicación.

Identifico que la autenticación utiliza un token JWT (JSON Web Token).

Un JWT puede contener declaraciones o _claims_ que describen al usuario autenticado. En un esquema de firma simétrica, el servidor utiliza una clave secreta para generar y verificar la firma del token.

Es importante distinguir que la firma de un JWT no cifra automáticamente su contenido: las declaraciones de un JWT convencional pueden ser legibles por quien obtenga el token.

Con el token de mi cuenta de prueba, examino su estructura y las declaraciones incluidas. A continuación, utilizo la clave `SECRET_KEY` encontrada en `Copy.py` para realizar la prueba de manipulación del token dentro del laboratorio.

Modifico las declaraciones pertinentes para que el token represente al usuario `admin` y verifico si la clave descubierta permite producir una firma que el servidor acepte.

![[banking7.png|700]]

Una vez obtenido el token que representa al usuario administrador, sustituyo el token de mi sesión de prueba por el token manipulado.

Recargo el portal y compruebo que la aplicación muestra información correspondiente a la cuenta administrativa.

![[banking8.png]]

Este resultado confirma que la manipulación del token ha permitido acceder a información protegida de otro usuario dentro del laboratorio.

## Análisis de vulnerabilidades

### Exposición de información sensible en una APK

- **Hallazgo:** la aplicación móvil contiene referencias que permiten descubrir recursos del servidor.
- **Causa raíz:** publicación de información que facilita la enumeración de recursos internos o sensibles.
- **Impacto:** puede reducir el esfuerzo necesario para localizar otros puntos de entrada.
- **Mitigación recomendada:** evitar incluir credenciales y secretos en aplicaciones cliente; revisar las referencias y configuraciones antes de distribuir el APK.
### Repositorio Git accesible desde el servidor web

- **Hallazgo:** es posible recuperar el repositorio mediante una ruta publicada en el servicio HTTP.
- **Causa raíz:** configuración incorrecta del servidor web o publicación de archivos internos del proyecto.
- **Impacto:** exposición del código, configuraciones, historial de cambios y posibles secretos.
- **Mitigación recomendada:** bloquear el acceso web a directorios `.git`, excluirlos de los artefactos de despliegue y revisar los archivos publicados.
### Exposición de la clave secreta de autenticación

- **Hallazgo:** el archivo `Copy.py` contiene una variable `SECRET_KEY` asociada al mecanismo de autenticación.
- **Causa raíz:** almacenamiento y publicación de un secreto en un archivo recuperable desde el servidor.
- **Impacto:** si se utiliza para firmar JWT y el backend acepta tokens generados con esa clave, un atacante podría falsificar declaraciones y suplantar usuarios.
- **Mitigación recomendada:** almacenar los secretos en un gestor de secretos o en variables de entorno protegidas; restringir su acceso; rotar la clave comprometida e invalidar las sesiones afectadas.

### Validación insuficiente de la identidad contenida en el JWT

- **Hallazgo:** el servidor acepta un token manipulado que permite acceder como administrador.
- **Causa raíz:** confianza en declaraciones de identidad que pueden ser falsificadas cuando la clave de firma queda comprometida, o validación insuficiente del proceso de autenticación y autorización.
- **Impacto:** suplantación de identidad y acceso no autorizado a información protegida.
- **Mitigación recomendada:** validar la firma y el algoritmo permitido, comprobar las declaraciones relevantes (`iss`, `aud`, `exp` y otras que correspondan), aplicar autorización en el servidor para cada recurso y proteger las claves de firma. Para privilegios administrativos, utilizar controles adicionales cuando el modelo de amenazas lo requiera.
## Conclusiones

Este laboratorio demuestra cómo varios fallos de seguridad aparentemente independientes pueden combinarse para comprometer una aplicación web.

La ruta de ataque comienza con la enumeración del servicio HTTP y el análisis estático de una aplicación Android. A partir de una ruta descubierta en la APK, identifico un repositorio Git expuesto y recupero un archivo que contiene una clave secreta asociada al mecanismo de autenticación.

Posteriormente, utilizo esa información para comprobar la posibilidad de manipular un JWT y suplantar al usuario administrador. El resultado es el acceso a información protegida de la aplicación.

Los principales aprendizajes técnicos son:

- La importancia del análisis estático de APK en una auditoría de seguridad móvil.
- Los riesgos de publicar repositorios Git en servidores web.
- La necesidad de proteger las claves criptográficas y los secretos de configuración.
- El funcionamiento de los JWT y la importancia de validar su firma y sus declaraciones.
- La diferencia entre una escalada de privilegios dentro de una aplicación y una escalada de privilegios en el sistema operativo.
    
La lección principal es que la seguridad debe abordarse de forma integral: una referencia expuesta en una aplicación móvil puede conducir a un repositorio accesible y, finalmente, a la suplantación de una identidad privilegiada.
## Referencias

- [JADX — Decompilador de aplicaciones Android](https://github.com/skylot/jadx)
- [Burp Suite — Documentación oficial](https://portswigger.net/burp/documentation)
- [JWT — RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)
- [OWASP MASVS — Mobile Application Security Verification Standard](https://mas.owasp.org/MASVS/)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)
    