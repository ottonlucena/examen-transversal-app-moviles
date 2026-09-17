# AvistAves — Informe técnico y demostración

**Asignatura:** Desarrollo de Aplicaciones Móviles
**Institución:** Instituto Profesional San Sebastián
**Equipo:** Otton Lucena y Valeria Gómez
**Framework:** React Native con Expo (SDK 57), TypeScript
**Repositorio:** `https://github.com/ottonlucena/examen-transversal-app-moviles`

---

## Índice

1. [El producto](#1-el-producto)
2. [Arquitectura del framework elegido](#2-arquitectura-del-framework-elegido)
3. [Patrones de diseño presentes en el framework y en nuestro código](#3-patrones-de-diseño-presentes-en-el-framework-y-en-nuestro-código)
4. [Comparación con otros dos frameworks](#4-comparación-con-otros-dos-frameworks)
5. [Optimización del consumo de la API](#5-optimización-del-consumo-de-la-api)
6. [Demostración de la aplicación](#6-demostración-de-la-aplicación)
7. [Declaración del uso de inteligencia artificial](#7-declaración-del-uso-de-inteligencia-artificial)
8. [Estado de verificación y limitaciones conocidas](#8-estado-de-verificación-y-limitaciones-conocidas)

---

## 1. El producto

AvistAves permite a los voluntarios de la Red de Observadores de Aves dejar constancia de un
avistamiento en terreno: qué vieron, dónde, con qué evidencia fotográfica y bajo qué condiciones
climáticas. Son tres pantallas: el listado de avistamientos, el formulario de registro y el detalle
de un avistamiento.

La aplicación se diseñó para su contexto real de uso, que no es un escritorio sino el campo abierto:
se usa de pie, con una mano, con sol directo sobre la pantalla y, con frecuencia, sin cobertura. De
ahí tres decisiones que atraviesan todo el producto:

- **Contraste alto y áreas de toque grandes.** Todo elemento pulsable mide al menos 48 puntos de
  alto y el texto principal mantiene una relación de contraste de 14,8:1 sobre el fondo.
- **Ninguna espera sin explicación.** Cada operación asíncrona (cámara, GPS, clima, lectura del
  almacenamiento) declara sus estados de carga, error y vacío, y ninguna puede girar de forma
  indefinida: todas tienen un tiempo máximo acotado.
- **La falta de red nunca impide registrar.** El clima es un dato deseable, no un requisito. Si la
  API no responde, el avistamiento se guarda igual y se marca explícitamente que no se pudo obtener
  el clima.

### Dónde se evalúa cada cosa

| Indicador | Dónde se evalúa | Máx |
|---|---|---|
| Conceptos del framework | Informe, sección 2 | 10 |
| Patrones de diseño | Informe, sección 3 | 12 |
| Comparación de frameworks | Informe, sección 4 | 12 |
| Principios de diseño de UI | RF-01 a RF-04 | 12 |
| Componentes de UI | RF-01, RF-03, RF-06 | 12 |
| Interfaces intuitivas | Estados, permisos y validaciones | 12 |
| Uso de periféricos | RF-01 (cámara y GPS) | 10 |
| Integración con la API | RF-02, RF-04 | 10 |
| Optimización de la API | Informe, sección 5 | 10 |

### Reparto del trabajo

El trabajo se organizó en dos frentes que avanzaron en paralelo. Las decisiones que
condicionaban la entrega las tomamos en conjunto: el framework, el router, la estrategia de
persistencia, el tercer framework de la comparación y el criterio de ordenamiento del listado.

Valeria Gómez se encargó de revisar y validar el informe, de contrastar que lo escrito
coincidiera con lo que la aplicación hace, y de probar la aplicación frente a los criterios de
aceptación definidos en BRIEF.md. Otton Lucena llevó la construcción del proyecto y la
ejecución de las fases de desarrollo.

Ambos revisamos el resultado final antes de la entrega.

---

## 2. Arquitectura del framework elegido

### 2.1 Qué es React Native y qué no es

React Native no es un navegador embebido ni un generador de código nativo. Es un framework que
ejecuta código JavaScript en un motor propio dentro de la aplicación y, a partir de la descripción
declarativa que ese código produce, **instancia y manipula componentes de interfaz nativos reales**
del sistema operativo. Cuando el programador escribe `<Text>`, lo que termina en pantalla en Android
es un `TextView` del sistema, no una imitación dibujada.

Esa distinción es la que explica casi todo lo demás: el rendimiento de la interfaz es el del sistema
operativo, pero a cambio hay que resolver el problema de comunicar dos mundos que hablan lenguajes
distintos, JavaScript y el código nativo de la plataforma.

### 2.2 Las piezas

**El motor de JavaScript: Hermes.** React Native incorpora Hermes, un motor de JavaScript
desarrollado específicamente para móviles. A diferencia de un motor de navegador, Hermes precompila
el código a *bytecode* durante la construcción de la aplicación, de modo que en el arranque no hay
que analizar ni compilar el código fuente. El efecto práctico es un tiempo de arranque menor y un
consumo de memoria más bajo, que en un dispositivo de gama media es exactamente donde se nota.

**La capa de interoperabilidad: JSI.** JSI (*JavaScript Interface*) es una interfaz en C++ que
permite que el código JavaScript tenga referencias directas a objetos nativos y los invoque de forma
síncrona. Es el reemplazo del antiguo *bridge*, que serializaba cada mensaje a JSON y lo enviaba de
forma asíncrona por una cola. La documentación oficial de arquitectura señala la reducción de esa
sobrecarga de serialización como una de las motivaciones centrales del rediseño.

**El renderizador: Fabric.** Fabric es el mecanismo que permite a React renderizar sobre vistas del
sistema anfitrión en lugar de nodos del DOM. Mantiene un núcleo compartido en C++ entre Android e
iOS, lo que reduce las divergencias de comportamiento entre plataformas, y genera código con
verificación de tipos de forma automática. Su ciclo tiene tres fases:

1. **Render.** React ejecuta los componentes y construye un árbol de nodos inmutable.
2. **Commit.** Se calcula la disposición y el árbol nuevo se promueve como el árbol vigente. Los
   nodos que no cambiaron se comparten en lugar de recrearse.
3. **Mount.** Las diferencias entre el árbol anterior y el nuevo se aplican sobre las vistas nativas.

Una optimización propia de esta fase es el **aplanado de vistas**: si un contenedor solo existe para
agrupar y no aporta nada visual, Fabric lo elimina de la jerarquía nativa. Un árbol de componentes
profundo en JavaScript no se traduce necesariamente en una jerarquía de vistas profunda en Android.

**Los módulos nativos: TurboModules.** Son la vía por la que el código JavaScript accede a
funcionalidad nativa (cámara, GPS, sistema de archivos). A diferencia del esquema antiguo, se cargan
de forma perezosa: un módulo que la aplicación nunca usa no se inicializa y no ocupa memoria.

**La disposición: Yoga.** React Native no usa el motor de disposición de Android. Usa Yoga, una
implementación de Flexbox en C++ compartida entre plataformas, que corre en su propio hilo. Por eso
un `flexDirection: 'row'` se comporta igual en Android que en iOS.

**El modelo de hilos.** El trabajo se reparte entre el hilo de JavaScript, el hilo de disposición y
el hilo principal de la interfaz. Que la disposición se calcule fuera del hilo principal es lo que
evita que una lista compleja bloquee el desplazamiento.

Desde React Native 0.76 esta arquitectura es la predeterminada. Este proyecto usa **React Native
0.86.3**, de modo que se ejecuta sobre ella.

### 2.3 Qué rol cumple Expo

Expo se confunde a menudo con React Native, y no son lo mismo. React Native es el framework. Expo es
una plataforma construida **encima** que resuelve los problemas que React Native deja abiertos.
Aporta cuatro cosas distintas:

**Un SDK de módulos nativos ya escritos y mantenidos.** `expo-camera`, `expo-location`,
`expo-file-system` y el resto son TurboModules con implementación en Kotlin para Android y en Swift
para iOS, expuestos a través de una API JavaScript uniforme. Sin Expo, acceder a la cámara implica
escribir o integrar código nativo, declarar permisos en el manifiesto y gestionar el ciclo de vida
de la actividad de Android a mano.

**`expo-modules-core`, la infraestructura común.** Es la capa que estandariza cómo un módulo nativo
se registra, cómo declara sus funciones y cómo convierte tipos entre JavaScript y el lenguaje
nativo. Es lo que hace que todos los módulos de Expo se usen igual.

**Los *config plugins*, que eliminan la edición manual de archivos nativos.** En lugar de abrir
`AndroidManifest.xml` para declarar el permiso de cámara, se declara la intención en `app.json`:

```json
[
  "expo-camera",
  {
    "cameraPermission": "AvistAves necesita la cámara para que puedas fotografiar el ave en el momento del avistamiento. La fotografía se guarda solo en tu dispositivo.",
    "recordAudioAndroid": false
  }
]
```

Durante la generación del proyecto nativo, el plugin de `expo-camera` añade
`android.permission.CAMERA` al manifiesto. El `recordAudioAndroid: false` es una decisión
deliberada: sin él, el plugin también declararía `android.permission.RECORD_AUDIO`, y nuestra
aplicación no graba video. Pedir un permiso que no se necesita es una mala práctica y aquí se evita
con una línea de configuración.

**El entorno de desarrollo y ejecución.** Expo aporta el empaquetador (Metro), el servidor de
desarrollo con recarga en caliente, y Expo Go, una aplicación contenedora que ya incluye compilados
todos los módulos del SDK. Como AvistAves usa exclusivamente módulos del SDK, se ejecuta dentro de
Expo Go sin compilar nada nativo: el ciclo de trabajo es recargar, no recompilar. Con menos de
cuarenta y ocho horas de plazo, eliminar la compilación nativa del ciclo fue la decisión de mayor
impacto sobre el riesgo del proyecto.

**El router.** `expo-router` deriva la navegación de la estructura de la carpeta `src/app/`: cada
archivo es una ruta. Por debajo monta un stack nativo sobre `react-native-screens`, de modo que las
transiciones y el botón atrás del sistema Android son los nativos, no una imitación.

### 2.4 Cómo se ve esta arquitectura en nuestro proyecto

| Pieza del framework | Dónde aparece en AvistAves |
|---|---|
| Componentes nativos vía Fabric | `FlatList` en `src/app/index.tsx` se traduce en un `RecyclerView` del sistema |
| TurboModule de cámara | `CameraView` en `src/componentes/CapturadorFoto.tsx` |
| TurboModule de ubicación | `Location.getCurrentPositionAsync` en `src/servicios/servicioUbicacion.ts` |
| TurboModule de archivos | `File`, `Directory` y `Paths` en `src/servicios/servicioFotos.ts` |
| Yoga y Flexbox | Todos los `StyleSheet.create` del proyecto |
| Config plugins | Bloque `plugins` de `app.json` |
| Router basado en archivos | Estructura de `src/app/` |

### 2.5 Una nota sobre la arquitectura observada en la práctica

Durante el desarrollo apareció un detalle que ilustra que estas capas son reales y no una
abstracción de manual. La plantilla de Expo SDK 57 genera `app.json` con el **React Compiler**
activado. El React Compiler es un compilador que reescribe los componentes de React para memorizar
resultados de forma automática, insertando llamadas a hooks adicionales durante la transformación.
Con esa opción activa, la aplicación fallaba en ejecución con el error "React has detected a change
in the order of Hooks", señalando una llamada a `useState` perfectamente correcta, y el listado
quedaba colgado en su estado de carga indefinidamente.

Desactivar el React Compiler resolvió el fallo sin modificar una sola línea de código de la
aplicación. El episodio deja una lección concreta: el código que se escribe no es el código que se
ejecuta, y entender qué transformaciones ocurren entre uno y otro es parte de trabajar con el
framework.

---

## 3. Patrones de diseño presentes en el framework y en nuestro código

Los tres patrones que siguen no fueron buscados después de escribir la aplicación. Se decidieron
antes de empezar, junto con el archivo donde cada uno debía quedar visible, y el código se escribió
para exhibirlos.

### 3.1 Observador (Observer)

**En el framework.** El modelo de renderizado de React es una implementación de publicación y
suscripción. Un componente que lee una porción de estado queda suscrito a ella; cuando ese estado
cambia, React notifica a los suscriptores volviéndolos a renderizar. La Context API generaliza el
mecanismo a un árbol completo de componentes: el proveedor publica y cada `useContext` es una
suscripción.

**En nuestro código.** El sujeto observable es `src/contexto/ContextoAvistamientos.tsx`. Mantiene la
colección de avistamientos y la publica hacia el árbol:

```tsx
// src/contexto/ContextoAvistamientos.tsx
export function ProveedorAvistamientos({ children }: { children: ReactNode }) {
  const [avistamientos, setAvistamientos] = useState<Avistamiento[]>([]);
  const [cargando, setCargando] = useState(true);
  const [error, setError] = useState<string | null>(null);

  const agregar = useCallback(async (avistamiento: Avistamiento) => {
    // Se publica lo que devuelve el repositorio, no una versión optimista:
    // así el estado en memoria y el almacén no pueden divergir.
    setAvistamientos(await repositorio.guardarAvistamiento(avistamiento));
  }, []);

  return (
    <ContextoAvistamientos.Provider value={valor}>{children}</ContextoAvistamientos.Provider>
  );
}
```

Los observadores se suscriben a través de un hook propio, que además falla de forma explícita si
alguien lo usa fuera del proveedor:

```ts
// src/hooks/usarAvistamientos.ts
export function usarAvistamientos(): EstadoAvistamientos {
  const contexto = useContext(ContextoAvistamientos);

  if (!contexto) {
    throw new Error('usarAvistamientos debe usarse dentro de ProveedorAvistamientos');
  }

  return contexto;
}
```

**Por qué esto es el patrón y no simplemente estado compartido.** La prueba está en lo que *no*
existe en el código. Cuando el formulario `src/app/registro.tsx` guarda un avistamiento, llama a
`agregar` y navega al listado. En ningún punto le comunica nada a `src/app/index.tsx`. No hay una
llamada, un evento ni una referencia entre ambas pantallas: el formulario ni siquiera sabe que el
listado existe. El listado se actualiza porque estaba suscrito al mismo sujeto. Ese
desacoplamiento entre quien publica el cambio y quien reacciona a él es exactamente lo que define al
Observador.

### 3.2 Fachada (Facade)

**En el framework.** Expo es, en su conjunto, una fachada. Cada módulo del SDK expone una superficie
JavaScript pequeña y uniforme que oculta dos implementaciones nativas distintas, la gestión de
permisos y el ciclo de vida de la actividad de Android. `Location.getCurrentPositionAsync()` es una
sola llamada; por detrás hay un `FusedLocationProviderClient` de Google Play Services, una petición
de permisos y una traducción de tipos entre Kotlin y JavaScript.

**En nuestro código.** Aplicamos el mismo patrón una capa más arriba. `src/servicios/servicioClima.ts`
expone **una** función pública, `obtenerClimaPara(latitud, longitud)`, y detrás de ella esconde
cinco responsabilidades: consulta de caché, construcción de la URL, petición con tiempo de espera y
reintento, validación de la respuesta y traducción del código meteorológico.

```ts
// src/servicios/servicioClima.ts
export async function obtenerClimaPara(
  latitud: number,
  longitud: number,
): Promise<ResultadoClima> {
  const cacheado = await obtenerDeCache(latitud, longitud);
  if (cacheado) {
    return { clima: cacheado, error: null, desdeCache: true };
  }

  try {
    const crudo = await obtenerJson(construirUrl(latitud, longitud));

    if (!esRespuestaValida(crudo)) {
      return {
        clima: null,
        error: 'El servicio de clima devolvió datos que no se pudieron interpretar.',
        desdeCache: false,
      };
    }

    const clima = adaptarRespuesta(crudo);
    await guardarEnCache(latitud, longitud, clima);

    return { clima, error: null, desdeCache: false };
  } catch (error) {
    const detalle = error instanceof Error ? error.message : 'Error desconocido';
    return {
      clima: null,
      error: `No se pudo obtener el clima: ${detalle.toLowerCase()}. Puedes guardar igual.`,
      desdeCache: false,
    };
  }
}
```

Nótese que la función **no lanza nunca**. Devuelve `clima: null` y un mensaje ya redactado en
español. Esa decisión es lo que garantiza, desde el diseño y no desde la disciplina de quien
programa, que un fallo de red no pueda impedir el registro de un avistamiento.

**Cómo se verifica que la fachada es real.** Una fachada que se filtra no es una fachada. La regla
del proyecto es que ninguna pantalla importe una biblioteca externa, y se comprueba de forma
mecánica:

```bash
grep -rn "expo-camera\|expo-location\|expo-file-system\|async-storage" src/app/ src/componentes/ src/hooks/
grep -rn "fetch(" src/app/ src/componentes/
```

El segundo comando no devuelve nada: ninguna pantalla llama a la red. El primero devuelve
exactamente dos líneas, que son las dos excepciones documentadas, y ambas existen por la misma razón
técnica: **React tiene construcciones que no se pueden envolver en una función.** `CameraView` es un
componente y `useForegroundPermissions` es un hook; ambos solo pueden invocarse desde el árbol de
React, de modo que meterlos en un servicio no es una cuestión de disciplina, es imposible. Su
alcance está acotado y documentado en `AGENTS.md`.

### 3.3 Compuesto (Composite)

**En el framework.** El árbol de componentes de React Native es un compuesto en el sentido clásico
del patrón. Un componente hoja (`Text`) y un componente contenedor (`View`, o uno propio que agrupa
otros) comparten exactamente la misma interfaz: reciben props y devuelven elementos. El cliente que
los usa no distingue entre una hoja y un compuesto, y esa uniformidad es la que permite anidarlos de
forma recursiva hasta llegar a la jerarquía de vistas nativas de Android.

**En nuestro código.** `src/componentes/TarjetaAvistamiento.tsx` es simultáneamente contenedor y
hoja, según desde dónde se mire. Como contenedor, compone la miniatura, tres bloques de texto y otro
componente propio, `InsigniaClima`:

```tsx
// src/componentes/TarjetaAvistamiento.tsx
<Pressable
  accessibilityRole="button"
  accessibilityLabel={`Ver detalle de ${avistamiento.nombreAve}`}
  onPress={() => onPress(avistamiento.id)}
  style={({ pressed }) => [estilos.tarjeta, pressed && estilos.presionada]}>
  <Image
    // La ruta se reconstruye en cada lectura desde el nombre persistido.
    source={{ uri: resolverUriFoto(avistamiento.fotoNombreArchivo) }}
    style={estilos.miniatura}
    contentFit="cover"
    transition={150}
  />

  <View style={estilos.datos}>
    <Text style={estilos.nombre} numberOfLines={1}>
      {avistamiento.nombreAve}
    </Text>
    <Text style={estilos.fecha}>{formatearFechaCorta(avistamiento.fechaAvistamiento)}</Text>
    <Text style={estilos.cantidad}>{formatearCantidad(avistamiento.cantidad)}</Text>
    <InsigniaClima clima={avistamiento.clima} compacta />
  </View>
</Pressable>
```

Como hoja, la `FlatList` de `src/app/index.tsx` la trata igual que trataría a un `Text`. No sabe ni
necesita saber que dentro hay una imagen, cuatro textos y un icono:

```tsx
// src/app/index.tsx
<FlatList
  data={ordenados}
  keyExtractor={(item) => item.id}
  renderItem={({ item }) => (
    <TarjetaAvistamiento avistamiento={item} onPress={irAlDetalle} />
  )}
/>
```

`InsigniaClima`, a su vez, es contenedor respecto del icono y el texto que agrupa. La composición es
recursiva y en ningún nivel cambia la interfaz.

### 3.4 Adaptador (Adapter)

Se incluye como cuarto patrón porque el archivo existe de todos modos y el contraste con la Fachada
resulta ilustrativo.

**En el framework.** `expo-modules-core` adapta los tipos nativos de Kotlin y Swift a los tipos de
JavaScript, de modo que un objeto nativo pueda consumirse con la interfaz que el código JavaScript
espera.

**En nuestro código.** Open-Meteo devuelve una estructura que no controlamos, con nombres en inglés
y un código meteorológico numérico. `adaptarRespuesta`, en `src/servicios/servicioClima.ts`, la
traduce al tipo de dominio `ClimaRegistrado`:

```ts
// src/servicios/servicioClima.ts
function adaptarRespuesta(respuesta: RespuestaOpenMeteo): ClimaRegistrado {
  const actual = respuesta.current;
  const traduccion = traducirCodigoClima(actual.weather_code);

  return {
    temperaturaC: actual.temperature_2m,
    humedadRelativa: actual.relative_humidity_2m,
    vientoKmh: actual.wind_speed_10m,
    codigoClima: actual.weather_code,
    descripcion: traduccion.descripcion,
    icono: traduccion.icono,
    obtenidoEn: new Date().toISOString(),
  };
}
```

La diferencia con la Fachada es de propósito, no de forma: la Fachada **simplifica** un subsistema
complejo; el Adaptador **traduce** entre dos interfaces incompatibles. Aquí el adaptador es lo que
permite que si mañana cambiáramos de proveedor de clima, solo este archivo tendría que cambiar: el
resto de la aplicación conoce `ClimaRegistrado`, no conoce Open-Meteo.

---

## 4. Comparación con otros dos frameworks

### 4.1 El eje de la comparación

Comparar frameworks enumerando ventajas produce un catálogo, no una comparación. Para que la
comparación sea tal, los tres se examinan sobre un mismo eje, elegido de forma explícita: **cómo
llega el código que escribe el desarrollador hasta la pantalla y hasta los periféricos del
dispositivo.**

Ese eje es el adecuado porque los tres frameworks resuelven el mismo problema con tres respuestas
estructuralmente distintas, y prácticamente todas sus fortalezas y debilidades se derivan de esa
diferencia inicial.

| Framework | Cómo se dibuja la interfaz | Cómo se accede al hardware | Lenguaje |
|---|---|---|---|
| React Native + Expo | Instancia **componentes nativos reales** del sistema | TurboModules a través de JSI | JavaScript / TypeScript |
| Ionic + Capacitor | **HTML y CSS dentro de un WebView**; los componentes imitan la apariencia nativa | Plugins de Capacitor que puentean el WebView con el código nativo | JavaScript / TypeScript |
| Flutter | **Dibuja cada píxel** en su propio lienzo con el motor Impeller | *Platform channels*, mensajería asíncrona entre Dart y el código nativo | Dart |

### 4.2 React Native con Expo

**Fortalezas.** Al instanciar componentes del sistema, la aplicación hereda gratis el aspecto, el
comportamiento y la accesibilidad de la plataforma: el desplazamiento, el teclado, el botón atrás y
las animaciones de navegación son los de Android, no una reproducción. El ecosistema de JavaScript
es reutilizable casi por completo, lo que en un equipo que ya conoce el lenguaje elimina la curva de
entrada. Expo, además, suprime la configuración nativa: los permisos se declaran en `app.json` y los
periféricos vienen resueltos.

**Debilidades.** La capa de interoperabilidad sigue siendo un punto de fricción cuando el volumen de
datos que cruza entre JavaScript y el código nativo es alto; JSI lo redujo de forma sustancial
respecto del antiguo *bridge*, pero no lo eliminó. La dependencia del ciclo de versiones del SDK de
Expo condiciona las actualizaciones: los paquetes deben instalarse con `npx expo install`, que fija
la versión compatible con el SDK, y actualizar el SDK implica revisar todo el conjunto. Por último,
el entorno gestionado tiene un límite: si el proyecto necesita código nativo propio, hay que generar
los proyectos nativos y asumir su mantenimiento.

**Dónde lo vivimos.** En este proyecto la fortaleza se notó en que cámara, GPS, sistema de archivos
y almacenamiento estuvieron operativos el primer día sin tocar Gradle. La debilidad se notó en la
rigidez de versiones: la versión de AsyncStorage que corresponde al SDK 57 es la 2.2.0, mientras que
el registro de npm publica la 3.1.1 como última; instalarla con `npm install` habría roto la
compilación.

### 4.3 Ionic con Capacitor

**Fortalezas.** Es la curva de entrada más corta de los tres: quien sabe desarrollo web ya sabe
Ionic, porque la interfaz es literalmente HTML, CSS y JavaScript. Permite una única base de código
para web, Android e iOS con un grado de reutilización que los otros dos no alcanzan. Capacitor,
además, no obliga a adoptar un framework de interfaz concreto: funciona con Angular, React, Vue o
sin ninguno.

**Debilidades.** El WebView impone un techo de rendimiento que no se puede superar con optimización
del código de la aplicación: en listas largas, desplazamiento continuo y animaciones complejas, la
diferencia con una interfaz nativa es perceptible. La apariencia nativa es una imitación mantenida a
mano, de modo que cada cambio de diseño de una versión de Android obliga a actualizar la biblioteca
de componentes para no quedar desfasado. Y el acceso a periféricos añade un salto adicional: el
código corre en el WebView, el plugin lo puentea hacia el código nativo y la respuesta vuelve por el
mismo camino.

**Por qué no lo elegimos.** El caso de uso de AvistAves es un listado con miniaturas y una vista de
cámara en vivo, que son precisamente las dos cosas en las que el WebView rinde peor. Además, la
pauta evalúa el uso de periféricos, y el camino hacia la cámara es más largo y más frágil en este
modelo.

### 4.4 Flutter

**Fortalezas.** Al dibujar cada píxel en su propio lienzo, Flutter obtiene un control total sobre la
interfaz: la aplicación se ve exactamente igual en Android y en iOS, sin las divergencias que
aparecen cuando se usan componentes de cada sistema. El código Dart se compila a código nativo
anticipadamente, sin motor de JavaScript intermedio, lo que le da el rendimiento más alto y más
predecible de los tres. Su catálogo de widgets es el más completo de los tres.

**Debilidades.** Exige aprender Dart, un lenguaje con mucho menor alcance fuera del desarrollo
móvil, lo que significa que el conocimiento se transfiere peor a otros contextos y que el
reclutamiento es más estrecho. Al no usar componentes del sistema, no hereda automáticamente los
cambios de diseño de cada versión de Android: la coherencia visual con el sistema depende de que el
equipo de Flutter actualice sus widgets. El tamaño base del binario es mayor, porque la aplicación
carga su propio motor de renderizado.

**Por qué no lo elegimos.** El equipo no conoce Dart, y con menos de cuarenta y ocho horas de plazo
el costo de aprender un lenguaje nuevo habría salido íntegro del tiempo destinado a los
requerimientos funcionales.

### 4.5 Síntesis

Ninguno de los tres es superior en abstracto; cada uno gana en un escenario distinto.

| Si el proyecto prioriza... | Conviene |
|---|---|
| Reutilizar una base de código web existente y salir rápido | Ionic + Capacitor |
| Rendimiento gráfico máximo e interfaz idéntica entre plataformas | Flutter |
| Aspecto y comportamiento nativos con un equipo que ya sabe JavaScript | React Native + Expo |

Para AvistAves, la combinación de restricciones fue decisiva: un plazo inferior a cuarenta y ocho
horas, un equipo que ya programa en JavaScript, la necesidad de cámara y GPS reales, y una
demostración sobre emulador. React Native con Expo es el único de los tres que satisface las cuatro
a la vez. Ionic habría penalizado justo los dos elementos que la pauta evalúa con más peso, y
Flutter habría consumido en aprendizaje el tiempo que hacía falta para construir.

---

## 5. Optimización del consumo de la API

Se implementaron dos medidas exigibles y una tercera de refuerzo. Las tres son localizables en un
archivo concreto del repositorio.

### 5.1 Medida 1: caché por ubicación con vencimiento de 15 minutos

**Archivo:** `src/servicios/cacheClima.ts`

Antes de salir a la red, el servicio de clima consulta una caché persistida en AsyncStorage. La
clave es el par de coordenadas redondeado a dos decimales, lo que agrupa en una misma celda todo lo
que ocurra dentro de un radio aproximado de 1,1 kilómetros.

```ts
// src/servicios/cacheClima.ts
export const VIGENCIA_CACHE_MS = 15 * 60 * 1000;
const DECIMALES_CELDA = 2;

export function calcularClaveCelda(latitud: number, longitud: number): string {
  return `${latitud.toFixed(DECIMALES_CELDA)},${longitud.toFixed(DECIMALES_CELDA)}`;
}

export async function obtenerDeCache(
  latitud: number,
  longitud: number,
): Promise<ClimaRegistrado | null> {
  const cache = await leerCache();
  const entrada = cache[calcularClaveCelda(latitud, longitud)];

  if (!entrada) return null;
  if (Date.now() - entrada.guardadoEn > VIGENCIA_CACHE_MS) return null;

  return entrada.clima;
}
```

**El vencimiento no es arbitrario.** La propia respuesta de Open-Meteo lo justifica. Consultando el
endpoint:

```json
"current": {
  "time": "2026-09-16T02:15", "interval": 900,
  "temperature_2m": 17.8, "relative_humidity_2m": 62,
  "wind_speed_10m": 1.5, "weather_code": 0
}
```

El campo `interval` vale 900 segundos: la API recalcula el dato cada quince minutos. Consultar antes
de que venza ese plazo devuelve exactamente el mismo valor, de modo que la petición no aportaría
información nueva. El vencimiento de la caché está fijado en ese número y no en una intuición.

**Efecto medible.** Un voluntario que registra varios avistamientos en la misma zona durante una
salida de terreno, que es el escenario real de uso, consume **una** petición en lugar de una por
avistamiento.

### 5.2 Medida 2: tiempo de espera de 6 segundos con un único reintento

**Archivo:** `src/servicios/clienteHttp.ts`

Toda petición se envía con un `AbortController` que la cancela a los 6 segundos.

```ts
// src/servicios/clienteHttp.ts
export const TIEMPO_ESPERA_MS = 6000;
export const ESPERA_ENTRE_INTENTOS_MS = 1000;
export const INTENTOS_MAXIMOS = 2;

async function peticionConTimeout(url: string): Promise<unknown> {
  const controlador = new AbortController();
  const temporizador = setTimeout(() => controlador.abort(), TIEMPO_ESPERA_MS);

  try {
    const respuesta = await fetch(url, { signal: controlador.signal });

    if (!respuesta.ok) {
      // 5xx es un fallo del servidor y puede resolverse solo; 4xx no.
      const esReintentable = respuesta.status >= 500;
      throw new ErrorHttp(
        `El servicio respondió con estado ${respuesta.status}`,
        respuesta.status,
        esReintentable,
      );
    }

    return await respuesta.json();
  } catch (error) {
    if (error instanceof Error && error.name === 'AbortError') {
      throw new ErrorHttp('El servicio tardó demasiado en responder', null, true);
    }
    throw new ErrorHttp('No hay conexión con el servicio', null, true);
  } finally {
    clearTimeout(temporizador);
  }
}
```

**Por qué un tiempo de espera propio.** Sin él, una petición en una red móvil deficiente puede
quedarse colgada más de un minuto antes de que el sistema la corte, con el indicador de actividad
girando y el usuario sin saber qué ocurre. Abortarla a los 6 segundos libera el socket y la interfaz
de inmediato.

**Por qué un solo reintento y solo ante fallo de red.** Repetir una petición mal formada, que es lo
que indica un `4xx`, no la va a arreglar: solo gasta batería y datos móviles del voluntario. Por eso
el reintento se reserva para los errores que pueden resolverse solos, que son los de red, los
vencimientos de tiempo y los `5xx`. El costo máximo total queda acotado y es conocido: 6 + 1 + 6 =
13 segundos, tras los cuales el avistamiento se guarda sin clima.

### 5.3 Medida 3 de refuerzo: renderizado eficiente del listado

**Archivos:** `src/app/index.tsx` y `src/componentes/TarjetaAvistamiento.tsx`

Tres decisiones concretas:

```tsx
// src/app/index.tsx — el orden se recalcula solo cuando cambian los datos o el criterio
const ordenados = useMemo(
  () => ordenarAvistamientos(avistamientos, orden),
  [avistamientos, orden],
);
```

```tsx
// src/componentes/TarjetaAvistamiento.tsx — la tarjeta no se vuelve a renderizar
// si sus datos no cambiaron
export default memo(TarjetaAvistamiento);
```

Además, la `FlatList` usa `keyExtractor` estable por `id` y `removeClippedSubviews`. El efecto
conjunto: cambiar el criterio de ordenamiento reordena el arreglo, pero las tarjetas cuyos datos no
cambiaron no se vuelven a renderizar ni recargan su imagen.

Se presenta como refuerzo y no como una de las dos medidas exigidas porque optimiza el renderizado
de la aplicación, no el consumo de la API.

---

## 6. Demostración de la aplicación

La demostración se realizó en dos entornos. Las secciones 6.1 a 6.8 corresponden al **emulador de
Android Pixel 8, API 34 (Android 14)**, imagen `google_apis_playstore`, sobre Linux Mint, donde la
ubicación se inyecta con los controles del emulador (`adb emu geo fix -70.6693 -33.4489`,
coordenadas de Santiago) y la cámara usa la escena virtual. La sección 6.9 corresponde a un
**dispositivo Android físico**, con cámara, GPS y conexión reales.

### 6.1 Verificación previa del entorno

Antes de construir la aplicación se comprobó que el emulador entregaba efectivamente cada periférico,
para no descubrir un bloqueo al final del plazo.

![Verificación del entorno](docs/evidencia-f0-sensores-ok.png)

GPS entregando coordenadas reales, `reverseGeocodeAsync` devolviendo una dirección legible y
Open-Meteo respondiendo con temperatura, humedad y código meteorológico.

Esta comprobación reveló un hallazgo que cambió el código definitivo: **`expo-location` exige
`Accuracy.High`**. Con `Accuracy.Balanced` la petición se resuelve por el proveedor *fused* de Google
Play Services, que usa red y wifi, y que en el emulador no recibe datos: la promesa no se resolvía
nunca. El diagnóstico se confirmó con `adb shell dumpsys location`, que mostraba el proveedor GPS en
`ProviderRequest[OFF]` y `mStarted=false`. La razón quedó documentada en el propio código.

### 6.2 Estado vacío del listado

![Estado vacío](docs/evidencia-estado-vacio.png)

RF-03 exige un estado vacío diseñado, no una pantalla en blanco: icono, título, explicación de qué
hacer y acción directa hacia el formulario.

### 6.3 Formulario de registro con ubicación y clima capturados

![Formulario de registro](docs/evidencia-formulario-registro.png)

Al abrir el formulario, la ubicación se captura **automáticamente**, sin que el usuario intervenga.
La dirección se muestra como texto legible y las coordenadas quedan como dato secundario, en modo
solo lectura: el usuario nunca escribe coordenadas a mano. El clima aparece ya traducido a
"19 grados · Despejado", nunca como el código numérico que devuelve la API.

La anotación "Dato reciente de esta zona" es la caché de la sección 5.1 funcionando: esa consulta no
tocó la red.

![Resto del formulario](docs/evidencia-formulario-completo.png)

El resto del formulario. La fecha y la hora vienen precargadas con el instante de apertura y son
editables; la cantidad de ejemplares viene precargada en 1 y solo admite enteros; las notas son
opcionales y así se indica.

Esta pantalla también reveló un defecto real del geocodificador de Android, corregido tras
inspeccionar la respuesta cruda: `reverseGeocodeAsync` devuelve el campo `streetNumber` con el código
postal pegado detrás (`"23, 8370042"`), de modo que concatenar los campos sin filtrarlos producía
direcciones como "Virginia Opazo 23, 8370042, Santiago". La dirección se arma ahora con campos
explícitos y saneados.

### 6.4 Validación antes de guardar

![Validación por campo](docs/evidencia-validacion-campos.png)

Al intentar guardar sin fotografía y sin nombre del ave, la aplicación no guarda y señala cada campo
faltante **junto al campo afectado**, no solo con una alerta global. El campo de nombre queda además
marcado con borde rojo.

### 6.5 Registro guardado y listado

![Guardado y listado](docs/evidencia-guardado-y-listado.png)

Al guardar correctamente, la aplicación confirma la operación y vuelve al listado, donde el
avistamiento recién creado aparece en primera posición. La tarjeta muestra miniatura, nombre, fecha
y temperatura registrada. Arriba, el selector de ordenamiento con sus tres criterios.

La navegación de vuelta descarta el formulario con `router.back()`, de modo que pulsar atrás desde
el listado sale de la aplicación en lugar de reabrir el formulario o mostrar un listado duplicado.

La primera implementación usaba `router.replace('/')`, siguiendo la idea de reemplazar el formulario
por el listado. La auditoría en el emulador mostró que no hace eso: `replace` no sustituye la
entrada del listado que ya estaba debajo en la pila, sino que apila una segunda encima. El síntoma
era una flecha de retroceso en la cabecera del listado, y pulsar atrás devolvía al mismo listado en
lugar de salir de la aplicación. Con `back()` se descarta el formulario y queda a la vista el
listado que ya existía.

#### Ordenamiento del listado

RF-03 exige un filtro o un ordenamiento a elección. Se implementaron **tres** criterios
intercambiables, definidos en `src/utilidades/estrategiasOrden.ts` como funciones de comparación:
el listado no sabe cómo se ordena, solo elige una clave.

![Orden por fecha](docs/evidencia-orden-por-fecha.png)

Orden por defecto, del avistamiento más reciente al más antiguo, como exige RF-03.

![Orden por cantidad](docs/evidencia-orden-por-cantidad.png)

El mismo conjunto ordenado por cantidad de ejemplares. El criterio activo queda marcado
visualmente y el listado se reordena de inmediato.

Añadir un criterio nuevo consiste en añadir una entrada a ese archivo, sin tocar la pantalla del
listado.

### 6.6 Detalle del avistamiento

![Detalle](docs/evidencia-detalle.png)

Fotografía en tamaño grande, todos los datos del avistamiento, y dos elementos que la pauta pide de
forma explícita:

- **La ubicación en formato entendible para una persona:** "Virginia Opazo 23, Santiago, Región
  Metropolitana", obtenida con `reverseGeocodeAsync`. Las coordenadas aparecen debajo, subordinadas,
  junto a la precisión en metros.
- **El clima presentado de forma legible:** icono, "Despejado", "19 grados", y las medidas de
  humedad y viento con sus unidades. El `weather_code` crudo no aparece en ninguna parte de la
  interfaz.

### 6.7 Permisos: explicación previa y degradación sin ruptura

Los permisos se solicitan **en el momento en que se necesitan**, no en una pantalla de bienvenida, y
siempre precedidos de una explicación propia de para qué sirven. El usuario decide informado.

![Permiso de ubicación no concedido](docs/evidencia-permiso-ubicacion.png)

Bloque de ubicación sin el permiso concedido. Se muestra la justificación y un botón para
concederlo. Obsérvese que **el resto del formulario sigue siendo utilizable**: el nombre del ave, la
fecha y los demás campos permanecen accesibles. Rechazar un permiso degrada una función concreta y
lo comunica; no cierra ni congela la aplicación.

![Permiso de cámara no concedido](docs/evidencia-permiso-camara.png)

El mismo criterio para la cámara. Hay siempre dos salidas: conceder el permiso o volver.

La aplicación distingue **tres** estados de permiso, no dos: concedido, denegado pero solicitable de
nuevo, y denegado de forma permanente. El tercero no puede resolverse con otra solicitud, de modo
que en ese caso el botón cambia a "Abrir ajustes" y lleva a la configuración del sistema. Sin esa
distinción, un usuario que hubiera denegado dos veces quedaría atrapado ante un botón que no hace
nada.

### 6.8 Comportamiento sin conexión

Prueba realizada desactivando la red del emulador con `adb shell svc wifi disable` y
`adb shell svc data disable`.

![Sin conexión con el servicio de clima](docs/evidencia-sin-red-clima.png)

La consulta del clima falla y la aplicación lo comunica en español, indicando explícitamente que el
registro puede continuar: "No se pudo obtener el clima: no hay conexión con el servicio. Puedes
guardar igual." El aviso es informativo y ofrece reintentar. El botón de guardar permanece
habilitado en todo momento.

Nótese que la ubicación **sí** se obtuvo sin red: el GPS es un sensor del dispositivo y no depende
de la conexión.

![Avistamiento guardado sin clima](docs/evidencia-guardado-sin-clima.png)

El avistamiento se guarda igual y el listado lo muestra con el indicador "Sin clima" y su icono
correspondiente, en lugar de la temperatura. No queda un hueco ni un valor vacío: la ausencia del
dato se comunica de forma explícita.

Esta prueba verifica el requisito más importante del diseño de la integración con la API: **un
problema de conexión nunca impide registrar un avistamiento.** Un voluntario en terreno, que es
justamente donde no hay cobertura, no pierde el registro.

### 6.9 Demostración en dispositivo físico

Las capturas anteriores provienen del emulador. Esta sección corresponde a la aplicación ejecutándose
sobre un **Samsung Galaxy Note 20 (SM-N980F) con Android 13**, mediante Expo Go, con la cámara, el
GPS y la conexión reales del teléfono.

![Listado en el teléfono](docs/evidencia-telefono-listado.png)

Dos avistamientos registrados en el dispositivo. Las miniaturas muestran fotografías con contenido
real, tomadas con la cámara del teléfono. Obsérvese el segundo registro, "No identidad": es el caso
que RF-01 contempla de forma explícita, el del voluntario que no logra identificar al ave y aun así
necesita dejar constancia.

![Detalle en el teléfono](docs/evidencia-telefono-detalle.png)

![Clima registrado en el teléfono](docs/evidencia-telefono-detalle-clima.png)

El detalle completo. La dirección obtenida con `reverseGeocodeAsync` y las coordenadas aparecen
**ocultadas de forma deliberada**: corresponden a la ubicación real de un integrante del equipo y
este repositorio es público. El bloque conserva su etiqueta y su posición, de modo que sigue siendo
verificable que la aplicación traduce las coordenadas a una dirección legible, que es lo que exige
RF-04. Todo lo demás es el dato auténtico: fotografía, fecha, cantidad, y el clima del momento con
sus tres medidas, obtenido de Open-Meteo con las coordenadas que entregó el GPS del teléfono.

Esta demostración cierra el único requisito que el emulador no podía acreditar. Con ella, los cuatro
periféricos y servicios que el proyecto necesita quedan verificados sobre hardware real: **cámara,
GPS, geocodificación inversa y API del clima.**

### 6.10 Persistencia tras cerrar la aplicación

![Persistencia](docs/evidencia-persistencia-tras-reinicio.png)

Captura tomada **después** de forzar el cierre de la aplicación y volver a abrirla. El avistamiento
y su fotografía siguen presentes.

Esto es lo que valida la decisión de diseño más importante del modelo de datos. La documentación de
`expo-camera` establece que las imágenes capturadas se escriben en el **directorio de caché** y que
el sistema operativo puede eliminarlas cuando necesite espacio. Persistir ese URI habría producido
exactamente el fallo que RF-05 prohíbe: un listado lleno de imágenes rotas tras un reinicio. Por eso
`src/servicios/servicioFotos.ts` mueve el archivo al directorio de documentos inmediatamente después
de la captura, y persiste **solo el nombre del archivo**, reconstruyendo la ruta en cada lectura:

```ts
// src/servicios/servicioFotos.ts
export async function guardarFotoPermanente(uriDeCache: string): Promise<string> {
  const destino = carpetaDestino();
  const archivo = new File(uriDeCache);

  if (!archivo.exists) {
    throw new Error('La fotografía capturada ya no está disponible');
  }

  await archivo.move(destino);
  return archivo.name;
}

export function resolverUriFoto(nombreArchivo: string): string {
  return new File(Paths.document, CARPETA_FOTOS, nombreArchivo).uri;
}
```

Se persiste el nombre y no la ruta absoluta porque la ruta del directorio de documentos pertenece al
sistema y no está garantizada entre instalaciones de la aplicación.

Un detalle de la API verificado durante la implementación: en SDK 57, `move()` y `copy()` devuelven
`Promise<void>` y **deben esperarse con `await`**, aunque varios ejemplos de la documentación general
los muestren como llamadas síncronas. Se comprobó contra las definiciones de tipos del paquete
instalado. Sin el `await`, la fotografía se persiste a medias.

---

## 7. Declaración del uso de inteligencia artificial

Se utilizó un asistente de inteligencia artificial (Claude, de Anthropic) durante el desarrollo de
esta entrega. Lo que se usó y para qué:

| Uso | Alcance |
|---|---|
| Análisis y planificación | Redacción del documento de análisis (`BRIEF.md`) y de las reglas técnicas del repositorio (`AGENTS.md`): traducción de los requerimientos a criterios de aceptación verificables, diseño del modelo de datos, definición de la matriz de estados y del plan por fases |
| Generación de código | Escritura del código de la aplicación en las fases 1 a 5, siguiendo la arquitectura y las convenciones fijadas previamente en `AGENTS.md` |
| Diagnóstico de fallos | Investigación de los cuatro fallos documentados en este informe: el proveedor de ubicación en el emulador, el React Compiler, el código postal en el geocodificador y la duplicación de la pila de navegación al guardar |
| Redacción del informe | Estructura y redacción de este documento |
| Verificación de versiones y APIs | Consulta de la documentación oficial de Expo y React Native, y del registro de npm, para fijar versiones y firmas de funciones |

**Criterio de trabajo aplicado.** Se estableció desde el inicio una regla explícita, recogida en
`CLAUDE.md`: ninguna funcionalidad se dio por terminada sin haberla ejecutado en el emulador y haber
visto el comportamiento funcionando. Que el código compile no es que el código funcione. Todas las
capturas de la sección 6 corresponden a ejecuciones reales.

**Qué no se delegó.** Las decisiones que condicionan la entrega las tomó el equipo: la elección del
framework, del router y de la estrategia de persistencia; el tercer framework de la comparación; el
criterio de ordenamiento del listado; el reparto del trabajo; y el alcance de cada fase.

**Verificación en lugar de confianza.** Ninguna versión de paquete de este proyecto se tomó de la
memoria del asistente. Todas se contrastaron contra el archivo `bundledNativeModules.json` del
paquete `expo@57.0.23`, que es la misma fuente que consulta `npx expo install`. Ese procedimiento
detectó una discrepancia relevante: la versión de AsyncStorage que corresponde al SDK 57 es la
**2.2.0**, mientras que el registro de npm publica la **3.1.1** como última. Instalar la última
habría roto la compilación nativa.

Del mismo modo, los nombres de los iconos meteorológicos se verificaron uno por uno contra el mapa
de glifos de `@expo/vector-icons` instalado, y la estructura de la respuesta de Open-Meteo se
comprobó ejecutando una petición real al endpoint antes de escribir el código que la consume.

---

## 8. Estado de verificación y limitaciones conocidas

Esta sección existe porque un informe que solo describa lo que funciona es un informe incompleto.

### 8.1 Verificado ejecutando la aplicación

| Requerimiento | Estado |
|---|---|
| RF-01 Registrar un avistamiento | Verificado: cámara embebida, ubicación automática, validación por campo, confirmación y vuelta al listado |
| RF-02 Obtener el clima | Verificado: consulta con las coordenadas capturadas, traducción del código, persistencia junto al avistamiento |
| RF-03 Listar los avistamientos | Verificado: orden más reciente primero, tres criterios de ordenamiento, estado vacío diseñado, acceso permanente al formulario |
| RF-04 Ver el detalle | Verificado: foto grande, datos completos, clima legible y dirección obtenida con `reverseGeocodeAsync` |
| RF-05 Persistencia | Verificado: tras forzar el cierre y reabrir, los datos y el archivo de la fotografía siguen presentes |
| RF-06 Navegación | Verificado: stack de Expo Router con las tres rutas y botón atrás del sistema operativo coherente |

Y los criterios transversales, que son los que evalúa el indicador de interfaces intuitivas:

| Criterio transversal | Estado |
|---|---|
| Validación que bloquea el guardado y señala el campo faltante | Verificado, sección 6.4 |
| Permiso de ubicación no concedido: explicación propia y formulario utilizable | Verificado, sección 6.7 |
| Permiso de cámara no concedido: explicación propia y salida disponible | Verificado, sección 6.7 |
| Fallo de la API del clima: aviso informativo, guardado permitido | Verificado sin red, sección 6.8 |
| Avistamiento sin clima: indicador explícito en el listado | Verificado, sección 6.8 |
| Estado de carga en cada operación asíncrona | Verificado durante la ejecución |
| Estado vacío del listado | Verificado, sección 6.2 |

### 8.2 Limitación conocida: captura de imagen en el emulador

`takePictureAsync` produce en el emulador un archivo JPEG de aproximadamente 15 kB completamente
negro. Ese tamaño, para una escena detallada, solo se explica si la imagen es negro uniforme.

Todo el resto de la cadena funciona y está verificado: la solicitud del permiso, la vista previa en
vivo, el disparo, el traslado del archivo desde la caché al directorio de documentos, la
persistencia del nombre, la reconstrucción de la ruta y el renderizado de la miniatura. Lo que falla
es únicamente el contenido del archivo que entrega el emulador.

![Vista previa de la cámara](docs/evidencia-f0-camara-preview.png)

La vista previa entrega fotogramas correctos, como muestra la captura. El problema aparece solo al
materializar el archivo.

Se descartaron seis hipótesis de forma sistemática:

| Hipótesis | Acción | Resultado |
|---|---|---|
| Disparo antes de que el sensor esté listo | Bloquear el obturador hasta `onCameraReady`, como exige la documentación | Sin cambio. **La corrección se mantuvo igualmente**, por ser correcta |
| Posprocesado de la imagen | `skipProcessing: true` y margen de 1,5 s | Sin cambio. Revertido, para no dejar código supersticioso |
| Renderizado por GPU del anfitrión | Reiniciar el emulador con `-gpu swiftshader_indirect` | Sin cambio |
| Fuente de la cámara virtual | Cambiar la cámara del dispositivo virtual a la webcam física del equipo | La vista previa mostró fotogramas reales de la webcam; **la captura siguió en negro** |
| Versión de Android | Repetir la prueba completa en un emulador Pixel 6 con API 33 (Android 13) | Sin cambio, y además en esa versión ni siquiera la vista previa funciona |
| Expo Go como intermediario | Compilar un *development build* con `npx expo run:android`, que integra CameraX en nuestro propio binario en lugar de usar el de Expo Go | Compilación correcta en 23 min 43 s. **La captura siguió en negro** |

La última hipótesis era la principal y también cayó. Un *development build* no usa el binario de
Expo Go: compila la aplicación con su propia integración nativa de CameraX. Que el resultado sea
idéntico descarta a Expo Go como causa.

### La prueba que cierra el diagnóstico

Al extraer el archivo directamente del sandbox de la aplicación
(`adb shell run-as cl.ipss.avistaves cat files/fotos/...`) aparece el dato concluyente:

![Captura cruda extraída del dispositivo](docs/evidencia-captura-emulador-cruda.jpg)

El archivo es un **JPEG válido de 1080x1911 píxeles y 17.026 bytes**, es decir 0,008 bytes por
píxel, cuando una fotografía real de esa escena rondaría entre 0,05 y 0,3. Pero lo decisivo no es
el tamaño: es que la imagen **lleva estampada una marca de tiempo en amarillo**, "17.09.26
16:20:51", que nuestro código no dibuja en ninguna parte.

Esa marca la genera la capa de cámara del propio emulador de Android. Su presencia demuestra tres
cosas a la vez:

1. La cadena de captura **funciona**: se generó un JPEG bien formado, a resolución completa, con el
   posprocesado del sistema aplicado.
2. El contenido negro **no lo produce nuestro código**, que se limita a recibir el archivo y moverlo
   al directorio de documentos.
3. El fallo está en el **componente de cámara del emulador**, que entrega su fotograma de captura
   en negro aunque alimente correctamente la vista previa. Son dos rutas distintas dentro del
   emulador, y solo una de ellas funciona.

### Resuelto en dispositivo físico

La hipótesis se confirmó. Al ejecutar la misma aplicación, sin cambiar una línea de código, sobre un
Samsung Galaxy Note 20 con Android 13, **la fotografía se captura con contenido correcto**. Las
capturas están en la sección 6.9.

El diagnóstico queda así cerrado por los dos extremos: se demostró que la causa no era el código
(seis hipótesis descartadas, más la marca de tiempo que identifica al emulador como autor del
archivo) y se demostró que en hardware real el problema no existe.

**Consecuencia para la evaluación.** RF-01 queda verificado en toda su extensión sobre un
dispositivo real. La limitación descrita en esta sección afecta únicamente al emulador y se
documenta porque el proceso de diagnóstico forma parte del trabajo realizado, no porque quede
ningún requisito sin cumplir.

### 8.3 Decisión pendiente documentada

La fecha y la hora del avistamiento son editables mediante dos campos de texto con validación
(`DD/MM/AAAA` y `HH:MM`) en lugar de un selector nativo de fecha. La razón es que un selector
requeriría añadir una dependencia fuera de la lista fijada en `AGENTS.md`, y la regla del proyecto
prohíbe incorporar dependencias sin acuerdo previo del equipo. La validación rechaza fechas
imposibles, incluidas las que `Date` desplazaría en silencio, como el 31 de febrero.
