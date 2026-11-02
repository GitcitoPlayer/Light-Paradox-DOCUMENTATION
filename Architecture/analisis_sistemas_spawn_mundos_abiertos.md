# Sistemas de Spawn en mundos abiertos

## Lógica de aparición, población, ubicación, persistencia y regeneración

Este documento reúne un análisis de diseño sobre cómo los videojuegos de
mundo abierto controlan la aparición de objetos, recursos y fauna. El
enfoque combina diseño de sistemas y conceptos técnicos aplicables a
motores como Unreal Engine y Unity.

------------------------------------------------------------------------

## 1. ¿Qué controla un sistema de Spawn?

No existe una única regla para generar todos los elementos de un mundo.
Un juego suele combinar distintos sistemas para decidir cuándo, dónde,
cuántos y bajo qué condiciones aparecen.

Cinco variables fundamentales:

1.  **Population Cap --- Límite de población:** cuántas instancias de un
    elemento pueden existir simultáneamente en una zona o en todo el
    mundo.
2.  **Spawn Locations --- Ubicaciones:** en qué lugares está permitido
    generar un elemento.
3.  **Spawn Timing --- Frecuencia:** cada cuánto tiempo el sistema
    intenta generar una nueva instancia.
4.  **Respawn Rules --- Reaparición:** qué sucede cuando el jugador
    recoge, destruye o elimina un elemento.
5.  **Spawn Conditions --- Condiciones:** qué circunstancias deben
    cumplirse: clima, hora, bioma, distancia, progresión, etc.

Estos conceptos permiten analizar gran parte de los sistemas de
población de un mundo abierto.

## 2. ¿Qué delimita cuántos elementos aparecen?

### Modelo A: población máxima

El sistema establece un límite de elementos que pueden existir
simultáneamente.

Ejemplo de límites para un bosque:

  Elemento      Máximo en el bosque
  ----------- ---------------------
  Hongos                         30
  Conejos                         8
  Ciervos                         5
  Minerales                      12

Si hay 30 hongos, el sistema no genera más. Si el jugador recoge 10, la
población baja a 20 y el sistema puede intentar reponer los faltantes.

Este modelo es útil para mantener una densidad relativamente constante.

### Modelo B: población mínima y máxima

Este modelo permite que la población fluctúe dentro de ciertos umbrales.

Ejemplo de parámetros:

-   **Spawn Minimum:** 5.
-   **Spawn Low:** 10.
-   **Spawn Maximum:** 20.

Comportamiento ilustrativo:

  -----------------------------------------------------------------------
  Población actual                    Comportamiento
  ----------------------------------- -----------------------------------
  20                                  No genera nada.

  15                                  Espera o genera lentamente.

  10                                  Comienza a reponer con mayor
                                      frecuencia.

  5                                   Intenta recuperar la población más
                                      rápidamente.

  0                                   Intenta recuperar la población
                                      mínima, si las condiciones lo
                                      permiten.
  -----------------------------------------------------------------------

Una referencia especialmente útil es la presentación de GDC de Leah
Miller, *Rewarding Exploration with Collectables and Gatherables*
(2019). Describe un sistema de grupos de spawn que utiliza parámetros
como:

-   **Spawn Group:** conjunto de puntos administrados en conjunto.
-   **Spawn Minimum:** población mínima que activa la reposición.
-   **Spawn Low:** umbral por debajo del cual aumenta la velocidad de
    aparición.
-   **Spawn Max:** límite máximo de objetos.

Referencia: [Rewarding Exploration with Collectables and Gatherables ---
GDC](https://www.gdcvault.com/).

### Modelo C: densidad por superficie

En lugar de establecer una cantidad total, el sistema determina cuántos
elementos pueden existir por unidad de espacio.

Ejemplos:

-   2 ciervos por cada 1,000 m².
-   5 hongos por cada 100 m².
-   1 mineral por cada 200 m².

Esto permite que una región grande tenga naturalmente más elementos que
una pequeña. También permite que una zona sea más abundante que otra sin
colocar manualmente cada objeto.

La distribución puede apoyarse en puntos calculados previamente y en
evaluaciones del terreno y de la densidad.

## 3. ¿Qué delimita dónde aparecen?

### A. Spawn Points

El diseñador coloca puntos o áreas específicas donde se permite generar
elementos.

Ejemplo de un bosque:

-   20 puntos potenciales para hongos.
-   8 puntos potenciales para conejos.
-   5 puntos potenciales para minerales.

No significa que todos deban estar ocupados simultáneamente. Cada punto
puede tener reglas propias:

-   Qué elementos pueden aparecer.
-   Qué probabilidad tiene cada elemento.
-   Qué distancia mínima debe existir respecto a otros elementos.
-   Qué condiciones ambientales necesita.
-   Si está disponible o temporalmente bloqueado.

Esta estrategia es útil cuando se necesita controlar la composición
visual del escenario.

### B. Biome-Based Spawning

El bioma determina qué elementos pueden aparecer.

  Bioma      Fauna posible             Recursos posibles
  ---------- ------------------------- --------------------
  Bosque     Conejos, ciervos          Hongos, bayas
  Desierto   Serpientes, escorpiones   Cactus, minerales
  Montaña    Cabras, águilas           Hierbas, cristales
  Pantano    Ranas, insectos           Plantas acuáticas

No es necesario que cada animal tenga un punto de spawn manual. Se
pueden establecer reglas que permitan su aparición dentro de
determinados ambientes.

Minecraft ofrece documentación técnica sobre filtros por bioma,
superficie, iluminación, densidad, probabilidad y tamaño de grupos.

Referencia: [Minecraft Creator Documentation --- Entity
Spawning](https://learn.microsoft.com/en-us/minecraft/creator/).

### C. Environmental Conditions

Estar en el bioma correcto puede no ser suficiente. La aparición también
puede depender de:

-   Hora del día.
-   Estación del año.
-   Clima.
-   Temperatura.
-   Altitud.
-   Tipo de suelo.
-   Proximidad al agua.
-   Presencia de vegetación.
-   Progresión del jugador.

Ejemplo: una especie de hongo podría necesitar estar en un bosque,
aparecer sobre suelo húmedo, tener una temperatura determinada y tener
mayor probabilidad durante la noche.

Estas condiciones permiten que los elementos tengan una relación
coherente con el mundo.

## 4. ¿Qué pasa cuando el jugador no recoge un elemento?

Aquí se separan los sistemas de spawn de los sistemas de persistencia.
Hay cuatro comportamientos habituales:

### A. Persistencia indefinida

El objeto permanece en el mundo hasta que el jugador lo recoge o
destruye.

Es habitual en recursos estáticos, cofres y coleccionables importantes.

### B. Persistencia con regeneración

El objeto permanece, pero puede cambiar de estado con el tiempo: una
planta vuelve a crecer, un árbol recupera sus frutos o un nodo regenera
sus recursos.

### C. Desaparición por tiempo

El objeto o criatura desaparece después de cierto tiempo, incluso si el
jugador nunca interactúa con él.

Puede utilizarse para fauna temporal, drops y objetos ambientales.

### D. Simulación de población

El elemento puede abandonar la zona, desplazarse o dejar de estar
representado físicamente. El sistema conserva o reconstruye la población
cuando el jugador regresa.

**Distinción importante:** que un elemento no esté visible no significa
necesariamente que haya dejado de existir en los datos del juego.

Esto es especialmente importante en mundos abiertos grandes, porque
mantener cada animal, insecto y objeto activo permanentemente sería
costoso.

## 5. ¿Qué sucede cuando el jugador abandona una zona?

Este problema pertenece a la simulación del mundo y al *level
streaming*.

Imaginemos que el jugador entra a un bosque y encuentra:

-   3 ciervos.
-   2 conejos.
-   15 hongos.
-   4 minerales.

Después se marcha a otra región durante 20 minutos y regresa. El juego
puede aplicar tres enfoques:

  -----------------------------------------------------------------------
  Enfoque                             Qué sucede al regresar
  ----------------------------------- -----------------------------------
  Persistencia                        Encuentra los mismos elementos y
                                      estados.

  Simulación simplificada             El juego calcula qué debería haber
                                      cambiado durante su ausencia.

  Regeneración                        El sistema vuelve a generar la
                                      población según sus reglas
                                      actuales.
  -----------------------------------------------------------------------

### Caso de estudio: State of Decay 2

*State of Decay 2* es una referencia útil para entender la
administración de población fuera de pantalla.

Su equipo explicó un sistema llamado **Ambient Population Manager**. La
idea es que no todos los zombis se mantienen físicamente activos en el
mundo. Cuando el jugador se desplaza hacia una zona nueva, el sistema
genera población según el contexto. Cuando abandona una zona, muchos
zombis pasan a una representación probabilística de la población.

Los enemigos especiales pueden conservar su ubicación y estado para que
el jugador pueda reencontrarlos posteriormente.

Referencia: [Game Developer --- Procedurally Generating Enemies, Places,
and Loot in State of Decay 2](https://www.gamedeveloper.com/).

## 6. Ejemplo integrado: sistema de recolección de hongos

Supongamos un bosque de un RPG con estas reglas:

### Configuración: Mushroom Population

  Parámetro                         Valor de ejemplo
  ------------------------------- ------------------
  Población mínima                                 5
  Población máxima                                20
  Puntos potenciales                              40
  Cooldown de regeneración                10 minutos
  Distancia mínima entre hongos             3 metros
  Probabilidad de aparición                      60%

### Condiciones de aparición

-   Bioma: bosque.
-   Terreno: suelo natural.
-   Distancia mínima: 3 metros entre hongos.
-   Probabilidad de aparición: 60%.
-   Máximo: 20 hongos simultáneos.

### Condiciones de regeneración

-   Cuando se recoge un hongo, su punto queda temporalmente inactivo.
-   Después de 10 minutos, vuelve a estar disponible.
-   El sistema comprueba que no se supere la población máxima.
-   Si no hay ubicaciones válidas, no genera nada.

### Ejemplo de comportamiento

  ------------------------------------------------------------------------
  Situación                                Población Respuesta del sistema
  --------------------- ---------------------------- ---------------------
  Inicio de exploración                           20 No necesita generar
                                                     más.

  El jugador recoge 8                             12 Sigue por encima del
                                                     mínimo; no necesita
                                                     reponer
                                                     inmediatamente.

  El jugador recoge                                3 Baja del mínimo; el
  otros 9                                            sistema busca puntos
                                                     disponibles para
                                                     regenerar.

  Transcurren 10                                   8 Los puntos cuyo
  minutos                                            cooldown terminó
                                                     pueden volver a
                                                     generar hongos,
                                                     respetando límites y
                                                     condiciones.
  ------------------------------------------------------------------------

Este es un modelo simplificado. En un juego real también hay que
considerar si los puntos están disponibles, si el jugador está cerca, si
el área está cargada y si las condiciones ambientales siguen siendo
válidas.

## 7. Material recomendado para estudiar

Las referencias cumplen funciones distintas:

1.  **Rewarding Exploration with Collectables and Gatherables --- GDC,
    Leah Miller (2019).** Presentación sobre distribución y
    administración de coleccionables y recursos. Especialmente relevante
    para Spawn Groups y umbrales de población.\
    [Consultar GDC Vault](https://www.gdcvault.com/)

2.  **Procedurally Generating Enemies, Places, and Loot in State of
    Decay 2 --- Game Developer / Undead Labs.** Caso de estudio sobre
    población dinámica, simulación fuera de pantalla y generación
    contextual.\
    [Buscar el artículo en Game
    Developer](https://www.gamedeveloper.com/)

3.  **Entity Spawning --- Minecraft Creator Documentation.**
    Documentación técnica sobre reglas de aparición, densidad, biomas,
    probabilidades y grupos.\
    [Consultar Microsoft
    Learn](https://learn.microsoft.com/en-us/minecraft/creator/)

4.  **Biome Painter: Populating Massive Worlds --- Game Developer.**
    Referencia sobre distribución de objetos en grandes superficies
    utilizando densidad y características del terreno.\
    [Consultar Game Developer](https://www.gamedeveloper.com/)

*Nota:* los enlaces anteriores apuntan a los portales o búsquedas
editoriales de las publicaciones, no necesariamente a una copia directa
de cada presentación o artículo.

## 8. Recomendación de arquitectura para un videojuego

Conviene evitar un único Spawn Manager que controle absolutamente todo.
Una arquitectura conceptual puede dividirse en tres niveles:

### Nivel 1: World Population Manager

Controla la población general y el presupuesto de entidades del mundo.

### Nivel 2: Administradores especializados

-   **Resource Manager:** hongos, minerales, plantas, cofres y
    coleccionables.
-   **Wildlife Manager:** animales, fauna ambiental, grupos y migración.

### Nivel 3: Spawn Point / Spawn Group

Controla ubicaciones, reglas particulares, cooldowns, densidad y
disponibilidad.

¿Por qué esta separación?

Porque un mineral y un ciervo tienen necesidades completamente
diferentes. Un mineral puede permanecer en el mismo lugar durante
semanas de tiempo de juego. Un ciervo puede desplazarse, formar grupos,
reproducirse o abandonar una zona.

Aunque ambos utilicen un sistema de spawn, sus reglas de vida son
distintas.

## 9. Preguntas de diseño para definir el sistema

Antes de implementar, conviene responder tres preguntas:

### Economía

-   ¿Qué tan abundantes deben ser los recursos?
-   ¿El jugador debe explorar para encontrarlos o debería encontrar
    suficientes en cada zona?
-   ¿La recolección es una actividad central o secundaria?
-   ¿Los recursos se regeneran para sostener un ciclo de juego o son
    limitados?

### Ecología

-   ¿La fauna debe parecer una población viva que habita el mundo?
-   ¿Los animales se desplazan entre zonas?
-   ¿Forman grupos o tienen comportamientos territoriales?
-   ¿El jugador puede afectar la población?

### Persistencia

-   ¿El mundo debe recordar lo que hizo el jugador?
-   ¿Qué elementos necesitan persistencia exacta?
-   ¿Qué elementos pueden simularse de forma simplificada fuera de
    pantalla?
-   ¿Qué elementos pueden regenerarse al volver a cargar una zona?

### Recomendación final

Estudia por separado los sistemas de recursos recolectables y los de
fauna dinámica. Son problemas de diseño distintos. Entender esa
diferencia ayuda a construir sistemas más sólidos en Unreal Engine o
Unity.

------------------------------------------------------------------------

## Glosario breve

  -----------------------------------------------------------------------
  Término                             Significado
  ----------------------------------- -----------------------------------
  Spawn                               Generación o aparición de una
                                      entidad.

  Respawn                             Reaparición de una entidad
                                      previamente eliminada o recogida.

  Population Cap                      Límite máximo de población.

  Spawn Point                         Ubicación potencial de aparición.

  Spawn Group                         Grupo de puntos administrados en
                                      conjunto.

  Spawn Rate                          Frecuencia o ritmo de aparición.

  Cooldown                            Tiempo de espera antes de permitir
                                      otra aparición.

  Biome-Based Spawning                Aparición condicionada por el
                                      bioma.

  Persistence                         Conservación del estado de una
                                      entidad.

  World Simulation                    Simulación del estado del mundo,
                                      incluso fuera de la zona activa.

  Level Streaming                     Carga y descarga de partes del
                                      mundo según necesidad.

  Despawn                             Eliminación o retirada de una
                                      entidad del mundo activo.
  -----------------------------------------------------------------------
