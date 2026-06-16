# 19 -- Blueprint: BP_PoisonTrigger
### Blueprint: BP_PoisonTrigger
### Tipo: Actor
### Base Asset: N/A -- Blueprint nuevo Light Paradox
### Fuente: Creacion directa -- sesion Light Paradox (Logica 6)
### Proyecto: Light Paradox - UE 5.4.4

---

## Contexto

`BP_PoisonTrigger` es el actor de prueba para el prototipo de Veneno (Logica 6).
Aplica `Effect_Poison` al jugador al detectar overlap con su collision.

Sirve ademas como **patron de referencia** para cualquier actor que necesite
aplicar un Estado al jugador: enemigos, trampas, areas de efecto, runas.

---

## Componentes

| Componente | Tipo | Notas |
|---|---|---|
| `Box` | Box Collision | Collision de activacion. Tamano sugerido: X=100, Y=100, Z=100 |
| (StaticMesh opcional) | StaticMesh | No necesario para funcionalidad |

---

## EventGraph confirmado

```
On Component Begin Overlap (Box)
  → Cast To BP_Character_Player (Object: Other Actor)
      Cast Failed → [termina]
      Cast Success →
        As BP_Character_Player
          → Get [AbilitySystemComponent]
          → Ability System Component (output)
        Make DataTableRowHandle
          Data Table: DT_StatusEffects
          Row Name: Effect_Poison
        Make STR_SaveData_StatusEffect
          Effect Handle: resultado del DataTableRowHandle
          Instigator Is Owner: False
          Stack: 1
          Time Remaining: [Duration de Effect_Poison]   ← ver nota critica
        Load Status Effect
          Target: Ability System Component
          Save Data: resultado del Make STR_SaveData_StatusEffect
```

---

## Nota critica -- Time Remaining

**Estado actual (prototipo):** `Time Remaining` esta hardcodeado con el valor
de `Duration` de `Effect_Poison` en el nodo `Make STR_SaveData_StatusEffect`.

**Pendiente para produccion:** Leer `Duration` directamente del DT para evitar
desincronizacion si el valor cambia.

### Fix pendiente -- leer Duration del DT

Agregar antes del `Make STR_SaveData_StatusEffect`:

1. Nodo `Get Data Table Row`
   - `Data Table`: `DT_StatusEffects`
   - `Row Name`: `Effect_Poison`
2. Del pin `Out Row` → nodo `Break STR_StatusEffectInstance`
3. Del Break → pin `Duration`
4. Conectar `Duration` al pin `Time Remaining` del `Make STR_SaveData_StatusEffect`

Con este fix, si se cambia `Duration` en el DT el trigger lo lee automaticamente.

---

## Patron reutilizable para otros Estados

Para crear un trigger de cualquier otro Estado, cambiar unicamente:

| Que cambiar | Donde |
|---|---|
| `Row Name` en Make DataTableRowHandle | Nombre de la fila del Estado en DT_StatusEffects |
| `Time Remaining` | Duration del nuevo Estado |

El resto del flujo es identico para cualquier Estado.

---

## Deuda tecnica registrada

| Problema | Notas | Estado |
|---|---|---|
| Time Remaining hardcodeado | Pendiente leer Duration del DT | Pendiente |
| Overlap en Box Collision del StaticMesh | Cambiado a Box Collision dedicado en sesion -- confirmado funcional | Resuelto |
| Sin Switch Has Authority | BP_Building_Trap_Base verifica Authority antes de activar. BP_PoisonTrigger no lo hace -- aceptable para prototipo, revisar en produccion si el juego es multiplayer | Pendiente evaluacion |

---

*Archivo creado -- sesion Light Paradox (Logica 6 -- prototipo Veneno)*
*Project: Light Paradox - Base: EasySurvivalRPGv5 - UE 5.4.4*
