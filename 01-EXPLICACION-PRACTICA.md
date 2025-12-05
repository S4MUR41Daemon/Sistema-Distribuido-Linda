# Explicación de la Práctica 5 - Sistema Distribuido LINDA

## ¿Qué es LINDA?

LINDA es un modelo de programación para sistemas distribuidos que permite la comunicación entre procesos mediante un **espacio de tuplas compartido**. Imagínatelo como una "pizarra compartida" donde diferentes procesos pueden dejar mensajes (tuplas) y otros procesos pueden leerlos o eliminarlos.

## Conceptos Clave

### 1. Tuplas
Una **tupla** es una secuencia ordenada de elementos (en este caso, todos de tipo String). Ejemplos:
- `["Alberto", "20", "Suspenso"]` - Tupla de 3 elementos
- `["Juan", "25"]` - Tupla de 2 elementos
- `["Producto1", "Precio", "100", "EUR", "Stock", "50"]` - Tupla de 6 elementos

### 2. Patrones (Patterns)
Un **patrón** es una tupla que puede contener **variables** para hacer búsquedas. Las variables se identifican con `?` seguido de una letra mayúscula (A-Z).

Ejemplos:
- `["Alberto", "?X", "?Y"]` - Busca tuplas que empiecen con "Alberto", y captura los elementos 2 y 3 en variables X e Y
- `["?Nombre", "20", "?Nota"]` - Busca tuplas donde el segundo elemento sea "20"

### 3. Operaciones Básicas

#### PostNote (PN) - Publicar una Tupla
- **Qué hace**: Añade una tupla al espacio compartido
- **Ejemplo**: `PN(["Alberto", "20", "Suspenso"])` - Guarda esta tupla en el servidor correspondiente
- **Características**: No bloquea, siempre tiene éxito (a menos que haya error de conexión)

#### ReadNote (ReadN) - Leer una Tupla
- **Qué hace**: Busca una tupla que coincida con el patrón y devuelve sus valores
- **Ejemplo**: `ReadN(["Alberto", "?X", "?Y"])` - Busca una tupla que empiece con "Alberto" y devuelve los valores
- **Características**: **NO elimina** la tupla del espacio, solo la lee. Si no encuentra coincidencia, espera hasta que aparezca.

#### RemoveNote (RN) - Eliminar una Tupla
- **Qué hace**: Busca una tupla que coincida con el patrón, la devuelve y **la elimina** del espacio
- **Ejemplo**: `RN(["Alberto", "?X", "?Y"])` - Busca, devuelve y elimina la tupla
- **Características**: Si no encuentra coincidencia, espera hasta que aparezca (bloquea hasta encontrar)

## Arquitectura del Sistema

### Distribución por Longitud de Tuplas

El sistema está dividido en **3 servidores** que se ejecutan en máquinas diferentes:

```
┌─────────────────────────────────────────┐
│         CLIENTES (Múltiples)            │
│  - Se conectan a los servidores         │
│  - Realizan operaciones PN, RN, ReadN   │
└──────────────┬──────────────────────────┘
               │
       ┌───────┴────────┐
       │                │
       ▼                ▼
┌──────────────┐  ┌──────────────┐
│  SERVIDOR 1  │  │  SERVIDOR 2  │
│  Tuplas 1-3  │  │  Tuplas 4-5  │
│  + RÉPLICA   │  │               │
└──────────────┘  └──────────────┘
                         │
                         ▼
                  ┌──────────────┐
                  │  SERVIDOR 3  │
                  │  Tuplas 6    │
                  └──────────────┘
```

#### Servidor 1: Tuplas de longitud 1 a 3
- Gestiona todas las tuplas con 1, 2 o 3 elementos
- **IMPORTANTE**: Tiene un **servidor réplica** para alta disponibilidad
- Si el servidor primario cae, el réplica toma el control
- Ejemplo de tuplas: `["Hola"]`, `["Juan", "25"]`, `["Alberto", "20", "Suspenso"]`

#### Servidor 2: Tuplas de longitud 4 a 5
- Gestiona tuplas con 4 o 5 elementos
- Ejemplo: `["Producto", "Precio", "100", "EUR"]`

#### Servidor 3: Tuplas de longitud 6
- Gestiona tuplas con exactamente 6 elementos
- Ejemplo: `["Producto1", "Precio", "100", "EUR", "Stock", "50"]`

### ¿Por qué esta distribución?
- **Balanceo de carga**: Distribuye el trabajo entre servidores
- **Escalabilidad**: Cada servidor maneja un rango específico
- **Alta disponibilidad**: El servidor 1 tiene réplica porque almacena datos críticos

## Flujo de Funcionamiento

### 1. Cliente se conecta
```
Cliente → Servidor: "Hola, quiero conectarme"
Servidor → Cliente: "OK, conexión establecida"
```

### 2. Cliente realiza operación
```
Cliente → Servidor 1: "PN(["Juan", "25"])"
Servidor 1: Almacena la tupla en memoria
Servidor 1 → Cliente: "OK, tupla guardada"
```

### 3. Otro cliente busca
```
Cliente 2 → Servidor 1: "ReadN(["Juan", "?X"])"
Servidor 1: Busca tupla que empiece con "Juan"
Servidor 1 → Cliente 2: "Encontrada: ["Juan", "25"]"
```

### 4. Cliente elimina
```
Cliente 2 → Servidor 1: "RN(["Juan", "?X"])"
Servidor 1: Busca, devuelve y elimina
Servidor 1 → Cliente 2: "Tupla eliminada: ["Juan", "25"]"
```

## Aspectos Técnicos Importantes

### 1. Comunicación entre Cliente y Servidor
- Necesitas un **protocolo de comunicación** (sockets TCP/IP, RMI, etc.)
- Los clientes deben saber a qué servidor conectarse según la longitud de la tupla

### 2. Almacenamiento en Memoria con Semáforos
- Cada servidor almacena tuplas en **memoria** (no en disco)
- Puedes usar estructuras de datos simples como `ArrayList` o `List`
- **Protección con Mutex**: Usa un `Semaphore` con 1 permiso (mutex) para proteger el acceso
- Todas las operaciones sobre el almacén deben:
  - Adquirir el mutex antes de acceder (`mutex.acquire()`)
  - Liberar el mutex después (`mutex.release()`)
  - Usar `try-finally` para asegurar la liberación
- Ejemplo:
  ```java
  Semaphore mutex = new Semaphore(1); // Mutex
  List<Tupla> almacen = new ArrayList<>();
  
  mutex.acquire(); // Entrar a sección crítica
  try {
      almacen.add(tupla);
  } finally {
      mutex.release(); // Salir de sección crítica
  }
  ```

### 3. Matching de Patrones
- Cuando un cliente busca con `["Alberto", "?X", "?Y"]`, el servidor debe:
  1. Buscar tuplas que empiecen con "Alberto"
  2. Capturar los valores de las posiciones 2 y 3 en variables X e Y
  3. Devolver la tupla completa o los valores capturados

### 4. Sincronización con Semáforos y Hilos
- Múltiples clientes pueden acceder simultáneamente
- Necesitas **sincronización** para evitar condiciones de carrera
- **Semáforos**: Usa `Semaphore` de Java para controlar acceso a recursos compartidos
- **Mutex**: Usa semáforos binarios (permisos = 1) para proteger secciones críticas
- **Hilos**: Cada cliente debe atenderse en un hilo separado (`Thread` o `ExecutorService`)
- **Secciones críticas**: Acceso al almacén de tuplas debe protegerse con mutex
- Ejemplo de uso:
  ```java
  Semaphore mutex = new Semaphore(1); // Mutex para sección crítica
  Semaphore semaforoLectores = new Semaphore(5); // Máximo 5 lectores simultáneos
  
  // En sección crítica:
  mutex.acquire(); // Entrar a sección crítica
  try {
      // Operación sobre almacén de tuplas
  } finally {
      mutex.release(); // Salir de sección crítica
  }
  ```

### 5. Servidor Réplica (Solo Servidor 1)
- El réplica debe estar sincronizado con el primario
- Si el primario cae, el réplica debe:
  - Detectar la caída
  - Asumir el rol de primario
  - Continuar atendiendo peticiones

## Ejemplo Práctico Completo

### Escenario: Sistema de Notas de Estudiantes

1. **Profesor publica nota**:
   ```
   Cliente Profesor → Servidor 1: PN(["Alberto", "20", "Suspenso"])
   Servidor 1: Guarda ["Alberto", "20", "Suspenso"]
   ```

2. **Estudiante consulta su nota**:
   ```
   Cliente Estudiante → Servidor 1: ReadN(["Alberto", "?Edad", "?Nota"])
   Servidor 1: Encuentra ["Alberto", "20", "Suspenso"]
   Servidor 1 → Cliente: "Edad=20, Nota=Suspenso"
   ```

3. **Estudiante elimina su registro** (después de verlo):
   ```
   Cliente Estudiante → Servidor 1: RN(["Alberto", "?X", "?Y"])
   Servidor 1: Encuentra, devuelve y elimina
   Servidor 1 → Cliente: "Tupla eliminada: ["Alberto", "20", "Suspenso"]"
   ```

## Puntos Clave a Recordar

✅ **Tuplas**: Secuencias de Strings (máx 6 elementos)  
✅ **Patrones**: Tuplas con variables `?A` a `?Z`  
✅ **3 Operaciones**: PN (añadir), ReadN (leer), RN (leer y eliminar)  
✅ **3 Servidores**: Distribuidos por longitud de tuplas  
✅ **Réplica**: Solo para el Servidor 1 (tuplas 1-3)  
✅ **Memoria**: Almacenamiento en RAM, no en disco  
✅ **Concurrencia**: Múltiples clientes simultáneos usando hilos  
✅ **Sincronización**: Semáforos y mutex para secciones críticas  
✅ **Java**: Todo implementado en Java con `Semaphore`, `Thread`, `ExecutorService`  

## Dificultades Comunes y Soluciones

### ¿Cómo saber a qué servidor conectarse?
- El cliente debe analizar la longitud de la tupla/patrón
- Si longitud ≤ 3 → Servidor 1
- Si longitud = 4 o 5 → Servidor 2
- Si longitud = 6 → Servidor 3

### ¿Cómo hacer matching de patrones?
- Comparar elemento por elemento
- Si el patrón tiene `?X`, acepta cualquier valor en esa posición
- Si el patrón tiene un valor concreto, debe coincidir exactamente

### ¿Cómo manejar la réplica?
- El réplica debe recibir todas las operaciones del primario
- Puedes usar un mecanismo de "heartbeat" para detectar caídas
- Cuando el primario cae, el réplica asume el control

