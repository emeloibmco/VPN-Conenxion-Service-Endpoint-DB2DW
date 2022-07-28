# VPN-Conexion-Service-Endpoint-DB2DW :computer:

<br />

<br />
<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/Arquitectura.png"></p>
<br />

## Tabla de contenido 📑

1. [Requisitos](#Requisitos-newspaper)

1. [Creación de DB2 Warehouse](#creación-de-db2-warehouse-filecabinet)
2. [Habilitación de VRF](#habilitación-de-vrf)
3. [Creación de la VPC y la subnet](#creación-de-la-vpc-y-la-subnet)
4. [Configuración de la VPN](#configuración-de-la-vpn-gear) 
5. [Configurar claves SSH y creación de la VSI](#configurar-claves-ssh-closedlockwithkey)
6. [Instalación del proxy Nginx](#instalar-nginx-bulb)
7. [Flow Log](#flow-log)
8. [Referencias](#Referencias-mag)
9. [Autores](#Autores-black_nib)
<br />

## Requisitos :newspaper:

- Tener una cuenta de [IBM Cloud](https://cloud.ibm.com/)

- :cloud: [IBM Cloud CLI](https://cloud.ibm.com/docs/cli?topic=cloud-cli-getting-started&locale=en)

- :satellite: [OpenVPN](https://openvpn.net/)

- [Git](https://git-scm.com/downloads)

## Creación de DB2 Warehouse :file_cabinet:
Ingrese a su cuenta de [IBM Cloud](https://cloud.ibm.com/), dé clic en la opción ```Catalog``` de la barra superior, posteriormente en la sección izquierda seleccione ```Databases``` y elija la opción ```Db2```.

Para configurar su instacia de DB2 seleccione la ubicación de su preferencia, el plan ```lite```, asigne un nombre al servicio y un grupo de recursos.

Finalmente, dé clic en la sección izquierda para aceptar los license agreements y cree el servicio con la opción ```Create```
<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/conf-db2.png"></p>

Ya que su servicio esté aprovisionado, acceda a él y seleccione ```Service Credentials``` en la sección de la izquierda, posteriormente dé click en ```New Credential```:

<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/crear-credencial-1.png"></p>

Asigne un nombre a su credencial y seleccione el rol ```Manager```:

<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/crear-credencial-2.png"></p>

Al dar click en las credenciales creadas podrá acceder a una lista donde puede consultar elementos como la apikey y el host.


## Habilitación de VRF
El Virtual Routing and Forwarding permite habilitar endpoints privados al crear recursos, lo que permite tener conexiones más seguras, ya que se realizan las conexiones a través de la red privada de IBM Cloud. Para habilitar el VRF en su cuenta de IBM Cloud hay dos opciones:

**Opción 1: Habilitar VRF a través de la línea de comandos**

Ingrese a IBM  Cloud Shell a través del banner superior de IBM Cloud. Verifique si los service endpoints están habilitados con el comando

```
ibmcloud account show
```

<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/service-endpoint.png"></p>

Si aparece Service Endpoint Enabled: false, como se puede ver en la imagen, habilite los service endpoint con el siguiente comando:

```
ibmcloud account update --service-endpoint-enable true
```

A continuación deberá generar un ticket para habilitar VRF, esto se hace ingresando la letra ```y```.

<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/VRF.png"></p>

Luego de que el VRF sea habilitado, ingrese nuevamente el comando para habilitar la conectividad por service endpoint.

**Opción 2: Habilitar VRF a través de la interfaz de usuario**

En el banner superior, dé click en ```Manage``` y seleccione la opción ```Account```. En la sección de la izquierda seleccione ```Account Settings```. Baje hasta la sección ```Virtual routing and Forwarding``` y dé click en ```Create case```

<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/UI.png"></p>

No modifique la descripción del caso, ingrese su número de cuenta y dé click en ```Submit```.

La habilitación del VRF debería tardar entre 15 y 30 minutos. Luego de que esté habilitado puede ingresar nuevamente a  ```Account Settings``` y en la sección ```Service Endpoints``` dé click en ```On```


## Creación de la VPC y la subnet

**Creación de la VPC**
<br/>
Para crear una VPC en su cuenta de IBM Cloud siga los pasos que se indican a continuación:

1. Dé click en el Menú de Navegación y seleccione la pestaña ```VPC Infrastructure```.

2. En la sección de ```Network``` seleccione la opción ```VPCs``` y posteriormente dé click en el botón ```Create```. 

<p align="center"><img width="600" src="https://github.com/emeloibmco/VPN-Conexion-Service-Endpoint-DB2DW/blob/main/images/Arquitectura.png"></p>

Una vez le aparezca la ventana para la configuración y creación de la *VPC*, complete lo siguiente:

* ```Name```: asigne un nombre exclusivo para la *VPC*.
* ```Resource Group```: seleccione el grupo de recursos en el cual va a trabajar.
* ```Location```: seleccione la ubicación en la cual desea implementar la *VPC*.
* ```Default security group```: deje seleccionadas las opciones *Permitir SSH* y *Permitir ping*.
* ```Classic access```: deje el campo SIN seleccionar.
* ```Default address prefixes```: deje el campo SIN seleccionar, ya que posteriormente se creará la subred en la que se va a trabajar.

Cuando ya tenga todos los campos configurados dé click en el botón ```Create virtual private cloud```.

3. Espere unos minutos mientras la *VPC* aparece en estado disponible y asegúrese de tener seleccionada la región en la cual la implementó.
4. Una vez haya sido aprovisonada la VPC, dé click en el nombre e ingrese a la pestaña ```Address prefixes```. En dicha pestaña, de click en ```Create``` e ingrese la dirección IP que desee junto con la máscara.

> NOTA: Puede utilizar la IP y máscara sugeridas en las subnets creadas por defecto cuando se estaba aprovisionando la VPC.

<br />

**Creación de la subnet**
<br />
El siguiente paso consiste en crear una Subnet en la *VPC*. Para ello, en la sección ```Network``` seleccione la opción ```Subnets``` y dé click en el botón ```Create```. Una vez le aparezca la ventana para la configuración y creación de la subnet, complete lo siguiente:

* ```Name```: asigne un nombre exclusivo para la subnet.
* ```Resource group```: seleccione el grupo de recursos en el cual va a trabajar (el mismo seleccionado en la creación de la *VPC*).
* ```Location```: seleccione la ubicación en la cual desea implementar la subnet (la misma seleccionada en la creación de la *VPC*).
* ```Virtual private cloud```: seleccione la *VPC* que creó anteriormente.
* Los demás parámetros no los modifique, deje los valores establecidos por defecto.

Cuando ya tenga todos los campos configurados dé click en el botón ```Create subnet```.

6. Espere unos minutos mientras la subnet aparece en estado disponible y asegúrese de tener seleccionada la región en la cual la implementó.

<br />


## Configuración de la VPN :gear:

**Crear una autorización IAM sevice-to-service**
<br/>
Para crear una autorización IAM sevice-to-service para su servidor VPN y certificate manager siga los siguientes pasos:
1. Desde la consola de IBM Cloud, vaya a la página [Manage Autorizations](https://cloud.ibm.com/iam/authorizations) y dé clic en el botón ```Create```
2. En el menú desplegable seleccione ```VPC Infrastructure Services``` y luego seleccione ```Resource based on selected attributes```
3. Seleccione ```Resource type``` > ```Client VPN for VPC```
4. En la opción Target Service seleccione ```Certificate Manager```
5. Seleccione la opción ```All resources``` y verifique la casilla ```Writer```
6. Dé clic en ```Authorize```

**Gestión de certificados de cliente y servidor VPN**
<br/>
Para la gestión de certificados hay dos opciones, usar OpenVPN para generar los certificados u ordenar un certificado usando Certificate Manager.

**Opción 1. Generación de certificados usando OpenVPN**
<br/>
A continuación se usará [OpenVPN easy-rsa](https://github.com/OpenVPN/easy-rsa) para generar los certificados y posteriormente importarlos al certificate manager.
1. Clone el repositorio Easy-RSA 3 en su carpeta local:

```
git clone https://github.com/OpenVPN/easy-rsa.git
cd easy-rsa/easyrsa3
```

2. Cree un nuevo PKI y CA:

```
./easyrsa init-pki
./easyrsa build-ca nopass
```

Verifique que el certificado CA esté generado en la ruta ```./pki/ca.crt```

3. Genere un certificado de servidor VPN:

```
./easyrsa build-server-full vpn-server.vpn.ibm.com nopass
```

Verifique que la llave pública haya sido generada en la ruta ```./pki/issued/vpn-server.vpn.ibm.com.crt``` y la llave privada en la ruta ```./pki/private/vpn-server.vpn.ibm.com.key```

4. Genere un certificado de cliente VPN:
```
./easyrsa build-client-full client1.vpn.ibm.com nopass
```
Verifique que la llave pública haya sido generada en la ruta ```./pki/issued/client1.vpn.ibm.com.crt``` y la llave privada en la ruta ```./pki/private/client1.vpn.ibm.com.key```
<br/>

Para importar el certificado del servidor al certificate manager siga estos pasos:
1. En el navegador Google Chrome diríjase a la página de [Certificate Manager](https://cloud.ibm.com/catalog/services/certificate-manager), complete la información y dé clic en ```Create``` para crear una instancia.
2. Diríjase a la página ```Your Certificates``` e importe el certificado del servidor según los siguientes pasos:

   * Elija un nombre para su certificado, este no puede contener guiones, números ni mayúsculas (ej. vpcdemo)
   * Dé clic al botón ```Browse``` y seleccione el archivo de certificado ```./pki/issued/vpn-server.vpn.ibm.com.crt```
   * Dé clic al botón ```Browse``` y seleccione el archivo de llave privada ```./pki/private/vpn-server.vpn.ibm.com.key```
   * Dé clic al botón ```Browse``` y seleccione el archivo de certificado intermediario ```./pki/ca.crt```
   * Dé clic al botón ```Import```
   <br/>

<br/>

**Opción 2. Ordenar un certificado usando Certificate Manager**
<br/>

Usted puede usar IBM Cloud Certificate Manager para ordenar un certificado público SSL/TLS como certificado de servidor VPN. Certificate Manager solo almacena certificados intermedios, por lo cual usted necesitará los root certificates de Let's Encrypt, guardados como archivos ```.pem```. Los dos archivos requeridos puede encontrarlos en [https://letsencrypt.org/certs/lets-encrypt-r3.pem](https://letsencrypt.org/certs/lets-encrypt-r3.pem) y [https://letsencrypt.org/certs/isrgrootx1.pem](https://letsencrypt.org/certs/isrgrootx1.pem). Cuando descargue y actualice el certificado de cliente VPN, use este root certificate para reemplazar la sección ```<ca>``` en el perfil de cliente.
<br/>

Los certificados ordenados son certificados públicos SSL/TLS y deben ser usados como certificados de servidor VPN únicamente. No deben ser usados para autenticar los clientes VPN.
<br/>

*Ubicar el certificado CRN*
<br/>

Al configurar la autenticación de un servidor VPN client-to-site usando la UI, usted puede especificar el Certificate Manager y el certificado SSL, o el CRN del certificado. Esto se puede hacer si usted no tiene acceso a la instancia de Certificate Manager. Tenga en cuenta que usted debe ingresar el CRN si está usando la API para crear el servidor VPN client-to-site.
<br/>
Para encontrar el CRN del certificado, siga estos pasos:

1. En la [consola de IBM Cloud](https://cloud.ibm.com/vpc-ext) Vaya al ícono de menú y seleccione ```Resource List```
2. Dé clic para expandir ```Services and software``` y posteriormente seleccione el Certificate Manager del que desea obtener el CRN.
3. Seleccione cualquier parte en esa fila de la tabla para abrir el panel lateral de detalles. El CRN del certificado se encuentra listado allí.

## Configurar claves SSH :closed_lock_with_key:
<br />
Para poder desplegar una *VSI* en *VPC* es necesario realizar la respectiva configuración para las claves *SSH*. Con base en esto, realice lo siguiente:

1. Para generar una clave *SSH* acceda al *IBM Cloud Shell* y coloque el comando:
```
ssh-keygen -t rsa -C "user_id"
```

2. Al colocar el comando anterior, en la consola se pide que especifique la ubicación, en este caso oprima la tecla Enter para que se guarde en la ubicación sugerida. Posteriormente, cuando se pida la ```Passphrase``` coloque una constraseña que pueda recordar o guárdela, ya que se utilizará más adelante.

3. Muévase con el comando ```cd .ssh``` a la carpeta donde están los archivos ```id_rsa.pub``` y ```id_rsa```. Estos archivos contienen las claves públicas y privadas respectivamente. 

4. Visualice la clave pública, ya que la necesitará para la creación de la *VSI*. Utilice el comando:
```
cat id_rsa.pub
```
> NOTA: Por defecto la clave empieza con ssh_rsa y termina con el user_ID. Copie la clave para emplearla más adelante.

<br />

**Desplegar VSI en VPC**
<br/>
Una vez ha configurado las claves *SSH* proceda con la creación de la *VSI* Linux en *VPC*. Complete los siguientes pasos:

1. Entre al menú desplegable y seleccione ```VPC Infrastructure```. En la sección de ```Compute``` seleccione la opción ```Virtual Server instances``` y posteriormente dé click en el botón ```Create```. Una vez le aparezca la ventana para la configuración y creación de la *VSI*, complete lo siguiente:

* ```Name```: asigne un nombre exclusivo para la *VSI*.
* ```Resource group```: seleccione el grupo de recursos en el cual va a trabajar (el mismo seleccionado en la creación de la *VPC*).
* ```Location```: seleccione la ubicación en la cual desea implementar la subnet (la misma seleccionada en la creación de la *VPC*).
* ```Hosting type```: seleccione la opción **Public**.
* ```Operating system```: seleccione la opción **Ubuntu Linux**.
* ```Profile```: deje seleccionado el perfil que viene por defecto (**Balanced | bx2-2x8**).
* ```SSH keys```: dé click en el botón ```Create key +```, asigne un nombre exclusivo para su clave *SSH*, seleccione el grupo de recursos y la ubicación y finalmente en **Public key** coloque la clave copiada en el ítem 3 del paso [Configurar claves SSH](#Configurar-claves-SSH-closed_lock_with_key). Posteriormente, dé click en el botón ```Create```.
* ```Virtual private cloud```: seleccione la *VPC* creada anteriormente.
* Los demás parámetros no los modifique, deje los valores establecidos por defecto.

Cuando ya tenga todos los campos configurados dé click en el botón ```Create virtual server instance```.

2. Espere unos minutos mientras la *VSI* aparece en estado disponible y asegúrese de tener seleccionada la región en la cual la implementó.

<br />


## Instalar Nginx :bulb:
Después de ingresar a su VSI por medio de la llave SSH, puede proceder a instalar nginx en su virtual server. Para esto use los siguientes comandos:
```
yum upgrade
```

```
yum install epel-release
```

```
yum install nginx
```

Ingrese a la carpeta de nginx que está ubicada en el path ```/etc/nginx/``` y luego de esto acceda al archivo de configuración con el siguiente comando:
```
cat nginx.conf
```

Finalmente, habilite el endpoint privado

## Flow Log
## Referencias :mag:

- [Guía VPN-for-VPC-Client-to-Site](https://github.com/emeloibmco/VPN-for-VPC-Client-to-Site)

- [IBM Cloud Docs - Enabling VRF and service endpoints](https://cloud.ibm.com/docs/account?topic=account-vrf-service-endpoint&interface=ui)

<br />

## Autores :black_nib:
Equipo *IBM Cloud Tech Sales Colombia*.
