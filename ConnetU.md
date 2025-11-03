# BYRON RODOLFO MALDONADO PALACIOS  - 202300076
# SEGUNDA EVALUACION PARCIAL
---
# Bases de Datos NoSQL: Grafos y Clave-Valor

## 1. ¿Qué es una base de datos basada en grafos y dos ventajas sobre una base relacional tradicional?

Una **base de datos basada en grafos** es un tipo de base de datos **NoSQL** que representa la información mediante **nodos** (entidades) y **relaciones** (conexiones) entre ellos.  
Este modelo se enfoca en **cómo los datos están conectados** más que en cómo se almacenan en tablas.

### Ventajas sobre una base relacional:
- **Eficiencia en consultas de relaciones complejas:** permite recorrer conexiones entre datos (como “amigos de mis amigos”) de forma más rápida que con uniones (`JOIN`) en SQL.  
- **Flexibilidad en el modelo de datos:** no necesita un esquema fijo; se pueden agregar nuevos tipos de nodos o relaciones sin alterar estructuras existentes.

---

## 2. ¿Qué tipo de problemas resuelve Redis y por qué se clasifica como una base de datos clave-valor?

**Redis** es una base de datos **NoSQL** que se utiliza para resolver problemas que requieren **alta velocidad de acceso a los datos**, como:
- Almacenamiento de sesiones de usuario  
- Cachés  
- Colas de mensajes  
- Contadores en tiempo real  

Se clasifica como una **base de datos clave-valor** porque almacena los datos en **pares únicos**, donde cada **clave** identifica de forma exclusiva un **valor**.  
Por ejemplo:

```
usuario:123 → "Ana López"
```

Esto permite acceder a la información de forma directa y muy rápida.

---

## 3. ¿Cuál es la diferencia entre un nodo y una relación en Neo4j?

En **Neo4j**, un **nodo** representa una entidad o actor del sistema, como un estudiante, un curso o un grupo.  
Una **relación**, en cambio, representa la conexión o vínculo entre dos nodos, como “SIGUE”, “PERTENECE_A” o “RECOMIENDA”.

**Ejemplo en la red social ConnectU:**

```
(Ana) -[:SIGUE]-> (Carlos)
```

Aquí **Ana** y **Carlos** son nodos, mientras que **SIGUE** es la relación que los conecta.

---

## 4. ¿Qué son las propiedades en Neo4j y cómo pueden aplicarse a un modelo de red social?

Las **propiedades** en Neo4j son pares **clave-valor** que se asignan a **nodos o relaciones** para guardar información adicional.  

**Ejemplo:**
```json
{
  "nombre": "Ana López",
  "carrera": "Ingeniería en Sistemas",
  "edad": 21
}
```

En una red social como *ConnectU*, las propiedades permiten personalizar el modelo, agregando detalles a los usuarios, publicaciones o grupos, y facilitando consultas específicas, como buscar a “todos los estudiantes de Ingeniería en Sistemas menores de 25 años”.

# Parte A – Modelado en Neo4j

## Descripción del caso ConnectU

Se diseña un modelo en **Neo4j** bajo las siguientes condiciones:

- Cada **Estudiante** puede `[:SIGUE]` a otros estudiantes.  
- Los estudiantes pueden `[:PERTENECE_A]` uno o varios **GruposDeEstudio**.  
- Los grupos están relacionados con un **Curso** mediante `[:ASOCIADO_A]`.  
- Los estudiantes pueden `[:RECOMIENDA]` a un **Profesor**.

---

## a) Descripción del grafo

### Pseudonodos principales

| Nodo | Descripción |
|------|--------------|
| **Estudiante** | Representa a un alumno dentro de ConnectU. |
| **GrupoDeEstudio** | Grupo al que pertenecen los estudiantes. |
| **Curso** | Curso académico asociado al grupo. |
| **Profesor** | Docente que puede ser recomendado por los estudiantes. |

### Relaciones del modelo

| Relación | Sentido | Significado |
|-----------|----------|-------------|
| `(:Estudiante)-[:SIGUE]->(:Estudiante)` | Dirigida | Un estudiante sigue a otro. |
| `(:Estudiante)-[:PERTENECE_A]->(:GrupoDeEstudio)` | Dirigida | Indica membresía en un grupo. |
| `(:GrupoDeEstudio)-[:ASOCIADO_A]->(:Curso)` | Dirigida | El grupo está vinculado a un curso. |
| `(:Estudiante)-[:RECOMIENDA]->(:Profesor)` | Dirigida | Un estudiante recomienda a un profesor. |

### Ejemplo de grafo (pseudonodos)

```
(Estudiante: Juan Pérez) -[:PERTENECE_A]-> (Grupo: Estudio Neo4j)
(Estudiante: Ana López) -[:PERTENECE_A]-> (Grupo: Estudio Neo4j)
(Grupo: Estudio Neo4j) -[:ASOCIADO_A]-> (Curso: Bases de Datos II)
(Estudiante: Juan Pérez) -[:RECOMIENDA]-> (Profesor: Carlos García)
(Estudiante: Ana López) -[:SIGUE]-> (Estudiante: Juan Pérez)
```

---

## b) Consultas Cypher

### 1️ Encontrar los compañeros de grupo de 'Juan Pérez'
```cypher
MATCH (juan:Estudiante {nombre:'Juan Pérez'})-[:PERTENECE_A]->(g:GrupoDeEstudio)<-[:PERTENECE_A]-(compañero:Estudiante)
WHERE compañero <> juan
RETURN compañero.nombre AS compañero_de_grupo;
```

### 2️ Contar cuántos estudiantes recomiendan al profesor 'Carlos García'
```cypher
MATCH (e:Estudiante)-[:RECOMIENDA]->(p:Profesor {nombre:'Carlos García'})
RETURN p.nombre AS profesor, count(e) AS total_recomendaciones;
```

### 3️ Obtener los grupos asociados al curso 'Bases de Datos II'
```cypher
MATCH (g:GrupoDeEstudio)-[:ASOCIADO_A]->(c:Curso {nombre:'Bases de Datos II'})
RETURN g.nombre AS grupo_asociado;
```

---

## Resumen

El modelo permite representar de forma intuitiva las interacciones sociales y académicas entre los estudiantes, sus grupos, los cursos y los profesores.  
Gracias a las relaciones `SIGUE`, `PERTENECE_A`, `ASOCIADO_A` y `RECOMIENDA`, Neo4j facilita consultas eficientes sobre redes de conexión, membresías y afinidades académicas.


# Parte B — Redis (ConnectU)
 **claves propuestas**, **comandos paso a paso** para demostrar los incisos 2(a) y 2(b), y la **justificación** del uso de Redis junto con Neo4j.

---

## 1) Propuesta de claves Redis y tipo de valor

| Clave | Tipo | ¿Qué guarda? | Operaciones típicas |
|---|---|---|---|
| `user:{id}:profile` | **HASH** | Campos del perfil de un estudiante (name, email, career, joined_at). | `HSET`, `HGET`, `HGETALL`, `HDEL` |
| `user:{id}:recent:courses` | **LIST** | Pila de **últimos cursos visitados** (máx. 5). | `LPUSH`, `LTRIM`, `LRANGE`, `EXPIRE` |
| `groups:popularity` | **ZSET (Sorted Set)** | **Ranking** de grupos por popularidad (score = miembros/visitas). | `ZINCRBY`, `ZREVRANGE`, `ZREM` |

> Ventajas: el **HASH** permite leer/escribir campos individuales; las **LIST** son ideales para pilas/recientes; el **ZSET** mantiene rankings en tiempo real.

---

## 2) Demostraciones paso a paso

> Puedes ejecutar todo desde `redis-cli` (ya sea dentro del contenedor Docker o con Redis instalado localmente).

### 2.1. Arranque rápido con Docker
```bash
docker run -d --name redis -p 6379:6379 redis:latest
docker exec -it redis redis-cli
```

---

### 2(a). “Los últimos 5 cursos visitados por un estudiante” (LIST)

**Crear perfil (opcional, HASH para contexto):**
```redis
HSET user:101:profile name "Luisa Martin" email "luisa@connectu.edu" career "Diseño Gráfico" joined_at "2025-02-10"
HGETALL user:101:profile
```

**Registrar visitas y limitar a 5:**
```redis
LPUSH user:101:recent:courses DWA-2025
LPUSH user:101:recent:courses BDG-101
LPUSH user:101:recent:courses IAL-2025
LPUSH user:101:recent:courses EST-2025
LPUSH user:101:recent:courses UX-200
LTRIM user:101:recent:courses 0 4
```

**Consultar los 5 más recientes (nuevo → antiguo):**
```redis
LRANGE user:101:recent:courses 0 4
```

**(Opcional) Caducidad automática de la lista:**
```redis
EXPIRE user:101:recent:courses 604800   # 7 días
```

---

### 2(b). “Los 3 grupos más populares del momento” (ZSET)

**Incrementar popularidad (por uniones/visitas):**
```redis
ZINCRBY groups:popularity 1 grupo:devweb
ZINCRBY groups:popularity 1 grupo:data
ZINCRBY groups:popularity 1 grupo:ux
ZINCRBY groups:popularity 5 grupo:devweb
ZINCRBY groups:popularity 3 grupo:data
ZINCRBY groups:popularity 2 grupo:ux
```

**Top 3 (de mayor a menor score):**
```redis
ZREVRANGE groups:popularity 0 2 WITHSCORES
```

**Nota (top por día):** usa claves por ventana de tiempo, p. ej. `groups:popularity:YYYY-MM-DD`, y aplica `EXPIRE` para mantener solo las ventanas activas.

---

## 3) ¿Por qué Redis es útil junto con Neo4j en este caso?

- **Caché de recomendaciones (read‑through):** las recomendaciones de grupos basadas en relaciones (intereses, amigos‑de‑amigos) se calculan en **Neo4j**, y se guardan en Redis como `user:{id}:reco:groups` (**LIST/ZSET**) con **TTL** (10–30 min). Las siguientes consultas responden en **milisegundos** sin recalcular el grafo.
- **Rankings y contadores en tiempo real:** los **ZSET** de Redis mantienen el **Top de grupos** con `ZINCRBY` sin hacer agregaciones costosas en el grafo.
- **Desacoplamiento y escalabilidad:** Neo4j asegura la **consistencia semántica** del grafo; Redis atiende **lecturas ultra rápidas**, últimos visitados, sesiones y rate‑limiting.
- **Invalidación simple:** si cambian relaciones en Neo4j (nuevo interés, unión a grupo), se invalida la caché con `DEL user:{id}:reco:groups` para regenerarla en la próxima lectura.

**Resultado:** menor latencia, menor carga sobre Neo4j y experiencia en tiempo real para el usuario.

---

# Sección III – Análisis y Diseño (Nivel Avanzado)

El equipo quiere agregar **recomendaciones personalizadas de grupos** según los intereses y conexiones de un estudiante.  
A continuación se explica cómo modelar la funcionalidad en Neo4j, cómo combinar Neo4j con Redis y por qué una base relacional no sería óptima.

---

## 1️ Modelado en Neo4j

### Descripción general

El modelo utiliza relaciones que permiten capturar afinidades, intereses y conexiones entre estudiantes y grupos.

### Entidades

| Nodo | Descripción |
|------|--------------|
| **Student** | Representa a un estudiante. |
| **Group** | Representa un grupo de estudio. |
| **Topic** | Representa un tema de interés o materia. |

### Relaciones

| Relación | Sentido | Propósito |
|-----------|----------|-----------|
| `(:Student)-[:INTEREST_IN]->(:Topic)` | Dirigida | Muestra los temas que interesan al estudiante. |
| `(:Group)-[:ABOUT]->(:Topic)` | Dirigida | Define sobre qué tema trata cada grupo. |
| `(:Student)-[:MEMBER_OF]->(:Group)` | Dirigida | Indica a qué grupo pertenece el estudiante. |
| `(:Student)-[:FRIEND_WITH]-(:Student)` | Bidireccional | Indica amistad o conexión entre estudiantes. |

### Ejemplo de modelo (pseudonodos)
```
(Student: Luisa) -[:INTEREST_IN]-> (Topic: Node.js)
(Student: Luisa) -[:FRIEND_WITH]-> (Student: Pedro)
(Student: Pedro) -[:MEMBER_OF]-> (Group: Ciencia de Datos)
(Group: Ciencia de Datos) -[:ABOUT]-> (Topic: Python)
```

### Consultas de recomendación

#### a) Por intereses
```cypher
MATCH (s:Student {id:'101'})-[:INTEREST_IN]->(t:Topic)<-[:ABOUT]-(g:Group)
WHERE NOT (s)-[:MEMBER_OF]->(g)
RETURN g.name AS grupo_recomendado, count(*) AS coincidencias
ORDER BY coincidencias DESC;
```

#### b) Por amigos
```cypher
MATCH (s:Student {id:'101'})-[:FRIEND_WITH]-(f:Student)-[:MEMBER_OF]->(g:Group)
WHERE NOT (s)-[:MEMBER_OF]->(g)
RETURN g.name AS grupo_por_amigos, count(DISTINCT f) AS amigos_en_grupo
ORDER BY amigos_en_grupo DESC;
```

#### c) Puntaje combinado
```cypher
MATCH (s:Student {id:'101'})
OPTIONAL MATCH (s)-[:INTEREST_IN]->(t:Topic)<-[:ABOUT]-(g:Group)
WHERE NOT (s)-[:MEMBER_OF]->(g)
WITH s, g, count(t) AS temas_comunes
OPTIONAL MATCH (s)-[:FRIEND_WITH]-(f:Student)-[:MEMBER_OF]->(g)
WITH g, temas_comunes, count(DISTINCT f) AS amigos_en_grupo
RETURN g.name AS grupo, 2*temas_comunes + amigos_en_grupo AS puntaje
ORDER BY puntaje DESC;
```

---

## 2️ Estrategia combinada Neo4j + Redis

| Objetivo | Descripción |
|-----------|-------------|
| **Optimizar rendimiento** | Redis se usa como caché para almacenar temporalmente las recomendaciones generadas por Neo4j. |
| **Reducir tiempo de respuesta** | Redis devuelve resultados en milisegundos mientras Neo4j se encarga del análisis de relaciones. |
| **Evitar consultas repetitivas** | Los resultados de recomendación se guardan con un TTL (tiempo de vida) y se actualizan solo cuando cambian las relaciones. |

### Ejemplo de implementación

```redis
# Guardar recomendaciones con TTL de 30 minutos
ZADD user:101:reco:groups 5 devweb 3 data 2 ux
EXPIRE user:101:reco:groups 1800

# Consultar top 3 grupos recomendados
ZREVRANGE user:101:reco:groups 0 2 WITHSCORES
```

**Flujo de uso:**
1. El backend consulta Redis para ver si existe `user:{id}:reco:groups`.
2. Si existe → responde al cliente en milisegundos.  
   Si no existe → ejecuta la consulta Cypher en Neo4j.
3. Los resultados se guardan en Redis con `EXPIRE 1800` (30 min).
4. Al cambiar intereses o grupos → `DEL user:{id}:reco:groups` para forzar recalculo.

---

## 3️ Por qué una base relacional no sería óptima

| Criterio | Base Relacional | Neo4j + Redis |
|-----------|----------------|---------------|
| Relaciones complejas | Requiere múltiples JOINs costosos | Navegación nativa por relaciones |
| Rendimiento en consultas | Disminuye con la profundidad | Estable en grafos grandes |
| Flexibilidad de esquema | Cambios estructurales costosos | Dinámico y sin esquema fijo |
| Latencia | Alta en consultas recursivas | Milisegundos con caché Redis |
| Escalabilidad | Limitada en relaciones N:M | Escalable horizontalmente |

