# Instrumentando microservicios en Go con Gin y AWS X-Ray

En una arquitectura de microservicios, las operaciones muchas veces abarcan múltiples servicios y recursos tales como gateways, microservicios, balanceadores de carga, bases de datos entre otros. La naturaleza distribuida de los microservicios es lo que hace invaluable la instrumentazión de software.

Si nuestro código provee información de traceo para requests, y logs, podemos decir que está instrumentado y que podemos observar cómo se está desempeñando nuestro sistema.

La instrumentación de servicios es especialmente útil para identificar y resolver problemas de rendimiento y errores. Los datos recolectados pueden ser usados para planear la capacidad de nuestros servicios al ayudarnos a entender el tráfico y patrones de uso en nuestras aplicaciones.

Existen varias soluciones para instrumentar nuestros servicios, como [OpenTelemetry](https://opentelemetry.io/), [Zipkin](https://zipkin.io/) y [datadog](https://www.datadoghq.com/). AWS también ofrece una [Distribución de OpenTelemetry](https://aws-otel.github.io/) para poder usar OpenTelemetry como backend de obserbabilidad mientras usas X-Ray o cualquier otra solución de terceros para recibir datos de telemetría y proveer procesamiento, agregación y visualización de éstos.

En este post, les voy a contar sobre mi experiencia al instrumentar un microservicio en Go usando Gin y AWS X-Ray.

## Gin

Gin es un framework para el lenguaje de programación Go para crear aplicaciones web, se destaca por ser ligero y tener un alto rendimiento, diseñado para facilitar la creación de aplicaciones web escalables de una manera rápida.

Ofrece una API minimalista, un router robusto, soporte para middleware y características de seguridad integradas, lo que lo convierte en una opción ideal para construir microservicios y otras aplicaciones web de alto rendimiento.

Si bien Gin puede tener una curva de aprendizaje empinada y características limitadas integradas, su simplicidad y capacidad de extensión lo convierten en una opción popular para los desarrolladores que priorizan el rendimiento y la escalabilidad.

**Crear un servicio de Gin desde cero está fuera del alcance de esta publicación**, pero puedes leer más sobre Gin en la [página oficial de su documentación]((https://gin-gonic.com/docs/)).

## AWS X-Ray

AWS X-Ray es un servicio de AWS que recolecta datos sobre los requests servidos por tu aplicación y provee herramientas para ver, filtrar y obtener información sobre esos datos para identificar problemas y oportunidades de optimización.

Algunos puntos a favor de X-Ray sobre otras herramientas similares son:

* Facilidad de integración con otros servicios de AWS.
* No hay infraestructura extra qué mantener (el daemon de X-Ray está incluído en las plataformas AWS Elastic Beanstalk y AWS Lambda).
* Puede funcionar sólo como visualizador (usando OpenTelemetry como tracer).
* Para servicios soportados, el SDK de X-Ray puede enviar y rastrear automáticamente los "ID de request" entre los servicios.
* Es administrado por AWS.
* Los primeros 100k rastreos del mes son gratis.
* El primer millón de rastreos obtenidos o escaneados cada mes es gratis.

Sin embargo algunos puntos en contra son:

* AWS X-Ray sólo puede ser usado con aplicaciones corriendo en Amazon EC2, Amazon EC2 containser service, AWS Lambda, y AWS Elastic Beanstalk.
* Después de agotar los rastreos gratuitos del mes, cada rastreo indexado y consultado tiene un costo.
* Soporte limitado de lenguajes: Mientras que el SDK de X-Ray tiene soporte para varios lenguajes de programación, no soporta todos los lenguajes o plataformas, lo cual puede limitar su utilidad en algunos casos.
* Vendor lock-in: El uso de X-Ray puede llevar a la dependencia exclusiva de AWS, ya que es un servicio propietario disponible sólo en la plataforma de AWS. Esto puede limitar su capacidad para cambiar a otros proveedores de nube o herramientas en el futuro.

Si, después de leer algunos de los pros y contras, aún estás inclinado a usar X-Ray, entonces puedes seguir leyendo.

### Requerimientos

Para ver la información de rastreo en AWS X-Ray, necesitas una cuenta de AWS y una aplicación corriendo en la infraestructura de AWS o que esté integrada con los servicios de AWS. Además, necesitarás:

* Una instancia del X-Ray daemon, que se puede ejecutar como un binario o como un contenedor de Docker. Puedes encontrar instrucciones detalladas sobre cómo ejecutar y configurar el daemon [aquí](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-local.html). Para éste artículo, estaré usando el binario para OS X.
* Tu aplicación debe de tener los permisos necesarios para interactuar con AWS X-Ray y otros servicios que use.

#### IAM Role

Para permitir que tu aplicación mande información sobre tus requests a X-Ray, tienes que proveerle al daemon de X-Ray un rol. Para crear un rol, vamos a entrar a nuestra consola web de AWS y de allí navegamos a la página principal de IAM y allí encontraremos el botón "Create Role" (o "Crear Rol" si tienes configurado tu panel de AWS en español).

![Crear nuevo rol](https://github.com/makkoman/blogposts/blob/main/x-ray/images/create-role.png)

En el asistente, selecciona "AWS Account" para Trusted Entity y da click en "Next"/"Siguiente". En la siguiente pantalla, busca por la política de permisos llamada "AWSXRayDaemonWriteAccess". Da click en "Next"/"Siguiente" para continuar..

![Nombra, Revisa y Crea](https://github.com/makkoman/blogposts/blob/main/x-ray/images/name-review-create.png)

Agrega un nombre y descripción para el rol, y después da click en "Create Role". Ésto te llevará a la lista de roles. Busca el rol que acabas de crear para ver y copiar su ARN.

![Detalles del rol](https://github.com/makkoman/blogposts/blob/main/x-ray/images/role-details.png)

### X-Ray Daemon

Ahora que ya tenemos el rol para el daemon, vamos a configurarlo.

Para mi proyecto de prueba, solo tuve que cambiar algunos valores de la configuración, como el nivel del logger, especificar el modo local a verdadero, y agregar el ARN del rol que creamos y la región de AWS en la que estamos operando nuestros servicios.

Aquí está la configuración que usé:

```yaml
# Send segments to AWS X-Ray service in a specific region
Region: "us-west-2"
Socket:
  # Change the address and port on which the daemon listens for UDP packets containing segment documents.
  UDPAddress: "127.0.0.1:2000"
  # Change the address and port on which the daemon listens for HTTP requests to proxy to AWS X-Ray.
  TCPAddress: "127.0.0.1:2000"
Logging:
  # Change the log level, from most verbose to least: dev, debug, info, warn, error, prod (default).
  LogLevel: "dev"
# Turn on local mode to skip EC2 instance metadata check.
LocalMode: true
# Assume an IAM role to upload segments to a different account.
RoleARN: "arn:aws:iam::269174633178:role/X-Ray_Daemon_role"
# Daemon configuration file format version.
Version: 2
```

En [la guía del desarrollador de AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/xray-daemon-configuration.html) puedes aprender más sobre otros valores que puedes configurar.

### Instrumentando tu microservicio en Go

Ahora que ya tenemos el X-Ray daemon configurado y corriendo, podemos proceder a instrumentar nuestro servicio.

AWS recomienda empezar agregando rastreo para requests entrantes envolviendo los controladores de servicio con `xray.Handler`. Pero, como estamos usando Gin, el enfoque que implementaremos es ligeramente diferente.

Mientras buscaba recursos sobre cómo instrumentar una aplicación con Gin, me encontré con éste [middleware](https://github.com/oroshnivskyy/go-gin-aws-x-ray), el cual está basado en la función [`xray.Handler`](https://github.com/aws/aws-xray-sdk-go/blob/1e154184282bb3b0166cb1b154f2b4abed0b1e6f/xray/handler.go#L99).

Éste middleware hace el mismo trabajo que `xray.Handler`, abrirá y cerrará un segmento para cada request recibido. También se encargará de manejar el header para IDs de rastreo (`"x-amzn-trace-id"`), que es un header que contiene un identificador que será generado para cada petición nueva y que será propagado a travéz de todos nuestros microservicios.

Así que vamos a agregar el middleware a las rutas que queremos intrumentar:

```go
// as part of my gin routes
v1.GET("/auth/roles", xraymid.Middleware(xray.NewFixedSegmentNamer("GetRoles")), controller.GetRoles)
```

Aquí estamos agregando el middleware de X-Ray (con el alias `xraymid`) a una ruta del grupo `v1`. El valor que estamos pasando como argumento a `NewFixedSegmentNamer` debe de ser un nombre descriptivo para tu ruta. Éste será el nombre para el grupo principal de rastreo para éste endpoint.

¡Bien! ¡Ahora veamos si funciona! Inicia tu servicio y verifica que el daemon esté corriendo.

Después de hacer un request, podemos ver en los logs del daemon algo como:
```
2023-03-21T13:10:47-06:00 [Debug] Received request on HTTP Proxy server : /GetSamplingRules
2023-03-21T13:10:48-06:00 [Debug] processor: sending partial batch
2023-03-21T13:10:48-06:00 [Debug] processor: segment batch size: 1. capacity: 50
2023-03-21T13:10:48-06:00 [Info] Successfully sent batch of 1 segments (0.109 seconds)
2023-03-21T13:10:49-06:00 [Debug] Send 1 telemetry record(s)
```

¡Parece que está funcionando! Vamos a ver qué dice la consola de AWS.

En tu consola web de AWS, ve a CloudWatch y en el panel lateral busca la opción para X-Ray, y da click en la opción "traces".

Si todo salió bien, deberías estar viendo el número de rastreos recibidos recientemente, y una tabla con la información de esos rastreos.

![Cloudwatch -> X-Ray -> Traces](https://github.com/makkoman/blogposts/blob/main/x-ray/images/cloudwatch-xray-traces.png)

En la tabla de registros, da click en alguno. Aparecerá la vista de rastreo/seguimiento, donde puedes ver la información registrada.

![Información de rastreo](https://github.com/makkoman/blogposts/blob/main/x-ray/images/simple-trace-info.png)

Aquí podemos ver los datos de seguimiento. Hasta el momento sólo estamos creando un segmento y cerrándolo para cada llamada, por lo que no tenemos mucha otra información, pero podemos ver el código de estado de respuesta, el tiempo que tomó para que se atendiera la solicitud y, por supuesto, el mapa de seguimiento, que por ahora incluye sólo el cliente y el servicio.

#### Creando sub segmentos

Ahora que tenemos nuestra configuración básica de instrumentación, ¿qué más podemos rastrear?

Hasta el momento, solo estamos rastreando una solicitud y algunos de sus metadatos. Pero, ¿qué pasa si queremos ser más detallados?

Digamos que tenemos un proceso intensivo que se ejecuta como parte de la solicitud; podemos agregar un subsegmento para monitorearlo.

En algún lugar de mi servicio, se ejecuta el siguiente código cuando llamo al endpoint `auth/roles`:

```go
// dentro de alguna función
roles := make([]Role, len(rolesList))
for i, roleItem := range rolesList {
	role, err := u.buildRole(roleItem)
	if err != nil {
		return model.RoleList{}, err
	}
	roles[i] = role
}
```

Aquí podemos envolver el bucle `for` en un subsegmento para ver cuánto tiempo del request tarda en ejecutar éste proceso.

Para crear el subsegmento, envolvemos el ciclo:

```go
err = xray.Capture(ctx, "BuildRolesDetail", func(ctx1 context.Context) error {
	for i, roleItem := range rolesList {
		role, err := u.buildRole(roleItem)
		if err != nil {
			return err
		}
		roles[i] = role
	}
	if err = xray.AddMetadata(ctx1, "No. roles built", len(roles)); err != nil {
		return nepErrors.InternalServerError.WithDetail(err.Error())
	}
	return nil
})
```

Vamos a correr nuestro servicio y llamemos de nuevo nuestro endpoint instrumentado.

Éste es el nuevo registro en AWS CloudWatch -> Traces:


![Rastreo con subsegmentos](https://github.com/makkoman/blogposts/blob/main/x-ray/images/trace-with-sub-segment.png)

Ahora podemos ver que la petición tomó **215ms**, y de esos, el ciclo `BuildRolesDetail` tomó **205ms**.

¿Ya estás pensando en las posibilidades? ¡Deberías! puedes usar `xray.AddMetadata` para agregar cualquier dato que te sea de utilidad. Únicamente toma en cuenta que el Daemon de X-Ray sólo envía a AWS [hasta 64KB de metadata por segmento](https://docs.aws.amazon.com/xray/latest/devguide/xray-api-segmentdocuments.html).

### Instrumentando clientes de AWS con X-Ray

Instrumentar clientes de AWS usando el SDK-V1 es bastante sencillo, puedes seguir la [guía oficial](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-go-awssdkclients.html) para hacerlo.

No hay mucha documentación sobre cómo instrumentar clientes de AWS usando el AWS SDK-v2, pero la configuración es bastante sencilla.

En algún lugar en el código de tu servicio, estás inicializando tu(s) cliente(s) de AWS. Para instrumentarlos, necesitas proveer a tus clientes con un cliente HTTP de X-Ray y pasar el contexto del request para cada llamada.

```go
cfg, err := config.LoadDefaultConfig(ctx)
if err != nil {
	return nil, err
}

// Create an HTTP client
httpClient := &http.Client{}

// Set the HTTP client as the AWS configuration's HTTP client
cfg.HTTPClient = httpClient

// Create an X-Ray client
xrayClient := xray.Client(httpClient)

dynamoClient := dynamodb.NewFromConfig(cfg, func(options *dynamodb.Options) {
	// Wrap the http.Client with an xray.Client
	options.HTTPClient = xrayClient
})
```

Aquí, estoy agregando el cliente HTTP de X-Ray al cliente de AWS DynamoDB.

Una vez hecho esto, llamemos de nuevo a nuestro endpoint instrumentado.

![Instrumentando el cliente de DynamoDB](https://github.com/makkoman/blogposts/blob/main/x-ray/images/instrumenting-ddb-client.png)

Yo estoy corriendo DynamoDB localmente, pero ya puedes ver qué tanto tiempo toma cada llamada a DynamoDB. También podemos ver que el mapa de rastreo ha sido acualizado para mostrar mi instancia local de DynamoDB.


## Conclusión
Instrumentar un servicio con X-Ray es relativamente sencillo, pero puede complicarse muy rápido dependiendo de las cosas que queremos monitorear. Debido a esto, el esfuerzo para agregar trazabilidad a su servicio puede variar de caso en caso.

Otra cosa a considerar es el límite de 64KB por segmento. Puede que no sea suficiente si deseas rastrear muchos subsegmentos o agregar más metadatos. Existen formas de evitar esto, pero están fuera del alcance de esta publicación.

En conclusión, implementar X-Ray en un microservicio en Go es un proceso sencillo que puede beneficiar enormemente la observabilidad y las capacidades de resolución de problemas de tu aplicación. El proceso de integración es relativamente fácil, y el SDK de X-Ray proporciona una serie de características útiles que facilitan la trazabilidad de las solicitudes y la identificación de cuellos de botella. Sin embargo, es importante tener en cuenta que X-Ray tiene algunas desventajas, como el costo asociado con su uso y las limitaciones de sus capacidades de muestreo.

No obstante, con una consideración cuidadosa y una implementación adecuada, X-Ray puede ser una herramienta invaluable para la depuración y optimización de tu arquitectura de microservicios. Así que no dudes en probarlo y ver cómo puede mejorar el rendimiento y la confiabilidad de tus microservicios en Go.
