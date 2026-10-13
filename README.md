# Análisis de Principios SOLID — Grano Dorado

*Instrumento de apoyo para justificar los principios SOLID dentro del alcance de un producto mínimo viable*

**Uso sugerido:** complete la plantilla durante el análisis del código o diseño y consérvela como evidencia para revisión, retroalimentación o refactorización.

## 1. Información general

| Campo | Detalle |
|---|---|
| **Lenguaje / Framework** | Aplicación: Kotlin Multiplatform + Compose Multiplatform (Android y web). API REST: Kotlin + Ktor (planeada). Base de datos: Firebase Cloud Firestore, a la que solo accede la API mediante Firebase Admin SDK (planeada). |
| **Tipo de análisis** | Revisión del diseño (niveles 3 y 4 del modelo C4 y diagrama de clases), las historias de usuario y el código del prototipo. |
| **Contexto** | Gestor de Turnos Laborales — Grano Dorado (ITM): programación, publicación y control de turnos de 45 empleados en 3 sucursales, de 06:00 a 22:00. |

## 2. Objetivo del análisis

Evaluar el cumplimiento de los cinco principios SOLID en el Gestor de Turnos, tomando el diseño del modelo C4 (componentes del nivel 3 y clases del nivel 4), el diagrama de clases del Entregable 1 y el código del prototipo. Se busca identificar qué responsabilidades conviene ajustar antes de implementar la API y conectar Firestore, y relacionar cada hallazgo con las historias de usuario y con los patrones GoF seleccionados (Chain of Responsibility, State y Adapter).

**Tipos de evidencia:**

- Diseño (C4 y diagrama de clases): lo que muestran los diagramas y las decisiones de arquitectura (la API y la capa de datos están planeadas).

- Código (prototipo): lo que revisamos en el repositorio, que son pantallas y ViewModels con datos de prueba, sin conexión a la API ni a Firebase.

## 3. Descripción breve del componente

**Función principal:** Programar, publicar y controlar los turnos de las tres sucursales: empleados, catálogo de turnos, asignación con validación de conflictos, consulta de horarios, cambios de turno, novedades y asistencia.

**Responsabilidades actuales:** La aplicación (prototipo) tiene pantallas y ViewModels con datos de prueba y aún no consume la API. La API REST (planeada) concentra las reglas de negocio, con un componente por grupo de historias, y es la única que accede a Firestore. Las reglas no viven en el cliente porque HU-15 exige la hora del servidor, CA-01.6 exige revalidar permisos aunque se entre por URL directa y HU-17 evalúa los turnos sin que nadie tenga la aplicación abierta.

**Clases / interfaces involucradas:** Nivel 4 de la API (diseño planeado): GestionEmpleados, Empleado y Rol; InicioSesion, Cuenta y RecuperacionClave; CatalogoTurnos, Turno, ProgramacionSemanal, Semana, Asignacion y ConsultaHorarios; MotorValidacion, ReglaValidacion y sus 7 reglas; ConsultaDisponibilidad; SolicitudesCambio, AprobacionCambios, SolicitudCambio, EstadoSolicitud y EstadoFinal; MarcacionEmpleado, RegistroSupervisor, Incumplimientos y Marca; ReporteNovedades, BandejaNovedades, RegistroNovedadSupervisor y Novedad; ServicioNotificaciones, CanalEnvio, AdaptadorCorreo y AdaptadorAvisos. Aplicación (nivel 3 y código): LoginViewModel, EmpleadosViewModel, MainViewModel y los demás ViewModels, y SessionManager.

**Dependencias principales:** Aplicación → API REST (JSON/HTTPS con token JWT) → Firestore (Firebase Admin SDK). Dentro de la API: rutas → servicios → Acceso a datos. Los servicios usan el Motor de validación, Notificaciones y Auditoría; Notificaciones usa Gmail API y un canal de avisos por definir.

**Relación con otros módulos:** ProgramacionSemanal, SolicitudesCambio y AprobacionCambios usan el Motor de validación, que consulta Disponibilidad (sin historia propia). Empleados, Autenticación, Turnos, Cambios de turno y Asistencia avisan por Notificaciones, y todos registran en Auditoría.

## 4. Resumen ejecutivo

|                                     |                           |                                                                                                                                                    |
|-------------------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| **Principio**                       | **Nivel de cumplimiento** | **Observación general**                                                                                                                            |
| S - Single Responsibility Principle | Medio                     | El diseño C4 separa los procesos en servicios por módulo; en el código, EmpleadosViewModel mezcla pantalla y validación.                           |
| O - Open/Closed Principle           | Medio                     | Chain of Responsibility y State permiten extender reglas y estados; falta sacar la validación de la entidad de asignación y usar Rol en el código. |
| L - Liskov Substitution Principle   | Medio                     | Las jerarquías del C4 solo existen en el diseño y el modelo de usuarios del diagrama de clases no coincide con el C4 ni con el código.             |
| I - Interface Segregation Principle | Medio                     | CanalEnvio es cohesionada para su cliente; todos los servicios dependen de un acceso a datos genérico (guardar/consultar).                         |
| D - Dependency Inversion Principle  | Bajo                      | Los servicios dependen del componente concreto Acceso a datos y, en el código, los ViewModels guardan los datos y usan un SessionManager global.   |

Escala: Cumple = Alto; Cumple parcialmente = Medio; No cumple = Bajo. En la versión corta se usa Sí / Parcial / No.

## Conclusión global del componente

El diseño de referencia reparte bien las responsabilidades por proceso y aplica patrones que favorecen OCP, SRP y DIP en puntos concretos, pero el acceso a datos no está abstraído en ninguna de las dos capas y, en el prototipo, la aplicación mezcla datos y reglas en los ViewModels.

**Nivel general de calidad del diseño:** Medio

**Prioridad de intervención:** Alta (resolver antes de implementar la API y conectar Firestore)

## 5. Análisis de Principios SOLID

### 5.1 S – Single Responsibility Principle (SRP)

*Definición. Una clase o módulo debe tener una sola razón para cambiar.*

**Elemento evaluado:** Servicios del nivel 4 de la API (GestionEmpleados, CatalogoTurnos, ProgramacionSemanal, AprobacionCambios, MarcacionEmpleado, RegistroSupervisor, ReporteNovedades, BandejaNovedades), la clase Administrador del diagrama de clases y EmpleadosViewModel del prototipo.

**Evaluación:** Cumple parcialmente.

**Evidencia observada:**

- Diseño (C4): cada proceso tiene su servicio: GestionEmpleados (HU-03, HU-04), CatalogoTurnos (HU-06), ProgramacionSemanal (HU-07), ConsultaHorarios (HU-08, HU-09), SolicitudesCambio y AprobacionCambios (HU-10, HU-13), MarcacionEmpleado y RegistroSupervisor (HU-15, HU-16), Incumplimientos (HU-17) y ReporteNovedades, BandejaNovedades y RegistroNovedadSupervisor (HU-11, HU-12, HU-14).

- Diseño (C4): el rol es un dato: Empleado tiene rol: Rol (5 valores) y el alcance por rol lo aplica el componente Seguridad y permisos (HU-01, CA-01.6). No hay una clase que ejecute todos los procesos.

- Diseño (C4): las reglas de conflicto están en MotorValidacion, una clase por regla (Chain of Responsibility), y las transiciones de la solicitud en clases de estado (State).

- Diagrama de clases (Entregable 1): Administrador registra empleados, crea y asigna turnos, aprueba solicitudes y gestiona asistencia y novedades.

- Código (prototipo): EmpleadosViewModel maneja la pantalla, la lista de empleados y las validaciones al guardar (campos vacíos, correo, documento repetido y el último Administrador General).

**Preguntas guía:**

- ¿La clase tiene una única responsabilidad claramente definida?

- ¿Mezcla lógica de negocio, acceso a datos, validación, presentación o infraestructura?

- ¿Existen múltiples razones de cambio en el mismo artefacto?

- ¿El nombre de la clase refleja una sola intención?

**Hallazgos:**

- En el diagrama de clases, Administrador reúne tareas de varios procesos; en el C4 esas tareas se reparten en servicios y el rol es un dato. Un rol indica quién puede actuar y los procesos los ejecutan servicios distintos, así que hay que dejar los dos documentos iguales.

- Las validaciones de EmpleadosViewModel son reglas de HU-03 y HU-04 que, según el C4, le corresponden a GestionEmpleados; quedarían duplicadas.

- ConsultaHorarios atiende a Turnos, Empleados, Asistencia y Novedades, pero solo con consultas de horario de solo lectura: tiene una sola razón de cambio y no es una violación.

- Separar las reglas en eslabones y las transiciones en estados mejora la cohesión: cada clase cambia por un solo motivo (un límite de horas, una transición permitida).

**Impacto técnico potencial:**

- Un cambio en una regla de HU-03 o HU-04 obligaría a modificar dos lugares y podría producir resultados distintos.

- Los ViewModels son más difíciles de probar porque mezclan presentación y reglas.

- La API, responsable de las reglas según el C4, podría quedar con validaciones incompletas.

**Recomendaciones:**

- Dejar claro en el diagrama de clases y en el C4 quién ejecuta los procesos: los servicios del nivel 4 y el rol como enumeración Rol, sin una clase Administrador que concentre procesos.

- Dejar en EmpleadosViewModel el estado de la pantalla y delegar las reglas de negocio de HU-03 y HU-04 a GestionEmpleados en la API.

- Implementar cada regla de conflicto como una subclase de ReglaValidacion y cada estado como un EstadoSolicitud.

### 5.2 O – Open Closed Principle (OCP)

*Definición. Las entidades de software deben estar abiertas para extensión, pero cerradas para modificación.*

**Elemento evaluado:** MotorValidacion, ReglaValidacion y sus reglas concretas (Chain of Responsibility); SolicitudCambio y EstadoSolicitud (State); entidad Asignacion; enumeración Rol.

**Evaluación:** Cumple parcialmente.

**Evidencia observada:**

- Diseño (C4): MotorValidacion arma cadenaAsignacion y cadenaIntercambio con ReglaValidacion (siguiente, validar(ctx), revisar(ctx)) y siete reglas: ReglaFechaNoPasada, ReglaEmpleadoActivo, ReglaDiaDisponible, ReglaUnTurnoPorDia, ReglaMaximoSemanal, ReglaDescansoMinimo y ReglaMismaSucursalPerfil (CA-07.2 a CA-07.5, CA-10.4, CA-10.5, CA-13.5).

- Diseño (C4): SolicitudCambio delega aceptar, rechazar, aprobar, aplicar, cancelar y vencer en EstadoSolicitud; los cinco estados terminales comparten EstadoFinal (HU-10, HU-13).

- Diseño (C4): Asignacion solo guarda empleado, turno, fecha y fechaModificacion; no valida. El tiempo efectivo se calcula en Turno.duracionEfectiva() y MarcacionEmpleado.tiempoEfectivo().

- Diagrama de clases (Entregable 1): AsignacionTurno tiene validarDisponibilidad y calcularTiempoEfectivo.

- Código (prototipo): los cargos se manejan como texto (por ejemplo "ADMINISTRADOR GENERAL"); el diseño usa la enumeración Rol (ADMINISTRADOR_GENERAL, ADMINISTRADOR_SUCURSAL, CAJERO, BARISTA, COCINERO).

**Preguntas guía:**

- ¿Agregar funcionalidad obliga a modificar código estable?

- ¿Se usan abstracciones, polimorfismo o estrategias de extensión?

- ¿Existen estructuras if/switch crecientes basadas en tipos o casos?

- ¿El diseño favorece agregar comportamientos sin romper lo ya construido?

**Hallazgos:**

- Mantener validarDisponibilidad en la entidad de asignación del diagrama de clases duplicaría las reglas, porque en el C4 la validación pertenece al Motor.

- Con la cadena, ajustar un límite (42 h o 12 h; HU-07 advierte que no están validados jurídicamente) modifica solo su regla, y agregar una regla es agregar un eslabón. Lo único que se modifica es la composición de la cadena en MotorValidacion.

- El corte de las reglas no está cerrado: el documento de HU las pone en HU-07 (CA-07.2 a CA-07.5) y el Story Map las separa en HU-07b (Corte 2). Esto no cambia el diseño, pero sí cuándo se implementa.

- Comparar cargos como texto obliga a repetir cadenas en cada regla que dependa del rol; la enumeración Rol lo evita.

**Impacto técnico potencial:**

- Cada cambio de límite o de regla tocaría un método largo compartido por la asignación y el intercambio.

- Una transición prohibida, como aprobar una solicitud vencida (CA-13.6), podría quedar permitida por un condicional olvidado.

- Un cargo mal escrito pasaría la compilación y fallaría en ejecución.

**Recomendaciones:**

- Validar solo en MotorValidacion, con una clase por regla, y retirar validarDisponibilidad de la entidad de asignación.

- Implementar los estados de la solicitud con EstadoSolicitud y EstadoFinal (State) en el Corte 2.

- Usar la enumeración Rol del nivel 4 en el código en lugar de texto.

- Confirmar si las reglas de conflicto pertenecen a HU-07 o a HU-07b antes de planear su implementación.

### 5.3 L – Liskov Substitution Principle (LSP)

*Definición. Los objetos de una subclase deben poder sustituir a los de su clase base sin alterar el comportamiento esperado del sistema.*

**Elemento evaluado:** Jerarquías del nivel 4: EstadoSolicitud con Solicitada, AceptadaPorElCompanero, Aprobada y EstadoFinal (Aplicada, RechazadaPorElCompanero, RechazadaPorElAdministrador, Cancelada y Vencida); ReglaValidacion y sus 7 reglas; CanalEnvio con AdaptadorCorreo y AdaptadorAvisos. Modelo de usuarios: Usuario, Empleado y Administrador (diagrama de clases) frente a Empleado con Rol (C4).

**Evaluación:** Cumple parcialmente.

**Evidencia observada:**

- Diagrama de clases (Entregable 1): Usuario es una clase abstracta de la que heredan Empleado y Administrador.

- Diseño (C4): no hay herencia de usuarios: Empleado es una sola entidad con rol: Rol. En el código tampoco existe esa jerarquía.

- Diseño (C4): EstadoSolicitud rechaza por defecto toda acción («Acción no permitida»); cada estado redefine solo las acciones que permite y EstadoFinal no permite ninguna. ReglaValidacion define validar(ctx) y cada regla redefine revisar(ctx). AdaptadorCorreo «convierte su respuesta o error en ResultadoEnvio».

- Código (prototipo): la pantalla de cambios de turno recibe isRolAdmin (true o false) para cambiar lo que muestra.

**Preguntas guía:**

- ¿Las subclases respetan el contrato de la superclase?

- ¿Alguna subclase lanza excepciones inesperadas o deshabilita comportamientos heredados?

- ¿Se redefinen precondiciones o postcondiciones de manera incompatible?

- ¿El cliente necesita conocer el subtipo concreto para funcionar correctamente?

**Hallazgos:**

- Clasificar a los usuarios con una enumeración no es herencia, así que en el C4 y en el código no hay contrato de subtipos de usuario que evaluar. Pero el diagrama de clases sí tiene la herencia, y hay que decidir cuál modelo queda para que coincidan.

- En diseño, las jerarquías del C4 respetan sus contratos: un estado terminal que rechaza cumple el «por defecto rechaza», una regla que corta devuelve el ResultadoValidacion esperado y los adaptadores devuelven ResultadoEnvio. Es una observación de diseño, todavía no se comprobó con código porque esas clases aún no están programadas.

- isRolAdmin no es un problema de LSP: la pantalla no sustituye subtipos, solo cambia lo que muestra según el rol. Es un tema de presentación por permisos.

- Decisión pendiente: HU-03 exige que todo empleado pertenezca a exactamente una sucursal y Empleado.sucursal es obligatoria en el nivel 4, pero HU-07 y HU-08 indican que el Administrador General actúa sobre las tres. No se cambia la multiplicidad a 0..1 hasta resolverlo.

**Impacto técnico potencial:**

- Mientras las jerarquías no estén programadas no se puede afirmar que los subtipos se comporten igual en ejecución; un adaptador que lance una excepción en lugar de devolver ResultadoEnvio rompería el contrato de CanalEnvio.

- Si el modelo de usuarios queda distinto en el diagrama y en el C4, los documentos y el código no coincidirían.

- Resolver la sucursal del Administrador General en un solo artefacto dejaría inconsistentes HU-03, el nivel 4 y las reglas de alcance.

- Decidir la pantalla con un booleano obliga a modificarla si aparece otro rol con permisos distintos.

**Recomendaciones:**

- Decidir cómo se modelan los usuarios y dejar igual el diagrama de clases y el C4.

- Decidir con el equipo cómo se modela el alcance del Administrador General y aplicarlo a la vez en HU-03, el nivel 4 de Empleados y Seguridad y permisos.

- Cuando se programen, verificar el LSP con pruebas de contrato: los 8 estados frente a las 6 acciones, cada regla con un caso válido y uno inválido, y cada adaptador con un envío exitoso y uno fallido.

- Derivar lo que muestra la pantalla de cambios de turno de los permisos de la sesión, en lugar de un booleano.

### 5.4 I – Interface Segregation Principle (ISP)

*Definición. Los clientes no deben depender de interfaces que no utilizan.*

**Elemento evaluado:** Interfaz CanalEnvio (Notificaciones); interfaz Autenticable del diagrama de clases; componente Acceso a datos con guardar(...) y consultar(...); contratos de repositorio propuestos.

**Evaluación:** Cumple parcialmente.

**Evidencia observada:**

- Diseño (C4): CanalEnvio tiene una sola operación, enviar(mensaje): ResultadoEnvio, y su cliente, ServicioNotificaciones, la usa completa para los correos y avisos de HU-02, HU-03, HU-07, HU-10, HU-13 y HU-17.

- Diseño (C4): todos los servicios de la API (GestionEmpleados, ProgramacionSemanal, MarcacionEmpleado, ReporteNovedades, entre otros) dependen del mismo componente Acceso a datos con guardar(...) y consultar(...) genéricos.

- Diseño: las operaciones de sesión son pocas y claras: Autenticable (diagrama de clases) tiene iniciarSesion y cerrarSesion, y en el C4 están en el servicio InicioSesion (HU-01).

**Preguntas guía:**

- ¿Las interfaces obligan a implementar métodos irrelevantes?

- ¿Existen interfaces demasiado amplias o genéricas?

- ¿Los clientes consumen solo una parte mínima de la interfaz?

- ¿Se podrían dividir los contratos en interfaces más cohesionadas?

**Hallazgos:**

- CanalEnvio cumple ISP no por tener pocos métodos, sino porque su único cliente necesita exactamente esa operación y los adaptadores no implementan nada que no usen.

- El acceso a datos genérico hace que cada servicio dependa de un contrato que cubre los datos de todos los módulos: un cambio en el almacenamiento de marcas afecta el contrato que usa Empleados.

- Las operaciones de sesión no mezclan nada ajeno, así que no hay problema ahí.

**Impacto técnico potencial:**

- Los servicios quedan acoplados a operaciones de otros módulos.

- Los dobles de prueba deben simular todo el acceso a datos aunque un servicio use una sola colección.

- Es más fácil que un módulo lea o escriba datos de otro sin control.

**Recomendaciones:**

- Definir en la API un contrato de repositorio por módulo con solo las operaciones que piden sus servicios: empleados, turnos, asignaciones, solicitudes, marcas y novedades (nombres por definir, por ejemplo RepositorioEmpleados).

- Mantener CanalEnvio con su única operación.

- En la aplicación, definir contratos de acceso a la API por funcionalidad, no uno solo para toda la aplicación.

### 5.5 D – Dependency Inversion Principle (DIP)

*Definición. Los módulos de alto nivel no deben depender de módulos de bajo nivel; ambos deben depender de abstracciones.*

**Elemento evaluado:** Servicios de la API frente a Acceso a datos y Firestore; ServicioNotificaciones frente a CanalEnvio; MotorValidacion frente a ReglaValidacion; ViewModels de la aplicación frente a la API; SessionManager.

**Evaluación:** No cumple.

**Evidencia observada:**

- Diseño (C4): nivel 2: la aplicación consume la API por JSON/HTTPS con token JWT y solo la API lee y escribe en Firestore mediante Firebase Admin SDK. El C4 registra la decisión «las reglas viven en la API, no en el cliente».

- Diseño (C4): nivel 4: los servicios dependen del componente concreto Acceso a datos; no hay interfaces de repositorio. El nivel 2 menciona rutas → servicios → repositorios con inyección de dependencias (Koin), sin reflejo en el nivel 4.

- Diseño (C4): ServicioNotificaciones depende de CanalEnvio y no de Gmail API (Adapter); MotorValidacion depende de ReglaValidacion (Chain of Responsibility).

- Código (prototipo): los datos de prueba están dentro de cada ViewModel, los ViewModels crean sus propias dependencias, el inicio de sesión usa usuarios de prueba y la sesión se guarda en un objeto global SessionManager. El nivel 3 de la aplicación marca «Sesión» y «Capa de datos» como planeados.

**Preguntas guía:**

- ¿La lógica de negocio depende directamente de implementaciones concretas?

- ¿Se inyectan dependencias o se crean con new dentro del componente?

- ¿Existen interfaces o abstracciones desacopladoras?

- ¿El componente puede probarse fácilmente con mocks, stubs o fakes?

**Hallazgos:**

- Conectar Firebase directo desde los ViewModels contradiría el C4. Los ViewModels deben depender de abstracciones de acceso a la API, y la implementación con Firestore va en la API.

- En la API, los servicios de negocio (alto nivel) dependen de un detalle de bajo nivel, Acceso a datos, sin abstracción intermedia; eso impide sustituir Firestore por un doble de prueba.

- Adapter y Chain of Responsibility son los puntos del diseño que ya invierten dependencias: el cliente conoce CanalEnvio y ReglaValidacion, no sus implementaciones.

- La inyección de dependencias con Koin está planeada, pero todavía no está en el código.

- Un SessionManager global hace que los ViewModels dependan de un detalle compartido.

**Impacto técnico potencial:**

- Pasar de los datos de prueba a la API obligará a modificar cada ViewModel.

- Probar los servicios de la API exigiría una base de datos Firestore real.

- Si un ViewModel se conectara directo a Firestore, las reglas de HU-15, CA-01.6 y HU-17 quedarían fuera de la API o duplicadas.

**Recomendaciones:**

- API: definir las interfaces de repositorio junto a los servicios y sus implementaciones con Firestore en Acceso a datos, inyectadas por constructor (Koin).

- Aplicación: definir repositorios por funcionalidad cuya implementación llame a la API con Ktor Client (componente «Capa de datos»), con una versión en memoria para los datos de prueba, e inyectarlos en los ViewModels por constructor.

- Reemplazar el SessionManager global por una interfaz de sesión inyectada y actualizar el estado de «Sesión» en el nivel 3 cuando esté en el código.

- Mantener ServicioNotificaciones dependiente de CanalEnvio y usar un adaptador falso en las pruebas.

## 6. Hallazgos transversales

**Problemas recurrentes identificados:**

- El diagrama de clases del Entregable 1 (Usuario, Administrador, Autenticable, AsignacionTurno, SolicitudCambioTurno, MarcaAsistencia) y el C4 (Empleado y Rol, InicioSesion, Asignacion, SolicitudCambio y Marca) nombran distinto varias clases. Hay que dejarlos coherentes.

- El acceso a datos es el punto más débil en las dos capas: es genérico en la API y está dentro de los ViewModels en la aplicación.

- Las reglas de negocio deben vivir en un solo lugar, la API: la validación de conflictos en el Motor, las transiciones en los estados y el envío en los adaptadores.

- Solo Chain of Responsibility, State y Adapter se usan como argumentos SOLID. Observer de la API está pendiente de validación y Factory Method fue descartado.

- Quedan pendientes el corte HU-07/HU-07b, la historia del registro de disponibilidad y la sucursal del Administrador General.

## 7. Priorización de mejoras

|                                                                                                        |               |               |              |               |                                                                                                                               |
|--------------------------------------------------------------------------------------------------------|---------------|---------------|--------------|---------------|-------------------------------------------------------------------------------------------------------------------------------|
| **Hallazgo**                                                                                           | **Principio** | **Severidad** | **Esfuerzo** | **Prioridad** | **Acción sugerida**                                                                                                           |
| Acceso a datos sin abstracción en la API y datos dentro de los ViewModels.                             | DIP           | Alta          | Medio        | Alta          | Repositorios en la API con implementación Firestore y repositorios hacia la API en la aplicación, inyectados por constructor. |
| Reglas de conflicto ubicadas en la entidad de asignación del diagrama de clases.                       | OCP           | Alta          | Medio        | Alta          | Validar solo en MotorValidacion con una clase por regla (Chain of Responsibility).                                            |
| Contrato de datos genérico guardar/consultar.                                                          | ISP           | Media         | Medio        | Media         | Al crear los repositorios de la primera fila, dividirlos por módulo según las operaciones de cada servicio.                   |
| Validaciones de negocio en EmpleadosViewModel.                                                         | SRP           | Media         | Bajo         | Media         | Delegar las reglas de HU-03 y HU-04 a GestionEmpleados y dejar en el ViewModel solo el estado de la pantalla.                 |
| SessionManager global y vista decidida con isRolAdmin.                                                 | DIP           | Media         | Bajo         | Media         | Interfaz de sesión inyectada y opciones de pantalla derivadas de los permisos.                                                |
| Modelo de usuarios distinto entre el diagrama y el C4; sucursal del Administrador General sin decidir. | LSP           | Media         | Bajo         | Media         | Decidir el modelo y la regla de HU-03 frente a HU-07; pruebas de contrato cuando se programen.                                |
| Transiciones de la solicitud de cambio sin implementar.                                                | OCP           | Media         | Medio        | Media         | Implementar State (EstadoSolicitud y EstadoFinal) en el Corte 2.                                                              |

## 9. Conclusión

**Síntesis del estado del diseño:** El diseño de referencia separa los procesos por servicio y aplica Chain of Responsibility, State y Adapter donde las historias lo justifican, lo que favorece SRP, OCP y DIP en esos puntos. El acceso a datos aún no está abstraído y, en el prototipo, la aplicación mezcla datos y reglas en los ViewModels.

**Principios más comprometidos:** DIP (acceso a datos sin abstracciones y SessionManager global), seguido de ISP (contrato de datos genérico) y SRP (reglas en la vista).

**Principios mejor implementados:** OCP y SRP en las reglas de conflicto (Chain of Responsibility) y en los estados de la solicitud (State), y DIP en Notificaciones (CanalEnvio, Adapter); los tres como diseño planeado.

**Nivel de deuda técnica asociada al diseño:** Media. El backend todavía no existe y la aplicación no está conectada, así que las correcciones pueden hacerse en el diseño antes de implementar.

**Recomendación final:** Antes de implementar la API y conectar Firestore: definir los repositorios por módulo en la API, conectar la aplicación solo a la API mediante repositorios inyectados, centralizar las reglas en el Motor de validación, dejar coherentes el diagrama de clases y el C4, y resolver las decisiones pendientes (HU-07/HU-07b, historia de disponibilidad y sucursal del Administrador General).

## Versión corta para revisión rápida

|               |                 |                                                                                        |                                                                                                  |                                                    |
|---------------|-----------------|----------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------|----------------------------------------------------|
| **Principio** | **¿Se cumple?** | **Evidencia**                                                                          | **Problema detectado**                                                                           | **Recomendación**                                  |
| SRP           | Parcial         | Servicios separados por proceso en el nivel 4; EmpleadosViewModel valida.              | Reglas de HU-03 en la vista.                                                                     | Delegar las reglas a GestionEmpleados.             |
| OCP           | Parcial         | MotorValidacion con 7 reglas y State en el diseño.                                     | Validación ubicada en la entidad; cargos como texto.                                             | Validar solo en el Motor y usar Rol.               |
| LSP           | Parcial         | Jerarquías del C4 respetan sus contratos en diseño.                                    | Modelo de usuarios distinto entre diagrama y C4; sucursal del Administrador General sin decidir. | Decidir el modelo y hacer pruebas de contrato.     |
| ISP           | Parcial         | CanalEnvio cohesionada; Acceso a datos genérico.                                       | Contrato de datos único.                                                                         | Repositorio por módulo.                            |
| DIP           | No              | Servicios dependen de Acceso a datos; datos en los ViewModels y SessionManager global. | Alto nivel depende de detalles.                                                                  | Repositorios inyectados; Firestore solo en la API. |
