# Análisis de Principios SOLID — Grano Dorado

*Instrumento de apoyo para justificar los principios SOLID dentro del alcance de un producto mínimo viable*

**Uso sugerido:** complete la plantilla durante el análisis del código o diseño y consérvela como evidencia para revisión, retroalimentación o refactorización.

---

## 1. Información general

| Campo | Detalle |
|---|---|
| **Lenguaje / Framework** | Aplicación web: Kotlin Multiplatform + Compose Multiplatform (Kotlin/Wasm). API REST: Kotlin + Ktor (planeada). Base de datos: Firebase Cloud Firestore, a la que solo accede la API mediante Firebase Admin SDK (planeada). |
| **Tipo de análisis** | Revisión de diseño sobre los niveles 3 y 4 del modelo C4, las historias de usuario y la auditoría de concordancia. |
| **Contexto** | Gestor de Turnos Laborales — Grano Dorado (ITM): programación, publicación y control de turnos de 45 empleados en 3 sucursales, de 06:00 a 22:00 (HU-01 a HU-17). |

---

## 2. Objetivo del análisis

Evaluar el cumplimiento de los cinco principios SOLID en el diseño del Gestor de Turnos tal como lo documenta el modelo C4 (componentes del nivel 3 y clases del nivel 4), identificar qué responsabilidades conviene ajustar antes de implementar la API y conectar Firestore, y relacionar cada hallazgo con las historias de usuario y con los patrones GoF aprobados (Chain of Responsibility, State y Adapter). Esta versión corrige el análisis anterior, que evaluaba un modelo de clases distinto del C4 y una conexión directa de la aplicación a Firebase (auditoría AUD-001 y AUD-002).

**Tipos de evidencia.**

- **Diseño documentado:** lo que muestran los diagramas C4 y las decisiones de arquitectura (diseño planeado de la API y componentes de la aplicación).
- **Implementación declarada:** lo que el equipo afirma sobre el código en el análisis anterior; no se pudo comprobar porque el código no se entregó.
- **Implementación verificada:** ninguna en esta revisión. Ningún principio se declara verificado en código.

---

## 3. Descripción breve del componente

**Función principal:** Programar, publicar y controlar los turnos de las tres sucursales: empleados, catálogo de turnos, asignación con validación de conflictos, consulta de horarios, cambios de turno, novedades y asistencia.

**Responsabilidades actuales:** La aplicación web (existente según el C4) tiene pantallas y ViewModels con datos de prueba y aún no consume la API. La API REST (planeada) concentra las reglas de negocio, con un componente por grupo de historias, y es la única que accede a Firestore. Las reglas no viven en el cliente porque HU-15 exige la hora del servidor, CA-01.6 exige revalidar permisos aunque se entre por URL directa y HU-17 evalúa los turnos sin que nadie tenga la aplicación abierta.

**Clases / interfaces involucradas:** Nivel 4 de la API (diseño planeado): GestionEmpleados, Empleado y Rol; InicioSesion, Cuenta y RecuperacionClave; CatalogoTurnos, Turno, ProgramacionSemanal, Semana, Asignacion y ConsultaHorarios; MotorValidacion, ReglaValidacion y sus 7 reglas; ConsultaDisponibilidad; SolicitudesCambio, AprobacionCambios, SolicitudCambio, EstadoSolicitud y EstadoFinal; MarcacionEmpleado, RegistroSupervisor, Incumplimientos y Marca; ReporteNovedades, BandejaNovedades, RegistroNovedadSupervisor y Novedad; ServicioNotificaciones, CanalEnvio, AdaptadorCorreo y AdaptadorAvisos. Aplicación (nivel 3): LoginViewModel, EmpleadosViewModel, MainViewModel y los demás ViewModels; SessionManager solo aparece en la declaración del equipo.

**Dependencias principales:** Aplicación → API REST (JSON/HTTPS con token JWT) → Firestore (Firebase Admin SDK). Dentro de la API: rutas → servicios → Acceso a datos. Los servicios usan el Motor de validación, Notificaciones y Auditoría; Notificaciones usa Gmail API y un canal de avisos por definir.

**Relación con otros módulos:** ProgramacionSemanal, SolicitudesCambio y AprobacionCambios usan el Motor de validación, que consulta Disponibilidad (sin historia propia, AUD-009). Empleados, Autenticación, Turnos, Cambios de turno y Asistencia avisan por Notificaciones, y todos registran en Auditoría.

---

## 4. Resumen ejecutivo

| Principio | Nivel de cumplimiento | Observación general |
|---|---|---|
| **S** - Single Responsibility Principle | Medio | El diseño C4 separa los procesos en servicios por módulo; según el equipo, EmpleadosViewModel mezcla pantalla y validación. |
| **O** - Open/Closed Principle | Medio | Chain of Responsibility y State permiten extender reglas y estados; falta sacar la validación de la entidad de asignación y confirmar el uso de Rol. |
| **L** - Liskov Substitution Principle | No verificable | El C4 no tiene herencia de usuarios; las jerarquías documentadas (EstadoSolicitud, ReglaValidacion, CanalEnvio) solo pueden revisarse como diseño. |
| **I** - Interface Segregation Principle | Medio | CanalEnvio es cohesionada para su cliente; todos los servicios dependen de un acceso a datos genérico (guardar/consultar). |
| **D** - Dependency Inversion Principle | Bajo | Los servicios dependen del componente concreto Acceso a datos y, según el equipo, los ViewModels guardan los datos y usan un SessionManager global. |

**Escala:** Cumple = Alto; Cumple parcialmente = Medio; No cumple = Bajo; No se puede verificar = «No verificable». En la versión corta se usa Sí / Parcial / No / No verificable.

### Conclusión global del componente

El diseño de referencia reparte bien las responsabilidades por proceso y aplica patrones que favorecen OCP, SRP y DIP en puntos concretos, pero el acceso a datos no está abstraído en ninguna de las dos capas y, según el equipo, la aplicación mezcla datos y reglas en los ViewModels. Ninguna conclusión está verificada en código.

**Nivel general de calidad del diseño:** Medio

**Prioridad de intervención:** Alta (resolver antes de implementar la API y conectar Firestore)

---

## 5. Análisis de Principios SOLID

### 5.1 S – Single Responsibility Principle (SRP)

*Definición.* Una clase o módulo debe tener una sola razón para cambiar.

**Elemento evaluado:** Servicios del nivel 4 de la API (GestionEmpleados, CatalogoTurnos, ProgramacionSemanal, AprobacionCambios, MarcacionEmpleado, RegistroSupervisor, ReporteNovedades, BandejaNovedades) y ViewModels de la aplicación (EmpleadosViewModel).

**Evaluación:** Cumple parcialmente

**Evidencia observada:**

- **Diseño documentado:** cada proceso tiene su servicio: GestionEmpleados (HU-03, HU-04), CatalogoTurnos (HU-06), ProgramacionSemanal (HU-07), ConsultaHorarios (HU-08, HU-09), SolicitudesCambio y AprobacionCambios (HU-10, HU-13), MarcacionEmpleado y RegistroSupervisor (HU-15, HU-16), Incumplimientos (HU-17, ficha rotulada HU-15 en el documento de HU, AUD-015) y ReporteNovedades, BandejaNovedades y RegistroNovedadSupervisor (HU-11, HU-12, HU-14).
- **Diseño documentado:** el rol es un dato: Empleado tiene rol: Rol (5 valores) y el alcance por rol lo aplica el componente Seguridad y permisos (HU-01, CA-01.6). No existe una clase Administrador que ejecute los procesos.
- **Diseño documentado:** las reglas de conflicto están en MotorValidacion, una clase por regla (Chain of Responsibility), y las transiciones de la solicitud en clases de estado (State).
- **Implementación declarada:** EmpleadosViewModel maneja la pantalla, la lista de empleados y las validaciones al guardar.

**Preguntas guía:**

1. ¿La clase tiene una única responsabilidad claramente definida?
2. ¿Mezcla lógica de negocio, acceso a datos, validación, presentación o infraestructura?
3. ¿Existen múltiples razones de cambio en el mismo artefacto?
4. ¿El nombre de la clase refleja una sola intención?

**Hallazgos:**

- La concentración de tareas que el análisis anterior atribuía a «Administrador» no corresponde al diseño de referencia. Un rol indica quién puede actuar; los procesos los ejecutan servicios distintos (AUD-002, AUD-004).
- Si EmpleadosViewModel valida las reglas de HU-03 (documento único, correo único, rango de la fecha de ingreso), esas reglas quedan duplicadas con GestionEmpleados, que es su responsable según el C4.
- ConsultaHorarios atiende a Turnos, Empleados, Asistencia y Novedades, pero solo con consultas de horario de solo lectura: tiene una sola razón de cambio y no es una violación.
- Separar las reglas en eslabones y las transiciones en estados mejora la cohesión: cada clase cambia por un solo motivo (un límite de horas, una transición permitida).

**Impacto técnico potencial:**

Si las reglas de negocio se reparten entre la aplicación y la API:

- Un cambio en una regla de HU-03 o HU-04 obliga a modificar dos lugares y puede producir resultados distintos.
- Los ViewModels son más difíciles de probar porque mezclan presentación y reglas.
- La API, responsable de las reglas según el C4, podría quedar con validaciones incompletas.

**Recomendaciones:**

- Modelar los procesos con los servicios del nivel 4 y el rol con la enumeración Rol y el componente Seguridad y permisos, sin una clase Administrador que concentre procesos.
- Dejar en EmpleadosViewModel el estado de la pantalla y delegar las reglas de negocio de HU-03 y HU-04 a GestionEmpleados en la API.
- Implementar cada regla de conflicto como una subclase de ReglaValidacion y cada estado como un EstadoSolicitud, para que cada clase tenga una sola razón de cambio.

---

### 5.2 O – Open Closed Principle (OCP)

*Definición.* Las entidades de software deben estar abiertas para extensión, pero cerradas para modificación.

**Elemento evaluado:** MotorValidacion, ReglaValidacion y sus reglas concretas (Chain of Responsibility); SolicitudCambio y EstadoSolicitud (State); entidad Asignacion; enumeración Rol.

**Evaluación:** Cumple parcialmente

**Evidencia observada:**

- **Diseño documentado:** MotorValidacion arma cadenaAsignacion y cadenaIntercambio con ReglaValidacion (siguiente, validar(ctx), revisar(ctx)) y siete reglas: ReglaFechaNoPasada, ReglaEmpleadoActivo, ReglaDiaDisponible, ReglaUnTurnoPorDia, ReglaMaximoSemanal, ReglaDescansoMinimo y ReglaMismaSucursalPerfil (CA-07.2 a CA-07.5, CA-10.4, CA-10.5, CA-13.5).
- **Diseño documentado:** SolicitudCambio delega aceptar, rechazar, aprobar, aplicar, cancelar y vencer en EstadoSolicitud; los cinco estados terminales comparten EstadoFinal (HU-10, HU-13).
- **Diseño documentado:** Asignacion solo guarda empleado, turno, fecha y fechaModificacion; no valida. El tiempo efectivo se calcula en Turno.duracionEfectiva() y MarcacionEmpleado.tiempoEfectivo().
- **Implementación declarada:** los cargos se manejan como texto; el diseño usa la enumeración Rol (ADMINISTRADOR_GENERAL, ADMINISTRADOR_SUCURSAL, CAJERO, BARISTA, COCINERO).

**Preguntas guía:**

1. ¿Agregar funcionalidad obliga a modificar código estable?
2. ¿Se usan abstracciones, polimorfismo o estrategias de extensión?
3. ¿Existen estructuras if/switch crecientes basadas en tipos o casos?
4. ¿El diseño favorece agregar comportamientos sin romper lo ya construido?

**Hallazgos:**

- El análisis anterior ubicaba validarDisponibilidad y calcularTiempoEfectivo en «AsignacionTurno», clase que no está en el C4. La validación pertenece al Motor; mantenerla también en la entidad duplicaría las reglas (AUD-007).
- Con la cadena, ajustar un límite (42 h o 12 h; HU-07 advierte que no están validados jurídicamente) modifica solo su regla, y agregar una regla es agregar un eslabón. Lo único que se modifica es la composición de la cadena en MotorValidacion.
- El corte de las reglas no está cerrado: el documento de HU las pone en HU-07 (CA-07.2 a CA-07.5) y el Story Map las separa en HU-07b (Corte 2). Esto no cambia el diseño, pero sí cuándo se implementa (AUD-008).
- Comparar cargos como texto obliga a repetir cadenas en cada regla que dependa del rol; la enumeración Rol lo evita.

**Impacto técnico potencial:**

Sin estas separaciones:

- Cada cambio de límite o de regla tocaría un método largo compartido por la asignación y el intercambio.
- Una transición prohibida, como aprobar una solicitud vencida (CA-13.6), podría quedar permitida por un condicional olvidado.
- Un cargo mal escrito pasaría la compilación y fallaría en ejecución.

**Recomendaciones:**

- Validar solo en MotorValidacion, con una clase por regla, y retirar validarDisponibilidad de la entidad de asignación.
- Implementar los estados de la solicitud con EstadoSolicitud y EstadoFinal (State) en el Corte 2.
- Usar la enumeración Rol del nivel 4 en el código en lugar de texto.
- Confirmar si las reglas de conflicto pertenecen a HU-07 o a HU-07b antes de planear su implementación.

---

### 5.3 L – Liskov Substitution Principle (LSP)

*Definición.* Los objetos de una subclase deben poder sustituir a los de su clase base sin alterar el comportamiento esperado del sistema.

**Elemento evaluado:** Jerarquías del nivel 4: EstadoSolicitud con Solicitada, AceptadaPorElCompanero, Aprobada y EstadoFinal (Aplicada, RechazadaPorElCompanero, RechazadaPorElAdministrador, Cancelada y Vencida); ReglaValidacion y sus 7 reglas; CanalEnvio con AdaptadorCorreo y AdaptadorAvisos. Modelo de usuarios: Empleado con Rol.

**Evaluación:** No se puede verificar

**Evidencia observada:**

- **Diseño documentado:** el C4 no tiene herencia de usuarios: Empleado es una sola entidad con rol: Rol. La jerarquía Usuario → Empleado / Administrador del análisis anterior no aparece en el C4 y, según el equipo, tampoco en el código.
- **Diseño documentado:** EstadoSolicitud rechaza por defecto toda acción («Acción no permitida»); cada estado redefine solo las acciones que permite y EstadoFinal no permite ninguna. ReglaValidacion define validar(ctx) y cada regla redefine revisar(ctx). AdaptadorCorreo «convierte su respuesta o error en ResultadoEnvio».
- **Implementación declarada:** la pantalla de cambios de turno recibe isRolAdmin (true o false) para cambiar lo que muestra.

**Preguntas guía:**

1. ¿Las subclases respetan el contrato de la superclase?
2. ¿Alguna subclase lanza excepciones inesperadas o deshabilita comportamientos heredados?
3. ¿Se redefinen precondiciones o postcondiciones de manera incompatible?
4. ¿El cliente necesita conocer el subtipo concreto para funcionar correctamente?

**Hallazgos:**

- Clasificar a los usuarios con una enumeración no es herencia, así que no hay contrato de subtipos de usuario que evaluar. Se retira esa evaluación del análisis anterior (AUD-002, AUD-004).
- En diseño, las jerarquías documentadas respetan sus contratos: un estado terminal que rechaza cumple el «por defecto rechaza», una regla que corta devuelve el ResultadoValidacion esperado y los adaptadores devuelven ResultadoEnvio. Es una observación de diseño, no una comprobación de comportamiento.
- isRolAdmin no es un problema de LSP: la pantalla no sustituye subtipos, solo cambia lo que muestra según el rol. Es un tema de presentación por permisos.
- Decisión pendiente: HU-03 exige que todo empleado pertenezca a exactamente una sucursal y Empleado.sucursal es obligatoria en el nivel 4, pero HU-07 y HU-08 indican que el Administrador General actúa sobre las tres. No se cambia la multiplicidad a 0..1 hasta resolverlo (AUD-013).

**Impacto técnico potencial:**

Mientras no exista código:

- No se puede afirmar que los subtipos se comporten igual en ejecución; un adaptador que lance una excepción en lugar de devolver ResultadoEnvio rompería el contrato de CanalEnvio.
- Resolver la sucursal del Administrador General en un solo artefacto dejaría inconsistentes HU-03, el nivel 4 y las reglas de alcance.
- Decidir la pantalla con un booleano obliga a modificarla si aparece otro rol con permisos distintos.

**Recomendaciones:**

- Verificar LSP con pruebas de contrato cuando exista el código: los 8 estados frente a las 6 acciones, cada regla con un caso válido y uno inválido, y cada adaptador con un envío exitoso y uno fallido.
- Decidir con el equipo cómo se modela el alcance del Administrador General y aplicarlo a la vez en HU-03, el nivel 4 de Empleados y Seguridad y permisos.
- Derivar lo que muestra la pantalla de cambios de turno de los permisos de la sesión, en lugar de un booleano.

---

### 5.4 I – Interface Segregation Principle (ISP)

*Definición.* Los clientes no deben depender de interfaces que no utilizan.

**Elemento evaluado:** Interfaz CanalEnvio (Notificaciones); componente Acceso a datos con guardar(...) y consultar(...); contratos de repositorio propuestos.

**Evaluación:** Cumple parcialmente

**Evidencia observada:**

- **Diseño documentado:** CanalEnvio tiene una sola operación, enviar(mensaje): ResultadoEnvio, y su cliente, ServicioNotificaciones, la usa completa para los correos y avisos de HU-02, HU-03, HU-07, HU-10, HU-13 y HU-17.
- **Diseño documentado:** todos los servicios de la API (GestionEmpleados, ProgramacionSemanal, MarcacionEmpleado, ReporteNovedades, entre otros) dependen del mismo componente Acceso a datos con guardar(...) y consultar(...) genéricos.
- **Diseño documentado:** las operaciones de sesión están en el servicio InicioSesion (iniciarSesion, iniciarSesionConGoogle, cerrarSesion; HU-01). La interfaz Autenticable solo aparece en el análisis anterior.

**Preguntas guía:**

1. ¿Las interfaces obligan a implementar métodos irrelevantes?
2. ¿Existen interfaces demasiado amplias o genéricas?
3. ¿Los clientes consumen solo una parte mínima de la interfaz?
4. ¿Se podrían dividir los contratos en interfaces más cohesionadas?

**Hallazgos:**

- CanalEnvio cumple ISP no por tener pocos métodos, sino porque su único cliente necesita exactamente esa operación y los adaptadores no implementan nada que no usen.
- El acceso a datos genérico hace que cada servicio dependa de un contrato que cubre los datos de todos los módulos: un cambio en el almacenamiento de marcas afecta el contrato que usa Empleados (AUD-006).
- Autenticable deja de evaluarse como interfaz del diseño aprobado porque no tiene correspondencia en el C4.

**Impacto técnico potencial:**

Con un contrato de datos único:

- Los servicios quedan acoplados a operaciones de otros módulos.
- Los dobles de prueba deben simular todo el acceso a datos aunque un servicio use una sola colección.
- Es más fácil que un módulo lea o escriba datos de otro sin control.

**Recomendaciones:**

- Definir en la API un contrato de repositorio por módulo con solo las operaciones que piden sus servicios: empleados, turnos, asignaciones, solicitudes, marcas y novedades (nombres por definir, por ejemplo RepositorioEmpleados).
- Mantener CanalEnvio con su única operación.
- En la aplicación, definir contratos de acceso a la API por funcionalidad, no uno solo para toda la aplicación.

---

### 5.5 D – Dependency Inversion Principle (DIP)

*Definición.* Los módulos de alto nivel no deben depender de módulos de bajo nivel; ambos deben depender de abstracciones.

**Elemento evaluado:** Servicios de la API frente a Acceso a datos y Firestore; ServicioNotificaciones frente a CanalEnvio; MotorValidacion frente a ReglaValidacion; ViewModels de la aplicación frente a la API; SessionManager.

**Evaluación:** No cumple

**Evidencia observada:**

- **Diseño documentado:** nivel 2: la aplicación consume la API por JSON/HTTPS con token JWT y solo la API lee y escribe en Firestore mediante Firebase Admin SDK. El C4 registra la decisión «las reglas viven en la API, no en el cliente».
- **Diseño documentado:** nivel 4: los servicios dependen del componente concreto Acceso a datos; no hay interfaces de repositorio. El nivel 2 menciona rutas → servicios → repositorios con inyección de dependencias (Koin), sin reflejo en el nivel 4.
- **Diseño documentado:** ServicioNotificaciones depende de CanalEnvio y no de Gmail API (Adapter); MotorValidacion depende de ReglaValidacion (Chain of Responsibility).
- **Implementación declarada:** los datos de prueba están dentro de cada ViewModel, los ViewModels crean sus dependencias y la sesión se guarda en un objeto global SessionManager. El nivel 3 de la aplicación marca «Sesión» y «Capa de datos» como planeados.

**Preguntas guía:**

1. ¿La lógica de negocio depende directamente de implementaciones concretas?
2. ¿Se inyectan dependencias o se crean con new dentro del componente?
3. ¿Existen interfaces o abstracciones desacopladoras?
4. ¿El componente puede probarse fácilmente con mocks, stubs o fakes?

**Hallazgos:**

- El análisis anterior proponía conectar Firebase desde los ViewModels, lo que contradice el C4. Los ViewModels deben depender de abstracciones de acceso a la API, y la implementación con Firestore va en la API (AUD-001).
- En la API, los servicios de negocio (alto nivel) dependen de un detalle de bajo nivel, Acceso a datos, sin abstracción intermedia; eso impide sustituir Firestore por un doble de prueba (AUD-006).
- Adapter y Chain of Responsibility son los puntos del diseño que ya invierten dependencias: el cliente conoce CanalEnvio y ReglaValidacion, no sus implementaciones.
- La inyección de dependencias con Koin está planeada, pero ningún código demuestra que funcione.
- Un SessionManager global hace que los ViewModels dependan de un detalle compartido; su existencia es declarada y contradice el estado «planeado» del componente Sesión (AUD-012).

**Impacto técnico potencial:**

Mientras las dependencias apunten a detalles concretos:

- Pasar de los datos de prueba a la API obligará a modificar cada ViewModel.
- Probar los servicios de la API exigiría una base de datos Firestore real.
- Si un ViewModel se conectara directo a Firestore, las reglas de HU-15, CA-01.6 y HU-17 quedarían fuera de la API o duplicadas.

**Recomendaciones:**

- API: definir las interfaces de repositorio junto a los servicios y sus implementaciones con Firestore en Acceso a datos, inyectadas por constructor (Koin).
- Aplicación: definir repositorios por funcionalidad cuya implementación llame a la API con Ktor Client (componente «Capa de datos»), con una versión en memoria para los datos de prueba, e inyectarlos en los ViewModels por constructor.
- Reemplazar el SessionManager global por una interfaz de sesión inyectada y actualizar el estado de «Sesión» en el nivel 3 cuando se confirme en el código.
- Mantener ServicioNotificaciones dependiente de CanalEnvio y usar un adaptador falso en las pruebas.

---

## 6. Hallazgos transversales

**Problemas recurrentes identificados:**

- El análisis anterior evaluaba un modelo de clases (Usuario, Administrador, Autenticable, AsignacionTurno, SolicitudCambioTurno, MarcaAsistencia) que no está en el C4. Esta versión usa los nombres del nivel 4: Empleado y Rol, InicioSesion, Asignacion, SolicitudCambio y Marca.
- El acceso a datos es el punto más débil en las dos capas: es genérico en la API y está dentro de los ViewModels en la aplicación.
- Las reglas de negocio deben vivir en un solo lugar, la API: la validación de conflictos en el Motor, las transiciones en los estados y el envío en los adaptadores.
- Solo Chain of Responsibility, State y Adapter se usan como argumentos SOLID. Observer de la API está pendiente de validación y Factory Method fue descartado. El StateFlow de NovedadesViewModel es comportamiento del framework, no una decisión GoF del equipo.
- Ninguna evaluación está verificada en código. Quedan pendientes el corte HU-07/HU-07b, la historia del registro de disponibilidad y la sucursal del Administrador General.

---

## 7. Priorización de mejoras

| Hallazgo | Principio | Severidad | Esfuerzo | Prioridad | Acción sugerida |
|---|---|---|---|---|---|
| Acceso a datos sin abstracción en la API y datos dentro de los ViewModels | DIP | Alta | Medio | Alta | Repositorios en la API con implementación Firestore y repositorios hacia la API en la aplicación, inyectados por constructor. |
| Reglas de conflicto ubicadas en la entidad de asignación en el análisis anterior | OCP | Alta | Medio | Alta | Validar solo en MotorValidacion con una clase por regla (Chain of Responsibility). |
| Contrato de datos genérico guardar/consultar | ISP | Media | Medio | Media | Al crear los repositorios de la primera fila, dividirlos por módulo según las operaciones de cada servicio. |
| Validaciones de negocio en EmpleadosViewModel (declarado) | SRP | Media | Bajo | Media | Delegar las reglas de HU-03 y HU-04 a GestionEmpleados y dejar en el ViewModel solo el estado de la pantalla. |
| SessionManager global y vista decidida con isRolAdmin (declarado) | DIP | Media | Bajo | Media | Interfaz de sesión inyectada y opciones de pantalla derivadas de los permisos. |
| Contratos de subtipos sin verificar y sucursal del Administrador General sin decidir | LSP | Media | Bajo | Media | Pruebas de contrato para estados, reglas y adaptadores; decidir la regla de HU-03 frente a HU-07. |
| Transiciones de la solicitud de cambio sin implementar | OCP | Media | Medio | Media | Implementar State (EstadoSolicitud y EstadoFinal) en el Corte 2. |

---

## 9. Conclusión

**Síntesis del estado del diseño:** El diseño de referencia separa los procesos por servicio y aplica Chain of Responsibility, State y Adapter donde las historias lo justifican, lo que favorece SRP, OCP y DIP en esos puntos. El acceso a datos aún no está abstraído y, según el equipo, la aplicación mezcla datos y reglas en los ViewModels. Ninguna conclusión se verificó en código.

**Principios más comprometidos:** DIP (acceso a datos sin abstracciones y SessionManager global), seguido de ISP (contrato de datos genérico) y SRP (reglas en la vista).

**Principios mejor implementados:** OCP y SRP en las reglas de conflicto (Chain of Responsibility) y en los estados de la solicitud (State), y DIP en Notificaciones (CanalEnvio, Adapter); los tres como diseño planeado. LSP no se puede verificar.

**Nivel de deuda técnica asociada al diseño:** Media. El backend todavía no existe y la aplicación no está conectada, así que las correcciones pueden hacerse en el diseño antes de implementar.

**Recomendación final:** Antes de implementar la API y conectar Firestore: definir los repositorios por módulo en la API, conectar la aplicación solo a la API mediante repositorios inyectados, centralizar las reglas en el Motor de validación y resolver las decisiones pendientes (HU-07/HU-07b, historia de disponibilidad y sucursal del Administrador General).

---

## Versión corta para revisión rápida

| Principio | ¿Se cumple? | Evidencia | Problema detectado | Recomendación |
|---|---|---|---|---|
| SRP | Parcial | Servicios separados por proceso en el nivel 4; EmpleadosViewModel valida (declarado). | Reglas de HU-03 en la vista. | Delegar las reglas a GestionEmpleados. |
| OCP | Parcial | MotorValidacion con 7 reglas y State en el diseño. | Validación ubicada en la entidad; cargos como texto (declarado). | Validar solo en el Motor y usar Rol. |
| LSP | No verificable | Sin herencia de usuarios en el C4; contratos de EstadoSolicitud, ReglaValidacion y CanalEnvio solo en diseño. | Sucursal del Administrador General sin decidir. | Pruebas de contrato y decisión sobre HU-03. |
| ISP | Parcial | CanalEnvio cohesionada; Acceso a datos genérico. | Contrato de datos único. | Repositorio por módulo. |
| DIP | No | Servicios dependen de Acceso a datos; datos en los ViewModels y SessionManager global (declarado). | Alto nivel depende de detalles. | Repositorios inyectados; Firestore solo en la API. |
