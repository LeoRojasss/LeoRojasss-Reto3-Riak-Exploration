# 🚀 Reto Riak – Parte Práctica

Este documento contiene la **instalación, exploración y ejemplos prácticos** de la base de datos **Riak** utilizando **Docker**.  
Forma parte del desarrollo del **Reto 3 – Bases de Datos Distribuidas**.

---

## 🐳 1. Instalación con Docker

### Paso 1: Crear el archivo `docker-compose.yml`

```yaml
version: '3'
services:
  riak:
    image: basho/riak-kv:latest
    container_name: riak-node1
    ports:
      - "8087:8087"   # Puerto Protocol Buffers
      - "8098:8098"   # Puerto HTTP REST API
    environment:
      - RIAK_HTTP_PORT=8098
      - RIAK_PROTOBUF_PORT=8087
    volumes:
      - riak_data:/var/lib/riak
volumes:
  riak_data:
```

### Paso 2: Levantar el contenedor

```bash
docker compose up -d
```

Verifica que esté corriendo:

```bash
docker ps
```

---

## 🔍 2. Verificar el estado del nodo

```bash
docker exec -it riak-node1 riak ping
```

Debe responder:

```
pong
```

---

## 🧰 3. Operaciones CRUD con la API HTTP

Riak expone una **API REST** para manipular datos.

### Crear un bucket y guardar un valor

```bash
curl -XPUT http://localhost:8098/buckets/usuarios/keys/user1   -H "Content-Type: application/json"   -d '{"nombre": "Leo Rojas", "edad": 23, "ciudad": "Medellín"}'
```

### Consultar el valor

```bash
curl http://localhost:8098/buckets/usuarios/keys/user1
```

**Resultado esperado:**

```json
{"nombre":"Leo Rojas","edad":22,"ciudad":"Medellín"}
```

### Actualizar un valor

```bash
curl -XPUT http://localhost:8098/buckets/usuarios/keys/user1   -H "Content-Type: application/json"   -d '{"nombre": "Leo Rojas", "edad": 25, "ciudad": "Bogotá"}'
```

### Eliminar un valor

```bash
curl -XDELETE http://localhost:8098/buckets/usuarios/keys/user1
```

---

## 🎵 4. Ejemplo práctico: Sistema tipo Spotify

Simulación de almacenamiento de preferencias musicales:

```bash
curl -XPUT http://localhost:8098/buckets/playlists/keys/user1   -H "Content-Type: application/json"   -d '{"usuario": "alrojasp", "generos": ["rock", "metal", "indie"], "favoritas": ["Nothing Else Matters", "In the End", "Creep"]}'
```

Consulta la playlist:

```bash
curl http://localhost:8098/buckets/playlists/keys/user1
```

---

## ⚙️ 5. Ejemplo de replicación con múltiples nodos

Puedes levantar un pequeño clúster de Riak:

```yaml
version: "2"
services:
  coordinator:
    image: basho/riak-kv
    ports:
      - "8087"
      - "8098"
    environment:
      - CLUSTER_NAME=riakkv
    labels:
      - "com.basho.riak.cluster.name=riak-kv"
    volumes:
      - schemas:/etc/riak/schemas
    network_mode: bridge
  member:
    image: basho/riak-kv
    ports:
      - "8087"
      - "8098"
    labels:
      - "com.basho.riak.cluster.name=riak-kv"
    links:
      - coordinator
    network_mode: bridge
    depends_on:
      - coordinator
    environment:
      - CLUSTER_NAME=riakkv
      - COORDINATOR_NODE=coordinator

volumes:
  schemas: {}
```

Luego, puedes hacer peticiones como las anteriores y apagar nodos y verificar que el cluster sigue funcionando perfectamente.

**Para escalar el cluster:**

```bash
docker compose scale coordinator=1 member=4
```

---

## 🤖 6. Integración con IA / LLM

Aunque Riak no es un VectorDB, puede almacenar **embeddings** o **resultados de IA**:

```bash
curl -XPUT http://localhost:8098/buckets/embeddings/keys/doc1   -H "Content-Type: application/json"   -d '{"texto": "La energía solar es limpia y renovable.", "vector": [0.12, 0.55, 0.98, 0.44]}'
```

Esto permite usar Riak como almacenamiento distribuido de contextos o resultados generados por **LLMs**.

---

## 7. Conclusión práctica

- Riak puede ejecutarse fácilmente en contenedores Docker.
- Permite operaciones CRUD mediante API HTTP.
- Su arquitectura distribuida facilita la replicación y la consistencia eventual.
- Es ideal como backend para aplicaciones **distribuidas, IoT o IA**.


