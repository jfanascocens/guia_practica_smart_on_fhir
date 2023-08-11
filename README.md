
# Guía de práctica de SMART on FHIR para servicios Backend

La guía SMART on FHIR APP Launch, orientado a servicios Backend define cómo establecer un proceso de autenticación y autorización entre aplicaciones clientes(en adelante clientes), sin requerir la autenticación de usuarios.
Entiendase como autenticación el proceso en que se verifica la identidad, en este caso de un cliente.
Entiendase como autorización el proceso en que, una vez autenticado un cliente, se le dá permiso para acceder a ciertos recursos protegidos(servidor FHIR protegido), por un tiempo acotado.

## Uso
La habilitación de servicios Backend se usa cuando la autorización a los recursos FHIR será igual para todos los usuarios de un cliente. Si se requiere autorización específica para los usuarios, se usa Authorization via SMART App Launch (de la misma guía Smart on FHIR)

## Flujo
Smart on FHIR define el uso de un flujo "Client Credentials" 

Client Credentials implica:

Comunicación usando autenticación Asimetrica, por lo tanto, el cliente emite una clave pública y privada.

Registro manual del cliente en el servidor de autenticación, indicando los permisos a los recursos, clave pública generada por el cliente, entre otros.

Estructrar los datos para la autenticación en formato JWT por parte del cliente

El cliente debe firmar el JWT usando su clave privada. Esto produce un "client assertion"

El cliente envía el client assertion al servidor de autenticación.

Si el client assertion es válido, el servidor entrega un token de acceso.

El cliente luego se comunica con el servidor de Recursos (FHIR), y solicita ejecuta el request enviando en cada request el mismo token de acceso.

Opcionalmente, el servidor de recursos puede ejecutar una Introspección, que significa validar contra el servidor de Autorización el token de acceso que recibió desde el cliente.


## Alcances

Se ejecutó el flujo hasta la autorización.
Además se ejecuta la introspección.
Quedó fuera el request hacia los recursos FHIR.

## Herramientas

Servidor de autenticación y autorización: [MITREID Connect](https://github.com/mitreid-connect/)
Cliente: [Script en Python 3](cliente_oauth2.ipynb)

# STEP BY STEP:

## 1) Instalar servidor de autenticación/autorización MitreID Connect

[Instrucciones aquí](https://github.com/mitreid-connect/OpenID-Connect-Java-Spring-Server/wiki/Build-instructions)


## 2) Configurar Cliente en servidor MitreID Connect

El grant type indicado por Smart on Fhir es "client Credentials"
Opcionalmente, permitir la introspección (checkbox en access)
En Credentials 

->Asymmetrically-signed JWT assertion

->algoritmo de firma puede ser RSA384, o ES384, o Allow Any para permitir cualquier algoritmo

->El public Key Set se genera a partir del JWT. Tanto JWT como Public Key Set se crean en el codigo Python Cliente: [Script en Python 3](cliente_oauth2.ipynb), y desde ahi se puede copiar y pegar como valores, siguiendo el formato del diccionario con clave "Keys":{[ aquí se pega el Public Key Set generado con el código ]} 

[Client Main](/images/client_main.png)

[Client Access](/images/cliente_access.png)

[Client Credentials](/images/client_credentials.png)

[Client Tokens](/images/client_tokens.png)

[Client Crypto](/images/client_crypto.png)

[Client Others](/images/client_others.png)

### 3) Al cliente se le asignan los permisos usando los Scopes.

Esto es una notación simplemente, la cual se debe registrar en el servidor de autorización y que también maneja el servidor de recursos.
No obstante que es una notación simplemente, se debe seguir las siguientes indicaciones de la [guia Smart on FHIR](https://www.hl7.org/fhir/smart-app-launch/scopes-and-launch-context.html)

## Los pasos a continuación se ejecutan a través de codigo en Python3(idealmente desde Jupyter Notebook o similar
Ver archivo [cliente_oauth2.ipynb] (cliente_oauth2.ipynb)

### 4) Crear claves publica y privada del cliente.
### 5) Crear JWT

Crear JWK a partir de una clave pública PEM.

El JWK se entrega al servidor de autenticación/autorización al momento de creación del registro del cliente.

        jwk = {
            "kty": "RSA",
            "e": base64.urlsafe_b64encode(e.to_bytes(ceil(e.bit_length() / 8))).decode().rstrip('='),
            "use": "sig",  # Use "sig" para firmar o "enc" para cifrar
            "kid": generate_kid(pem_public_key),
            "alg": "RS384",  # Algoritmo de firma (puedes cambiarlo si es necesario)
            "n": base64.urlsafe_b64encode(n.to_bytes(ceil(n.bit_length() / 8))).decode().rstrip('=')
        }


### 6) Registrar el cliente en el servidor
En la sección "Credentials" de la configuración del cliente en el servidor de autenticación, hay un campo de Public Key Set, que si se selecciona la opción by Value permite ingresar el jwk creado en el paso anterior.

![Client Credentials](/images/client_credentials.png)


### 7)Crear el client assertion

Client Assertion es el JWK firmado con la clave privada.


### 8) Ejecutar el request de autorización.

Preparar y ejecutar Request con los headers y el body donde se incluye el client_assertion
    
### (Opcional) Servidor de recursos ejecuta introspección contra servidor de autenticación/autorización

La introspección es un proceso en el cual el servidor de recursos (FHIR), quien recibió un token de autorización de parte de un cliente, se lo pasa al servidor de autenticación/autorización para determinar su validez y verificar sus datos.

## License

[Apache 2.0](https://choosealicense.com/licenses/apache-2.0/)
