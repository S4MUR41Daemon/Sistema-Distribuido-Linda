# Guía de Semáforos, Hilos y Concurrencia para LINDA

## Introducción

Esta práctica requiere el uso de **semáforos** (incluyendo mutex) y **hilos** para manejar la concurrencia. Esta guía explica cómo implementarlo correctamente.

---

## 1. Conceptos Fundamentales

### Semáforos en Java

Un **semáforo** (`Semaphore`) controla el acceso a un recurso compartido mediante un contador de permisos.

```java
import java.util.concurrent.Semaphore;

// Crear semáforo con N permisos
Semaphore semaforo = new Semaphore(N);

// Adquirir permiso (bloquea si no hay permisos disponibles)
semaforo.acquire();

// Liberar permiso
semaforo.release();
```

### Mutex (Semáforo Binario)

Un **mutex** es un semáforo con **1 solo permiso**. Garantiza que solo un hilo puede acceder a la sección crítica a la vez.

```java
Semaphore mutex = new Semaphore(1); // Mutex: 1 permiso = exclusión mutua
```

### Hilos en Java

Un **hilo** (`Thread`) permite ejecutar código de forma concurrente.

```java
// Crear y ejecutar hilo
Thread hilo = new Thread(() -> {
    // Código que se ejecuta en el hilo
    System.out.println("Ejecutando en hilo");
});
hilo.start();
```

---

## 2. Arquitectura del Sistema con Hilos

### Servidor con Múltiples Hilos

Cada cliente debe atenderse en un **hilo separado**:

```java
public class ServidorLinda {
    private ServerSocket serverSocket;
    private Semaphore mutexAlmacen;
    private AlmacenTuplas almacen;
    
    public ServidorLinda(int puerto) {
        this.mutexAlmacen = new Semaphore(1); // Mutex para el almacén
        this.almacen = new AlmacenTuplas(mutexAlmacen);
        // ...
    }
    
    public void iniciar() {
        try {
            serverSocket = new ServerSocket(puerto);
            System.out.println("Servidor escuchando en puerto " + puerto);
            
            while (true) {
                Socket clienteSocket = serverSocket.accept();
                // Crear hilo para cada cliente
                Thread hiloCliente = new Thread(() -> {
                    atenderCliente(clienteSocket);
                });
                hiloCliente.start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    private void atenderCliente(Socket socket) {
        // Cada cliente se atiende en su propio hilo
        try {
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream())
            );
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true
            );
            
            String mensaje;
            while ((mensaje = in.readLine()) != null) {
                String respuesta = procesarMensaje(mensaje);
                out.println(respuesta);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### Alternativa con ExecutorService

Para mejor gestión de recursos, usa `ExecutorService`:

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ServidorLinda {
    private ExecutorService poolHilos;
    
    public ServidorLinda() {
        // Pool de 10 hilos máximo
        this.poolHilos = Executors.newFixedThreadPool(10);
    }
    
    public void iniciar() {
        while (true) {
            Socket clienteSocket = serverSocket.accept();
            // Enviar tarea al pool de hilos
            poolHilos.submit(() -> atenderCliente(clienteSocket));
        }
    }
}
```

---

## 3. Protección del Almacén con Mutex

### AlmacenTuplas con Semáforo

El almacén de tuplas es un **recurso compartido** que debe protegerse:

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Semaphore;

public class AlmacenTuplas {
    private List<Tupla> tuplas;
    private Semaphore mutex;
    
    public AlmacenTuplas(Semaphore mutex) {
        this.tuplas = new ArrayList<>();
        this.mutex = mutex;
    }
    
    public void agregarTupla(Tupla tupla) {
        try {
            mutex.acquire(); // Entrar a sección crítica
            tuplas.add(tupla);
            System.out.println("Tupla agregada. Total: " + tuplas.size());
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            mutex.release(); // Salir de sección crítica
        }
    }
    
    public Tupla buscarTupla(Patron patron) {
        try {
            mutex.acquire();
            for (Tupla tupla : tuplas) {
                if (coincide(patron, tupla)) {
                    return tupla; // NO elimina
                }
            }
            return null; // No encontrada
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        } finally {
            mutex.release();
        }
    }
    
    public Tupla eliminarTupla(Patron patron) {
        try {
            mutex.acquire();
            for (Tupla tupla : tuplas) {
                if (coincide(patron, tupla)) {
                    tuplas.remove(tupla);
                    return tupla; // Eliminada
                }
            }
            return null; // No encontrada
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        } finally {
            mutex.release();
        }
    }
    
    private boolean coincide(Patron patron, Tupla tupla) {
        // Lógica de matching
        // ...
    }
}
```

### Puntos Importantes

1. **Siempre usar `finally`**: Asegura que el mutex se libere incluso si hay excepción
2. **Manejar `InterruptedException`**: Interrumpir el hilo si es necesario
3. **Sección crítica mínima**: Mantén el código dentro del `try` lo más corto posible

---

## 4. Operaciones Bloqueantes (ReadN y RN)

### Problema: Bloqueo con Mutex

Si `ReadN` o `RN` no encuentran la tupla y deben **bloquear esperando**, no pueden mantener el mutex bloqueado (otros hilos no podrían agregar tuplas).

### Solución: Liberar Mutex Mientras Espera

```java
public Tupla readNote(Patron patron) {
    while (true) {
        try {
            mutex.acquire();
            Tupla encontrada = buscarEnAlmacen(patron);
            
            if (encontrada != null) {
                mutex.release();
                return encontrada; // Encontrada, devolver
            }
            
            // No encontrada: liberar mutex y esperar
            mutex.release();
            
            // Esperar un poco antes de volver a intentar
            Thread.sleep(100);
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return null;
        }
    }
}
```

### Solución Mejorada: Usar Wait/Notify

Para mejor eficiencia, usa `wait()` y `notify()`:

```java
public class AlmacenTuplas {
    private List<Tupla> tuplas;
    private Semaphore mutex;
    
    public Tupla readNote(Patron patron) {
        while (true) {
            try {
                mutex.acquire();
                Tupla encontrada = buscarEnAlmacen(patron);
                
                if (encontrada != null) {
                    mutex.release();
                    return encontrada;
                }
                
                // No encontrada: liberar mutex y esperar
                mutex.release();
                
                synchronized (this) {
                    wait(); // Esperar hasta que se agregue una tupla
                }
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return null;
            }
        }
    }
    
    public void agregarTupla(Tupla tupla) {
        try {
            mutex.acquire();
            tuplas.add(tupla);
            
            // Notificar a hilos que esperan
            synchronized (this) {
                notifyAll(); // Despertar a todos los hilos esperando
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            mutex.release();
        }
    }
}
```

---

## 5. Estructura Completa del Servidor

### Ejemplo Completo

```java
import java.io.*;
import java.net.*;
import java.util.concurrent.Semaphore;

public class Servidor1 {
    private ServerSocket serverSocket;
    private AlmacenTuplas almacen;
    private Semaphore mutex;
    
    public Servidor1(int puerto) {
        this.mutex = new Semaphore(1); // Mutex para exclusión mutua
        this.almacen = new AlmacenTuplas(mutex);
    }
    
    public void iniciar() {
        try {
            serverSocket = new ServerSocket(8001);
            System.out.println("Servidor 1 iniciado en puerto 8001");
            
            while (true) {
                Socket clienteSocket = serverSocket.accept();
                System.out.println("Cliente conectado: " + 
                    clienteSocket.getInetAddress());
                
                // Crear hilo para cada cliente
                Thread hiloCliente = new Thread(() -> {
                    atenderCliente(clienteSocket);
                });
                hiloCliente.start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    private void atenderCliente(Socket socket) {
        try {
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream())
            );
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true
            );
            
            String mensaje;
            while ((mensaje = in.readLine()) != null) {
                String respuesta = procesarMensaje(mensaje);
                out.println(respuesta);
                
                if (mensaje.equals("DESCONECTAR")) {
                    break;
                }
            }
            
            socket.close();
            System.out.println("Cliente desconectado");
            
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    
    private String procesarMensaje(String mensaje) {
        String[] partes = mensaje.split("\\|");
        String operacion = partes[0];
        
        switch (operacion) {
            case "PN":
                Tupla tupla = parsearTupla(partes);
                if (validarLongitud(tupla, 1, 3)) {
                    almacen.agregarTupla(tupla);
                    return "OK|Tupla guardada";
                } else {
                    return "ERROR|Longitud inválida";
                }
                
            case "ReadN":
                Patron patron = parsearPatron(partes);
                Tupla encontrada = almacen.readNote(patron);
                if (encontrada != null) {
                    return "OK|" + encontrada.toString();
                } else {
                    return "ESPERANDO";
                }
                
            case "RN":
                Patron patronRN = parsearPatron(partes);
                Tupla eliminada = almacen.removeNote(patronRN);
                if (eliminada != null) {
                    return "OK|" + eliminada.toString();
                } else {
                    return "ESPERANDO";
                }
                
            default:
                return "ERROR|Operación desconocida";
        }
    }
}
```

---

## 6. Buenas Prácticas

### ✅ Hacer

1. **Siempre usar `finally`** para liberar el mutex
2. **Manejar `InterruptedException`** correctamente
3. **Mantener secciones críticas cortas**
4. **Usar nombres descriptivos** para semáforos
5. **Documentar** qué protege cada semáforo

### ❌ Evitar

1. **No olvidar liberar** el mutex (causa deadlock)
2. **No mantener el mutex** mientras esperas (bloquea otros hilos)
3. **No usar `synchronized`** si el requisito es usar semáforos
4. **No crear demasiados hilos** (usa `ExecutorService` con límite)

---

## 7. Debugging de Problemas Comunes

### Deadlock (Interbloqueo)

**Síntoma**: El programa se queda colgado.

**Causa común**: Olvidar liberar el mutex.

**Solución**: Siempre usar `finally`:
```java
try {
    mutex.acquire();
    // código
} finally {
    mutex.release(); // SIEMPRE se ejecuta
}
```

### Race Condition (Condición de Carrera)

**Síntoma**: Resultados inconsistentes o impredecibles.

**Causa común**: Acceso al almacén sin mutex.

**Solución**: Proteger TODAS las operaciones sobre el almacén:
```java
// ❌ MAL
tuplas.add(tupla); // Sin protección

// ✅ BIEN
mutex.acquire();
try {
    tuplas.add(tupla);
} finally {
    mutex.release();
}
```

### Hilos que No Terminan

**Síntoma**: El programa no termina aunque cierres los clientes.

**Causa común**: Hilos esperando indefinidamente.

**Solución**: Usar `ExecutorService` con `shutdown()`:
```java
ExecutorService pool = Executors.newFixedThreadPool(10);
// ...
pool.shutdown(); // Cuando quieras terminar
```

---

## 8. Resumen de Conceptos Clave

| Concepto | Uso en LINDA |
|----------|-------------|
| **Semaphore** | Control de acceso a recursos |
| **Mutex (Semaphore(1))** | Proteger almacén de tuplas |
| **Thread** | Atender cada cliente en hilo separado |
| **ExecutorService** | Gestionar pool de hilos |
| **Sección crítica** | Acceso al almacén (agregar, buscar, eliminar) |
| **acquire()** | Entrar a sección crítica |
| **release()** | Salir de sección crítica |

---

## 9. Checklist de Implementación

- [ ] Crear `Semaphore mutex = new Semaphore(1)` en el servidor
- [ ] Pasar el mutex al `AlmacenTuplas`
- [ ] Proteger `agregarTupla()` con `mutex.acquire()`/`release()`
- [ ] Proteger `buscarTupla()` con mutex
- [ ] Proteger `eliminarTupla()` con mutex
- [ ] Crear un `Thread` por cada cliente conectado
- [ ] Usar `finally` para asegurar liberación del mutex
- [ ] Manejar `InterruptedException` correctamente
- [ ] Probar con múltiples clientes simultáneos

---

## 10. Ejemplo de Prueba

```java
// Crear servidor
Servidor1 servidor = new Servidor1(8001);
new Thread(() -> servidor.iniciar()).start();

// Crear múltiples clientes (simulados)
for (int i = 0; i < 5; i++) {
    final int id = i;
    new Thread(() -> {
        ClienteLinda cliente = new ClienteLinda();
        cliente.conectar("localhost", 8001);
        cliente.postNote(new Tupla("Cliente" + id, "Mensaje" + id));
    }).start();
}
```

Esto prueba que múltiples clientes pueden acceder simultáneamente sin problemas de concurrencia.

---

¡Con esta guía deberías poder implementar correctamente la sincronización con semáforos y hilos! 🚀

