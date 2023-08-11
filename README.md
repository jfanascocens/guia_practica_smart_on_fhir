
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

Instalar servidor de autenticación/autorización MitreID Connect

[Instrucciones aquí](https://github.com/mitreid-connect/OpenID-Connect-Java-Spring-Server/wiki/Build-instructions)


## Configurar Cliente en servidor MitreID Connect

El grant type indicado por Smart on Fhir es "client Credentials"
Opcionalmente, permitir la introspección (checkbox en access)
En Credentials 
->Asymmetrically-signed JWT assertion
->algoritmo de firma puede ser RSA384, o ES384, o Allow Any para permitir cualquier algoritmo
->El public Key Set se genera a partir del JWT. Tanto JWT como Public Key Set se crean en el codigo Python Cliente: [Script en Python 3](cliente_oauth2.ipynb), y desde ahi se puede copiar y pegar como valores, siguiendo el formato del diccionario con clave "Keys":{[ aquí se pega el Public Key Set generado con el código ]} 

Registro de cliente
![Pestaña Credentials](/images/client_credentials.png)


### Al cliente se le asignan los permisos usando los Scopes.

Esto es una notación simplemente, la cual se debe registrar en el servidor de autorización y que también maneja el servidor de recursos.
No obstante que es una notación simplemente, se debe seguir las siguientes indicaciones de la [guia Smart on FHIR](https://www.hl7.org/fhir/smart-app-launch/scopes-and-launch-context.html)

### Crear claves publica y privada del cliente.
Se guardan en archivos locales private1.pem y public1.pem

     from Crypto.PublicKey import RSA
    key = RSA.generate(2048)
    pv_key_string = key.exportKey()
    publicPem=str()
    
    with open ("private1.pem", "w") as prv_file:
        print("{}".format(pv_key_string.decode()), file=prv_file)
    
    pb_key_string = key.publickey().exportKey()
    with open ("public1.pem", "w") as pub_file:
        print("{}".format(pb_key_string.decode()), file=pub_file)
		
	#Mostrar los archivos de claves y guardarlos en variables útiles en todo el notebook.
    publicPem=str()
    privatePem=str()
    with open ("public1.pem", "r") as pub_file:
        #print(pub_file.read())
        publicPem=pub_file.read()
    with open ("private1.pem", "r") as pub_file:
        #print(pub_file.read())
        privatePem=pub_file.read()
    print(publicPem)



###Crear JWT

    # Crear JWK a partir de una clave pública PEM.
    #### el JWK se entrega al servidor de autenticación/autorización al momento de creación del registro del cliente.
    import base64
    import hashlib
    import json
    from cryptography.hazmat.primitives import serialization
    from math import ceil
    from datetime import datetime, timedelta
    
    def pem_to_jwk(pem_public_key):
        # Cargar la clave pública desde el formato PEM
        public_key = serialization.load_pem_public_key(pem_public_key.encode(), backend=None)
    
        # Obtener los componentes de la clave pública
        e = public_key.public_numbers().e
        n = public_key.public_numbers().n
    
        # Construir el objeto JWK
        jwk = {
            "kty": "RSA",
            "e": base64.urlsafe_b64encode(e.to_bytes(ceil(e.bit_length() / 8))).decode().rstrip('='),
            "use": "sig",  # Use "sig" para firmar o "enc" para cifrar
            "kid": generate_kid(pem_public_key),
            "alg": "RS384",  # Algoritmo de firma (puedes cambiarlo si es necesario)
            "n": base64.urlsafe_b64encode(n.to_bytes(ceil(n.bit_length() / 8))).decode().rstrip('=')
        }
    
        return jwk
    
    def generate_kid(pem_public_key):
        # Calcular el hash SHA-256 de la clave pública para obtener el kid
        public_key = serialization.load_pem_public_key(pem_public_key.encode(), backend=None)
        sha256_hash = hashlib.sha256(public_key.public_bytes(
            encoding=serialization.Encoding.PEM,
            format=serialization.PublicFormat.SubjectPublicKeyInfo
        )).digest()
    
        # Tomar los primeros 16 caracteres del hash como kid
        kid = base64.urlsafe_b64encode(sha256_hash)[:16].decode()
    
        return kid
    
    # Ejemplo de uso:
    pem_public_key = publicPem
    
    jwk = pem_to_jwk(pem_public_key)
    print(json.dumps(jwk, indent=2))
    




Registrar el cliente en el servidor

[imagen](imagenMitreID_config_cliente_Credentials)



Crear el client assertion


    #Crear client assertion firmando el JWT con clave privada
    
    import jwt
    import time
    
    import timedelta
    from datetime import datetime, timedelta
    
    def create_client_assertion(client_id, private_key_path, token_endpoint):
        # Leer el contenido de la clave privada
        #with open(private_key_path, 'r') as f:
            
        # Crear el payload para el JWT
        iat = int(time.time())
        exp = iat + 3600  # El token expirará en 1 hora (3600 segundos)
        payload = {
            "iss": client_id,
            "sub": client_id,
            "aud": token_endpoint,
            "iat": iat,
            "exp": datetime.utcnow()+timedelta(minutes=4)
        }
    
        # Crear el client_assertion JWT utilizando la clave privada
        client_assertion = jwt.encode(payload, private_key, algorithm='RS384')
    
        return client_assertion
    
    # Ejemplo de uso:
    #El Client ID lo entrega el servidor de autenticación, al momento de crear el registro del cliente en el sistema
    client_id = "8bc6bc6d-2da5-424b-be6a-d0491dd1c6ce"
    
    private_key= privatePem
    token_endpoint = "http://localhost:8080/openid-connect-server-webapp/token" # este es el endpoint del servidor de autorización
    
    client_assertion = create_client_assertion(client_id, private_key, token_endpoint)
    print("Client Assertion JWT:", client_assertion)
    #print(client_assertion.decode('utf-8'))
    
    


Ejecutar el request de autorización.

    # Preparar Request con los headers y el body que incluye el client_assertion
    
    
    import requests
    
    def make_token_request():
        url = 'http://localhost:8080/openid-connect-server-webapp/token'
        headers = {
            'accept': 'application/json',
            'Content-Type': 'application/x-www-form-urlencoded'
        }
        data = {
            'grant_type': 'client_credentials',
            'scope': 'profile',
            'client_assertion_type': 'urn:ietf:params:oauth:client-assertion-type:jwt-bearer',
            'client_assertion': client_assertion
        }
    
        response = requests.post(url, headers=headers, data=data)
    
        if response.status_code == 200:
            token_data = response.json()
            access_token = token_data.get('access_token')
            
            print("Access Token:", access_token)
            return access_token
        else:
            print(f"Error en la solicitud: {response.status_code} - {response.text}")
        
    
    
    # Ejecutar Request.
    access_token=make_token_request()
    


###(Opcional) Servidor de recursos ejecuta introspección contra servidor de autenticación/autorización

    body_intro = {
        'token': access_token,
        'client_assertion': encoded_jwt.decode('utf-8'),
        'client_assertion_type': 'urn:ietf:params:oauth:client-assertion-type:jwt-bearer',
    
    
    }
    print(body_intro)
    
    intr = requests.post('http://localhost:8080/openid-connect-server-webapp/introspect',
                         headers=headers,
                         data=urlencode(body_intro))
    print(intr.content)


## License

[Apache 2.0](https://choosealicense.com/licenses/apache-2.0/)
