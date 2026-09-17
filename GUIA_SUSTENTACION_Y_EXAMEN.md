# 📚 GUÍA MAESTRA DE PREPARACIÓN: SUSTENTACIÓN ORAL Y PRUEBA ESCRITA
**Proyecto:** Oasis Guayaquil Spa — Sistema de Gestión y Reserva de Citas  
**Stack Tecnológico:** React 18 + Vite (Frontend) | Node.js + Express (Backend) | SQLite3 (Base de Datos)

---

## 🎯 OBJETIVO DE ESTA GUÍA
Esta guía te proporciona todo el dominio conceptual, técnico y práctico necesario para:
1. **Sustentación Oral Individual:** Explicar con fluidez, seguridad y sin titubeos cualquier línea o bloque de código que el docente seleccione al azar, describiendo con precisión el flujo de datos de extremo a extremo (*End-to-End*).
2. **Prueba Escrita Teórica/Práctica:** Responder preguntas de examen sobre la arquitectura cliente-servidor, ciclo de vida de React, asincronía en Node.js, middlewares, protocolo HTTP/REST y bases de datos relacionales, además de resolver ejercicios de código en papel.

---

## 🧭 TABLA DE CONTENIDOS
1. [Arquitectura General del Sistema](#1-arquitectura-general-del-sistema)
2. [Flujo de Datos de Extremo a Extremo (End-to-End)](#2-flujo-de-datos-de-extremo-a-extremo-end-to-end)
3. [Explicación Bloque por Bloque del Código (Defensa Oral)](#3-explicación-bloque-por-bloque-del-código-defensa-oral)
   - [3.1 Backend: Servidor, Rutas, Controlador, Modelo y Base de Datos](#31-backend)
   - [3.2 Frontend: App, Formularios, Listado, Servicios y Animaciones](#32-frontend)
4. [Balotario Conceptual para la Prueba Escrita](#4-balotario-conceptual-para-la-prueba-escrita)
5. [Resolución de Problemas Prácticos (Ejercicios Típicos de Examen)](#5-resolución-de-problemas-prácticos-ejercicios-típicos-de-examen)
6. [Simulador de Preguntas Trampa del Docente y Respuestas Modelo](#6-simulador-de-preguntas-trampa-del-docente-y-respuestas-modelo)

---

# 1. ARQUITECTURA GENERAL DEL SISTEMA

El sistema opera bajo el modelo **Cliente-Servidor desacoplado** utilizando una **API REST**:

```
+-----------------------------------------------------------------------------------+
|                              FRONTEND (React + Vite)                              |
|  Puerto: 5173                                                                     |
|  - Componentes: App.jsx, ReservationForm.jsx, ReservationList.jsx, Header.jsx     |
|  - Estado y Reactividad: useState, useEffect, useMemo, Framer Motion              |
|  - Cliente HTTP: fetch() en services/api.js                                       |
+------------------------------------------+----------------------------------------+
                                           |
                              Peticiones HTTP (JSON)
                       [GET, POST, PUT, DELETE] /api/citas
                                           |
                                           v
+-----------------------------------------------------------------------------------+
|                              BACKEND (Node.js + Express)                          |
|  Puerto: 5000                                                                     |
|  - app.js (CORS, Express JSON, Logger Visual ANSI, Error Handler Centralizado)     |
|  - Rutas: routes/cita.routes.js (Mapeo de endpoints)                              |
|  - Controlador: controllers/cita.controller.js (Validación de negocio, HTTP codes)|
|  - Modelo: models/cita.model.js (Lógica de colisiones, SQL con Promesas)          |
+------------------------------------------+----------------------------------------+
                                           |
                         Consultas SQL Parametrizadas
                                           |
                                           v
+-----------------------------------------------------------------------------------+
|                           BASE DE DATOS (SQLite3)                                 |
|  Archivo físico: backend/src/db/spa.db                                            |
|  - Tabla: citas (id, nombre_cliente, telefono, servicio, fecha, hora, ...)        |
|  - Migración automática: PRAGMA table_info + ALTER TABLE ADD COLUMN terapeuta     |
+-----------------------------------------------------------------------------------+
```

---

# 2. FLUJO DE DATOS DE EXTREMO A EXTREMO (END-TO-END)

Si el profesor te dice: *"Explícame qué pasa exactamente desde que el usuario hace click en Guardar Cita hasta que se refleja en la pantalla"*, esta es la respuesta perfecta:

### Paso 1: Captura en el Frontend (`ReservationForm.jsx`)
1. El usuario llena los inputs del formulario (controlados por el hook `useState(formData)`).
2. Al presionar el botón **"Confirmar Reserva"**, se dispara el evento `onSubmit` del formulario, activando la función `handleSubmit(e)`.
3. `e.preventDefault()` cancela la recarga natural del navegador.
4. **Validación en el cliente:**
   - Se limpian espacios con `.trim()`.
   - Se comprueba la expresión regular del teléfono: `/^09\d{8}$/` (10 dígitos, iniciando con `09`).
   - Se valida que la fecha no sea anterior a hoy.
5. Si pasa la validación, invoca `onSubmit(formData)` que fue pasada como prop desde `App.jsx`.

### Paso 2: Envío Asíncrono (`api.js`)
6. En `App.jsx`, la función `handleCreateCita` activa el estado `isSubmitting = true` (para mostrar animación de carga o deshabilitar botones).
7. Invoca `createCita(formData)` ubicada en `src/services/api.js`.
8. `fetch('http://localhost:5000/api/citas', { method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(citaData) })` convierte el objeto JavaScript a un string en formato JSON y lo envía por la red al puerto 5000.

### Paso 3: Recepción y Procesamiento en el Backend (`app.js`, `cita.routes.js`, `cita.controller.js`)
9. **`app.js`**:
   - El middleware `cors()` permite la petición proveniente de `localhost:5173`.
   - El middleware `express.json()` analiza el body de la petición HTTP y lo convierte en el objeto `req.body`.
   - El logger personalizado registra en consola: `[HTTP] Method: POST | Route: /api/citas`.
10. **`cita.routes.js`**: Enruta el método `POST /` hacia `citaController.crear`.
11. **`cita.controller.js`**:
    - **Validación defensiva de campos obligatorios:** Comprueba que no vengan nulos.
    - **Validación de formato de teléfono ecuatoriano:** Si falla, responde con `400 Bad Request`.
    - **Validación de Disponibilidad (Lógica de Conflicto):** Invoca a `CitaModel.checkAvailability(...)`.

### Paso 4: Capa de Datos y Persistencia (`cita.model.js` y `database.js`)
12. En `CitaModel.checkAvailability`:
    - Ejecuta un `SELECT * FROM citas WHERE fecha_cita = ? AND estado != 'Cancelada'`.
    - Convierte la hora solicitada a minutos desde medianoche: `HH:MM -> minutos`.
    - Suma la duración del tratamiento (ej: Hidroterapia = 45 min).
    - Evalúa si el terapeuta asignado (ej: Mario) ya tiene una cita cuyo intervalo se solapa:
      $$\text{Solapamiento} = \max(\text{inicio}_1, \text{inicio}_2) < \min(\text{fin}_1, \text{fin}_2)$$
    - Si hay solapamiento con el mismo terapeuta, el controlador retorna `HTTP 409 Conflict` con un mensaje explicativo y los horarios ocupados.
13. Si no hay colisión, `CitaModel.create` genera un identificador único global `uuidv4()` y ejecuta un `INSERT INTO citas (...) VALUES (?, ?, ...)` usando **consultas parametrizadas** (evitando inyecciones SQL).
14. La base de datos SQLite guarda el registro en el disco local (`spa.db`).

### Paso 5: Respuesta y Actualización Reactiva (`App.jsx`)
15. El controlador responde con código `HTTP 201 Created`:
    `{ message: 'Cita creada exitosamente', data: { id, nombre_cliente, ... } }`.
16. El middleware de logs en `app.js` imprime en verde: `Status: 201`.
17. En `api.js`, la promesa de `fetch` se resuelve, `response.json()` entrega la nueva cita al `App.jsx`.
18. `App.jsx` ejecuta `setCitas(prev => [...prev, nuevaCita.data])`, actualizando el estado de React.
19. Se dispara un Toast visual: *"Cita reservada con éxito"*.
20. El hook `useMemo` recalcula automáticamente las métricas (ingresos totales, citas activas, servicio top) y la lista filtrada.
21. React reconcilia el **Virtual DOM** y `ReservationList.jsx` renderiza la nueva tarjeta con una animación suave de entrada de Framer Motion.

---

# 3. EXPLICACIÓN BLOQUE POR BLOQUE DEL CÓDIGO (DEFENSA ORAL)

## 3.1 BACKEND

### Archivo: `backend/app.js`
* **Líneas 1-3:** Importación de módulos principales: `express` (framework web), `cors` (permite comunicación entre diferentes puertos/orígenes) y el enrutador modular `citaRoutes`.
* **Línea 6:** `const PORT = process.env.PORT || 5000;`  
  *¿Por qué?* Permite que el puerto sea configurable por variables de entorno en producción o use el 5000 por defecto.
* **Líneas 9-10:** 
  - `app.use(cors())`: Evita el bloqueo del navegador por política de mismo origen (*Same-Origin Policy*).
  - `app.use(express.json())`: Middleware que parsea el cuerpo de peticiones entrantes con encabezado `Content-Type: application/json`.
* **Líneas 13-37: Middleware de Auditoría y Logs Visuales:**
  - Registra fecha/hora formateada, método HTTP y ruta.
  - Escucha el evento `res.on('finish')` para obtener el `statusCode` una vez que la respuesta ya fue emitida.
  - Aplica códigos de escape ANSI (`\x1b[32m`, etc.) para dar color en la consola (Verde = GET/200, Amarillo = POST/400, Rojo = 500/DELETE).
* **Línea 40:** `app.use('/api/citas', citaRoutes);`  
  Prefijo global de ruta. Toda petición que empiece con `/api/citas` será delegada al router.
* **Líneas 42-45: Manejo de 404:** Si ninguna ruta coincidió, responde `{ error: 'Ruta no encontrada' }` con estado 404.
* **Líneas 48-54: Middleware Global de Errores (Anti-Crash 500):**
  - Posee 4 parámetros: `(err, req, res, next)`. Express lo reconoce exclusivamente como manejador de errores por tener 4 argumentos.
  - Captura excepciones asíncronas no controladas enviadas con `next(error)`, evitando que el proceso de Node.js se caiga (crash) y devolviendo JSON estructurado.

---

### Archivo: `backend/src/db/database.js`
* **Líneas 1-4:** Uso del paquete `sqlite3.verbose()` y resolución de la ruta del archivo con `path.resolve(__dirname, 'spa.db')`.
* **Líneas 6-25: Conexión y Creación de Tabla:**
  - `new sqlite3.Database(...)`: Abre la conexión o crea el archivo físico si no existe.
  - `CREATE TABLE IF NOT EXISTS citas (...)`: Define el esquema relacional con clave primaria `id TEXT PRIMARY KEY`, tipos `TEXT`, `REAL` para el precio y valores por defecto (`estado DEFAULT 'Pendiente'`).
* **Líneas 27-42: Migración en Caliente (Hot Migration):**
  - Consulta `PRAGMA table_info(citas)` para inspeccionar las columnas de la tabla existente.
  - Si la columna `terapeuta` no existe (bases de datos creadas en versiones previas), ejecuta `ALTER TABLE citas ADD COLUMN terapeuta TEXT DEFAULT 'Mario'` dinámicamente sin perder los registros previos.

---

### Archivo: `backend/src/routes/cita.routes.js`
* **Línea 2:** `const router = express.Router();` Crea un enrutador aislado y modular.
* **Líneas 6-16: Mapeo REST Semántico:**
  - `GET /` $\rightarrow$ `listarTodas`
  - `POST /` $\rightarrow$ `crear`
  - `PUT /:id` y `PATCH /:id` $\rightarrow$ `actualizar` (ambos métodos soportados para actualización completa o parcial).
  - `DELETE /:id` $\rightarrow$ `eliminar` (utiliza parámetro de ruta `:id`).

---

### Archivo: `backend/src/controllers/cita.controller.js`
* **Líneas 4-7: Función `isValidPhone`:** Valida que el teléfono tenga exactamente 10 dígitos numéricos mediante regex (`/^[0-9]{10}$/`).
* **Método `listarTodas` (Líneas 10-18):**
  - Extrae `req.query.fecha` (permite filtrar citas por día vía query string: `?fecha=2026-09-18`).
  - Llama a `CitaModel.findAll(fecha)` y retorna `200 OK` con el arreglo de citas.
* **Método `crear` (Líneas 20-58):**
  - Sanitiza con `.trim()`.
  - Valida obligatoriedad de campos $\rightarrow$ `400 Bad Request`.
  - Valida formato celular de Ecuador (`/^09\d{8}$/`) $\rightarrow$ `400 Bad Request`.
  - Comprueba disponibilidad horaria llamando a `CitaModel.checkAvailability`. Si hay solape, responde con código **`409 Conflict`**, informando al usuario el rango ocupado y sugiriendo cambiar terapeuta u hora.
  - Si todo es correcto, persiste en la base y responde **`201 Created`**.
* **Método `actualizar` (Líneas 60-115):**
  - Busca primero si la cita existe con `CitaModel.findById(id)`. Si no, `404 Not Found`.
  - Fusiona los datos viejos con los nuevos (`{ ...citaExistente, ...req.body }`).
  - Si se modifican hora, fecha, terapeuta o estado, vuelve a comprobar colisiones pasando `idToIgnore: id` para no detectar como conflicto la propia cita que se está editando.
* **Método `eliminar` (Líneas 117-131):**
  - Valida existencia previa (`404`) y ejecuta el borrado devolviendo `200 OK`.

---

### Archivo: `backend/src/models/cita.model.js`
* **Funciones Auxiliares:**
  - `getServiceDuration(servicio)`: Extrae con regex los minutos contenidos en el nombre (ej. `"Hidroterapia (45 min)"` $\rightarrow$ `45`).
  - `timeToMinutes(timeStr)`: Convierte `"HH:MM"` a minutos enteros desde las 00:00 (ej: `"09:30"` $\rightarrow$ $9 \times 60 + 30 = 570$). Facilita comparaciones matemáticas de intervalos.
  - `minutesToTime(min)`: Convierte de nuevo de minutos a formato `"HH:MM"`.
* **Algoritmo de Colisión (`checkAvailability`, Líneas 53-105):**
  - Consulta citas del día excluyendo las canceladas.
  - Si el terapeuta es diferente, no hay colisión (permite atención simultánea en cabinas distintas).
  - Si es el mismo terapeuta, calcula `inicioNuevo`, `finNuevo`, `inicioExistente`, `finExistente`.
  - Fórmula de intersección de intervalos:
    `Math.max(inicioExistente, inicioNuevo) < Math.min(finExistente, finNuevo)`
  - Si es verdadero, hay traslape y retorna los detalles del conflicto.
* **Consultas Parametrizadas (Seguridad):**
  En `create`, `update`, `delete`: se usa `db.run('... VALUES (?, ?, ?)', [val1, val2, val3])`.  
  *Defensa Oral:* "Usamos marcadores de posición `?` para que el driver de SQLite trate las entradas estrictamente como datos literales y nunca como código SQL ejecutable, anulando cualquier intento de **Inyección SQL**".

---

## 3.2 FRONTEND

### Archivo: `frontend/src/App.jsx`
* **Líneas 24-44: Gestión del Estado con `useState`:**
  - `citas`: Arreglo principal de reservas.
  - `isLoading`: Estado booleano para feedback de carga.
  - `dateFilter`: Filtro rápido ('all' vs fecha de hoy).
  - `toasts`: Arreglo para notificaciones flotantes temporales.
* **Líneas 52-67: `useEffect` para carga de datos:**
  ```javascript
  useEffect(() => {
    fetchCitas();
  }, [dateFilter]);
  ```
  *Defensa Oral:* "El hook `useEffect` sincroniza el componente con el backend. Su arreglo de dependencias contiene `[dateFilter]`, lo que significa que cada vez que el usuario alterne entre 'Hoy' y 'Todas', React disparará una nueva consulta al servidor".
* **Líneas 123-130: Optimización con `useMemo` (Filtrado de Citas):**
  - Memoriza el resultado del filtrado por texto (`searchTerm`) y categoría (`selectedCategory`).
  - Solo se recalcula cuando cambian `citas`, `searchTerm` o `selectedCategory`, evitando filtros innecesarios en re-renders ajenos.
* **Líneas 132-160: `useMemo` (Cálculo de Métricas del Dashboard):**
  - Recorre las citas activas para calcular: Ingreso total estimado, citas confirmadas y cuál es el servicio más demandado (*Top Servicio*).
* **Líneas 84-96: Actualización Optimista (`handleUpdateStatus`):**
  - Cambia el estado localmente de inmediato en la interfaz para máxima velocidad percibida (`setCitas(...)`).
  - Luego envía la petición al backend (`updateCita`).
  - Si el backend responde con error, revierte los cambios llamando a `fetchCitas()` y muestra un toast de error.

---

### Archivo: `frontend/src/components/ReservationForm.jsx`
* **Componente Controlado (*Controlled Component*):**
  - Cada input tiene su propiedad `value={formData.campo}` y su manejador `onChange={handleChange}`.
  - React es la "única fuente de la verdad" (*Single Source of Truth*) del formulario.
* **Cálculo de Fin de Sesión en Tiempo Real (`calculateEndTime`):**
  - Cuando el usuario escoge la hora (ej: 10:00) y el servicio ("Limpieza Facial (60 min)"), la interfaz calcula e imprime dinámicamente: *"Sesión: 10:00 a 11:00 (Duración: 60 min)"*.
* **Manejo de Errores y Conflictos 409:**
  - Si el backend rechaza la cita por solapamiento de agenda del terapeuta, el bloque `catch (err)` activa `setIsConflict(true)` y muestra una alerta visual destacada recomendando cambiar de terapeuta o de hora.

---

### Archivo: `frontend/src/components/ReservationList.jsx`
* **Props recibidas:** `citas`, `onUpdateStatus`, `onDeleteRequest`, `isLoading`, `processingId`.
* **Cálculo de Iniciales (`getInitials`):** Extrae las dos primeras letras del nombre del cliente para crear un avatar minimalista.
* **Integración con WhatsApp:**
  - Función `handleWhatsApp(cita)`:
    - Remueve el '0' inicial del número y le antepone el prefijo internacional de Ecuador (`593`).
    - Compone un mensaje personalizado con el nombre del cliente, tratamiento, terapeuta asignado y rango de horas.
    - Utiliza `encodeURIComponent()` para escapar caracteres especiales y abre la ventana mediante `window.open('https://api.whatsapp.com/send?...')`.
* **Animaciones con Framer Motion:**
  - Utiliza `<AnimatePresence>` y `<motion.div layout>`: Si se elimina o cancela una cita, las demás tarjetas se reacomodan suavemente con física de resortes (*springs*).

---

# 4. BALOTARIO CONCEPTUAL PARA LA PRUEBA ESCRITA

### 1. ¿Qué es React y qué problema resuelve el Virtual DOM?
* **Respuesta:** React es una biblioteca declarativa de JavaScript para construir interfaces de usuario basada en componentes.  
* El **Virtual DOM** es una representación virtual ligera del DOM real en memoria. Cuando el estado de un componente cambia, React genera un nuevo Virtual DOM, lo compara con el anterior mediante un algoritmo de diferenciación (*Diffing Algorithm*) y aplica en el DOM real únicamente las diferencias mínimas indispensables (*Reconciliation*), evitando costosos redibujados de la página completa.

### 2. ¿Qué es el flujo de datos unidireccional (*One-way Data Flow*) en React?
* **Respuesta:** En React, los datos fluyen exclusivamente de arriba hacia abajo (de componentes padres a componentes hijos) a través de **`props`**. Los componentes hijos no pueden modificar directamente las props de sus padres; para comunicarse hacia arriba, el padre debe pasar una función callback que el hijo ejecuta (concepto conocido como *Lifting State Up*).

### 3. Diferencia fundamental entre `useState`, `useEffect` y `useMemo`
| Hook | Propósito principal | ¿Cuándo se ejecuta? |
| :--- | :--- | :--- |
| **`useState`** | Declara y actualiza variables de estado reactivas en el componente. | Provoca un re-render cada vez que se llama a su función actualizadora. |
| **`useEffect`** | Ejecuta efectos secundarios (peticiones HTTP, suscripciones, timers). | Después del renderizado, según cambien las variables de su arreglo de dependencias `[]`. |
| **`useMemo`** | Memoriza el resultado de un cálculo pesado para no recalcularlo en cada render. | Durante el renderizado, únicamente cuando alguna dependencia del arreglo cambia. |

### 4. ¿Qué es Node.js y cómo maneja la concurrencia si es "Single Threaded"?
* **Respuesta:** Node.js es un entorno de ejecución (*runtime*) para JavaScript construido sobre el motor V8 de Google Chrome.  
* Aunque su hilo principal es de un solo subproceso (*Single Threaded*), maneja miles de conexiones concurrentes gracias a su arquitectura orientada a eventos no bloqueante (**Event Loop**) respaldada por la biblioteca **libuv**, la cual delega operaciones pesadas de Entrada/Salida (I/O) como lectura de disco o consultas a base de datos al pool de hilos del sistema operativo.

### 5. ¿Qué es un Middleware en Express y cuál es el rol de `next()`?
* **Respuesta:** Un middleware es una función intermedia que tiene acceso al objeto de solicitud (`req`), al objeto de respuesta (`res`) y a la siguiente función middleware en el ciclo de solicitud-respuesta de la aplicación (`next`).
* Si el middleware actual no finaliza el ciclo emitiendo una respuesta (ej. `res.json()`), está obligado a invocar **`next()`** para ceder el control al siguiente middleware en la cadena; de lo contrario, la petición se queda "colgada" indefinidamente.

### 6. ¿Qué es CORS y por qué ocurre?
* **Respuesta:** **CORS** (*Cross-Origin Resource Sharing*) es un mecanismo de seguridad implementado por los navegadores web que restringe peticiones HTTP originadas desde un dominio, protocolo o puerto distinto al del servidor que sirve los recursos.  
* En nuestro spa, el frontend corre en el puerto `5173` y el backend en el `5000`. Sin el paquete `cors()` en Express, el navegador bloquearía cualquier intento del frontend de comunicarse con la API.

### 7. Códigos de Estado HTTP fundamentales (*HTTP Status Codes*)
* **`200 OK`**: Petición exitosa estándar (ej. listar o actualizar citas).
* **`201 Created`**: Recurso creado exitosamente (ej. tras un `POST` de nueva cita).
* **`400 Bad Request`**: Datos inválidos enviados por el cliente (ej. faltan campos o teléfono con formato erróneo).
* **`404 Not Found`**: El recurso o endpoint solicitado no existe.
* **`409 Conflict`**: Conflicto con el estado actual del servidor (ej. choque de horarios/solapamiento con el mismo terapeuta).
* **`500 Internal Server Error`**: Error inesperado no controlado en el backend.

---

# 5. RESOLUCIÓN DE PROBLEMAS PRÁCTICOS (EJERCICIOS TÍPICOS DE EXAMEN)

### Ejercicio 1: Algoritmo de Detección de Colisión de Horarios
**Problema:** Escribe una función pura en JavaScript que reciba dos intervalos de tiempo expresados en minutos y determine si se solapan.
```javascript
// Solución Matemática Estándar:
// Dos intervalos [A_inicio, A_fin] y [B_inicio, B_fin] se intersecan SI Y SOLO SI:
// max(A_inicio, B_inicio) < min(A_fin, B_fin)

function haySolapamiento(inicio1, fin1, inicio2, fin2) {
  return Math.max(inicio1, inicio2) < Math.min(fin1, fin2);
}

// Ejemplo de prueba:
// Cita 1: 09:00 a 09:45 (540 min a 585 min)
// Cita 2: 09:30 a 10:15 (570 min a 615 min)
// max(540, 570) = 570
// min(585, 615) = 585
// ¿570 < 585? -> TRUE (Hay conflicto)
```

---

### Ejercicio 2: Detección y Corrección de SQL Injection
**Código Vulnerable:**
```javascript
// PELIGROSO: Concatenación directa de strings
const query = "SELECT * FROM citas WHERE id = '" + req.params.id + "'";
db.all(query, (err, rows) => { ... });
```
* **Explicación del riesgo:** Un atacante podría enviar como ID: `' OR '1'='1`, transformando la consulta en `SELECT * FROM citas WHERE id = '' OR '1'='1'`, extrayendo o borrando todos los registros de la base.
* **Solución Segura (Consultas Preparadas / Parametrizadas):**
```javascript
// SEGURO: Uso de Placeholders (?)
const query = "SELECT * FROM citas WHERE id = ?";
db.get(query, [req.params.id], (err, row) => {
  if (err) return next(err);
  res.json(row);
});
```

---

### Ejercicio 3: Escribir un Middleware de Express para Validar Datos
**Problema:** Diseña un middleware llamado `validarCitaBody` que asegure que el campo `nombre_cliente` y `telefono` no vengan vacíos.
```javascript
const validarCitaBody = (req, res, next) => {
  const { nombre_cliente, telefono } = req.body;
  
  if (!nombre_cliente || typeof nombre_cliente !== 'string' || !nombre_cliente.trim()) {
    return res.status(400).json({ error: "El nombre del cliente es obligatorio." });
  }
  
  if (!telefono || !/^09\d{8}$/.test(telefono.trim())) {
    return res.status(400).json({ error: "El teléfono debe ser ecuatoriano (10 dígitos iniciando con 09)." });
  }
  
  // Si todo está correcto, permite el paso al siguiente middleware o controlador:
  next();
};

module.exports = validarCitaBody;
```

---

### Ejercicio 4: ¿Por qué este código en React imprime el valor anterior?
**Código con error de novato:**
```javascript
const [contador, setContador] = useState(0);

const handleClick = () => {
  setContador(contador + 1);
  console.log(contador); // <-- ¿Por qué imprime 0 y no 1?
};
```
* **Explicación para examen:** La función `setContador` de React es **asíncrona** y no muta la variable inmediatamente en la línea actual; programa una actualización de estado para el siguiente ciclo de render. Para observar el valor actualizado se debe leer en el siguiente renderizado o mediante un `useEffect(() => { console.log(contador); }, [contador])`.

---

# 6. SIMULADOR DE PREGUNTAS TRAMPA DEL DOCENTE Y RESPUESTAS MODELO

### Pregunta 1: *"¿Por qué usaron SQLite y no una base de datos como MySQL o MongoDB?"*
* **Respuesta Modelo:**  
  *"SQLite es una base de datos relacional serverless, autocontenida y embebida en un solo archivo físico (`spa.db`), lo que garantiza portabilidad absoluta y cero latencia de red entre el backend y la base de datos para esta escala de aplicación. Soporta el estándar SQL completo, integridad referencial y transacciones ACID. Para producción a escala masiva con alta concurrencia de escrituras distribuidas migraríamos a PostgreSQL, pero la arquitectura en capas que implementamos desacopla el Modelo del Controlador, permitiendo cambiar el motor de persistencia sin alterar una sola línea de lógica de negocio en las rutas ni en el frontend."*

---

### Pregunta 2: *"¿Dónde está la View (Vista) en tu backend si dices que aplicas MVC?"*
* **Respuesta Modelo:**  
  *"En una arquitectura desacoplada moderna basada en APIs RESTful, el backend no genera vistas HTML en el servidor (como lo harían motores de plantillas clásicos tipo EJS o Blade). El backend actúa estrictamente como un proveedor de servicios de datos; la **Vista** se encuentra completamente del lado del cliente, delegada a los componentes declarativos de **React**, los cuales consumen las respuestas en formato JSON generadas por el Controlador."*

---

### Pregunta 3: *"Si dos personas intentan reservar con Mario a la misma hora, ¿cómo garantiza el sistema que no se crucen?"*
* **Respuesta Modelo:**  
  *"El backend ejecuta una validación de disponibilidad antes de cualquier inserción mediante `CitaModel.checkAvailability`. Compara las reservas existentes para la fecha seleccionada con el terapeuta indicado, transformando los horarios a minutos y sumando la duración en minutos de cada servicio. Si se cumple la condición matemática de solapamiento de intervalos `max(inicio1, inicio2) < min(fin1, fin2)`, el servidor bloquea la operación y responde con código HTTP `409 Conflict`. No obstante, si el cliente asigna la cita a otro terapeuta disponible (como Juan, Ana o Sofía), el sistema sí permite la reserva simultánea porque cada terapeuta atiende en una cabina independiente."*

---

### Pregunta 4: *"¿Por qué el listado de citas no usa `window.location.reload()` para actualizarse tras agregar una cita?"*
* **Respuesta Modelo:**  
  *"Porque React es una **Single Page Application (SPA)**. Recargar la página destruye el estado en memoria, reinicia el ciclo de vida y genera una experiencia de usuario deficiente con parpadeos de pantalla. En su lugar, aplicamos reactividad del estado: la promesa de creación retorna el registro insertado y actualizamos el estado inmutablemente con `setCitas(prev => [...prev, nuevaCita])`. React detecta la nueva referencia en el estado y actualiza únicamente la fila correspondiente en el DOM a través de su algoritmo de reconciliación."*

---

### Pregunta 5: *"¿Qué sucede si la base de datos se apaga o falla una consulta en el backend?"*
* **Respuesta Modelo:**  
  *"Las operaciones del modelo están encapsuladas en `Promises`. Si ocurre un fallo en SQLite, la promesa se rechaza con `reject(err)`. El controlador envuelve la llamada en un bloque `try/catch` y delega el fallo mediante `next(error)`. Este llega al **Middleware Centralizado de Manejo de Errores** definido al final de `app.js` con firma `(err, req, res, next)`, el cual registra la traza en consola y devuelve una respuesta con código `500 Internal Server Error` en formato JSON, garantizando que el servidor de Node.js no colapse (*anti-crash*) y que el frontend reciba un mensaje claro para alertar al usuario con un Toast de error."*

---

### 💡 CONSEJOS DE ORO PARA EL DÍA DE LA EVALUACIÓN:
1. **Habla con propiedad técnica:** No digas *"esta cosa manda la información"*; di *"el hook ejecuta una petición asíncrona mediante el API Fetch enviando un payload en formato JSON con cabecera Content-Type"*.
2. **Usa la regla de las 3 capas:** Cada vez que te pregunten por una funcionalidad, menciona: **Frontend (React)** $\rightarrow$ **Controlador/Ruta (Express)** $\rightarrow$ **Modelo/Persistencia (SQLite)**.
3. **Muestra calma:** Si te piden un bloque al azar, lee primero la firma de la función, identifica qué entra (parámetros/props), qué procesa (validación/algoritmo) y qué sale (retorno/estado/código HTTP).
