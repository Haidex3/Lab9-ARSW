### Escuela Colombiana de Ingeniería
### Arquitecturas de Software - ARSW

## Escalamiento en Azure con Maquinas Virtuales, Sacale Sets y Service Plans

### Dependencias
* Cree una cuenta gratuita dentro de Azure. Para hacerlo puede guiarse de esta [documentación](https://azure.microsoft.com/es-es/free/students/). Al hacerlo usted contará con $100 USD para gastar durante 12 meses.

### Parte 0 - Entendiendo el escenario de calidad

Adjunto a este laboratorio usted podrá encontrar una aplicación totalmente desarrollada que tiene como objetivo calcular el enésimo valor de la secuencia de Fibonnaci.

**Escalabilidad**
Cuando un conjunto de usuarios consulta un enésimo número (superior a 1000000) de la secuencia de Fibonacci de forma concurrente y el sistema se encuentra bajo condiciones normales de operación, todas las peticiones deben ser respondidas y el consumo de CPU del sistema no puede superar el 70%.

### Parte 1 - Escalabilidad vertical

1. Diríjase a el [Portal de Azure](https://portal.azure.com/) y a continuación cree una maquina virtual con las características básicas descritas en la imágen 1 y que corresponden a las siguientes:
    * Resource Group = SCALABILITY_LAB
    * Virtual machine name = VERTICAL-SCALABILITY
    * Image = Ubuntu Server 
    * Size = Standard B1ls
    * Username = scalability_lab
    * SSH publi key = Su llave ssh publica

![Imágen 1](images/part1/part1-vm-basic-config.png)

2. Para conectarse a la VM use el siguiente comando, donde las `x` las debe remplazar por la IP de su propia VM (Revise la sección "Connect" de la virtual machine creada para tener una guía más detallada).

    `ssh scalability_lab@xxx.xxx.xxx.xxx`

3. Instale node, para ello siga la sección *Installing Node.js and npm using NVM* que encontrará en este [enlace](https://linuxize.com/post/how-to-install-node-js-on-ubuntu-18.04/).
4. Para instalar la aplicación adjunta al Laboratorio, suba la carpeta `FibonacciApp` a un repositorio al cual tenga acceso y ejecute estos comandos dentro de la VM:

    `git clone <your_repo>`

    `cd <your_repo>/FibonacciApp`

    `npm install`

5. Para ejecutar la aplicación puede usar el comando `npm FibinacciApp.js`, sin embargo una vez pierda la conexión ssh la aplicación dejará de funcionar. Para evitar ese compartamiento usaremos *forever*. Ejecute los siguientes comando dentro de la VM.

    ` node FibonacciApp.js`

6. Antes de verificar si el endpoint funciona, en Azure vaya a la sección de *Networking* y cree una *Inbound port rule* tal como se muestra en la imágen. Para verificar que la aplicación funciona, use un browser y user el endpoint `http://xxx.xxx.xxx.xxx:3000/fibonacci/6`. La respuesta debe ser `The answer is 8`.

![](images/part1/part1-vm-3000InboudRule.png)

7. La función que calcula en enésimo número de la secuencia de Fibonacci está muy mal construido y consume bastante CPU para obtener la respuesta. Usando la consola del Browser documente los tiempos de respuesta para dicho endpoint usando los siguintes valores:
    * 1000000
    * 1010000
    * 1020000
    * 1030000
    * 1040000
    * 1050000
    * 1060000
    * 1070000
    * 1080000
    * 1090000    

8. Dírijase ahora a Azure y verifique el consumo de CPU para la VM. (Los resultados pueden tardar 5 minutos en aparecer).

![Imágen 2](images/part1/part1-vm-cpu.png)

9. Ahora usaremos Postman para simular una carga concurrente a nuestro sistema. Siga estos pasos.
    * Instale newman con el comando `npm install newman -g`. Para conocer más de Newman consulte el siguiente [enlace](https://learning.getpostman.com/docs/postman/collection-runs/command-line-integration-with-newman/).
    * Diríjase hasta la ruta `FibonacciApp/postman` en una maquina diferente a la VM.
    * Para el archivo `[ARSW_LOAD-BALANCING_AZURE].postman_environment.json` cambie el valor del parámetro `VM1` para que coincida con la IP de su VM.
    * Ejecute el siguiente comando.

    ```
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
    newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
    ```

10. La cantidad de CPU consumida es bastante grande y un conjunto considerable de peticiones concurrentes pueden hacer fallar nuestro servicio. Para solucionarlo usaremos una estrategia de Escalamiento Vertical. En Azure diríjase a la sección *size* y a continuación seleccione el tamaño `B2ms`.

![Imágen 3](images/part1/part1-vm-resize.png)

11. Una vez el cambio se vea reflejado, repita el paso 7, 8 y 9.
12. Evalue el escenario de calidad asociado al requerimiento no funcional de escalabilidad y concluya si usando este modelo de escalabilidad logramos cumplirlo.
13. Vuelva a dejar la VM en el tamaño inicial para evitar cobros adicionales.

**Preguntas**

1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?
2. ¿Brevemente describa para qué sirve cada recurso?
3. ¿Al cerrar la conexión ssh con la VM, por qué se cae la aplicación que ejecutamos con el comando `npm FibonacciApp.js`? ¿Por qué debemos crear un *Inbound port rule* antes de acceder al servicio?
4. Adjunte tabla de tiempos e interprete por qué la función tarda tando tiempo.
5. Adjunte imágen del consumo de CPU de la VM e interprete por qué la función consume esa cantidad de CPU.
6. Adjunte la imagen del resumen de la ejecución de Postman. Interprete:
    * Tiempos de ejecución de cada petición.
    * Si hubo fallos documentelos y explique.
7. ¿Cuál es la diferencia entre los tamaños `B2ms` y `B1ls` (no solo busque especificaciones de infraestructura)?
8. ¿Aumentar el tamaño de la VM es una buena solución en este escenario?, ¿Qué pasa con la FibonacciApp cuando cambiamos el tamaño de la VM?
9. ¿Qué pasa con la infraestructura cuando cambia el tamaño de la VM? ¿Qué efectos negativos implica?
10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta? Si/No ¿Por qué?
11. Aumente la cantidad de ejecuciones paralelas del comando de postman a `4`. ¿El comportamiento del sistema es porcentualmente mejor?

### Parte 2 - Escalabilidad horizontal

#### Crear el Balanceador de Carga

Antes de continuar puede eliminar el grupo de recursos anterior para evitar gastos adicionales y realizar la actividad en un grupo de recursos totalmente limpio.

1. El Balanceador de Carga es un recurso fundamental para habilitar la escalabilidad horizontal de nuestro sistema, por eso en este paso cree un balanceador de carga dentro de Azure tal cual como se muestra en la imágen adjunta.

![](images/part2/part2-lb-create.png)

2. A continuación cree un *Backend Pool*, guiese con la siguiente imágen.

![](images/part2/part2-lb-bp-create.png)

3. A continuación cree un *Health Probe*, guiese con la siguiente imágen.

![](images/part2/part2-lb-hp-create.png)

4. A continuación cree un *Load Balancing Rule*, guiese con la siguiente imágen.

![](images/part2/part2-lb-lbr-create.png)

5. Cree una *Virtual Network* dentro del grupo de recursos, guiese con la siguiente imágen.

![](images/part2/part2-vn-create.png)

#### Crear las maquinas virtuales (Nodos)

Ahora vamos a crear 3 VMs (VM1, VM2 y VM3) con direcciones IP públicas standar en 3 diferentes zonas de disponibilidad. Después las agregaremos al balanceador de carga.

1. En la configuración básica de la VM guíese por la siguiente imágen. Es importante que se fije en la "Avaiability Zone", donde la VM1 será 1, la VM2 será 2 y la VM3 será 3.

![](images/part2/part2-vm-create1.png)

2. En la configuración de networking, verifique que se ha seleccionado la *Virtual Network*  y la *Subnet* creadas anteriormente. Adicionalmente asigne una IP pública y no olvide habilitar la redundancia de zona.

![](images/part2/part2-vm-create2.png)

3. Para el Network Security Group seleccione "avanzado" y realice la siguiente configuración. No olvide crear un *Inbound Rule*, en el cual habilite el tráfico por el puerto 3000. Cuando cree la VM2 y la VM3, no necesita volver a crear el *Network Security Group*, sino que puede seleccionar el anteriormente creado.

![](images/part2/part2-vm-create3.png)

4. Ahora asignaremos esta VM a nuestro balanceador de carga, para ello siga la configuración de la siguiente imágen.

![](images/part2/part2-vm-create4.png)

5. Finalmente debemos instalar la aplicación de Fibonacci en la VM. para ello puede ejecutar el conjunto de los siguientes comandos, cambiando el nombre de la VM por el correcto

```
git clone https://github.com/daprieto1/ARSW_LOAD-BALANCING_AZURE.git

curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash
source /home/vm1/.bashrc
nvm install node

cd ARSW_LOAD-BALANCING_AZURE/FibonacciApp
npm install

npm install forever -g
forever start FibonacciApp.js
```

Realice este proceso para las 3 VMs, por ahora lo haremos a mano una por una, sin embargo es importante que usted sepa que existen herramientas para aumatizar este proceso, entre ellas encontramos Azure Resource Manager, OsDisk Images, Terraform con Vagrant y Paker, Puppet, Ansible entre otras.

#### Probar el resultado final de nuestra infraestructura

1. Porsupuesto el endpoint de acceso a nuestro sistema será la IP pública del balanceador de carga, primero verifiquemos que los servicios básicos están funcionando, consuma los siguientes recursos:

```
http://52.155.223.248/
http://52.155.223.248/fibonacci/1
```

2. Realice las pruebas de carga con `newman` que se realizaron en la parte 1 y haga un informe comparativo donde contraste: tiempos de respuesta, cantidad de peticiones respondidas con éxito, costos de las 2 infraestrucruras, es decir, la que desarrollamos con balanceo de carga horizontal y la que se hizo con una maquina virtual escalada.

3. Agregue una 4 maquina virtual y realice las pruebas de newman, pero esta vez no lance 2 peticiones en paralelo, sino que incrementelo a 4. Haga un informe donde presente el comportamiento de la CPU de las 4 VM y explique porque la tasa de éxito de las peticiones aumento con este estilo de escalabilidad.

```
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10 &
newman run ARSW_LOAD-BALANCING_AZURE.postman_collection.json -e [ARSW_LOAD-BALANCING_AZURE].postman_environment.json -n 10
```

**Preguntas**

* ¿Cuáles son los tipos de balanceadores de carga en Azure y en qué se diferencian?

* **Azure Load Balancer (Capa de transporte TCP/UDP)**

   * Funciona en la capa de transporte, lo que significa que distribuye tráfico basado en dirección IP y puerto.
   * No inspecciona contenido HTTP o HTTPS.
   
   * Puede ser público (recibe tráfico desde Internet) o interno (balancea dentro de una VNet).
      
   * Realiza el enrutamiento de paquetes mediante hash de IP de origen, IP de destino, puerto de origen y puerto de destino.
      
   * Permite configurar health probes (TCP o HTTP) para detectar el estado de los servidores backend.
   
   * Ofrece reglas de Network Address Translation (NAT) para publicar puertos específicos de VMs.
   
   * Su rendimiento es muy alto y de baja latencia.

* **Azure Application Gateway (Capa de aplicación HTTP/HTTPS)**

   * Opera en la capa de aplicación, por lo que inspecciona y entiende el tráfico HTTP/HTTPS.
   
   * Permite hacer balanceo basado en contenido, es decir, puede enrutar según: el host del encabezado, la ruta de URL, cabeceras HTTP o cookies.
   
   * Desencripta el tráfico HTTPS en el gateway y lo envía como HTTP a los backends.
   
   * Incluye un Web Application Firewall (WAF) integrado para proteger contra ataques como XSS, SQL Injection o CSRF.
      
   * Ofrece integración nativa con Azure Kubernetes Service (AKS) y Azure App Services.

* **Azure Front Door**

   * Opera en la capa de aplicación (HTTP/HTTPS), pero es un servicio global que usa la red perimetral (edge network) de Azure.
   
   * Proporciona balanceo de carga global: distribuye tráfico entre aplicaciones desplegadas en distintas regiones geográficas.
   
   * Incluye aceleración mediante CDN (Content Delivery Network), lo que reduce la latencia para usuarios distribuidos globalmente.
   
   * Realiza enrutamiento inteligente basado en: latencia del cliente, disponibilidad regional, estado de salud del backend.


* **Azure Traffic Manager (Nivel DNS)**

   * Funciona a nivel de DNS (capa 3), no balancea tráfico directamente sino que resuelve nombres de dominio hacia diferentes endpoints.
   
   * No toca los paquetes de datos, solo redirecciona al cliente hacia la IP o región más adecuada según políticas.
   
   * Es independiente del protocolo, ya que actúa antes de que se establezca la conexión (HTTP, TCP, etc.).
   
   * Usa métodos de enrutamiento configurables:
   
      * Priority: dirige el tráfico al primer endpoint activo (para alta disponibilidad).
      
      * Weighted: reparte el tráfico según pesos asignados.
      
      * Performance: envía a la región con menor latencia.
      
      * Geographic: asigna regiones por país o continente.

* ¿Qué es SKU, qué tipos hay y en qué se diferencian?, ¿Por qué el balanceador de carga necesita una IP pública?

SKU (Stock Keeping Unit) se usa en Azure para distinguir versiones o configuraciones de un mismo recurso. Define qué características, límites y precios tiene un servicio. Por ejemplo:

   * Su alcance (regional o zonal)

   * Su nivel de seguridad

   * Su rendimiento y disponibilidad
     
Tipos: 

**Basic SKU**

   * Solo puede existir dentro de una misma red virtual (VNet).
   No puede balancear tráfico entre redes o zonas de disponibilidad.
   
   * Soporta un número limitado de instancias backend (hasta 300).
   
   * No tiene soporte para alta disponibilidad entre zonas (Redundancia zonal).
   
   * El Health Probe es básico (solo TCP o HTTP simple).
   
   * No tiene integración con Azure Monitor ni métricas avanzadas.
      
   * Las reglas de salida (outbound) están fijas y no se pueden personalizar.
   

**Standard SKU**

   * Es la versión empresarial y de producción, diseñada para alta disponibilidad, resiliencia y control granular.
      
   * Alta disponibilidad zonal: el balanceador puede distribuir tráfico entre zonas de disponibilidad (AZs), ofreciendo tolerancia a fallos a nivel físico.
   
   * Soporta hasta 1000 instancias backend o más (depende de la región).
   
   * Permite tanto balanceo interno como público, con múltiples IPs front-end.
   
   * Incluye métricas y diagnósticos detallados integrados con Azure Monitor y Log Analytics.
   
   * Es seguro por defecto: el tráfico de entrada se bloquea hasta que se configuren Network Security Groups (NSGs) explícitamente.
      
   * Permite reglas NAT múltiples, reutilización de puertos y configuración avanzada de outbound.

**Un balanceador de carga necesita una IP pública** porque expone un endpoint de nivel de red (Network Frontend) asociado a una dirección IP pública estática.
Esta IP pública actúa como punto de terminación para el tráfico proveniente de Internet, permitiendo la traducción de direcciones públicas a privadas (NAT) y la distribución de conexiones hacia instancias backend dentro de una red virtual (VNet).

* ¿Cuál es el propósito del *Backend Pool*?

El Backend Pool en Azure Load Balancer es el conjunto lógico de endpoints destino (interfaces de red, IPs privadas o instancias de un Scale Set) donde el balanceador distribuye el tráfico entrante definido en sus reglas. Los paquetes que llegan al Frontend IP son reenrutados mediante NAT hacia una de las instancias del pool, seleccionada según el algoritmo de distribución (hash de IP y puerto) y el estado reportado por el Health Probe.

* ¿Cuál es el propósito del *Health Probe*?

El Health Probe en Azure Load Balancer es el mecanismo de supervisión activa que verifica periódicamente la disponibilidad de los endpoints del Backend Pool mediante sondeos TCP o HTTP.
Su propósito técnico es actualizar dinámicamente el plano de control del balanceador, marcando las instancias como Healthy o Unhealthy para que solo los destinos operativos reciban tráfico.
  
* ¿Cuál es el propósito de la *Load Balancing Rule*? ¿Qué tipos de sesión persistente existen, por qué esto es importante y cómo puede afectar la escalabilidad del sistema?.

La Load Balancing Rule en Azure define el mapeo de tráfico entre el Frontend y el Backend Pool, especificando puerto, protocolo, Health Probe y política de sesión persistente.
Controla cómo el balanceador distribuye las conexiones entrantes hacia las instancias disponibles del backend.

Los tipos de sesión persistente son:

   * None: Sin afinidad. Cada solicitud puede ir a cualquier backend (stateless, mayor escalabilidad).
   
   * Client IP: Todas las solicitudes desde una misma IP se envían al mismo backend (stateful, mantiene sesión).
   
   * Client IP + Protocol: Afinidad por IP y protocolo, útil para servicios mixtos TCP/UDP. Por ejemplo, cuando una misma IP de cliente abre conexiones por diferentes servicios (ej., HTTP y WebSocket).

La persistencia es importante porque garantiza continuidad de sesión, pero limita la capacidad del balanceador para distribuir carga equitativamente, reduciendo la escalabilidad y tolerancia a fallos en sistemas que dependen de afinidad.
  
* ¿Qué es una *Virtual Network*? ¿Qué es una *Subnet*? ¿Para qué sirven los *address space* y *address range*?

Una **Virtual Network (VNet)** en Azure es una red privada virtual y aislada dentro de la nube, que permite comunicar de forma segura los recursos (como máquinas virtuales, bases de datos o aplicaciones) entre sí y con otras redes. Define un espacio de direcciones IP (address space) propio, por ejemplo 10.0.0.0/16 (usado en el laboratorio) dentro del cual se crean subredes (subnets), reglas de ruteo y seguridad (NSG) para controlar cómo fluye el tráfico dentro y fuera de esa red.

Una **Subnet** es una división lógica del address space de la VNet, que agrupa recursos bajo un mismo rango de IP y políticas de red. Cada subred tiene su propio address range (subconjunto CIDR) y puede tener asociadas reglas de Network Security Groups (NSG) o ruteo personalizado (UDR), por ejemplo 10.0.0.0/24 (usado en el laboratorio).

**Address Space:**
Es el bloque principal de direcciones IP asignado a toda la VNet, expresado en formato CIDR.
Define los límites de IPs disponibles en esa red virtual.

**Address Range:**
Es el subconjunto del address space que se asigna a cada subnet. Determina las IPs que pueden usar los recursos dentro de esa subred.
  
* ¿Qué son las *Availability Zone* y por qué seleccionamos 3 diferentes zonas?. ¿Qué significa que una IP sea *zone-redundant*?

Es una unidad física de aislamiento dentro de una región de Azure, compuesta por uno o varios centros de datos independientes con su propia energía, refrigeración y red. Estas zonas están interconectadas con baja latencia, pero lo suficientemente aisladas para que una falla eléctrica, de red o de hardware en una zona no afecte las otras.

Se distribuyeron las 3 máquinas virtuales en distintas zonas para lograr alta disponibilidad y tolerancia a fallos:

* Si una zona completa sufre una caída física o de red, las otras dos zonas continúan operando.

* El Load Balancer Standard detecta automáticamente, mediante los Health Probes, qué VMs siguen activas y redirige el tráfico solo a las que están disponibles.

* Esto implementa un modelo de resiliencia zonal, en el que el servicio permanece activo incluso ante una falla local.

Que una IP sea **zone-redundant** significa que no pertenece físicamente a una sola zona de disponibilidad, sino que existe y funciona al mismo tiempo en todas las zonas de la región. La IP no se cae si una zona falla, porque Azure la tiene replicada en la infraestructura de red de todas las zonas.

* ¿Cuál es el propósito del *Network Security Group*?

El Network Security Group (NSG) en Azure tiene como propósito controlar y filtrar el tráfico de red entrante y saliente hacia los recursos de una VNet, como máquinas virtuales, subnets o interfaces de red. Técnicamente, un NSG actúa como un firewall de capa de transporte, que aplica reglas de seguridad basadas en IP, puerto y protocolo (TCP/UDP). Cada regla define si se permite o bloquea el tráfico, según origen, destino, dirección y prioridad.
  
* Informe de newman 1 (Punto 2)
* Presente el Diagrama de Despliegue de la solución.




