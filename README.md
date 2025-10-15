
VIDEO EXPLICATIVO https://youtu.be/BTiY50b4aDk

# PASO 0 – FUNDAMENTACIÓN DE LA BASE DE DATOS **RIAK**

## 1. Introducción general

**Riak** es una base de datos **NoSQL distribuida** que trabaja bajo el modelo **clave-valor (key-value)** y fue desarrollada por **Basho Technologies**. Está inspirada en el proyecto **Amazon Dynamo**, y su enfoque principal es ofrecer **alta disponibilidad**, **tolerancia a fallos** y **escalabilidad horizontal**, más que consistencia estricta.  

Riak no es un solo producto, sino una **plataforma** que tiene varias versiones y soluciones, todas basadas en el mismo núcleo distribuido. Las principales son:

- **Riak KV:** la versión más básica, donde se guardan datos como pares clave-valor.
- **Riak TS:** pensada para datos de **series de tiempo**, por ejemplo sensores o IoT.
- **Riak S2 (o Riak CS):** usada como **almacenamiento de objetos**, tipo Amazon S3.

En general, Riak se usa cuando se necesitan sistemas que sigan funcionando aunque haya caídas o desconexiones, como en entornos distribuidos, IoT, redes sociales o backends de aplicaciones grandes.

---

## 2. Conceptos de bases de datos distribuidas aplicados en Riak

Riak trabaja con una arquitectura **peer-to-peer** totalmente **descentralizada**. Esto significa que todos los nodos son iguales y no existe un “nodo maestro” que controle a los demás.  

Cada nodo:
- Sabe qué parte de los datos le toca gracias a un **hash consistente**, que divide el espacio de claves en un **anillo lógico (ring)**.  
- Puede recibir cualquier solicitud (lectura o escritura) y, si no es el dueño de esa clave, redirigirla al nodo correcto.  
- Tiene mecanismos internos para **replicar**, **recuperar** y **mantener consistencia eventual** con los otros nodos.

### 🔹 Coordinador lógico (en tiempo de ejecución)

Aunque todos los nodos son iguales, cuando un cliente envía una petición (por ejemplo, un `PUT` o un `GET`), el nodo que la recibe **actúa temporalmente como “coordinador”**.  
Ese nodo:
1. Recibe la solicitud del cliente.  
2. Calcula qué nodos del anillo deben manejar esa clave.  
3. Envía la operación a esos nodos.  
4. Espera las respuestas y devuelve el resultado al cliente.  

Si ese nodo coordinador se cae durante la operación, **no pasa nada grave**:
- Otros nodos pueden asumir el rol en las próximas peticiones.  
- Si los datos ya se habían replicado, se mantienen gracias a mecanismos como **read repair** o **hinted handoff**.  
- Si no se había confirmado la operación, el cliente puede reenviar la solicitud sin riesgo de corrupción (porque las operaciones son idempotentes).

En resumen, el coordinador lógico **no es un líder permanente**, solo coordina una operación puntual.

---

### 🔹 Coordinador de despliegue (en Docker o la nube)

En la documentación de Riak (por ejemplo, en `docker-compose.yml`) a veces aparece un nodo llamado **“coordinator”**, y puede confundir un poco.  
Ese nodo **no es el coordinador lógico** del sistema, sino un **punto de arranque** del clúster (también llamado *seed node* o *bootstrap node*).

Cuando se levanta Riak con Docker, este “coordinator” solo sirve para que los otros nodos (los `member`) puedan conectarse y formar el clúster inicial.  
Después de eso, todos los nodos son iguales.  
Si apagas el contenedor `coordinator` **una vez el clúster está formado**, Riak **sigue funcionando sin problemas**.  

Solo si lo apagas **antes** de que los demás se unan, los otros nodos no sabrán con quién conectarse.  
Pero una vez el clúster existe, el sistema sigue sirviendo peticiones, replicando y respondiendo, sin depender del nodo inicial.

> 🔸 En pocas palabras: el `coordinator` del YAML es solo un nodo de arranque, no un maestro.  
> Si se cae, los demás nodos siguen operando normalmente gracias a la arquitectura descentralizada de Riak.

---

## 3. Escalabilidad y consistencia

Riak sacrifica un poco de **consistencia inmediata** para ganar en **disponibilidad y escalabilidad**.  
Trabaja con un modelo de **consistencia eventual**, donde las actualizaciones se propagan de forma asíncrona entre nodos.

El comportamiento del sistema se puede ajustar con tres parámetros:

- **N:** número de réplicas totales.  
- **W:** número de réplicas que deben confirmar una escritura.  
- **R:** número de réplicas que deben responder a una lectura.

Por ejemplo:
- Si se prioriza `W` alto, hay más consistencia.  
- Si se prioriza `A` (Availability), se puede aceptar respuestas de menos nodos.

Esto le da flexibilidad al sistema para adaptarse a distintos escenarios.

---

## 4. Teorema CAP aplicado a Riak

El **teorema CAP** dice que una base de datos distribuida solo puede cumplir **dos de tres** propiedades:  
**Consistencia (C)**, **Disponibilidad (A)** y **Tolerancia a particiones (P)**.

Riak está diseñada principalmente para ser **AP**:
- Siempre responde (alta disponibilidad).  
- Tolera caídas o particiones de red (tolerancia a particiones).  
- La consistencia puede ser eventual, no estricta.

Este enfoque hace que Riak sea ideal para sistemas donde **lo importante es no dejar de responder**, incluso si hay nodos caídos.

---

## 5. Particionamiento, replicación y modelo de fallos

Riak usa un **anillo de hash consistente** para repartir los datos entre los nodos.  
Cada nodo tiene asignado un rango de claves y guarda una copia de los datos junto con otros nodos (replicación).

### Mecanismos de tolerancia a fallos:
- **Hinted Handoff:** cuando un nodo cae, otro guarda temporalmente sus datos.  
- **Read Repair:** si en una lectura se detectan versiones desactualizadas, se corrigen.  
- **Anti-Entropy:** sincronización periódica entre nodos para evitar divergencias.

Gracias a eso, Riak sigue funcionando aunque varios nodos estén inactivos, y se recupera automáticamente cuando vuelven a estar en línea.

---

## 6. Modelado de datos y comparación con modelo relacional

Riak KV no tiene tablas ni columnas como una base relacional.  
Usa **buckets** (contenedores) con pares clave-valor.  
Los valores pueden ser JSON, binarios o cualquier tipo de dato serializado.

| Aspecto | Relacional | Riak KV |
|----------|-------------|----------|
| Estructura | Tablas con esquema fijo | Buckets sin esquema |
| Relaciones | Llaves foráneas y joins | Se manejan desde la aplicación |
| Transacciones | ACID | BASE (Basically Available, Soft-state, Eventual consistency) |
| Escalabilidad | Vertical | Horizontal |

**Migrar de relacional a Riak** implica desnormalizar.  
Por ejemplo, en lugar de tener una tabla `Usuarios` y otra `Pedidos`, se podría guardar un solo objeto JSON con toda la info del usuario y sus pedidos.

### Riak TS  
Usa una estructura similar, pero adaptada a **series de tiempo**, con columnas como `timestamp`, `sensor_id`, `valor`, etc.  
Permite hacer consultas por rangos de tiempo (`WHERE time > t1 AND time < t2`).

### Riak S2  
Está hecho para guardar **objetos grandes** (imágenes, documentos, videos).  
Cada objeto tiene una clave y metadatos asociados, parecido a Amazon S3.

---

## 7. Oportunidades en VectorDB, LLM y GenIA

Aunque Riak no es una base de datos vectorial como tal, se puede integrar en ese tipo de ecosistemas por su arquitectura distribuida.

Algunas ideas:
- **Riak KV:** para guardar embeddings de usuarios o textos como vectores binarios.  
- **Riak TS:** para registrar métricas o logs de entrenamiento de modelos.  
- **Riak S2:** para guardar datasets, modelos o archivos grandes de IA.  
- Combinado con **Apache Spark** o **Solr**, se pueden hacer búsquedas más complejas o análisis distribuidos.

En general, Riak puede funcionar como una base de almacenamiento confiable dentro de pipelines de **IA generativa o vector search**.

---

## Conclusión

Riak es una base de datos **distribuida, tolerante a fallos y sin un nodo maestro**, lo que le da alta disponibilidad y escalabilidad horizontal.  
Su arquitectura tipo Dynamo y sus mecanismos de replicación y reparación automática la hacen muy robusta para entornos donde la disponibilidad es más importante que la consistencia estricta.  

Además, el hecho de que tenga distintas versiones (KV, TS, S2) le permite cubrir varios casos de uso: desde almacenamiento simple hasta series de tiempo o almacenamiento de objetos.  
Y aunque no es una VectorDB como tal, puede integrarse fácilmente con entornos de IA gracias a su flexibilidad y APIs distribuidas.
