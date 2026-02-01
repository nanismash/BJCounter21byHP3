# BJ Counter — Contador de Cartas Blackjack

**byHachePe** · PWA · Sistema Hi-Lo · Basic Strategy + Illustrious 18 + Fab 4

---

## Contenido

1. [Descripción](#descripción)
2. [Arquitectura del archivo](#arquitectura-del-archivo)
3. [Estado global](#estado-global)
4. [Módulos JavaScript](#módulos-javascript)
   - [Splash & Menu](#splash--menu)
   - [Audio](#audio)
   - [Límites y clamp](#límites-y-clamp)
   - [Matemáticas principales](#matemáticas-principales)
   - [Actualización del display](#actualización-del-display)
   - [Acciones del usuario](#acciones-del-usuario)
   - [Motor de desviaciones](#motor-de-desviaciones)
   - [Motor de recomendaciones](#motor-de-recomendaciones)
   - [Basic Strategy](#basic-strategy)
5. [Lógica de inferencia de mazos](#lógica-de-inferencia-de-mazos)
6. [Insurance / Even Money](#insurance--even-money)
7. [Sistema de apuestas (Kelly)](#sistema-de-apuestas-kelly)
8. [Desviaciones implementadas](#desviaciones-implementadas)
9. [PWA y Service Worker](#pwa-y-service-worker)
10. [Bugs conocidos pendientes](#bugs-conocidos-pendientes)

---

## Descripción

Aplicación web progresiva (PWA) para contar cartas en Blackjack usando el sistema Hi-Lo. Calcula Running Count, True Count y unidades de apuesta según Kelly Criterion. Proporciona recomendaciones de jugada basadas en Basic Strategy con las desviaciones Illustrious 18 y Fab 4 superpuestas según el True Count actual.

Diseñada para uso educativo. Un solo archivo HTML (~695 líneas) sin dependencias externas.

---

## Arquitectura del archivo

```
blackjack-counter.html
├── <head>
│   ├── Meta tags (viewport, PWA, theme-color)
│   ├── Service Worker registration (1 línea inline)
│   ├── <style> — Todo el CSS (~200 líneas)
│   │   ├── Reset & layout (body, .app-scroll, .container)
│   │   ├── Splash screen
│   │   ├── Menu de sistema de conteo
│   │   ├── Cards (mazos, stats, betting)
│   │   ├── Botones de cartas y quick-tap
│   │   ├── Strategy box & recomendaciones
│   │   └── Footer & disclaimer
│   └── </head>
├── <body>
│   ├── #splash — Pantalla de carga (3s)
│   ├── #menuSistema — Overlay de selección Hi-Lo / KO / Omega II
│   ├── .app-scroll > .container — Todo el contenido principal
│   │   ├── Botones de mazos (1-8)
│   │   ├── Input de cartas restantes
│   │   ├── Running Count (editable) + True Count (calculado)
│   │   ├── Unidades a apostar
│   │   ├── Quick-tap (-1 / 0 / +1) + botones carta por carta
│   │   ├── Strategy box (3 selects + botón recomendación)
│   │   ├── Botón reset
│   │   ├── Disclaimer
│   │   └── Footer (byHachePe)
│   └── <script> — Todo el JS (~350 líneas)
```

---

## Estado global

Tres variables mantienen todo el estado de la sesión:

```javascript
let decksSelected = 8;      // Cantidad de mazos del shoe actual (1-8)
let cardsRemaining = 416;   // Cartas restantes en el shoe
let runningCount = 0;       // Conteo acumulado Hi-Lo
```

El True Count y las unidades de apuesta se **calculan en tiempo real** a partir de estas tres variables; no se almacenan.

---

## Módulos JavaScript

### Splash & Menu

- **Splash:** Se oculta tras 3 segundos mediante la clase `.hide` (transición opacity).
- **Menu:** Overlay fixed con los sistemas de conteo. Solo Hi-Lo está activo; KO y Omega II están visualmente inhabilitados (`disabled`). `selectSistema()` actualmente solo cierra el menu (extensión futura).

### Audio

```javascript
ensureAudio()  // Crea AudioContext (requerido tras gesture del usuario)
playTap()      // Oscilador sine: 420Hz → 180Hz en 60ms, gain 0.18 → 0
```

Usa Web Audio API nativa. El contexto se inicializa en el primer `pointerdown` o `touchstart` para cumplir la política de autoplay del browser.

### Límites y clamp

```javascript
clampCards(v)          // Retorna v dentro de [0, decksSelected*52]
clampRunningCount(v)   // Retorna v dentro de [-(decksSelected*4), +(decksSelected*4)]
```

El límite del RC se basa en que cada mazo tiene exactamente 4 ases y 4×(10,J,Q,K) = 20 cartas de valor 10. El RC teórico máximo absoluto es cuando todas las cartas jugadas fueron del mismo signo. En la práctica el límite es `decks × 4` (4 aces por mazo = máximo desequilibrio puro por ases).

**Nota:** El clamp del RC solo se aplica en edición manual (`updateFromRunning`). Los métodos `addCard` y `addCount` no limitan porque el decremento natural de `cardsRemaining` impone límites implícitos.

### Matemáticas principales

```javascript
getDecksFraction()     // cardsRemaining / 52  →  "mazos equivalentes restantes"
getTrueCountRaw()      // runningCount / getDecksFraction()  →  TC sin redondeo
roundToHalf(n)         // Redondea al múltiplo más cercano de 0.5
calcBettingUnits(tc)   // Tabla Kelly: TC ≤1 → 1 unidad ... TC >9 → 12 unidades
```

El True Count se redondea a medio punto (0.5) para la UI y para las comparaciones de desviaciones. Esto es estándar en la práctica real de conteo.

### Actualización del display

`updateDisplay()` es la función central que sincroniza la UI con el estado. Se llama tras **cualquier** cambio de estado:

1. Escribe `cardsRemaining` en el input
2. Escribe y recolora `runningCount` (verde/rojo/neutro)
3. Calcula y muestra `trueCount` con color
4. Calcula y muestra `bettingUnits` con color y animación de pulse para valores altos

### Acciones del usuario

| Función | Efecto |
|---|---|
| `setDecks(d, btn)` | Resetea todo: decksSelected=d, cartas=d×52, RC=0, activa botón |
| `updateFromCards()` | Edición manual de cartas → infiere el botón de mazos correcto |
| `updateFromRunning()` | Edición manual de RC → aplica clamp |
| `addCard(c)` | Carta específica vista: RC += valor Hi-Lo, cartas-- |
| `addCount(v)` | Quick-tap: RC += v, cartas-- |
| `resetCount()` | Vuelve RC a 0, cartas al máximo del shoe, limpia recomendaciones |

### Motor de desviaciones

`checkDeviations(hv, dv, tc, isPair, pairCard)` — Se ejecuta **antes** de Basic Strategy. Retorna un objeto `{action, detail, warning?, alt?, altDetail?}` si hay desviación aplicable, o `null` si no.

**Orden de prioridad dentro de la función:**
1. Fab 4 (Surrender) — máxima prioridad
2. Illustrious 18 Stand
3. Illustrious 18 Hit (complementos de stand)
4. Illustrious 18 Double
5. Split deviations

Esta jerarquía importa porque algunas condiciones se solapan en los mismos `hv/dv` pero a diferentes rangos de TC.

### Motor de recomendaciones

`getRecommendation()` — Flujo principal:

```
1. Validar selección (3 cartas)
2. Contar las 3 cartas vistas (RC y cartas restantes)
3. Calcular TC actual
4. Si BJ vs no-As dealer → retorno inmediato
5. Si dealer muestra A:
   a. Si jugador tiene BJ → Even Money advice + BJ → retorno
   b. Si no → Insurance advice (continúa al paso 6)
6. checkDeviations() → si retorna algo, mostrar y retornar
7. Basic Strategy según: isPair → isSoft → hard
```

### Basic Strategy

Implementada como cascada de if/else según la clasificación de la mano:

- **Pares:** A, 10, 9, 8, 7, 6, 5, 4, 2-3 (cada uno con reglas específicas por dealer)
- **Soft:** 20 (stand), 19 (double vs 6 si TC≥1), 18, 17, 16, 15, 14, 13
- **Hard:** 21→17 (stand), 16, 15, 14, 13, 12, 11, 10, 9, ≤8

Todas las reglas corresponden a la variante **6-8 decks, Dealer hits Soft 17 (H17), Double After Split permitido (DAS)**.

---

## Lógica de inferencia de mazos

Cuando el usuario edita manualmente el campo "Cartas restantes":

```javascript
// Ejemplo: usuario pone 200 cartas
Math.ceil(200 / 52) = 4  →  botón "4" se activa
// Ejemplo: usuario pone 52 cartas  
Math.ceil(52 / 52) = 1   →  botón "1" se activa
// Ejemplo: usuario pone 417 (> 416 máximo)
Se clampea a 416         →  botón "8" se activa
// Ejemplo: usuario pone 0
Math.ceil(0/52) = 0 → se clampea a 1 → botón "1" se activa
```

El botón representa **el shoe más pequeño que puede contener esa cantidad de cartas**. No cambia la lógica matemática (el TC siempre usa `cardsRemaining/52` directamente), solo es un indicador visual.

---

## Insurance / Even Money

Cuando el dealer muestra un **Ace**, la app diferencia dos escenarios:

### Jugador tiene Blackjack → oferta Even Money
- **TC ≥ +3:** Acepta. La probabilidad del dealer de tener BJ es ≥ 1/3 (hay suficientes Tens). Even Money es matemáticamente neutral o ligeramente positivo.
- **TC < +3:** Rechaza. La probabilidad es < 1/3, cobrar 3:2 tiene mejor EV esperado.

### Jugador NO tiene Blackjack → oferta de Seguro (Insurance)
- **TC ≥ +3:** Toma seguro. Mismo razonamiento probabilístico.
- **TC < +3:** No tomes seguro. Pierde valor a largo plazo.

**Base matemática:** El seguro paga 2:1. Para que sea +EV, la probabilidad de que el dealer tenga un 10 debajo debe ser ≥ 1/3. En un mazo neutro (sin contar) esa probabilidad es ~30.8% (16/52), es decir, < 1/3. Solo cuando el TC sube a +3 o más, la concentración de Tens supera ese umbral.

---

## Sistema de apuestas (Kelly)

`calcBettingUnits(tc)` implementa una tabla de apuestas basada en Kelly Criterion adaptada para Blackjack:

| True Count | Unidades |
|---|---|
| ≤ 1 | 1 |
| 1.5 | 1.5 |
| 2 | 2 |
| 2.5 | 2.5 |
| 3 | 3 |
| 3.5 | 3.5 |
| 4 | 4 |
| 4.5 | 5 |
| 5 | 6 |
| 5.5 | 7 |
| 6 | 8 |
| 6.5-7 | 9 |
| 7.5-8 | 10 |
| 8.5-9 | 11 |
| > 9 | 12 |

El spread es 1:12 (ratio mínimo:máximo). Los valores intermedios (1.5, 2.5, 3.5) permiten una progresión más suave que reduce la varianza comparada con saltos enteros.

La visualización usa 4 niveles de color:
- **low** (blanco): 1-2 unidades
- **medium** (amarillo): 3-5 unidades
- **high** (verde): 6-9 unidades
- **very-high** (verde brillante + pulse): 10-12 unidades

---

## Desviaciones implementadas

### Fab 4 — Surrender

| Mano | Dealer | Condición | Acción |
|---|---|---|---|
| 15 | 10 | TC ≤ 0 | Surrender |
| 15 | A | TC ≤ +2 | Surrender |
| 16 | 9 | TC < +1 | Surrender |
| 16 | 10 | TC ≤ +3 | Surrender |

### Illustrious 18 — Stand

| Mano | Dealer | Condición | Vs Basic Strategy |
|---|---|---|---|
| 12 | 2 | TC ≥ +3 | BS: Hit → Dev: Stand |
| 12 | 3 | TC ≥ +2 | BS: Hit → Dev: Stand |
| 12 | 4 | TC ≥ 0 | BS: Stand (confirma) |
| 12 | 5 | TC ≥ -2 | BS: Stand (amplía rango) |
| 12 | 6 | TC ≥ -1 | BS: Stand (amplía rango) |
| 13 | 2 | TC ≥ -1 | BS: Stand (confirma en TC negativo) |
| 13 | 3 | TC ≥ -2 | BS: Stand (amplía rango) |
| 15 | 10 | TC ≥ +4 | BS: Surrender/Hit → Dev: Stand |
| 15 | 9 | TC ≥ +5 | BS: Hit → Dev: Stand |
| 16 | 9 | TC ≥ +1 | BS: Hit → Dev: Stand |
| 16 | 10 | TC ≥ +4 | BS: Surrender/Hit → Dev: Stand |
| 16 | A | TC ≥ +2 | BS: Hit → Dev: Stand |

### Illustrious 18 — Hit (complementos)

| Mano | Dealer | Condición | Vs Basic Strategy |
|---|---|---|---|
| 13 | 2 | TC < -1 | BS: Stand → Dev: Hit |
| 13 | 3 | TC < -2 | BS: Stand → Dev: Hit |

### Illustrious 18 — Double

| Mano | Dealer | Condición | Vs Basic Strategy |
|---|---|---|---|
| 9 | 2 | TC ≥ +1 | BS: Hit → Dev: Double |
| 9 | 3 | TC ≥ 0 | BS: Hit → Dev: Double |
| 9 | 7 | TC ≥ +3 | BS: Hit → Dev: Double |
| 10 | 10 | TC ≥ +4 | BS: Hit → Dev: Double |
| 10 | A | TC ≥ +4 | BS: Hit → Dev: Double |
| 11 | A | TC ≥ +1 | BS: Hit → Dev: Double |

### Split Deviations

| Par | Dealer | Condición | Acción |
|---|---|---|---|
| 4-4 | 5, 6 | TC ≥ +2 | Split (confirma BS) |
| 10s | 4 | TC ≥ +4 | Split (vs BS: Stand) |
| 10s | 5, 6 | TC ≥ +5 | Split (vs BS: Stand) |
| 9-9 | 7 | TC ≥ +3 | Stand en 18 (no split) |
| 8-8 | 10 | TC ≤ -1 | Hit (no split) |
| 8-8 | A | TC ≤ -1 | Hit (no split) |

### Insurance / Even Money

| Situación | Condición | Acción |
|---|---|---|
| BJ propio vs A dealer | TC ≥ +3 | Acepta Even Money |
| BJ propio vs A dealer | TC < +3 | Rechaza Even Money |
| Sin BJ vs A dealer | TC ≥ +3 | Toma Seguro |
| Sin BJ vs A dealer | TC < +3 | No tomes Seguro |

---

## PWA y Service Worker

Tres archivos adicionales hacen la app instalable:

- **manifest.json** — Nombre, iconos, orientación portrait, theme color
- **sw.js** — Service Worker que cachea todos los assets para uso offline
- **index.html** — Copia del HTML principal (algunos browsers requieren este nombre para la instalación PWA)

---

## Bugs conocidos pendientes

Identificados en el último audit sistemático. Ordenados por severidad:

### 🔴 Bug real — RC no se reclampea al cambiar mazos via cartas

**Situación:** El usuario tiene RC=+20 con 8 mazos (válido, máximo ±32). Luego edita las cartas restantes a 100. La función `updateFromCards` infiere 2 mazos → `decksSelected=2`. Pero el RC permanece en +20, que excede el nuevo máximo (±8). El TC resultante es +10.4, absurdo para un shoe de 2 mazos.

**Ubicación:** `updateFromCards()` — falta llamar a `clampRunningCount` después de cambiar `decksSelected`.

**Fix propuesto:**
```javascript
// Al final de updateFromCards(), antes de updateDisplay():
runningCount = clampRunningCount(runningCount);
document.getElementById('runningCount').value = runningCount;
```

---

### 🟡 Minor — TC=3.5 con 16 vs 10 cae a Surrender en lugar de Stand

**Situación:** Fab 4 cubre 16v10 con `tc<=3` (Surrender). I18 cubre 16v10 con `tc>=4` (Stand). El valor tc=3.5 no es capturado por ninguna desviación y cae a Basic Strategy, que da Surrender contra 10.

En la práctica, a TC=3.5 el Stand ya sería marginalmente mejor. Es un edge case rarísimo (TC debe estar exactamente en 3.5).

**Fix propuesto:** Cambiar la condición de I18 stand 16v10 de `tc>=4` a `tc>=3.5`.

---

### 🟡 Minor — 0 cartas restantes muestra botón "1"

**Situación:** Cuando el shoe se agota completamente (0 cartas), `Math.ceil(0/52)=0` se clampea a 1 y el botón "1" se ilumina. Visualmente confuso si el shoe original era de 8 mazos.

**Fix propuesto:** Guardar el shoe original en una variable separada (`deckOrigin`) y usarlo como fallback cuando `cardsRemaining===0`, o simplemente no cambiar el botón cuando el valor es 0.

---

### 🔵 Arquitectura — Desviación Soft 19 vs 6 incrustada en Basic Strategy

La regla "Dobla Soft 19 contra 6 cuando TC≥+1" está codificada directamente en el bloque de Basic Strategy en lugar de en `checkDeviations()`. Funciona correctamente pero es inconsistente con todas las demás desviaciones que pasan por el motor centralizado. No afecta el comportamiento.
