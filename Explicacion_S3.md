# Crear un sitio estático en S3

Inicialmente tenemos una arquitectura

![Arquitectura](/images/arquitectura.jpeg)

Con esta arquitectura, los clientes tendrán acceso al website que se deployee en el servicio de Amazon S3, en este caso se utilizará la AWS CLI


El primer paso es conectarse a la CLI de Amazon y configurar por lo que se descarga una KEY de SSH con extensión .pem

Ingresamos al 
```bash
cd ~/Descargas
```

Cambiamos los permisos de la clave a solo lectura  
```bash
chmod 400 cafe.pem
```

Ahora podemos ingresar o conectarnos a la terminal, para esto utilizamos SSH especificando el inicio con clave o key y especificamos usuario así como IP pública de la instancia [EC2](https://aws.amazon.com/es/ec2/)
```bash
ssh -i cafe.pem ec2-user@18.237.109.40
```

Con lo que tenemos: 

![SSH](/images/ssh.jpeg)
## Configurando el AWS CLI 
Una vez hecho, es momento de configurar la CLI de AWS:
```bash
aws configure
```

Para conocer el Access Key es necesario ir a [AWS IAM](https://docs.aws.amazon.com/es_es/IAM/latest/UserGuide/id_users.html), dentro de usuario, seleccionamos el usuario, y al momento de crear el usuario mostrará el AccessKey junto al SecretKey 

El resto es especificar la región, en este caso utilizaremos us-west-2 y formato de salida json, con lo que se puede ver algo similar a: 

![AWS Configure](/images/aws_configure.jpeg)

## Creando el bucket de S3 utilizando AWS CLI 

Para crear el bucket utilizaremos la [S3 API](https://docs.aws.amazon.com/cli/latest/reference/s3api/), el nombre de nuestro __bucket__ será __cafecafe255__

Para esto escribimos: 
```bash
aws s3api create-bucket --bucket cafecafe255 --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
```

Si todo sale bien, obtendremos un JSON como: 
```json
{
    "Location": "http://cafecafe255.s3.amazonaws.com/"
}
```
## Creando un nuevo usuario con AWS IAM en nuestra cuenta para acceder a S3

Para [crear el usuario](https://docs.aws.amazon.com/cli/latest/reference/iam/create-user.html)

```bash
aws iam create-user --user-name awsS3user
```

Al hacerlo, obtenemos un JSON con información del nombre y fecha de creación, además del ID del usuario y su ARN
```json
{
    "User": {
        "UserName": "awsS3user", 
        "Path": "/", 
        "CreateDate": "2023-11-29T01:52:46Z", 
        "UserId": "AIDA4R3YW4T5WFMZVUL4Q", 
        "Arn": "arn:aws:iam::863004714235:user/awsS3user"
    }
}
```

Ahora es necesario [crear un perfil de inicio de sesión](https://docs.aws.amazon.com/cli/latest/reference/iam/create-login-profile.html)

```bash
aws iam create-login-profile --user-name awsS3user --password Training123!
```

Al hacerlo obtenemos un JSON, que contiene el nombre de usuario, fecha de creación y el una variable booleana para restablecer la contraseña.

```json
{
    "LoginProfile": {
        "UserName": "awsS3user", 
        "CreateDate": "2023-11-29T01:56:58Z", 
        "PasswordResetRequired": false
    }
}
```

Ahora iniciamos sesión en la consola a través de IAM, para esto requerimos el ID, usuario y contraseña. 

![Inicio sesión IAM](/images/inicio_IAM.jpeg)

De regreso en la terminal, [buscamos la política que otorgue acceso completo](https://docs.aws.amazon.com/cli/latest/reference/iam/list-policies.html) al usuario __awsS3user__

```bash
aws iam list-policies --query "Policies[?contains(PolicyName,'S3')]"
```

Ahora otorgamos acceso completo con el nombre de la política S3FullAccess que en JSON sale: 
```json
{
        "PolicyName": "AmazonS3FullAccess", 
        "PermissionsBoundaryUsageCount": 0, 
        "CreateDate": "2015-02-06T18:40:58Z", 
        "AttachmentCount": 0, 
        "IsAttachable": true, 
        "PolicyId": "ANPAIFIR6V6BVTRAHWINE", 
        "DefaultVersionId": "v2", 
        "Path": "/", 
        "Arn": "arn:aws:iam::aws:policy/AmazonS3FullAccess", 
        "UpdateDate": "2021-09-27T20:16:37Z"
    }
```
Entonces utilizamos [attach-user-policy](https://docs.aws.amazon.com/cli/latest/reference/iam/attach-user-policy.html), que nos permitirá otorgar el acceso completo al usuario. 

```bash
aws iam attach-user-policy \ 
--policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
--user-name awsS3user
```

## Subiendo los archivos a S3 utilizando AWS CLI

Ingresamos al directorio en el que se encuentre nuestro contenido para el sitio, dependiendo de cual sea el directorio en el que se encuentre este paso puede ser diferente.

```bash
cd static-website
```
En esta carpeta se debe encontrar nuestro __index.html__, dicho archivo lo subiremos al bucket mediante: 

En este caso reemplazamos el nombre del bucket y además y colocamos el nombre de nuestro HTML, que es __index.html__
```bash
aws s3 website s3://cafecafe255/ --index-document index.html
```

Para subir los archivos y que sean visibles, necesitamos cambiar la configuración del bucket S3, por lo tanto en la consola: 

![Bloque acceso](/images/bloqueacceso.jpeg)

Quedaría tal que: 

![Configuración Bucket](/images/desactivado.jpeg)

Lo siguiente es habilitar las ACL colocamos y confirmamos que se active el ACL
![ACL habilitado](/images/image.png)


Para subir archivos al bucket es necesario copiar la ruta absoluta donde se encuentran los archivos: 
```bash
aws s3 cp /home/ec2-user/sysops-activity-files/static-website/ s3://cafecafe255/ --recursive --acl public-read
```

Con lo que en la terminal observamos: 
![Archivos](/images/archivos.jpeg)

Para verificar que se hayan subido correctamente escribimos: 

```bash
aws s3 ls cafecafe255
```

Hasta este punto casi hemos finalizado, ahora es necesario regresar a la __consola__ de AWS, y en el servicio de __S3__, seleccionamos el bucket y vamos a __Propiedades__, luego a la sección de __Static website hosting__. Si observamos se habilitaó la sección de __Bucket hosting__, en consecuencia se podría ejecutar

```bash
aws s3 website
```
Clickeamos en el link para observar nuestro sitio: 
![Sitio web estático](/images/sitios_estaticos.jpeg)

Y podemos ver nuestro sitio web estático.
![Cafe](/images/pagina.jpeg)

 El sitio contiene en este caso un link con el nombre del bucket, en este caso: 

 http://cafecafe255.s3-website-us-west-2.amazonaws.com
 
