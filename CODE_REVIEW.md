# Code review

Las referencias siguen la nomenclatura `Clase#línea` o `Clase#método`. Cada comentario indica su prioridad.

- **[BLOQUEANTE]** No aprobaría la PR sin corregirlo, porque no cumple una historia de usuario, falla en ejecución o introduce un riesgo que no aceptaría en producción.
- **[IMPORTANTE]** Pediría arreglarlo en esta PR o en la siguiente.
- **[SUGERENCIA]** Mejora que propondría y hablaría con el equipo.
- **[NIT]** Detalle menor.

## Resumen

**No aprobaría la PR en su estado actual.** La estructura del proyecto es una buena base, pero ninguna de las cuatro historias de usuario se cumple, y el único test pasa en verde sin detectar ninguno de los fallos.

| Historia de usuario | Estado | Motivo principal |
|---|---|---|
| 1. Puntuación | No se cumple | La completitud sobrescribe la puntuación y las palabras clave con mayúscula o puntuación no se detectan |
| 2. Ocultar irrelevantes | No se cumple | El listado público devuelve un error 500 si los anuncios no están puntuados |
| 3. Fecha de irrelevancia | No se cumple | La fecha se reinicia en cada recálculo |
| 4. Orden | No se cumple | El listado se ordena de peor a mejor |

Además de los fallos de cada historia, hay dos bloqueantes más. `GET /ads/score` modifica datos, y no hay tests que comprueben las reglas de negocio.

Para revisar el código lo he arrancado en local y he calculado a mano la puntuación esperada de los anuncios de prueba. Con las reglas de la historia, el listado público debería devolver los anuncios 2, 4, 5 y 6 con 85, 80, 75 y 50 puntos. Ahora mismo, los cuatro tienen 40.

Si tuviera que priorizar, el orden sería este.

1. Corregir los fallos bloqueantes de las cuatro historias, cada uno con su test.
2. Cambiar `GET /ads/score` por `POST` y proteger los endpoints internos.
3. Llevar las reglas de puntuación y relevancia al dominio y quitar la dependencia de la aplicación hacia los DTOs.
4. El resto de comentarios, que se pueden tratar en PRs posteriores.

---

## Historia de usuario 1. Puntuación del anuncio

> Asignar a cada anuncio una puntuación entre 0 y 100 según sus fotos, su descripción y lo completo que esté.

**No se cumple.** De los ocho anuncios de prueba, la puntuación es incorrecta justo en los cuatro que deberían ser relevantes (el 2, el 4, el 5 y el 6).

Lo he comprobado calculando a mano el anuncio 2 de `InMemoryPersistence`, un piso con una foto HD, tamaño 300 y la descripción "Nuevo ático céntrico recién reformado. No deje pasar la oportunidad y adquiera este ático de lujo". Según la historia debería tener 85 puntos (20 por la foto HD, 5 por la descripción, 20 por cuatro palabras clave y 40 por estar completo), pero el código le da 40.

### La completitud sobrescribe la puntuación en lugar de sumar (`AdsServiceImpl#118-120`) [BLOQUEANTE]

```java
if (ad.isComplete()) {
    score = Constants.FORTY;
}
```

La historia dice que un anuncio completo "proporciona otros 40 puntos", pero aquí se asigna 40 y se pierde todo lo acumulado. Cualquier anuncio completo vale exactamente 40, así que ordenar por calidad deja de tener sentido. Debería ser `score += Constants.FORTY`.

### Las palabras clave con mayúscula o puntuación no se detectan y el recuento de palabras falla (`AdsServiceImpl#93` y `AdsServiceImpl#110-114`) [BLOQUEANTE]

```java
List<String> wds = Arrays.asList(description.split(" "));
if (wds.contains("luminoso")) score += Constants.FIVE;
```

- La comparación distingue mayúsculas, así que "Nuevo" o "Ático" no puntúan, aunque el propio README las escribe así.
- Separar por un espacio deja la puntuación pegada a la palabra, y "reformado." o "céntrico," no coinciden.
- Los espacios repetidos cuentan como palabras y los saltos de línea no separan, y ese recuento decide los puntos por longitud.

Con los datos de prueba se detectan 5 de las 9 palabras clave que aparecen. En el anuncio 2 no se detectan "Nuevo" ni "reformado.", y en el 4 no se detectan "Ático" ni "reformado,". Cada uno pierde 10 puntos.

Normalizaría la descripción una sola vez (minúsculas con `Locale` explícito y separando por todo lo que no sea letra o número) y usaría esa lista para contar palabras y para buscar palabras clave.

```java
List<String> words = Arrays.stream(description.toLowerCase(Locale.ROOT).split("[^\\p{L}\\p{N}]+"))
        .filter(word -> !word.isEmpty())
        .collect(Collectors.toList());
```

### El chalet puntúa con 50 palabras y la historia dice "más de 50" (`AdsServiceImpl#104-108`) [IMPORTANTE]

Para los pisos la historia dice "50 o más" y para los chalets "más de 50", así que la diferencia parece intencionada. El código usa `>= 50` en los dos casos. Lo confirmaría con producto y lo cubriría con tests de 49, 50 y 51 palabras.

### Errores con anuncios sin fotos o con la descripción en blanco (`AdsServiceImpl#71` y `Ad#isComplete`) [IMPORTANTE]

- Si la lista de fotos es `null`, `ad.getPictures().isEmpty()` y `Ad#isComplete` lanzan un `NullPointerException`.
- Una descripción con solo espacios suma 5 puntos y hace que el anuncio cuente como completo. Usaría `trim().isEmpty()`.
- `Ad#isComplete` repite las mismas condiciones en una sola expresión muy larga. Un `switch` por tipología sería más claro.

### Todas las reglas están en un único método y fuera del dominio (`AdsServiceImpl#calculateScore`) [IMPORTANTE]

El método mezcla los puntos por fotos, descripción, longitud, palabras clave y completitud, el ajuste entre 0 y 100, la regla de irrelevancia y el guardado. Lo que me preocupa no son sus 70 líneas, sino que tiene varios motivos para cambiar, en contra del principio de responsabilidad única. Por eso no se puede probar cada regla por separado, añadir una regla obliga a tocarlo (en contra de abierto/cerrado) y `Ad` se queda sin comportamiento.

Movería el cálculo a un servicio de dominio, `AdScoreCalculator`, y el servicio de aplicación solo orquestaría la lectura, el cálculo y el guardado. Empezaría con un método privado por regla, y si las reglas empiezan a cambiar a menudo pasaría a una lista de `ScoreRule` (patrón Strategy). Con cinco reglas no lo haría desde el principio, para no sobrediseñar.

```java
interface ScoreRule {
    int score(Ad ad);
}

int calculate(Ad ad) {
    int total = rules.stream().mapToInt(rule -> rule.score(ad)).sum();
    return Math.max(MIN_SCORE, Math.min(MAX_SCORE, total));
}
```

### Los nombres de `Constants` describen el valor y no su significado (`Constants`) [IMPORTANTE]

`TEN` o `FORTY_NINE` no eliminan los números mágicos, solo les cambian el nombre. Además, `Constants.TWENTY` sirve a la vez para los puntos de una foto HD, el mínimo de palabras de un piso y los puntos de la descripción de un chalet, así que si cambia una regla cambian las tres. Usaría nombres de negocio como `HD_PICTURE_POINTS` o `COMPLETE_AD_POINTS`.

### Otros detalles

- **[SUGERENCIA]** `AdsServiceImpl#110-114`. Las cinco palabras clave se repiten en cinco `if`. Un `Set` sería más legible y fácil de llevar a configuración.
- **[SUGERENCIA]** `AdsServiceImpl#84-87`. `Optional` se usa como un simple `if (description != null)`. Está pensado como tipo de retorno.
- **[NIT]** `AdsServiceImpl#71-115`. Hay demasiado anidamiento de `if` y `else`. Extrayendo un método por regla con retornos tempranos, la mayoría de `else` desaparecen. No los quitaría todos por norma, porque el de `AdsServiceImpl#132-136` representa dos caminos distintos.
- **[NIT]** `AdsServiceImpl#122-130`. Se guarda la puntuación y luego se corrige. Es más claro ajustarla antes con `Math.max(0, Math.min(100, score))`.
- **[NIT]** `AdsServiceImpl#70-78`. Los comentarios repiten lo que dice el código.
- **[NIT]** `AdsServiceImpl#75`. Una foto con `quality` a `null` cuenta como SD sin avisar, porque `Quality.HD.equals(null)` devuelve `false`. Lo validaría al construir `Picture`.
- **[NIT]** `Constants`. La clase no es `final` y no tiene constructor privado, así que se puede instanciar o extender sin sentido.

### Dudas para producto

- Si una palabra clave aparece dos veces, ¿suma una vez o dos? El código suma una vez y me parece razonable, pero lo fijaría en un test.
- ¿Cuentan las palabras clave escritas sin tilde, como "centrico"?
- ¿Qué cuenta como palabra? ¿Los números o los símbolos sueltos?

---

## Historia de usuario 2. Ocultar los anuncios irrelevantes

> Los usuarios no deben ver anuncios con menos de 40 puntos.

**No se cumple.** El listado público falla si se consulta antes de calcular las puntuaciones, y la regla está en la capa equivocada.

### El listado público devuelve un error 500 si los anuncios no están puntuados (`InMemoryPersistence#66` y `InMemoryPersistence#75`) [BLOQUEANTE]

```java
.filter(x -> x.getScore() >= Constants.FORTY)
```

`score` es `null` hasta que alguien llama a `/ads/score`, y comparar un `Integer` nulo con un `int` lanza un `NullPointerException`. Lo he reproducido arrancando la aplicación, y `GET /ads/public` responde con un 500. Lo mismo pasa con `findIrrelevantAds`.

Habría que decidir qué es un anuncio sin puntuar. Yo lo trataría como no relevante, para no mostrar nunca contenido sin evaluar.

### La regla de relevancia está en infraestructura y duplicada (`InMemoryPersistence#66`, `InMemoryPersistence#75` y `AdsServiceImpl#132`) [IMPORTANTE]

El umbral de 40 puntos es una regla de negocio, pero se aplica en el repositorio y se repite en el servicio. Si cambia el umbral hay que tocar dos capas y es fácil que acaben siendo distintas. La llevaría al dominio con un método `Ad#isRelevant()` y una constante `RELEVANCE_THRESHOLD`, y la infraestructura la usaría para filtrar. Con una base de datos real, el filtro iría en la consulta, pero el valor seguiría saliendo del dominio.

```java
public boolean isRelevant() {
    return score != null && score >= RELEVANCE_THRESHOLD;
}
```

### Dudas para producto

- Un anuncio nuevo no tiene puntuación hasta que alguien llama a `/ads/score`. ¿Debe ocultarse hasta entonces? Esto refuerza la idea de calcular la puntuación al crear o editar el anuncio.

---

## Historia de usuario 3. Anuncios irrelevantes y fecha desde la que lo son

> El encargado de calidad quiere ver los anuncios irrelevantes y desde qué fecha lo son, para medir la calidad media del portal.

**No se cumple.** La fecha se reinicia en cada recálculo, así que nunca indica desde cuándo es irrelevante un anuncio.

### La fecha de irrelevancia se sobrescribe en cada recálculo (`AdsServiceImpl#132-136`) [BLOQUEANTE]

```java
if (ad.getScore() < Constants.FORTY) {
    ad.setIrrelevantSince(new Date());
} else {
    ad.setIrrelevantSince(null);
}
```

Cada llamada a `/ads/score` pone la fecha actual a todos los anuncios irrelevantes. Un anuncio que es irrelevante desde el lunes aparecerá el miércoles como "irrelevante desde el miércoles". La fecha solo debería fijarse cuando el anuncio pasa de relevante a irrelevante, mantenerse en los recálculos siguientes y borrarse si vuelve a ser relevante. Si no se borrara, un anuncio que se recupera y vuelve a caer mostraría la fecha de la primera vez.

Llevaría esta regla al dominio, para que nadie pueda dejar el anuncio en un estado incoherente desde fuera.

```java
public void updateScore(int newScore, Instant now) {
    score = newScore;
    if (isRelevant()) {
        irrelevantSince = null;
        return;
    }
    if (irrelevantSince == null) {
        irrelevantSince = now;
    }
}
```

### La fecha se crea con `new Date()` y no se puede testear (`AdsServiceImpl#133`) [IMPORTANTE]

Crear la fecha dentro del servicio impide comprobar en un test qué fecha se guarda o que se conserva entre recálculos. Inyectaría un `java.time.Clock`, que en producción es el reloj del sistema y en los tests una hora fija.

### El listado de calidad no tiene ninguna protección (`AdsController#17-20`) [IMPORTANTE]

`/ads/quality` es un listado interno para el equipo de calidad que expone puntuaciones y fechas, pero es tan accesible como el listado público. Debería requerir autenticación o un rol concreto. Si eso ya lo resuelve un gateway, lo dejaría documentado.

### Otros detalles

- **[SUGERENCIA]** `Ad`, `AdVO` y `QualityAd`. `java.util.Date` es mutable (el getter expone el estado interno) y está obsoleta desde Java 8. Usaría `Instant` en el dominio y un formato ISO 8601 con zona horaria explícito en la respuesta.
- **[NIT]** `AdsServiceImpl#findQualityAds`. El listado no tiene orden. Ordenarlo por `irrelevantSince` ayudaría a ver primero los anuncios que llevan más tiempo siendo irrelevantes.

### Dudas para producto

- La historia habla de medir la calidad media del portal, pero el endpoint solo devuelve una lista. ¿Necesitan métricas agregadas, como el número de anuncios irrelevantes o el tiempo medio que llevan así, o les basta la lista?

---

## Historia de usuario 4. Ordenar de mejor a peor

> Como usuario de idealista quiero ver los anuncios ordenados de mejor a peor.

**No se cumple.** El listado se ordena al revés, de peor a mejor.

### El orden es ascendente (`AdsServiceImpl#21`) [BLOQUEANTE]

```java
ads.sort(Comparator.comparing(Ad::getScore));
```

`Comparator.comparing` usa el orden natural, de menor a mayor, así que el primer anuncio del listado es el de peor puntuación. Debería ser `Comparator.comparing(Ad::getScore).reversed()`.

Además, ahora mismo el orden es arbitrario. Por el fallo de la completitud de la historia 1, todos los anuncios relevantes tienen exactamente 40 puntos, como comprobé al arrancar la aplicación.

### No hay criterio de desempate (`AdsServiceImpl#21`) [IMPORTANTE]

Cuando dos anuncios tienen la misma puntuación, `List.sort` respeta el orden en que llegan del repositorio, que es el orden de inserción de `InMemoryPersistence`. Ahora el recálculo completo lo deja igual, pero es un detalle de implementación. `InMemoryPersistence#save` mueve el anuncio guardado al final, así que cualquier `save` individual (por ejemplo, al editar un anuncio) ya lo alteraría, y con una base de datos sin `ORDER BY` el orden sería indefinido. Añadiría un segundo criterio fijo, como el id o la fecha de publicación, y lo consultaría con producto.

```java
Comparator.comparing(Ad::getScore).reversed().thenComparing(Ad::getId)
```

### La ordenación se hace en memoria y sin paginación (`AdsServiceImpl#20-21`) [SUGERENCIA]

Con una base de datos real, el orden debería pedirse en la propia consulta del repositorio, junto con una paginación. Ahora `/ads/public` carga y devuelve todos los anuncios relevantes de golpe, y eso no escala con el volumen de un portal como idealista.

### Otros detalles

- **[NIT]** `AdsServiceImpl#21`. `sort` modifica directamente la lista que devuelve el repositorio, y `Collectors.toList()` no garantiza que sea modificable. Es más seguro ordenar dentro del stream con `sorted`.

---

## Arquitectura y diseño

### La capa de aplicación depende de la infraestructura (`AdsService#3-4` y `AdsServiceImpl#4-5`) [IMPORTANTE]

`AdsService` devuelve `PublicAd` y `QualityAd`, que son los DTOs de la API. En la arquitectura hexagonal que sugieren los paquetes, las dependencias solo apuntan hacia dentro, de `infrastructure` a `application` y de `application` a `domain`, y este import va al revés. Por eso un cambio en el JSON obliga a tocar el servicio, y el caso de uso no se puede reutilizar desde otro adaptador (un consumidor de eventos o un batch) sin arrastrar los DTOs web.

Los DTOs están bien ubicados en infraestructura, porque son el contrato HTTP. Lo que cambiaría es quién convierte. El servicio devolvería objetos de dominio y el controlador, o un mapper de `infrastructure.api`, los transformaría.

```java
// application
List<Ad> findPublicAds();

// infrastructure.api
return ResponseEntity.ok(adsService.findPublicAds().stream()
        .map(PublicAd::from)
        .collect(toList()));
```

La otra dirección sí está bien resuelta. `AdRepository` es un puerto del dominio e `InMemoryPersistence` es el adaptador que lo implementa.

### Inyección de dependencias por campo (`AdsServiceImpl#15-16` y `AdsController#14-15`) [IMPORTANTE]

```java
@Autowired
private AdRepository adRepository;
```

Con `@Autowired` sobre un campo privado, las dependencias no se ven desde fuera, no pueden ser `final` y para testear la clase hace falta reflexión, que es lo que hace `@InjectMocks` en el test actual. Usaría inyección por constructor. Deja claro qué necesita cada clase, permite campos inmutables y se puede instanciar en un test con un simple `new`.

```java
private final AdRepository adRepository;

public AdsServiceImpl(AdRepository adRepository) {
    this.adRepository = adRepository;
}
```

### `Ad` se puede modificar libremente desde fuera (`Ad#56-131`) [IMPORTANTE]

`Ad` tiene setters públicos para todos sus campos, incluido el `id`. Cualquier clase puede poner una puntuación de 500 o una fecha de irrelevancia a un anuncio relevante, y ninguna regla lo impide. Quitaría los setters y dejaría que el estado cambie solo a través de métodos de dominio como el `updateScore` propuesto en la historia 3.

Además, `equals` y `hashCode` comparan todos los campos. En una entidad la identidad debería ser solo el `id`. Con la implementación actual, dos instancias del mismo anuncio (mismo `id`) dejan de ser iguales si difieren en la puntuación. Además, si cambia la puntuación de un anuncio guardado en un `HashSet` o usado como clave de un `HashMap`, su `hashCode` cambia y ya no se encuentra.

### Otros detalles

- **[SUGERENCIA]** `PublicAd`, `QualityAd`, `AdVO` y `PictureVO`. Los getters y setters escritos a mano no aportan nada. Si el equipo lo usa, propondría Lombok, o `record` con Java 16 o superior. No lo aplicaría a `Ad`, porque `@Data` generaría setters públicos y un `equals` sobre todos los campos, justo lo que hay que evitar en una entidad.
- **[SUGERENCIA]** `AdsServiceImpl#23-35` y `AdsServiceImpl#42-57`. Las conversiones a `PublicAd` y a `QualityAd` copian los mismos seis campos. Un mapper común evitaría que se desincronicen.
- **[SUGERENCIA]** `AdsServiceImpl#61-65`. El recálculo guarda los anuncios uno a uno y sin transacción. Si uno falla a mitad, por ejemplo por una foto `null`, unos quedan recalculados y otros no. Además, recalcula todos los anuncios aunque no hayan cambiado.
- **[NIT]** `AdsServiceImpl#3` y `AdsServiceImpl#9`. Los imports con comodín (`com.idealista.domain.*` y `java.util.*`) esconden qué clases se usan realmente.
- **[NIT]** `Ad#36-48`. El segundo constructor deja `score` e `irrelevantSince` a `null` de forma implícita. Si se mantiene, debería delegar en el constructor completo con `this(...)`.
- **[NIT]** `AdsService`. Una interfaz con una sola implementación y el sufijo `Impl` añade una capa que hoy no aporta mucho. Es una convención de equipo, así que no la cambiaría sin hablarlo.
- **[NIT]** `AdsServiceImpl#93`. La variable `wds` está abreviada, y el código mezcla comentarios en español con nombres en inglés. Elegiría un único idioma para todo.

---

## API

### `GET /ads/score` modifica el estado y devuelve un `202` que no corresponde (`AdsController#27-31`) [BLOQUEANTE]

```java
@GetMapping("/ads/score")
public ResponseEntity<Void> calculateScore() {
    adsService.calculateScores();
    return ResponseEntity.accepted().build();
}
```

Un `GET` no debe modificar nada en el servidor, y este recalcula y guarda todas las puntuaciones. Un crawler o el prefetch del navegador pueden lanzarlo sin querer, y una caché puede responder sin que llegue a ejecutarse. Debería ser un `POST`, y solo para usuarios autorizados.

Además, `202 Accepted` indica que la petición se procesará más tarde, pero `calculateScores()` es síncrona y cuando se responde el cálculo ya ha terminado. O se mantiene síncrono y devuelve `204 No Content`, o se hace asíncrono de verdad, encolando el trabajo y devolviendo `202`.

A medio plazo me plantearía si el recálculo debe ser un endpoint. Podría lanzarse al crear o editar un anuncio, o con un proceso programado.

### Otros detalles

- **[SUGERENCIA]** `AdsController`. No hay gestión de errores. Sin un `@ControllerAdvice`, cualquier excepción devuelve la respuesta por defecto de Spring con un 500, como ocurre con el `NullPointerException` del listado público. Definiría un formato de error común y códigos de estado adecuados.
- **[SUGERENCIA]** No hay logs ni métricas. No se puede saber cuánto tarda el recálculo, cuántos anuncios se procesan ni cuántos pasan a ser irrelevantes. Esto último encaja con la necesidad de medir la calidad del portal de la historia 3.

---

## Persistencia

### Una lista compartida sin protección frente a accesos concurrentes (`InMemoryPersistence#15-16` y `InMemoryPersistence#49-55`) [IMPORTANTE]

El repositorio es un singleton, y Tomcat atiende varias peticiones a la vez con la misma instancia. Si alguien llama a `/ads/score` mientras otra persona consulta `/ads/public`, el `ArrayList` se modifica mientras se recorre. Eso puede provocar una `ConcurrentModificationException`, que desaparezcan anuncios leídos justo entre el `removeIf` y el `add` de `save`, o un listado con unas puntuaciones recalculadas y otras no.

Usaría un `ConcurrentHashMap<Integer, AdVO>` indexado por id, que además hace que guardar y buscar no tengan que recorrer toda la lista. Entiendo que este repositorio es provisional, y con una base de datos real esto lo resolverían las transacciones, por eso no lo bloquearía.

### Las fotos que no existen se convierten en `null` (`InMemoryPersistence#91-98`) [IMPORTANTE]

```java
.map(pictureVO -> new Picture(...))
.orElse(null);
```

Si un anuncio referencia una foto que no existe, se añade un `null` a su lista de fotos. Después falla con un `NullPointerException` en `AdsServiceImpl#30` (`Picture::getUrl`) y en el cálculo de la puntuación (`picture.getQuality()`). Descartaría esas fotos o lanzaría una excepción explícita de datos inconsistentes. Además, buscar cada foto recorriendo toda la lista es lento, y un `Map` por id lo resuelve.

La conversión entre `PictureVO` y `Picture` sí está bien ubicada. Es responsabilidad del adaptador de persistencia, y no la movería al dominio.

### Otros detalles

- **[SUGERENCIA]** `InMemoryPersistence#53-54`. Al guardar un anuncio se vuelven a guardar todas sus fotos, aunque puntuar no las cambia.
- **[NIT]** `InMemoryPersistence#82` y `InMemoryPersistence#96`. `Typology.valueOf` y `Quality.valueOf` lanzan una excepción sin contexto si hay un dato incorrecto, y no indican qué anuncio falla.
- **[NIT]** `AdVO` y `PictureVO`. "VO" suele significar Value Object, pero son modelos de persistencia mutables y con identidad. `AdEntity` o `AdRecord` serían nombres más precisos.
- **[NIT]** `InMemoryPersistence#20-27`. Los datos de prueba están en el constructor, y tienen incoherencias, como el anuncio 1, que es un chalet con la descripción "Este piso...", o garajes con tamaño de vivienda. Además, ningún chalet tiene tamaño de jardín, así que la regla de completitud de los chalets nunca se prueba con estos datos. Los cargaría desde un fichero.

---

## Tests

### El único test no comprueba ninguna puntuación (`AdsServiceImplTest#calculateScoresTest`) [BLOQUEANTE]

```java
verify(adRepository).findAllAds();
verify(adRepository, times(2)).save(any());
```

El test solo verifica interacciones, es decir, que se lee del repositorio y que se llama dos veces a `save` con cualquier anuncio. No comprueba qué puntuación se calcula ni qué se guarda. Por eso pasa en verde con todos los fallos descritos en este documento, y da una falsa sensación de seguridad. Con reglas de negocio tan concretas, no aprobaría la PR sin tests que las cubran.

Añadiría estos tests.

- **Cada regla por separado.** Fotos HD, SD y sin fotos, descripción vacía y en blanco, cada palabra clave con mayúsculas y puntuación, y completitud por tipología.
- **Los límites.** 19, 20, 49 y 50 palabras en pisos y 50 y 51 en chalets, además del mínimo de 0 y el máximo de 100.
- **La relevancia y la fecha.** Que se fije al pasar a irrelevante, que se conserve en el siguiente recálculo y que se borre al volver a ser relevante, usando un `Clock` fijo.
- **El orden del listado público**, incluido el desempate.
- **Un test de integración con los datos de `InMemoryPersistence`.** Según mis cálculos, las puntuaciones esperadas son 0, 85, 20, 80, 75, 50, 0 y 25 para los anuncios del 1 al 8, y `/ads/public` debería devolver los anuncios 2, 4, 5 y 6 en ese orden.
- **Tests de la API con `@WebMvcTest`** para comprobar los verbos HTTP, los códigos de estado y el formato del JSON.

### Otros detalles

- **[NIT]** `AdsServiceImplTest#34` y `AdsServiceImplTest#43`. Los dos anuncios del test tienen `id = 1`, así que representan el mismo anuncio con datos distintos. Con el repositorio real, el segundo `save` pisaría al primero.
- **[NIT]** `AdsServiceImplTest#relevantAd`. Por el fallo de la completitud, este anuncio obtiene 40 puntos cuando debería obtener 100 (40 por las fotos, 5 por la descripción, 30 por sus 51 palabras y 40 por estar completo, con el máximo de 100), y el test no lo detecta.
- **[NIT]** `AdsServiceImplTest#23`. El campo se llama `scoreService`, pero la clase probada es `AdsServiceImpl`. Además, en JUnit 5 los métodos de test no necesitan ser `public`.

---

## Build y proyecto

### El jar generado no se puede ejecutar (`pom.xml`) [IMPORTANTE]

El proyecto compila, pero el `pom.xml` no incluye `spring-boot-maven-plugin`. Sin él, `mvn package` genera un jar sin clase principal ni dependencias, y `java -jar` falla con el error "no hay ningún atributo de manifiesto principal". Solo se puede arrancar con `mvn spring-boot:run`, lo que impide desplegarlo de la forma habitual.

### Versiones sin soporte (`pom.xml#15` y `pom.xml#20`) [SUGERENCIA]

Spring Boot 2.1.3 es de 2019, y toda la rama 2.x dejó de tener soporte gratuito a finales de 2023. Java 8 es de 2014 y ya hay cuatro versiones LTS posteriores. Con estas versiones no llegan parches de seguridad gratuitos. Spring Boot 4 salió en noviembre de 2025 y exige como mínimo Java 17. No es algo de esta PR, pero propondría planificar la migración a Spring Boot 4 con Java 21 o 25 como una tarea técnica aparte, porque mezclarla con estos cambios haría imposible revisarlos.

### Otros detalles

- **[NIT]** `README.md#3`. El badge de integración continua apunta a otro repositorio y a travis-ci.org, que ya no existe.
- **[NIT]** `.gitignore`. No ignora los ficheros de IntelliJ (`.idea/` y `*.iml`) ni `.DS_Store`.

---

## Lo que está bien

- **La estructura de paquetes.** La separación en `domain`, `application` e `infrastructure` deja clara la intención de seguir una arquitectura hexagonal y hace fácil orientarse en el proyecto.
- **`AdRepository` como puerto del dominio.** El dominio define qué necesita y la infraestructura decide cómo lo hace. Cambiar la memoria por una base de datos solo requeriría un adaptador nuevo, y permite probar el servicio con un mock.
- **Modelos de persistencia separados del dominio.** `AdVO` guarda las fotos como ids y `Ad` las tiene como objetos, y la conversión vive en el adaptador. El dominio no depende de cómo se almacenan los datos.
- **Enums para `Typology` y `Quality`.** Dentro del dominio no se trabaja con cadenas, así que un error al escribirlas no compila. Las cadenas solo aparecen en persistencia y en la API.
- **Un DTO para cada consumidor.** `PublicAd` no expone la puntuación ni la fecha de irrelevancia, mientras que `QualityAd` sí las incluye para el equipo de calidad.
- **`Ad#isComplete` en la entidad.** Es una regla de negocio dentro del dominio y es el ejemplo a seguir para el resto de reglas.
- **Un controlador fino.** `AdsController` no tiene lógica, solo delega en el servicio y construye la respuesta.
