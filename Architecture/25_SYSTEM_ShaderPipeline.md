# 25 — Sistema: Pipeline de Importación USD y Shaders
### Sistema: Importación de assets USD/USDA (artista externa) + estructura de materiales resultante
### Base Asset: N/A — pipeline de arte propio de Light Paradox (independiente de EasySurvivalRPGv5)
### Fuente: Flujo de trabajo confirmado por el usuario — sesión Light Paradox (Shader Pipeline)
### Proyecto: Light Paradox · UE 5.4.4

---

## Contexto

La artista entrega los modelos en formato **USD/USDA**, no en formatos tradicionales
de malla (FBX, etc.). Cada entrega consiste en **4 archivos**, de los cuales **3 llevan
el label**:

| Label | Contenido |
|---|---|
| `geo` | Geometría del modelo |
| `material` | Definición de materiales (USD Preview Surface) |
| `textures` | Texturas referenciadas por los materiales |

El **4º archivo** es el archivo principal (sin label de los tres anteriores) — es el
único que se importa manualmente. Este archivo principal referencia internamente a
los otros tres (`geo`, `material`, `textures`) y Unreal los resuelve automáticamente
al importarlo.

> **Regla confirmada:** solo se importa el archivo principal. Nunca importar `geo`,
> `material` o `textures` por separado — el importador USD de Unreal sigue las
> referencias internas del archivo principal y trae todo el set correctamente
> vinculado.

---

## Procedimiento de importación

### Paso 1 — Importar el archivo principal

1. Content Browser → botón **Import** (o arrastrar el archivo `.usd`/`.usda` principal
   directamente al Content Browser).
2. Seleccionar el archivo principal entregado por la artista (el que NO lleva label
   `geo`/`material`/`textures`).
3. Unreal abre el menú **USD Import Options**.

### Paso 2 — USD Import Options — ajuste obligatorio por material

> **✅ Confirmado funcional por el usuario.**

Dentro del menú de importación, **para cada material** que trae el archivo, hay que:

1. Localizar la propiedad de shader correspondiente a **Opacity Texture**.
2. **Desmarcar (desactivar)** esa opción.

**Motivo:** con `Opacity Texture` activo durante la importación, el shader resultante
produce un **error visual equivalente a normales invertidas** — el material deja de
ocluir correctamente y se ven otras mallas "a través" del objeto. Ver diagnóstico
completo en la sección "Problema conocido — Opacity Texture" más abajo.

> **Regla:** repetir el desmarcado de `Opacity Texture` para **cada material** listado
> en el menú de importación, no solo el primero. El menú USD Import Options lista los
> materiales del archivo individualmente — cada uno tiene su propio checkbox.

### Paso 3 — Confirmar importación

Con `Opacity Texture` desmarcado en todos los materiales del archivo, continuar con la
importación normal. Con este único ajuste el shader funciona correctamente — **no se
requiere ningún paso adicional** para que el modelo se vea bien en el nivel.

---

## Problema conocido — Opacity Texture activo causa oclusión rota

### Síntoma

La textura del material se ve correctamente, pero el mesh muestra otras mallas que
están detrás de él — comportamiento visualmente idéntico a cuando las normales de
una malla están invertidas, aunque la causa real es distinta.

### Causa raíz (diagnóstico)

Al dejar `Opacity Texture` activo en el importador USD, Unreal arma el material
resultante con `Blend Mode` en un modo translúcido (en vez de `Opaque`), guiado por la
textura de opacidad del USD. Un material en modo translúcido no escribe al depth
buffer de la misma forma que uno opaco y no hace depth-sorting correctamente contra
otras mallas — el resultado es que se "ve a través" del objeto, aunque la textura en
sí esté bien.

### Fix aplicado (confirmado)

Desmarcar `Opacity Texture` en el menú **USD Import Options**, por material, antes de
confirmar la importación (ver Paso 2 arriba). Con esto el material se genera en modo
`Opaque` (o el modo correcto según el material) y el problema no ocurre.

> **Nota:** si en el futuro se necesita opacidad/transparencia real para algún
> material específico (vidrio, cristal, efectos), no usar este checkbox del
> importador — en su lugar, configurar manualmente `Blend Mode = Translucent` en el
> material ya importado y revisar iluminación (Skylight, Build Lighting, Reflection
> Captures) según la sección siguiente.

---

## Estructura de materiales resultante tras la importación

### Jerarquía confirmada

El pipeline de importación USD de Unreal genera una cadena de materiales, no un
material plano:

```
UsdPreviewSurface1                          ← Material base (Engine/USD plugin), editable
      ↑ Parent
UsdPreviewSurfaceTranslucent1                ← Material Instance intermedio (generado por el importador)
      ↑ Parent
MI_[NombreAsset]Base (ej. MI_CuerpoBase)     ← Material Instance final, el que se aplica al mesh
```

> **⚠️ Nota de nomenclatura:** pese al nombre `UsdPreviewSurfaceTranslucent1`, este
> material intermedio no implica necesariamente que el `Blend Mode` final sea
> `Translucent` — depende de cómo haya quedado configurado tras el fix del Paso 2.
> Confirmar `Blend Mode` real inspeccionando el material, no asumir por el nombre.

### Por qué la primera ventana que se abre no tiene grafo de nodos

Al abrir `MI_[NombreAsset]Base` (el Material Instance final, el que está asignado al
mesh en el nivel) se abre una ventana con una lista de parámetros (**Global Scalar
Parameters**, **Global Texture Parameters**, **Global Vector Parameters**), **sin**
grafo de nodos visible. Esto es esperado — los Material Instance nunca muestran el
grafo; solo exponen los parámetros que el Material padre decidió dejar editables.

Para editar el grafo (agregar nodos, cambiar conexiones) hay que subir en la
jerarquía hasta `UsdPreviewSurface1`, que es el único nivel editable con grafo
completo.

### Parámetros confirmados en `MI_[NombreAsset]Base` (ejemplo: `MI_CuerpoBase`)

| Grupo | Parámetro | Notas |
|---|---|---|
| Global Texture Parameters | `AmbientOcclusionTexture` | |
| Global Texture Parameters | `BaseColorTexture` | |
| Global Texture Parameters | `EmissiveColorTexture` | Desmarcado por defecto (usa `WhiteSquareTexture`, sin efecto) |
| Global Texture Parameters | `MetallicTexture` | |
| Global Texture Parameters | `NormalTexture` | |
| Global Texture Parameters | `OpacityTexture` | Desmarcado por defecto (usa `127grey`) — resultado directo del fix del Paso 2 |
| Global Texture Parameters | `RefractionTexture` | Desmarcado por defecto (usa `127grey`) |
| Global Texture Parameters | `RoughnessTexture` | |
| Global Scalar Parameters | `Opacity` | Valor confirmado `1.0` |
| Global Scalar Parameters | `Refraction` | Valor confirmado `1.4` |
| Global Scalar Parameters | (patrón por canal) `[Canal]Rotation`, `[Canal]UVIntensity` | Confirmado para BaseColor, Metallic, Normal, Emissive, Opacity, Refraction — cada canal de textura tiene su propio control de rotación/tiling de UV expuesto |

> **Nota:** `OpacityTexture` y `RefractionTexture` aparecen desmarcados y en gris en
> instancias ya importadas con el fix del Paso 2 aplicado — es el estado esperado,
> no un error.

---

## Cómo editar el shader (flujo para el custom shader próximo)

### Regla de oro — nunca editar material de Engine Content directamente

`UsdPreviewSurface1` (y `UsdPreviewSurfaceTranslucent1` si corresponde) son
materiales que vienen con el plugin USD de Unreal — son Engine Content, no assets del
proyecto. Mismo principio que **Rule 4.1** de `03_LIGHTPARADOX_PROJECT_RULES.md`
(nunca editar el asset base directamente, siempre trabajar en una copia/clase hija):
no se edita el grafo de `UsdPreviewSurface1` original.

### Flujo correcto — duplicar y reconectar jerarquía

1. Content Browser → click derecho sobre el material base (`UsdPreviewSurface1` o
   equivalente) → **Duplicate**.
2. Guardar el duplicado dentro de la carpeta del proyecto (ej.
   `Content/LightParadox/Materials/M_[NombreAsset]_Custom`).
3. Abrir `MI_[NombreAsset]Base` → panel Details → scroll hasta arriba → campo
   **Parent**.
4. Cambiar el `Parent` para que apunte al material duplicado editable, en vez de a
   `UsdPreviewSurfaceTranslucent1`/`UsdPreviewSurface1` original.
   - Si existen varios Material Instance intermedios en la cadena y alguno de ellos
     también es editable (no Engine Content), se puede optar por redirigir ese
     intermedio hacia el duplicado en vez de saltarlo — a elección, ambos caminos
     son válidos.
5. `Save`. El mesh no debería cambiar de aspecto en este punto — el duplicado es
   idéntico al original hasta que se edite su grafo. Si el aspecto visual se
   mantiene igual, la reconexión fue exitosa.
6. A partir de acá, todo el grafo del duplicado es editable libremente — este es el
   punto de partida para el custom shader tipo Genshin.

### Nota — Emissive Color no sirve para saturar colores

Confirmado en sesión: subir la intensidad de `Emissive Color` **no** satura/vibra los
colores del material — **suma** luz blanca por encima de todo lo demás (es un canal
additive), lavando el resultado hacia blanco cuanto más alto se sube.

- Para colores más vibrantes/saturados: usar un nodo `Desaturation` con `Fraction`
  negativo (ej. `-0.3` a `-0.6`) sobre el `Base Color`, expuesto como Scalar
  Parameter (ej. `BaseColorSaturation`) para ajustar por instancia.
- Para que el material brille sin lavar el color: el Vector Parameter que alimenta
  `Emissive Color` debe tener un color saturado específico (no blanco/gris) antes de
  multiplicarlo por la intensidad — así el brillo mantiene el tono del material en
  vez de blanquearlo.

---

## Checklist de diagnóstico — material se ve negro tras importar

Aplica sobre todo a materiales que quedan en modo `Translucent` (intencional o no).

1. **Skylight en el nivel** — los materiales Translucent dependen de reflejos de
   entorno para no verse negros (a diferencia de los opacos). Verificar que exista un
   `Sky Light` en la escena.
2. **Lighting Mode del material** (panel Details del material, sección
   `Translucency`) — si está en `Surface TranslucencyVolume` (default), requiere
   Volumetric Lightmaps construidos (`Build Lighting`).
3. **Build Lighting nunca corrido** — causa muy común de "todo se ve negro" en
   translúcidos mientras los opacos se ven bien.
4. **Reflection Capture** cerca del mesh — `Place Actors` → `Sphere Reflection
   Capture` → colocar cerca → `Update Captures`/Build.
5. **Valores de `Opacity`** — confirmar que no esté en un valor bajo combinado con
   `OpacityTexture` en gris medio (`127grey` ≈ 0.5), lo cual puede leerse como "casi
   invisible = negro" sobre fondo oscuro.
6. **Modo de vista** — confirmar `Lit`, no `Unlit`/`Wireframe`.

---

## Pendientes / deuda técnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Confirmar si `UsdPreviewSurfaceTranslucent1` es editable o Engine Content en todos los casos | Depende de cómo el importador USD genera la cadena — puede variar | ⏳ Pendiente verificar caso a caso |
| Documentar nombre real del material duplicado tras `Duplicate` | Unreal asigna sufijo automático (`_Copy` u otro) — no hay convención de nombre fijada todavía | ⏳ Pendiente — definir convención de nombres para materiales custom (ej. `M_[NombreAsset]_Custom`) |
| Validar si el fix de `Opacity Texture` desmarcado aplica igual para materiales que sí necesitan transparencia real (vidrio, cristal) | El Paso 2 asume que ningún material de la entrega actual necesita opacidad real vía USD — revisar si la artista entrega algún material con transparencia intencional en el futuro | ⏳ Pendiente — sin casos de prueba todavía |
| Formalizar convención de nombres de Vector/Scalar Parameters nuevos agregados en shaders custom (ej. `BaseColorSaturation`, `EmissiveIntensity`) | Para mantener consistencia entre materiales distintos del proyecto | ⏳ Pendiente — definir al iniciar el custom shader tipo Genshin |

---

## Ver también

- `03_LIGHTPARADOX_PROJECT_RULES.md` — Rule 4.1, principio de no editar assets base directamente (aplicado aquí a materiales de Engine Content).
- Próxima sesión: implementación de custom shader tipo Genshin Impact sobre el material duplicado (`M_[NombreAsset]_Custom`) como base de partida.

---

*Archivo creado — sesión Light Paradox (Shader Pipeline)*
*Cambios: documentación inicial del flujo de importación USD (4 archivos, solo se importa el principal), fix de Opacity Texture en USD Import Options, estructura de jerarquía de materiales resultante (UsdPreviewSurface1 → UsdPreviewSurfaceTranslucent1 → MI_[NombreAsset]Base), flujo de duplicación para edición custom, nota sobre Emissive vs saturación, checklist de diagnóstico de material negro.*
*Project: Light Paradox · UE 5.4.4*
