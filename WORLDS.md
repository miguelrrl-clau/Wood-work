# Análisis sistemático y plan para multi-mundo

> Objetivo: entender los **flujos** que ya tenemos en el mundo "Aserradero", encontrar qué está
> acoplado/hardcodeado, y proponer una abstracción (`WorldConfig`) para replicar el mismo juego
> en otros temas (hotel, restaurante, huerto, etc.) sin reescribir la lógica.

---

## 1. El flujo universal (lo que el juego realmente modela)

Todos estos juegos idle son **el mismo grafo de producción**:

```
  FUENTE ──cosecha──> BUFFER_CRUDO ──proceso──> BUFFER_PROCESADO ──transporte──> ESTANTE
     │                                                                              │
   (árboles)                                                                    (demanda)
                                                                                   ▼
                                          DINERO_EN_CAJA <──pago── CLIENTES ──compran──┘
                                                │
                                          (cobrar) ──> CUENTA ──reinvertir──> mejoras
```

En el mundo actual:

| Etapa abstracta | En "Aserradero" | Variable / objeto |
|---|---|---|
| Fuente | Árboles del bosque | `trees[]` |
| Cosechar | Talar (jugador / leñador / tractor) | `doAction`, `updateWorkers` |
| Buffer crudo | Patio de entrada del aserradero | `s.millIn` (cap `millInCap`) |
| Procesar | Sierra corta madera→tablas | `s.millProcRate`, animación `cutT` |
| Buffer procesado | Salida del aserradero | `s.millOut` (cap `millOutCap`) |
| Transporte | Llevar tablas a la tienda | jugador / vendedor / banda |
| Estante | Repisa de la tienda | `s.shelf` (cap `shelfCap`) |
| Demanda | Clientes que llegan | `demandPerSec`, `customers[]` |
| Pago | Cae dinero en la caja | `s.cash` |
| Cobrar | Pasar caja→cuenta | zona `cash` |
| Cuenta | Dinero gastable | `s.money` |
| Mejoras | Spots en el piso / tienda | `BUYS[]`, `WORKERS`, `MACHINES`, `TOOLS` |

**La lección central** (igual en cualquier tema): *puedes automatizar toda la producción, pero
si no hay demanda no vendes; y cada etapa tiene un límite (cuello de botella).*

---

## 2. Los "actores" (cómo se escala cada etapa)

- **Manual**: el jugador hace la tarea acercándose (talar, meter, sacar, surtir, cobrar).
- **Trabajadores (NPC humanos)**: automatizan una etapa (leñador=cosecha, carpintero=proceso,
  vendedor=transporte). Tope visible por rol (`WCAP`).
- **Máquinas**: reemplazan trabajadores cuando topan (banda=transporte; tractor→cosechadora=cosecha).
  Al comprarlas, los humanos de ese tramo **se van**.
- **Marketing**: sube la **demanda** (único modo de crecer ventas).
- **Tiers de edificio**: forma/color/tamaño cambian por nivel (`millTiers`, `storeTiers`).

---

## 3. Qué está acoplado / hardcodeado hoy (lo que hay que generalizar)

Todo vive en `index.html` con valores y posiciones fijas:

1. **Nombres de etapas y recursos**: `millIn/millOut/shelf/cash`, "madera/tablas". → deberían ser
   `stages[]` y `resources[]` con ids genéricos.
2. **Geometría/posiciones**: coordenadas mágicas (`SAW_X`, `MILLIN_POS`, `STORE_DROP`, zonas, spots).
   → deberían derivarse de la posición de cada **estación**.
3. **Modelos 3D**: `makeTractor`, `makeBuyModel`, `millTiers`, `storeTiers` están escritos a mano.
   → necesitamos un registro de "model builders" por id.
4. **Economía**: tasas/costos/caps dispersos (`woodDeliver`, `millProcRate`, `vendorRate`,
   `*Cap`, `WORKERS`, `MACHINES`, `TOOLS`, `MKT`). → un solo bloque `config.economy`.
5. **Reglas de reemplazo** (`beltActive`, `harvesterTier`, `roleReplaced`): lógica ad-hoc por rol.
   → declarar en config qué máquina reemplaza qué trabajador y a partir de qué nivel.
6. **Tiers de edificio**: thresholds y modelos hardcodeados. → `station.tiers[]`.
7. **Tema/colores**: cielo, suelo, paleta. → `config.theme`.

> Síntoma de este acoplamiento: el bug que acabas de ver — el aserradero solo cambiaba a los 2
> carpinteros porque el threshold estaba escrito a mano y desalineado con la tienda.

---

## 4. La abstracción propuesta: `WorldConfig`

Un mundo se define con **datos**, no con código. La misma máquina de juego (`engine`) lo interpreta.

```js
const WORLD = {
  id: "sawmill",
  theme: { sky: 0x8fc4e8, ground: 0x4f9a4f, accent: 0x7CFC9C },

  // recursos que fluyen por la cadena
  resources: {
    raw:       { id:"log",   label:"Madera", model:"log",   stack:"log" },
    processed: { id:"plank", label:"Tablas", model:"plank", stack:"plank" },
  },

  // la cadena = lista ordenada de etapas. El engine genera buffers, zonas y NPCs.
  chain: [
    { id:"harvest", type:"source",  from:"field(trees)",   to:"raw",
      manual:"chop", workers:["lumberjack"], machines:["tractor"] },
    { id:"process", type:"convert", station:"mill", in:"raw", out:"processed",
      rate:(s)=>2.5+ s.lvl.process*2.5, outCap:(s)=>20+s.lvl.process*8, inCap:()=>999,
      machineAnim:"saw" },
    { id:"deliver", type:"move", from:"processed", to:"shelf",
      manual:"carry", workers:["seller"], machines:["belt"] },
    { id:"sell", type:"demand", shelf:"shelf",
      price:{min:3,max:40,ref:10,elasticity:1.25}, customers:true, payTo:"cash" },
    { id:"collect", type:"bank", from:"cash", to:"money" },
  ],

  // estaciones físicas (posición + modelos por tier)
  stations: {
    mill:  { pos:[-12,12], levelVar:"process",
             tiers:[ millT0, millT1, millT2, millT3 ], tierAt:[0,1,3,6] },
    store: { pos:[ 12,12], levelVar:"sell",
             tiers:[ storeT0, storeT1, storeT2, storeT3 ], tierAt:[0,1,3,6] },
  },

  // agentes (humanos) y máquinas que los reemplazan
  agents: {
    lumberjack:{ kind:"human", model:"lumberjack", task:"harvest", rate:0.8, capVisible:5 },
    seller:    { kind:"human", model:"seller",     task:"deliver", rate:3,   capVisible:2 },
    process:   { kind:"human", model:"carpenter",  task:"process", rate:2.5, capVisible:3, stationary:"mill" },
  },
  machines: {
    tractor:{ replaces:"lumberjack", rate:6, model:"tractor",
              evolve:[{at:3,model:"harvester",scale:1.4},{at:6,scale:1.9}] },
    belt:   { replaces:"seller", rate:9, model:"belt", route:["mill.out","store.in"] },
  },

  // mejoras compradas en spots del piso
  upgrades: [
    { id:"tool",    spot:[4,5],    label:"Sierra mejor", kind:"toolLevel", costs:[40,250,1500] },
    { id:"process", spot:[-17,9],  label:"+Aserradero",  kind:"worker", base:60, scale:1.20 },
    { id:"sell",    spot:[17,9],   label:"+Tienda",      kind:"worker", base:90, scale:1.22 },
    { id:"tractor", spot:[-22,13], label:"Tractor",      kind:"machine", base:300, scale:1.25 },
    { id:"belt",    spot:[3,20.5], label:"Banda",        kind:"machine", base:400, scale:2.2 },
  ],

  marketing: [ /* mismos canales: suben demanda */ ],
};
```

Con esto, el `engine` (lo que hoy es el loop + helpers) genera **solo a partir de datos**:
buffers, caps, zonas del jugador, spots de compra, NPCs, tiers de edificio, demanda y caja.

---

## 5. Cómo se mapean otros mundos (mismas piezas, otro tema)

| Pieza | 🌲 Aserradero | 🏨 Hotel | 🍝 Restaurante | 🍓 Huerto |
|---|---|---|---|---|
| Fuente | Árboles | Huéspedes que llegan | Despensa / ingredientes | Plantas/parcelas |
| Cosechar | Talar | Check-in / limpiar cuarto | Tomar orden | Recolectar fruta |
| Crudo | Troncos | Cuarto sucio | Ingredientes crudos | Fruta sin lavar |
| Procesar | Sierra | Limpieza/servicio | Cocina | Lavado/empaque |
| Procesado | Tablas | Cuarto listo | Platillo | Caja de fruta |
| Transporte | A la tienda | A recepción | Mesero a la mesa | A la tienda |
| Estante/Demanda | Repisa+clientes | Cuartos disponibles vs reservas | Mesas vs comensales | Anaquel vs clientes |
| Máquinas | Banda, tractor | Carrito de limpieza, ascensor | Banda de cocina, robot | Tractor, banda de lavado |
| Tiers edificio | Cabaña→fábrica | Motel→hotel→resort | Fondita→restaurante→cadena | Parcela→invernadero→agro |

**Conclusión clave**: no cambia la *lógica*, solo cambian **datos, modelos y etiquetas**. Por eso
conviene invertir en el `engine` data-driven antes de hacer un segundo mundo.

---

## 6. Roadmap de refactor (incremental, sin romper lo que funciona)

1. ✅ **HECHO — Extraer `CFG`** (1 bloque arriba en `index.html`): tema (cielo/suelo/niebla),
   economía (tasas, caps, demanda, precio/elasticidad) y umbrales de tier/consolidación. Los
   helpers (`woodDeliver`, `millProcRate`, `vendorRate`, `*Cap`, `tierOf`, `harvesterTier`, etc.)
   ya leen de `CFG`. Comportamiento idéntico; ahora se tunea/replica desde un solo objeto.
   *(Se llama `CFG` y no `WORLD` porque `WORLD` ya era el radio del mapa.)*
2. **Registro de modelos** `MODELS = { log, plank, tractor, harvester, lumberjack, belt, millT0.. }`
   y que `makeBuyModel`/NPCs/tiers lean de ahí.
3. **Estaciones como objetos**: derivar `SAW_X`, zonas y spots de `station.pos`.
4. **Cadena genérica**: convertir el bloque "SIMULACIÓN DE NEGOCIO" del loop en un intérprete de
   `WORLD.chain` (source→convert→move→demand→bank).
5. **Agentes/máquinas genéricos**: una sola IA parametrizada (seek→act→haul) leyendo `agents`/`machines`.
6. **Tiers genéricos**: `station.tiers[]` + `tierAt[]`.
7. Recién entonces: **crear `worlds/hotel.js`** etc. como puros datos.

Cada paso deja el juego funcionando; el mundo "Aserradero" se vuelve el primer `WorldConfig`.

---

## 7. Estado de flujos (correcciones y pendientes)

**Corregido**
- Edificios sí evolucionan (antes el aserradero pedía 2 carpinteros; ahora cambia desde la 1ª
  compra, con pop de construcción).
- Botón **Reiniciar** para probar flujos desde 0.
- Cobrar instantáneo; cargar/descargar rápido.
- Colisión dura (nada atraviesa nada salvo árboles); NPCs rodean obstáculos.

**A vigilar / siguiente**
- Cuellos de botella deben quedar claros en cada etapa (la lección ya los detecta).
- Balance de costos al consolidar máquinas (que no sea trivial pasar a cosechadora).
- El refactor del punto 6 es lo que habilita los otros mundos "fácil".
