# Guía Rápida - Sistema LINDA

## Resumen Ejecutivo

### ¿Qué hay que hacer?
Crear un sistema distribuido donde múltiples clientes pueden compartir información mediante **tuplas** almacenadas en **3 servidores diferentes**.

### Conceptos en 30 segundos
- **Tupla**: Lista de Strings, ej: `["Juan", "25", "Aprobado"]`
- **Patrón**: Tupla con variables, ej: `["Juan", "?X", "?Y"]` (busca cualquier valor en X e Y)
- **3 Operaciones**:
  - `PN`: Guardar tupla
  - `ReadN`: Leer tupla (no la elimina)
  - `RN`: Leer y eliminar tupla
- **3 Servidores**: Cada uno maneja tuplas de diferente longitud (1-3, 4-5, 6)
- **Réplica**: El Servidor 1 tiene una copia de seguridad

---

## Flujo de Trabajo Simplificado

```
1. Cliente se conecta → Servidor correspondiente
2. Cliente envía operación → PN/ReadN/RN + tupla/patrón
3. Servidor procesa → Busca/Guarda/Elimina
4. Servidor responde → Resultado al cliente
5. Cliente se desconecta
```

---

## Ejemplo Práctico Paso a Paso

### Escenario: Sistema de Mensajería

**Paso 1: Cliente A publica mensaje**
```
Cliente A → Servidor 1: PN(["Usuario1", "Hola", "Mundo"])
Servidor 1: Guarda en memoria
Respuesta: "OK, tupla guardada"
```

**Paso 2: Cliente B busca mensajes de Usuario1**
```
Cliente B → Servidor 1: ReadN(["Usuario1", "?Mensaje1", "?Mensaje2"])
Servidor 1: Busca tupla que empiece con "Usuario1"
Servidor 1: Encuentra ["Usuario1", "Hola", "Mundo"]
Servidor 1 → Cliente B: "Mensaje1=Hola, Mensaje2=Mundo"
```

**Paso 3: Cliente B elimina el mensaje después de leerlo**
```
Cliente B → Servidor 1: RN(["Usuario1", "?X", "?Y"])
Servidor 1: Busca, encuentra y elimina
Servidor 1 → Cliente B: "Tupla eliminada: ["Usuario1", "Hola", "Mundo"]"
```

---

## Decisiones Técnicas Clave

### 1. ¿Cómo comunicarse?
**Opciones:**
- **Sockets TCP/IP**: Más control, más trabajo
- **RMI (Remote Method Invocation)**: Más fácil, más abstracto
- **Recomendación**: Sockets TCP/IP para aprender más

### 2. ¿Cómo almacenar tuplas?
**Requisito**: Debes usar **semáforos** (no `synchronized` ni `ConcurrentHashMap`)

**Solución:**
- `ArrayList<Tupla>` o `List<Tupla>`: Estructura simple
- **Proteger con Mutex**: `Semaphore mutex = new Semaphore(1)`
- Todas las operaciones adquieren/liberan el mutex
- **Recomendación**: `List<Tupla>` protegida con semáforo mutex

### 3. ¿Cómo hacer matching?
**Algoritmo:**
```
Para cada tupla en almacenamiento:
  Para cada posición i en patrón:
    Si patrón[i] es variable (?X):
      Continuar (acepta cualquier valor)
    Si patrón[i] == tupla[i]:
      Continuar (coincide)
    Si no:
      Esta tupla no coincide, probar siguiente
  Si llegamos aquí: ¡Coincidencia encontrada!
```

### 4. ¿Cómo manejar la réplica?
**Estrategia:**
1. Servidor primario envía todas las operaciones al réplica
2. Réplica mantiene copia sincronizada
3. Réplica verifica "heartbeat" del primario periódicamente
4. Si primario no responde: réplica asume control

---

## Estructura de Mensajes (Propuesta)

### Formato de Mensaje Cliente → Servidor
```
OPERACION|elemento1|elemento2|...|elementoN
```

**Ejemplos:**
```
CONECTAR
PN|Alberto|20|Suspenso
ReadN|Alberto|?X|?Y
RN|Alberto|?X|?Y
DESCONECTAR
```

### Formato de Respuesta Servidor → Cliente
```
OK|mensaje
ERROR|tipo_error|descripcion
RESULTADO|valor1|valor2|...|valorN
```

**Ejemplos:**
```
OK|Tupla guardada correctamente
OK|Tupla encontrada: Alberto|20|Suspenso
ERROR|TUPLA_NO_ENCONTRADA|No se encontró tupla que coincida
```

---

## Checklist Mínimo para Funcionar

### Servidor debe:
- [ ] Escuchar en un puerto
- [ ] Aceptar múltiples clientes (threads)
- [ ] Validar longitud de tuplas según servidor
- [ ] Almacenar tuplas en memoria
- [ ] Buscar tuplas por patrón
- [ ] Responder a cliente

### Cliente debe:
- [ ] Conectarse a servidor correcto (según longitud)
- [ ] Enviar operaciones en formato correcto
- [ ] Recibir y mostrar respuestas
- [ ] Manejar errores de conexión

### Sistema completo debe:
- [ ] 3 servidores funcionando (o simulados)
- [ ] Clientes pueden conectarse a cualquiera
- [ ] Operaciones PN, ReadN, RN funcionan
- [ ] Réplica del Servidor 1 funciona

---

## Errores Comunes a Evitar

❌ **No validar longitud de tuplas** → Servidor debe rechazar tuplas incorrectas  
❌ **Olvidar sincronización** → Múltiples clientes pueden causar problemas  
❌ **Matching incorrecto** → Variables deben aceptar cualquier valor  
❌ **No manejar desconexiones** → Servidor debe limpiar recursos  
❌ **Réplica no sincronizada** → Debe recibir todas las actualizaciones  

---

## Recursos Útiles

### Conceptos Java necesarios:
- **Sockets**: `ServerSocket`, `Socket`
- **Threads**: `Thread`, `Runnable`, `ExecutorService`
- **Semáforos**: `Semaphore`, `mutex.acquire()`, `mutex.release()`
- **Mutex**: `Semaphore(1)` para exclusión mutua
- **I/O**: `BufferedReader`, `PrintWriter`

### Importantes:
- **NO uses `synchronized`**: Debes usar `Semaphore`
- **NO uses `ConcurrentHashMap`**: Usa `List` normal con semáforo
- **Cada cliente en un hilo**: `Thread` o `ExecutorService`

### Estructura de proyecto mínima:
```
src/
├── tuplas/
│   ├── Tupla.java
│   └── Patron.java
├── servidor/
│   ├── ServidorLinda.java (interfaz/clase base)
│   ├── Servidor1.java
│   ├── Servidor2.java
│   ├── Servidor3.java
│   └── ReplicaServidor1.java
└── cliente/
    └── ClienteLinda.java
```

---

## Preguntas Frecuentes

### ¿Los servidores deben estar en máquinas diferentes?
**Respuesta**: Idealmente sí, pero para desarrollo puedes simularlo con diferentes puertos en la misma máquina.

### ¿Qué pasa si dos clientes buscan la misma tupla?
**Respuesta**: El primero que haga `RN` la elimina. El segundo debe esperar a que aparezca otra tupla que coincida.

### ¿Las variables pueden repetirse en un patrón?
**Respuesta**: Sí, pero deben tener el mismo valor. Ej: `["?X", "?X"]` busca dos elementos iguales.

### ¿Cómo probar sin tener 3 máquinas?
**Respuesta**: Ejecuta cada servidor en un puerto diferente (ej: 8001, 8002, 8003) en la misma máquina.

---

## Siguiente Paso

1. **Lee** `01-EXPLICACION-PRACTICA.md` para entender todo en detalle
2. **Revisa** `02-PASOS-A-SEGUIR.md` para el plan completo
3. **Consulta** `03-DIVISION-TRABAJO.md` para repartir el trabajo
4. **Empieza** con la implementación más simple (un servidor, un cliente)

¡Mucha suerte con la práctica! 🚀

