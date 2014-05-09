#Grand Central Dispatch (GCD)

Este documento es una traducción libre y resumida de dos artículos/tutoriales publicados en [Ray Wenderlich](http://www.raywenderlich.com/) ([Parte 1](http://www.raywenderlich.com/60749/grand-central-dispatch-in-depth-part-1) y [Parte 2](http://www.raywenderlich.com/63338/grand-central-dispatch-in-depth-part-2)). Para obtener más información sobre el funcionamiento práctico de Grand Central Dispatch, por favor acudir a la fuente.

### ¿Qué es GCD?

GCD es el nombre comercial de _libdispatch_, la librería de Apple que permite ejecutar código de forma concurrente en hardware multicore iOS y OS X.

### Terminología GCD

**_Serial y Concurrente_**: Las tareas ejecutadas de forma _serializada_ son ejecutadas una cada vez, mientras que las tareas ejecutadas de forma _concurrente_ son ejecutadas al mismo tiempo.

**_Síncrono y Asíncrono_**: Una función _síncrona_ sólo devuelve su valor cuando ha completado completamente su tarea, por lo que el hilo de jecución se detendrá hasta que dicha función termine. Las funciones _asíncronas_ permitirán al hilo de ejecución continuar aunque no hayan terminado su tarea.

**_Sección Crítica (Critical Section)_**: Es una sección de código que no debe ejecutarse de forma concurrente, normalmente porque accede a algún recurso compartido que puede ser modificado en ese mismo instante, provocando un error o datos erróneos.

**_Condición de Carrera (Race Condition)_**: Esto ocurre cuando el comportamiento del programa depende de una secuencia de eventos que se ejecutan de forma no controlada. 

**_Bloque Mortal (Deadlock)_**: Cuando dos tareas, A y B, se están ejecutando de forma concurrente y A depende de B para continuar y viceversa, provocando la pausa de ambas tareas y del programa.

**_Thread Safe_**: El código _thread safe_ es aquel que puede ser llamado desde diversos hilos de ejecución sin problemas.

**_Context Switch_**: Se trata del proceso de almacenaje y recuperación del estado de ejecución cuando esta pasa de un hilo de ejecución a otro.

### Concurrencia y Paralelismo

Los dispositivos multicore ejecutan varios hilos al mismo tiempo usando _paralelismo_, sin embargo, para que un dispositivo de un sólo core consiga esto, debe ejecutar un hilo, realizar un _context switch_ y entonces ejecutar otro hilo o proceso. Esta imagen ilustra estos dos comportamientos.

![](http://img.readitlater.com/i/cdn1.raywenderlich.com/wp-content/uploads/2014/01/Concurrency_vs_Parallelism/RS/w640.png)

Paralelismo necesita concurrencia, pero la concurrencia no garantiza el paralelismo.

### Colas

GCD proporciona _dispatch queues_ (colas) para manejar bloques de código. Estas colas manejan las tareas que se le dan a GCD y las ejecutan en orden FIFO (_First In, First Out_). Todas las _dispatch queues_ son _thread safe_.

##### Colas en serie (Serial Queues)

Las tareas en una _serial queue_ se ejecutan de una en una y en orden. Además, es imposible saber el tiempo que transcurrirá entre que un bloque de código termine de ejecutarse y el siguiente bloque comience su ejecución.

![](http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/Serial-Queue-480x272.png)

GCD es quien controla los tiempos de ejecución de estas tareas. Lo único garantizado es que las tareas se ejecutarán en el orden en el que fueron añadidas a la cola.

##### Colas concurrentes (Concurrent Queues)

Las tareas de las _concurrent queues_ también se ejecutarán en el mismo orden en el que fueron añadidas a la cola, pero dentro de esta el orden de ejecución de los bloques y el número de estos que se ejecutarán concurrentemente queda bajo el control de GCD.

![image](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/Concurrent-Queue-480x272.png)

##### Tipos de cola

**_Main Queue_**: Es una _serial queue_ en la que deben ejecutarse todas las tareas relacionadas con la interacción con la interfaz del usuario.

**_Global Dispatch Queues_**: Se tratan de cuatro _concurrent queues_ definidas por la prioridad de las tareas que ejecutan, _background_, _low_, _default_ y _high_.

**_Colas propias_**: Es posible crear nuestras propias _serial queues_ y _concurrent queues_ si así lo necesitamos.

### dispatch_sync

_dispatch_sync_ permite enviar de forma asíncrona un bloque de código a la cola que queramos.

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
	// Aquí va el código que deseamos ejecutar
});
```
En este ejemplo se ha usado una _global dispath queue_, concretamente la de alta prioridad, _high_. Existen las siguientes:

+ DISPATCH_QUEUE_PRIORITY_DEFAULT
+ DISPATCH_QUEUE_PRIORITY_HIGH
+ DISPATCH_QUEUE_PRIORITY_LOW
+ DISPATCH_QUEUE_PRIORITY_BACKGROUND

### dispatch_after

_dispatch_after_ permite retrasar la ejecución de un bloque de código durante una cantidad de tiempo determinada

```
double delayInSeconds = 1.0;
dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC));
dispatch_after(popTime, dispatch_get_main_queue(), ^(void){
	// Aquí va el código que deseamos ejecutar con retraso
});
```
En este ejemplo se puede ver el uso de `dispatch_get_main_queue()` para obtener la _main queue_.

### Hacer que nuestros singletons sean thread-safe

```
+ (instancetype)sharedManager
{
    static SingletonClass *singletonInstance = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        singletonInstance = [[SingletonClass alloc] init];
        // Realizar inicializaciones adicionales si fuera necesario
    });
    return singletonInstance;
}
```

### Ejemplo visual de dispatch_sync


![image](http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/dispatch_sync_in_action.gif)


### Ejemplo visual de dispatch_async

![image](http://cdn1.raywenderlich.com/wp-content/uploads/2014/01/dispatch_async_in_action.gif)


### Grupos de dispatch

Es útil utilizar _grupos de dispatch_ cuando queremos que se ejecute una tarea al finalizar otro grupo de tareas pero que no sabemos cuándo terminarán.

```
// Creamos un grupo de dispatch
dispatch_group_t dispatchGroup = dispatch_group_create();
(...)
dispatch_group_enter(dispatchGroup);
// A partir de aquí, el código forma parte del grupo de dispatch como un elemento
(...)
dispatch_group_leave(dispatchGroup);
// A partir de aquí el código deja de ser un elemento del grupo de dispatch
(...)
dispatch_group_notify(dispatchGroup, dispatch_get_main_queue(), ^{
	// Código que deseamos ejecutar cuando no queden elementos en el grupo
});
```

