# Al Día

App organizadora para Android: notas ordenadas por categorías, recordatorios que suenan
como alarma, y control de gastos diarios con balance por semana y por mes. Todo se guarda
en el teléfono — sin cuenta, sin internet.

El diseño sale del proyecto de Claude Design **"Diseño de app Android multipropósito"**,
sobre el sistema **Nocturne**.

## Cómo compilarla

En esta PC **no hay Android Studio ni JDK instalados**, así que el proyecto está escrito
pero todavía no se compiló ni se probó en un teléfono.

1. Instalar [Android Studio](https://developer.android.com/studio) (trae el JDK y el SDK).
2. Abrir la carpeta `nwpn` con **File → Open**. Android Studio va a bajar Gradle 8.9 y las
   dependencias en la primera sincronización; puede tardar varios minutos.
3. Conectar el teléfono con depuración USB activada, o crear un emulador, y darle a **Run**.

Para generar un APK instalable a mano:
**Build → Build Bundle(s) / APK(s) → Build APK(s)** → `app/build/outputs/apk/debug/`.

> El proyecto no incluye `gradlew` ni `gradle-wrapper.jar` (son binarios). Android Studio no
> los necesita para sincronizar. Para compilar desde la terminal, generalos una vez con
> `gradle wrapper`.

## El sistema de diseño

Nocturne es un sistema **solo oscuro**. No hay tema claro ni selector de tema, y el acento
es fijo: el blurple `#9184D9`.

| Token | Valor |
|---|---|
| Fondo | `#161826` |
| Superficie | `#232532` |
| Texto | `#E9E9ED` |
| Acento | `#9184D9` |
| Divisor | `#E9E9ED` al 16 % |

Rasgos del sistema que la app respeta:

- Las tarjetas no llevan sombra: llevan un **anillo de 1 px** (`--shadow-sm`). La sombra
  aparece recién en `elev-md` / `elev-lg`.
- Los botones son **de contorno**, no rellenos: borde y texto en acento, fondo transparente.
- Los rótulos de tarjeta (`card-kicker`) van en acento, 10 px, versalitas espaciadas.
- Las cifras usan numerales tabulares (`tnum`), no una tipografía monoespaciada.
- La regla horizontal se **desvanece en las puntas** (`FadingRule`), 48 px por lado.

Como el CSS pide Inter y el proyecto no empaqueta la fuente, la app usa la sans del sistema
con el mismo peso de titular (500). Si querés Inter de verdad, hay que agregar el `.ttf` a
`res/font/` y apuntar `Type.kt` ahí.

### El color de las categorías

El acento dejó de ser configurable; ahora **cada categoría de notas elige su color** de una
paleta de 8 tonos. Esos tonos no están escritos a mano: se derivan con la misma receta que
Nocturne usa para su propio acento —misma luminosidad y croma, hue rotado— así que
`Color.kt` incluye una conversión **OKLCH → sRGB** y calcula los tres colores de cada
categoría (punto, fondo y texto de la etiqueta) a partir del hue.

Hues: Blurple 289 · Azul 250 · Celeste 205 · Verde 155 · Lima 108 · Ámbar 75 · Naranja 38 ·
Rosa 340. «Sin categoría» no tiene hue y cae en los neutros.

## Cómo está armado

| Capa | Qué hay |
|---|---|
| `data/` | Room: `Category`, `Note`, `Reminder`, `Expense` + repositorios. Ajustes en DataStore. |
| `alarm/` | `AlarmScheduler`, receptores, y la pantalla que suena sobre el bloqueo. |
| `ui/theme/` | Tokens de Nocturne, conversión OKLCH, escala tipográfica. |
| `ui/common/` | Tarjeta, etiqueta, botón de contorno, interruptor, regla, íconos. |
| `ui/` | Una pantalla por sección: Inicio, Notas, Alarmas, Gastos, Ajustes. |

No usa inyección de dependencias: las piezas se arman en `AlDiaApp` y se toman con
`(context.applicationContext as AlDiaApp)`.

Los íconos no vienen de Material Icons: son los mismos trazos SVG del diseño, reconstruidos
como `ImageVector` en `ui/common/AlDiaIcons.kt`.

### Las alarmas

- Un recordatorio con **Alarma** se programa con `setAlarmClock`: queda exento del ahorro de
  batería y muestra el ícono de alarma en la barra de estado.
- Uno con **Aviso** usa `setExactAndAllowWhileIdle`: llega igual, pero sin sonar.
- Al saltar, `AlarmReceiver` publica una notificación con *full-screen intent*. Eso es lo que
  hace que Android abra `AlarmActivity` con el teléfono bloqueado y la app cerrada.
- **Sin permiso de notificaciones no suena nada**, porque sin notificación no hay pantalla
  completa. La app lo pide al arrancar y lo avisa arriba de todo en Recordatorios.
- Al apagar el teléfono se borran las alarmas del sistema: `BootReceiver` las vuelve a poner
  al encender, al actualizar la app y al cambiar la hora o la zona horaria.

### Las categorías de notas

Las notas no tienen clave foránea contra su categoría, a propósito. Al borrar una categoría,
`NotesRepository.deleteCategory` reasigna sus notas a «Sin categoría» y recién después la
elimina, todo en la misma transacción. «Sin categoría» está marcada con `isDefault` y no se
puede borrar.

## Diferencias con el diseño

Cosas que el diseño no cubría y hubo que resolver:

- **Crear categorías.** El diseño solo permite cambiarles el color desde Ajustes. La app
  agrega un tile «Nueva categoría» al final de la grilla de Notas.
- **Editar notas.** El diseño solo tiene vista de lectura. La app le suma un botón «Editar
  nota» de contorno al pie de esa vista, que abre la misma hoja que usa para crear.
- **Borrar.** Mantener apretada una nota, categoría, recordatorio o gasto abre el diálogo.
- **Alarmas exactas.** En el diseño es un interruptor común; acá refleja el permiso real del
  sistema y al tocarlo abre la pantalla de Android correspondiente.
- La pantalla de alarma sonando no está en el diseño: usa los tokens de Nocturne con un
  botón principal relleno, más legible de madrugada que uno de contorno.

## Qué falta

- Buscador dentro de una categoría (por ahora la búsqueda es global).
- Editar un gasto ya cargado — se puede borrar y volver a cargarlo.
- Elegir la fecha de un gasto: siempre se guarda con la fecha y hora del momento.
- Exportar a CSV y copia de seguridad manual.
- Los recordatorios de «Una vez» suenan la próxima vez que sean esa hora, sin elegir el día.

---

`prototipo-al-dia.html` es el prototipo original, anterior a Nocturne. Quedó como registro;
el diseño vigente es el del proyecto de Claude Design.
