# Anaire Cloud
- [¿Qué background necesito para poner este proyecto en marcha?](#qu%C3%A9-background-necesito-para-poner-este-proyecto-en-marcha)
- [Descripción de la solución Cloud](#descripci%C3%B3n-de-la-soluci%C3%B3n-cloud)
- [Detalle de la solución cloud](#detalle-de-la-soluci%C3%B3n-cloud)
    - [Componentes](#componentes)
- [Instrucciones paso a Paso](#instrucciones-paso-a-paso)
    - [Crear cuenta AWS y configuración básica](#crear-cuenta-aws-y-configuraci%C3%B3n-b%C3%A1sica)
    - [Crear Volumen](#crear-volumen)
    - [Crear Elastic IP](#crear-elastic-ip)
    - [Crear Security Group](#crear-security-group)
    - [Crear par de claves](#crear-par-de-claves)
    - [Crear imagen base](#crear-imagen-base)
    - [Crear template de VM](#crear-template-de-vm)
    - [Crear Scaling Group](#crear-scaling-group)
- [¿Por qué lo hacemos así?](#por-qu%C3%A9-lo-hacemos-as%C3%AD)
    - [¿Cómo funciona el Scaling Group?](#c%C3%B3mo-funciona-el-scaling-group)
    - [Alternativas](#alternativas)
- [Instalación en cluster K8s genérico](#instalaci%C3%B3n-en-cluster-k8s-gen%C3%A9rico)
- [Subvenciones Cloud](#subvenciones-cloud)
- [¿Cómo revisar el consumo en AWS?](#c%C3%B3mo-revisar-el-consumo-en-aws)

# ¿Qué background necesito para poner este proyecto en marcha?
El objetivo es que seas capaz de desplegar esta solución sin tener conocimientos previos de aplicaciones en la nube, Kubernetes o programación.

¿Y si siguiendo las instrucciones no soy capaz? Entonces no hemos conseguido hacer las instrucciones lo suficiéntemente fáciles. Por favor, avísanos para que te intentemos echar una mano y mejoremos la documentación.
[![Twitter URL](https://img.shields.io/twitter/url/https/twitter.com/anaire_co2.svg?style=social&label=Estamos%20para%20ayudar%20%40anaire_co2)](https://twitter.com/anaire_co2)

Te recomendamos que leas este README al completo y que lleves a cabo los pasos enumerados en sección [Instrucciones Paso a Paso](#instrucciones-paso-a-paso).

**IMPORTANTE** Si vas a utilizar Amazon Web Services consulta la sección de [subvenciones cloud](#subvenciones-cloud) por si pudieses beneficiarte de créditos para hospedar de forma gratuita la solución.

Si tienes experiencia con Kubernetes puedes atreverte con [algunas alternativas](#alternativas) desplegando sobre otra plataforma de tu elección o incluso sobre tu propia máquina física.

# Descripción de la solución Cloud
La solución en la nube consiste en un conjuto de contenedores que se orquestan sobre un cluster Kubernetes.

Nosotros lo hemos orientado como la creación de una máquina virtual que contiene todos los componentes y que ejecutamos en la [nube de Amazon (AWS)](https://aws.amazon.com/). Esto nos da unas ventajas que se enumeran más adelante, pero con unos conocimientos básicos de Kubernetes se podría hacer en cualquier otro hyperscaler (si se desea alojar la solución en la nube) o en un servidor/ordenador/raspberry en el propio colegio/centro en el que se instale la solución.

# Detalle de la solución cloud
## Componentes
En nuestra máquina virtual (o física) instalaremos un cluster Kubernetes (K8s). Dentro de este desplegaremos todos los componentes necesarios para nuestra solución. Estos son:
* **Grafana:** Es la interfaz gráfica en la que se pueden consultar los valores de CO2, temperatura y humedad de cada sensor. Obtiene los datoss consultando a Prometheus.
* **Prometheus:** Es el encargado de almacenar el histórico de todos los datos recibido por los sensores.
* **Pushgateway:** Prometheus está pensado para consultar datos, no para que se los envíen. Ahí entra Pushgateway, quien es capaz de recibir los datos de los sensores y presentarlos de forma que Prometheus pueda consultarlos.
* **MQTT broker:** Mosquitto (MQTT) es un servidor de mensajes ampliamente utilizado en el Internet de las Cosas (IoT). El broker es el servidor de mensajería en el que los dispositivos publicarán las mediciones en el topic 'measurement'. Esta manera de enviar los datos es muy ligera y tiene ventajas adicionales que planteamos incorporar al código en un futuro cercano.
* **MQTT forwarder:** Se trata de un cliente Mosquitto que se subscribirá al broker para que se notifique cada vez que se reciban nuevas medidas. Este forwarder se encarga de mandar los datos a Pushgateway por HTTP, que es el protocolo que este entiende.
![Componentes Anaire Cloud](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/componentes.jpg)
# Instrucciones paso a Paso
Para entender por qué hemos elegido instalar nuestra solución de esta forma por favor consulta la sección [¿Por qué lo hacemos así?](#por-qué-lo-hacemos-así)

**IMPORTANTE:** En los pasos siguientes se va a crear una cuenta en Amazon Web Services y desplegar recursos sobre esta. Desplegar recursos tiene un coste asociado que se factura mensualmente. Estas instrucciones están pensadas para desplegar la solución con un coste mínimo, pero es IMPRESCINDIBLE mantener una monitorización de cuánto consumo se está haciendo para no llevarnos un susto con la factura a fin de mes. Estas instrucciones no garantizan el importe de la factura.

**IMPORTANTE** Asegurate de mantener tus credenciales de la cuenta AWS en lugar seguro y de impedir que nadie tenga acceso a tu VM. Si alguien tiene acceso a tus credenciales podría desplegar recursos incrementando tu factura.

**IMPORTANTE** Intentamos crear unas instrucciones todo lo detalladas que podemos y estaremos encantados de intentarte ayudar a arrancar tu copia de este software, pero no podemos hacernos responsables del consumo ni de la seguridad de tu cuenta en Amazon Web Services ni en ningún otro proveedor cloud. En caso de no estar seguro de ser capaz de poder controlar estos aspectos por tí mismo pide ayuda a alguien con experiencia de despliegues cloud.

## Crear cuenta AWS y configuración básica
[Crea una nueva cuenta](https://portal.aws.amazon.com/billing/signup#/start)
Rellena la información de contacto solicitada y la información de pago.

Esto te proporcionará un usuario raiz. Es una mala práctica usar el usuario raiz para crear recursos. En su lugar crearemos un usuario IAM.
1. [Inicia sesión en la consola](https://console.aws.amazon.com/console/home?nc2=h_ct&src=header-signin) como usuario raíz con los datos que acabas de obtener
2. En la barra de búsqueda de servicios busca IAM y accede a este servicio.
3. En el menú de la izquierda pulsa sobre 'Usuarios'
4. Pulsa el botón 'Añadir usuario(s)'
5. Selecciona nombre de usuario y marca las opciones 'Acceso mediante programación' y 'Acceso a la consola de administración de AWS'
El 'Acceso mediante programación' proporionará las credenciales para que más tarde seamos capaces de gestionar nuestra cuenta mediante un script.
el 'Acceso a la consola de administración de AWS' nos proporcionará un usuario y contraseña que usar en lugar del usuario raiz.
Si lo deseas puedes elegir qué contraseña quieres y si en el próximo login será necesario cambiarla
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/crear_usuario_1.jpg)
Y pulsa 'Siguiente'
6. Selecciona el grupo 'admin' para proporcionar acceso a todos los recursos
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/crear_usuario_2.jpg)
Y pulsa 'Siguiente'
7. Añade alguna etiqueta si lo deseas y pulsa 'siguiente'
8. Revisa la información y pulsa 'Crear un usuario'

Deberías ver una ventana de confirmación con este aspecto. Descarga el archivo CSV y asegurate de dejarlo almacenado en un sitio seguro. El usuario creado tiene permisos de administrador y por tanto puede crer todos los recursos que quiera, lo que podría suponer una factura considerable.
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/crear_usuario_3.jpg)

En el archivo descargado deberías tener los campos "User name,Password,Access key ID,Secret access key,Console login link"
* **User name:** Nombre del usario IAM
* **Password:** Contraseña asignada al usuario IAM
* **Access key ID:** ID que se debe utilizar en el script para permitir al mismo crear recursos en AWS.
* **Secret access key:** Contraseña que se debe usar junto al 'Access key ID' en el script para permitir al mismo crear recursos en AWS
* **Console login link:** Enlace directo para hacer login. El ID de 12 dígitos con el que comienza es el ID de cuenta.

A partir de ahora este será el usuario que utilicemos para crear los recursos y no el usuario raíz.

## Crear Volumen
1. [Inicia sesión en la consola](https://console.aws.amazon.com/console/home?nc2=h_ct&src=header-signin) con ID de cuenta y tu usuario IAM creado en [la sección anterior](#crear-cuenta-aws-y-configuración-básica)
2. En la parte superior derecha se puede seleccionar dónde se quieren crear los recursos. Irlanda es la zona en Europa donde más barato resulta desplegar recursos, te recomendamos esta opción.
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/volumen_1.jpg)
3. En buscar servicios selecciona 'EC2'
4. En el menú lateral izquierdo, en la sección 'Elastic Block Storage' selecciona Volúmenes
5. Pulsa el botón 'Create Volume'
6. Selecciona 'General Purpose SSD (gp2)' y 'Size (GiB)' 5. La availability zone puede ser la que prefieras, pero recuerda en qué Availability Zone (AZ) creas el volumen puesto que la máquina virtual (VM) debe crearse en la misma AZ.
7. Ahora deberías tener un volumen con state 'available'
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/volumen_2.jpg)
Apunta el 'Volume ID', es el identificador que más adelante tendremos que indicar en el script para asociar nuestra máquina virtual a este volumen.

## Crear Elastic IP
1. Asumiendo que acabas de terminar la sección anterior, en el menú lateral izquierdo selecciona 'Direcciones IP elásticas' dentro de la sección 'Red y seguridad'.
2. Pulsa el botón 'Asignar la dirección IP elástica'
3. Deja la selección por defecto 'Grupo de direcciones IPv4 de Amazon' y pulsa el botón asignar.
4. Deberías llegar a una pantalla como la siguiente:
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/eip_1.jpg)
Apunta tanto la 'Dirección IPv4 asignada' como el 'ID de asignnación'. La primera es la IP que usaremos para poder consultar la interfaz gráfica y a la que los dispositivos mandarán las mediciones. La segunda es el identificador que más adelante tendremos que indicar en el script para asociar nuestra máquina virtual a esta IP.
## Crear Security Group
1. Asumiendo que acabas de terminar la sección anterior, en el menú lateral izquierdo selecciona 'Security Groups' dentro de la sección 'Red y seguridad'.
2. Pulsa el botón 'Crear grupo de seguridad'
3. Rellena un nombre y una descripción. En VPC te aparecerá la PVC por defecto, a menos que estés usando una cuenta AWS en la que hayas creado VPCs adicionales y quieras usar una de estas no lo toques.
4. Añade las siguientes reglas de entrada:
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/security_group_1.jpg)
NOTA: Sólo estamos permitiendo comunicación con el puerto que usará el broker MQTT y con el puerto que usaremos para consultar Grafana. No estamos permitiendo SSH pues eso haría nuestra máquina más vulnerable. Si fuese necesario hacer troubleshooting se habilitaría el SSH sólo desde la IP que estemos usando (esto estará descrito más adelante en una sección de troubleshooting).
5. Mantén las reglas de salida como están (permitiendo todo el tráfico).
6. Pulsa el botón 'Crear grupo de seguridad'

## Crear par de claves
1. Asumiendo que acabas de terminar la sección anterior, en el menú lateral izquierdo selecciona 'Par de claves' dentro de la sección 'Red y seguridad'.
2. Pulsa el botón 'Crear par de claves'
3. Escribe un nombre para el par de claves
4. Selecciona el formato deseado del archivo. Tipicamente será pem, pero si tu intencción es usar PuTTy para conectar con la máquina selecciona ppk.
5. Pulsa 'Crear par de claves'. Al hacerlo se descargará automáticamente tu clave privada. Guárdala en lugar seguro porque será la forma de acceder a tu máquina virtual.

## Crear imagen base
El siguiente paso es crear temporalmente una máquina virtual que usaremos para obtener una imagen modificada, que es la que realmente usaremos para la máquina virtual corriendo la funcionalidad.

1. Asumiendo que acabas de terminar la sección anterior, en el menú lateral izquierdo selecciona 'Instancias' dentro de la sección 'Instancias'.
2. Pulsa el botón en la parte superior derecha 'Lanzar instancias'
3. Select 'Minimal Ubuntu 18.04 LTS - Bionic' form 'AWS Marketplace'
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_1.jpg)

4. Selecciona 'Continue'
![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_2.jpg)

5. Selecciona 't3a.small' y pulsa 'Next: Configure Instance Details'

**OJO: Instrucciones incompletas:** Hay que asegurarse de que se elige la subnet correspondiente a la Availability Zone en la que se creó el volumen. La máquina virtual y el volumen deben pertenecer a la misma Availability Zone.

6. Desplazate hasta la caja 'User data' dentro de la sección 'Advanced Details'.
En esta caja tienes que copiar el contenido del [script para configurar la VM de nuestro repositorio Github](https://github.com/anaireorg/anaire-cloud/blob/main/stack/user_data_new_vm.sh)

**OJO** Tienes que editar las variables en la sección AWS credentials con los datos de tu access key and secret, tu región (si seleccionaste Irlanda es eu-west-1) el id del volumen que has creado y el id de la IP elástica que has creado.

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_4.jpg)

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_5.jpg)

7. Selecciona 'Review and Launch'.

8. Selecciona 'Launch'.

9. Una ventana aparecerá para seleccionar el par de claves que creamos anteriormente. Marca 'Choose an existing key pair' y seleccionalo en la segunda caja. Marca el checkbox y pulsa 'Launch Instances'.

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_3.jpg)

10. Deberías obtener una imagen de confirmación como la siguiente

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_6.jpg)

11. Pulsa el botón 'View instances' para ir a ver tu instancia en el panel de instancias. Después de un rato debería tener este aspecto. Si no lo tiene pulsa el símbolo para actualizar.

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_7.jpg)

12. Espera hasta que el script que hemos copiado en el 'user data' complete su ejecución. Esto lo podríamos hacer haciendo ssh a la máquina con la clave privada que hemos descargado y comprobando el contenido de /home/ubuntu/userdata.txt, pero podemos simplemente esperar unos 10 minutos para estar seguros y proceder.

13. Selecciona la instancia, pulsa en 'Acciones', 'Image and templates' y 'Crear imagen'.

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_8.jpg)

14. Selecciona un nombre y asegúrate de quitar el dispositivo /dev/sdb (de 5GB), sólo queremos hacer el snapshot del disco de 8GB. Si no te apareciese el disco de 5GB es que el script de User Data no se ha ejecutado correctamente, puede ser un problema con las variables o por haber esperado poco tiempo a que se aplicase.

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_9.jpg)

15. Pulsa 'Crear Imagen'.

16. Cuando la creación acabe vuelve a instancias.

17. Selecciona la instancia, y pulsa 'Terminar instancia' en el desplegable 'Estado de instancia'.

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/image_10.jpg)

En este punto tenemos:
* Un volumen preparado para almacenar los datos de nuestras aplicaciones
* Una IP elástica, que será la IP que usarán tanto nuestros dispositivos para enviar los datos como nosotros para acceder a la interfaz gráfica.
* Una imagen de VM con todos los paquetes que necesitamos para poder correr nuestra aplicación. 

## Crear template de VM
1. En el menú laterar de EC2, ve a 'Plantillas de lanzamiento' dentro de la sección 'Instancias'.
2. Pulsa el botón 'Crear plantilla de lanzamiento'.
3. Da un nombre a la plantilla.
4. Selecciona como 'Imagen de Amazon Machine (AMI)' la imagen que creamos anteriormente
5. Selecciona t3a.small como 'Tipo de instancia'
6. Selecciona el par de claves que creamos anteriormente
7. Abre el desplegable 'Detalles avanzados'
8. Desplázate hasta el final de la página donde se encuentra la caja 'Datos de usuario'
En esta caja tienes que copiar el contenido del [script para configurar scaling group de nuestro repositorio Github](https://github.com/anaireorg/anaire-cloud/blob/main/stack/user_data_scaling_group.sh)

**OJO** Tienes que editar las variables en la sección AWS credentials con los datos de tu access key and secret, tu región (si seleccionaste Irlanda es eu-west-1) el id del volumen que has creado y el id de la IP elástica que has creado.

## Crear Scaling Group
Ahora crearemos un scaling group que contendrá una sola VM. Este se basará en el template que acabamos de crear y que se encarga de cada vez que la vm se arranque enlazarla con el volumen, con la IP elástica y volver a arrancar las aplicaciones.

1. En el menú laterar de EC2, ve a 'Grupos de Auto Scaling' dentro de la sección 'Auto Scaling'.
2. Pulsa el botón Crear grupo de Auto Scaling
3. Elige nombre y selecciona la plantilla que hemos creado. Pulsa siguiente.
4. En la sección 'Red', en el apartado 'Subredes' selecciona únicamente la subred correspondiente a la Availability Zone en la que creaste el volumen anteriormente.
5. Pulsa siguiente dejando los parámetros por defecto hasta que llegues a la pantalla 'Revisar'.
6. Pulsa el botón 'Crear grupo de Auto Scaling'

# ¿Por qué lo hacemos así?
AWS, al igual que el resto de hyperscalers tienen servicios de Kubernetes gestionados que ofrecen alta disponibilidad y te ocultan la complejidad de la gestión del cluster. Pese a ello, la aproximación seguida aquí tiene ventajas:
* Ejecutar en un hyperscaler (AWS en este caso) permite 
    * Tener acceso a los datos a través de Internet desde cualquier ubicación.
    * Reduce la probabilidad de que falle la VM
    * Posibilidad de uso de Scaling Group, maximizando la resiliencia de la solución
* Ejecutar en una VM autocontenida permite
    * Minimizar el coste
    * Poder migrar la solución fácilmente a otra plataforma
    * Ampliar el setup incluyendo una segunda VM autocontenida que sean copia una de otra sería relativamente sencillo haciendo que los broker MQTT funcionasen como cluster.

## ¿Cómo funciona el Scaling Group?
Básicamente el scaling group lo que hace es monitorizar en número de instancias que tienes en el grupo y asegurarse de que siempre se esté en el número deseado de instancias.

En nuestro caso el número deseado es 1, y en el template hemos hecho que esa instancia esté ligada a nuestro volumen y a nuestra elastic IP. Esto consigue que si la VM es terminada, ya sea porque por error la borramos o porque hay un fallo catastrófico en AWS que provoca que la VM muera, otra VM la reemplazaría, pero con la misma IP y con los mismos datos almacenados, por lo que a todos los efectos sería como si no hubiese fallado.

En nuestra experiencia el tiempo que tarda la nueva máquina virtual en estar accesible prestando servicio es entre 5 y 10 minutos desde que se borra la VM original.

![image](https://github.com/anaireorg/anaire-cloud/raw/main/screenshots/scaling_group.jpg)

## Alternativas
Realmente la aplicación no tiene requisitos especiales, sólo necesita un cluster Kubernetes y crear el directorio /data, que es donde se almacenan los datos persistentes.
Podría utilizarse hospedaje para la VM en cualquier otro hyperscaler, usar un servicio de Kubernetes gestionado o instalar el cluster Kubernetes en un equipo físico dentro del centro (colegio) a monitorizar.

# Instalación en cluster K8s genérico
Los ficheros yaml para lanzar la aplicación están en [el directorio stack de Github](https://github.com/anaireorg/anaire-cloud/tree/main/stack).
Es importante tener en cuenta que hay dependencias. El orden correcto de despliegue es:
1. **Pushgateway**
2. **Prometheus**. Para desplegarlo es necesario sustituir la variable del ClusterIP de Pushgateway en el manifest.
3. **Grafana**. Para desplegarlo es necesario sustituir la variable del ClusterIP de Prometheus en el manifest.
4. **MQTT Broker**
5. **MQTT Forwarder**
# Subvenciones Cloud
Comprueba si gracias a alguno de estos programas podrías obtener créditos para correr la aplicación de forma gratuita.
* [Open data sponsorship program](https://aws.amazon.com/es/opendata/open-data-sponsorship-program/)
* [AWS Educate](https://aws.amazon.com/es/education/awseducate/)
# Cómo revisar el consumo en AWS
Accede a la sección de Billing para tener acceso al consumo actual y a una predicción del consumo esperado a fin de mes. 
Por defecto los usuarios administradores no tienen acceso completo al billing. Puedes conceder estos permisos siguiendo la documentación de AWS o usar el usuario raiz para realizar estas consultas.
