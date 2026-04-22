# View Specifications

Este documento describe el sistema de diseño y los componentes visuales del Advisory Board.

## Referencias Visuales

- **Dia** (dia.so) — estética base: fondo muy claro con gradiente sutil, mucho espacio, sensación premium y respirada. CTA pill-shaped.
- **Consensus** (consensus.app) — estructura de pantalla: input centrado, acciones secundarias como pills, jerarquía clara.
- **Propelland Brand Guidelines** — identidad corporativa: tipografía, paleta de color oficial.

---

## Sistema de Diseño

### Tipografía

Dos fuentes: **Eiko** para identidad de marca (títulos y destacados), **Inter** para todo el UI.

#### Escala tipográfica Propelland (adaptada a web)

| Estilo | Fuente | Peso | Tamaño web | Letter-spacing | Uso en la app |
|--------|--------|------|------------|----------------|---------------|
| Title | Eiko | Medium | `2.5rem` | `-0.02em` | Título del board en estado idle |
| Highlights | Eiko | Medium | `2rem` | `0.02em` | Headings de sección, Decision Brief title |
| Subheading | Inter | Medium | `1rem` | `0` | Nombre de experto, headings de card |
| Subheading Caps | Inter | Medium | `0.75rem` | `0.15em` | Labels de sección en uppercase (SEAT, VERDICT) |
| Body text | Inter | Regular | `0.875rem` | `0` | Respuestas de agentes, body general |
| Caption | Inter | Regular | `0.75rem` | `0` | Credenciales, metadata, timestamps |
| Section Title | Inter | Light | `0.75rem` | `0` | Labels de tabla, títulos de panel, uppercase |

---

### Paleta de Color

#### Colores Propelland (base)

| Nombre | Hex | Uso en la app |
|--------|-----|---------------|
| Propelland Blue | `#0042FF` | Color primario del board por defecto, CTAs |
| Black | `#130E1E` | Texto principal |
| White | `#FDFDFD` | Fondo de página, superficies |
| Spa Mint | `#4EF0B6` | Acento secundario, indicador de streaming activo |
| Propelland Chalk | `#1F1F1F` | Texto en fondos oscuros (no usado en light mode) |
| Dark Grey | `#34343B` | Texto secundario en contextos de alto contraste |
| Medium Grey | `#778186` | Labels secundarios, credenciales |
| Light Grey | `#CBD6DC` | Bordes, separadores |

#### Color primario por board

Cada board define su color primario en `board.yaml` bajo `theme.primary`. Se carga como CSS custom property:

```css
:root {
  --board-primary: #0042FF; /* valor del board.yaml */
}
```

El color primario por defecto es Propelland Blue. Cada board puede sobreescribirlo:

```yaml
# board.yaml
theme:
  primary: "#0042FF"   # Veterinary — Propelland Blue
  primary: "#4EF0B6"   # B2B — Spa Mint
  primary: "#130E1E"   # Executives — Black
```

**Usos del color primario:**
- Borde o acento del título del board
- Focus ring del input
- Pills de preguntas de ejemplo
- Borde izquierdo de la expert card en estado streaming

#### Colores de veredicto (fijos — no cambian con el board)

Colores semánticos universales. Para "Endorse" se usan los verdes de Propelland.

| Veredicto | Color texto | Color fondo |
|-----------|------------|-------------|
| Endorse | `#177E59` (Propelland Dark Green) | `#BEF2DF` (Propelland Light Green) |
| Endorse with conditions | `#B45309` | `#FEF3C7` |
| Defer / Uncertain | `#C2410C` | `#FFEDD5` |
| Do Not Endorse | `#B91C1C` | `#FEE2E2` |
| Pending / No verdict | `#778186` (Medium Grey) | `#F1F5F9` |

---

### Espaciado y Radio

- **Sistema de espaciado**: escala de 4px (Tailwind default)
- **Radio de botones y pills**: `9999px` (full pill)
- **Radio de cards**: `12px` (`rounded-xl`)
- **Radio de inputs**: `12px` (`rounded-xl`)
- **Radio de badges de veredicto**: `9999px` (pill)

---

### Sombras

| Elemento | Sombra |
|----------|--------|
| Cards en reposo | `0 1px 3px rgba(19,14,30,0.06)` |
| Cards en streaming | `0 4px 12px rgba(19,14,30,0.08)` |
| Input en focus | ring de `--board-primary` al 25% de opacidad |
| Modal | `0 20px 60px rgba(19,14,30,0.12)` |

---

## Layout

### Pantalla inicial (estado idle)

Centrado vertical y horizontal. Inspirado en Dia + Consensus.

```
┌─────────────────────────────────────┐
│                                     │
│         [Board Name]                │  ← Eiko, acento --board-primary
│                                     │
│   ┌─────────────────────────────┐   │
│   │   Ask the board...          │   │  ← input pill, Inter
│   │                         [→] │   │  ← CTA oscuro pill
│   └─────────────────────────────┘   │
│   [Example 1] [Example 2] [Ex 3]   │  ← pills con --board-primary
│                                     │
│   ○  ○  ○  ○  ○  ○                 │  ← expertos idle: avatar + nombre + seat
│                                     │
└─────────────────────────────────────┘
```

---

### Pantalla de resultados (estado done)

```
┌─────────────────────────────────────┐
│  [Board Name]            [input]    │  ← header compacto
├─────────────────────────────────────┤
│  Consensus Bar                      │  ← barra semáforo horizontal
├──────────────────┬──────────────────┤
│                  │  [Card] [Card]   │
│  Synthesis       │  [Card] [Card]   │  ← grid 2–3 cols según nº agentes
│  Panel           │  [Card] [Card]   │
└──────────────────┴──────────────────┘
```

---

## Componentes

### QuestionInput

- Textarea pill, fondo `#FDFDFD`, borde `#CBD6DC`
- Focus: ring de `--board-primary` al 25%
- Botón envío: pill oscuro `#130E1E`, icono flecha, dentro del input
- Desactivado mientras hay consulta en curso

### ExamplePills

Pills con las preguntas de ejemplo del `board.yaml`.
- Fondo: `--board-primary` al 8%
- Texto: `--board-primary`
- Borde: `--board-primary` al 20%
- Al hacer clic: populan el input

### ExpertCard

**Idle:**
- Avatar circular (foto o iniciales, fondo `--board-primary`)
- Nombre Inter `500`, credenciales Inter `400` en `#778186`
- Seat label en `#CBD6DC`

**Streaming:**
- Borde izquierdo `2px` en `--board-primary`
- Pulsing dot `#4EF0B6` (Spa Mint) en esquina superior derecha
- Campos con `fade-in` suave al llegar del stream

**Done:**
- Badge de veredicto pill (colores semáforo)
- `PRIMARY_CONCERN` en una línea, Inter `400`
- `PATH_FORWARD` en una línea, Inter `400`
- Botón "View full response" — texto link
- Botón "Chat" — icono, esquina inferior derecha
- Si el veredicto cambió tras reconsideración: badge anterior tachado en `#778186`

### ConsensusBar

Barra horizontal de `8px` de altura, segmentos proporcionales a los veredictos, colores semáforo. Aparece cuando el primer agente termina y se actualiza en tiempo real.

```
[████ Endorse ████][██ Conditions ██][░ Defer ░][░ No ░]
  2 Endorse          2 Conditions      1 Defer    1 No
```

- Leyenda debajo: dot de color + etiqueta + número
- Radio `9999px`
- Transición de ancho suave `300ms`

### SynthesisPanel

- Resumen ejecutivo: máx 4 líneas, `#130E1E`, Inter `400`
- Botón "View full analysis": texto link con chevron
- Expandido: Decision Brief en markdown
- Transición de altura `300ms ease`

### ExpertChatModal

- Modal centrado, fondo `#FDFDFD`, sombra de modal
- Header: avatar + nombre (Eiko) + seat (Inter)
- Burbujas usuario: fondo `--board-primary` al 10%, alineadas a la derecha
- Burbujas agente: fondo `#F8FAFC`, alineadas a la izquierda
- Input igual que QuestionInput pero compacto

---

## Fondo de Página

```css
background: linear-gradient(160deg, #FDFDFD 0%, #F0F4FF 100%);
```

Blanco Propelland hacia un tinte muy sutil de azul. No cambia con el board primary.

---

## Animaciones

| Elemento | Animación |
|----------|-----------|
| Agente pensando | `pulse` suave en avatar, dot `#4EF0B6` |
| Campos llegando del stream | `fade-in` `200ms` |
| Consensus Bar actualizando | transición de ancho `300ms ease` |
| Veredicto cambiando | nuevo badge `fade-in`, anterior `opacity-40 line-through` |
| SynthesisPanel expandiendo | `height` animado `300ms ease` |
