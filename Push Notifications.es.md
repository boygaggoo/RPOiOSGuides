#Notificationes Push de iOS (APNS)

![img](http://core0.staticworld.net/images/article/2013/09/ios_7_notification_center-100054497-poster.jpg)

Esta guía te ayudará a empezar a trabajar con las **notificaciones push de iOS** en dispositivos reales (ya que en el simulador no están soportadas) aprovechando el *Servicio de notificaciones push de Apple*.


##¿Qué son las APNS?

Las Notificaciones Push de Apple (Apple Push Notifications en inglés o APNS) es la pieza central del sistema de notificaciones push. Es un servicio robusto y altamente eficiente para propagar información a dispositivos iOS y OS X. Cada dispositivo establece una conexión IP acreditada y encriptada con el servicio y recibe notificaciones mientras se mantenga la conexión. Si una notificación para una aplicación es recibida cuando la aplicación no está en funcionamiento, el dispositivo alerta al usuario que la aplicación tiene nuevos datos a la espera de ser revisados.

##¿Cómo funciona APNS?

![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/registration_sequence_2x.png)

Si el servicio de notificaciones está activado en tu aplicación, iOS generará un ID único, llamado **token de dispositivo**. Este token se genera cuando la aplicación se abre por primera vez y el usuario acepta que la app le envíe notificaciones push. Necesitamos enviar este token a nuestro servidor y almacenarlo en una base de datos para poder enviar peticiones al servidor APNS. Entonces, tendremos que usar una librería para interactuar con el servidor APNS. Veamos cuáles son estas librerías.

##Librerías APNS de servidor

###PHP

* [**EasyAPNS**](http://www.easyapns.com/)
* [**ApnsPHP**](https://code.google.com/p/apns-php/)

###Node.js

* [**apnagent**](http://apnagent.qualiancy.com/) (*Usaremos esta en esta guía*)

##Configurando el soporte de notificaciones push en nuestra app

Antes de nada, tenemos que estar registrado como [Desarrollador Apple](http://developer.apple.com) ya que necesitamos crear un App ID para nuestra app que soporte notificaciones push así como un _provisioning profile_ enlazado a ese App ID y los dispositivos en los que probemos las notificaciones. Este _provisioning profile_ debe ser importado a XCode.

Una vez hecho esto, debemos decirle a iOS que nuestra app quiere recibir notificaciones push. Así que debemos añadir el siguiente código en el método ```application:didFinishLaunchingWithOptions:``` de nuestro AppDelegate:

	[[UIApplication sharedApplication] registerForRemoteNotificationTypes: (UIRemoteNotificationTypeBadge | UIRemoteNotificationTypeSound | UIRemoteNotificationTypeAlert)];

Esto hará que nuestra app pueda cambiar el número que muestra el globo de notificaciones, reproducir sonidos mientras está cerrada y mostrar notificaciones de un servidor remoto. Si no necesitas alguna de estas opciones,  puedes quitarlas sin problema.	
	
![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/token_generation_2x.png)

Cuando abramos nuestra app y esta detecte que debería recibir notificaciones push, conectará con el servidor APNS solicitando un **token de dispositivo**. Este token es un string de 32 bytes con 64 carácteres hexadecimales. Usaremos este ID único para enviar notificaciones a este dispositivo. Si queremos enviar una notificación a un usuario, por ejemplo, que tiene varios dispositivos, tendremos que almacenar un token por dispositivo en nuestro servidor y enviar una notificación por dispositivo.

Con ```-didRegisterForRemoteNotificationsWithDeviceToken:``` obtendremos el token de dispositivo que vamos a usar en nuestro servidor.

	-(void)application:(UIApplication*)application didRegisterForRemoteNotificationsWithDeviceToken:(NSData*)deviceToken {
	
        NSString *token = [deviceToken description];
	}
Después, lo único que debemos hacer es subirlo a nuestro servidor con una petición simple.	

##Lado del servidor: configurando la comunicación APNS

![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/remote_notif_simple_2x.png)

El sistema de notificaciones push funcionará de la siguiente manera: tenemos un proveedor, que es nuestro server, que enviará una petición con datos al servidor de APNS de Apple. El dispositivo recibirá una notificación y si los datos coinciden con la información del dispositivo y las notificaciones están activadas en este, se mostrarán al usuario.

De acuerdo, ahora nuestro dispositivo está listo para recibir notificaciones push. Pero necesitamos un servidor con el que enviar los datos al servidor APNS de Apple.

Estos datos vienen en un formato JSON que especifica como va a ser notificado el usuario a través del dispositivo. Contiene el mensaje, el contador que aparece en el globo, el nombre del fichero de sonido que sonará con la alerta y campos propios si el desarrollador quiere enviar información que no sea parte de la notificación en si pero que puede ser usada por la app. Por ejemplo:



	{
    	"aps" : {
        	"alert" : "Hey! I'm a push notification! :D",
        	"badge" : 0,
        	"sound" : "chime"
    	},
    	"acme1" : "bar",
    	"acme2" : ["data1", "data2"]
	}

![](https://developer.apple.com/library/ios/documentation/NetworkingInternet/Conceptual/RemoteNotificationsPG/Art/token_trust_2x.png)

Antes de poder crear la API en nuestro servidor, necesitamos algunos certificados.

###Certificates

1. Entra en el _iOS Provision Portal_ desde la zona de miembros de la web de Apple Developer. Selecciona "App IDs" en el menú de la izquierda, después elige la opción "configure" de la aplicación para la que deseas generar el certificado.

2. Ahora necesitamos activar el soporte de APNS en esta aplicación. Si no está activado ya, haz check en la opción "Enable for Apple Push Notification server".

3. Selecciona "Configure" en el entorno (desarrollo y/o producción) para el que deseas generar los certificados. Sigue las instrucciones en pantalla para generar un fichero de tipo CSR. Asegúrate de guardar este fichero CSR en un sitio seguro para que puedas reutilizarlo con futuros certificados.

4. Tras subir tu fichero CSR se generará un "Apple Push Notification service SSL Certificate". Descárgalo y guárdalo en un sitio seguro en tu disco.

5. Una vez descargado, localiza el archivo con Finder y haz doble click sobre él para importarlo en la aplicación "Acceso a Llaveros". Usa los filtros de la parte izquierda para localizar el nuevo certificado que has importado si no lo ves a simple vista. Aparecerá listado bajo el llavero "login" en la categoría de "Certificados". Una vez localizado, haz clic con el botón derecho y elige "Exportar".

6. Cuando la ventana de exportación aparezca, asegúrate que la opción "Formato de Fichero" indica *.p12*. Ponle un nombre acorde al entorno para el que está creado el certificado, como **apnscert-dev.p12** o **apnscert-prod.p12** y guárdalo en el directorio de tu proyecto. Se solicitará un password por si queremos proteger el certificado exportado. Es opcional, pero si lo indicamos, tendremos que configurar la librería que vayamos a usar con dicho password.

7. Las clases de apnagent soportan ficheros *.p12* a través de las opciones de configuración pfx para autenticar conexiones. El último paso es opcional pero enseña a convertir un fichero *.p12* en un par "clave" y "certificado" (formato *.pem*).

Localiza tu fichero *.p12* con el terminal y ejecuta los siguientes comandos:

	openssl pkcs12 -clcerts -nokeys -out apns-dev-cert.pem -in apnscert-dev.p12
	openssl pkcs12 -nocerts -out apns-dev-key.pem -in apnscert-dev.p12

El primero generará un fichero *.pem* para usarlo como certificado. El segundo generará un fichero *.pem* para usarlo como clave. El segundo necesita un password para asegurar la clave. Se puede quitar ejecutando el siguiente comando:

	openssl rsa -in apnagent-dev-key.pem -out apnagent-dev-key.pem

###Construyendo la API

Ahora tenemos que construir una API sencilla en nuestro servidor para manejar la recepción del token de dispositivo y el control de las notificaciones remotas.

En esta guía usaremos [**apnagent**](http://apnagent.qualiancy.com/), un módulo node.js para APNS. Primero de todo, debemos tener instalado node.js. Una vez hecho esto, ejecutamos el siguiente comando en el terminal:

	npm install apnagent

En nuestro directorio del servidor, creamos un fichero **apns.js** (o con cualquier otro nombre) y un directorio con los certificados que hemos generado, tanto de desarrollo como de producción. En el fichero **apns.js** escribimos lo siguiente:

	var apnagent = require('apnagent'),
		agent = module.exports = new apnagent.Agent();

La variable *agent* será nuestro manejador de APNS. Debemos decirle a apnagent los certificados que vamos a usar:		

	agent
		.set('cert file', join(__dirname, 'certs/apns-dev-cert.pem'))
		.set('key file', join(__dirname, 'certs/apns-dev-key.pem'))
		.enable('sandbox');

Si vamos a usar nuestro servidor en producción, cambiaremos los certificados por los de dicho entorno:	
		
	agent
		.set('cert file', join(__dirname, 'certs/apns-prod-cert.pem'))
		.set('key file', join(__dirname, 'certs/apns-prod-key.pem'));
				
Una vez configurado, estamos listos para enviar notificaciones push al servidor APNS. Echa un vistazo a las funciones que vamos a asignar, como ```alert```, ```badge```, ```sound```, etc. Toda la información que vamos a enviar en este código coincide con los datos que pusimos anteriormente en la guía. *apnagent* transformará toda esta información en un JSON para enviarlo al servidor APNS.
		
	agent.createMessage()
    	.device("<a1b56d2c 08f621d8 7060da2b c3887246 f17bb200 89a9d44b fb91c7d0 97416b30>")
    	.alert("Hey! I'm a push notification! :D")
    	.badge(0)
    	.sound("chime")
    	.set("acme1", "bar")
    	.set("acme2", ["data1", "data2"])
    	.send(function (err) {

      	if (err && err.toJSON) {
        	res.json(400, { error: err.toJSON(false) });
        	
        	// Handle the error
      	} 

      	else if (err) {
        	res.json(400, { error: err.message });
        	
        	// Handle anything else (not likely)
      	}

      	else {
        	res.json({ success: true });
        	
        	//Success!
      	}
    });
    
Y si todo ha ido bien, la notificación aparecerá en el dispositivo.    


##Autor

Hola, soy Álvaro Franco, puedes contactar conmigo en:

* [**@alvarofr_** en Twitter](http://twitter.com/alvarofr_)
* [**@alvarofranco** en Github](http://github.com/alvarofranco)

Si tienes cualquier duda sobre esta guía, contacta conmigo a través del email [alvarofrancoayala@gmail.com](mailto:alvarofrancoayala@gmail.com)