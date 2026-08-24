# raytop

Monitor de procesos **de pantalla completa** para el terminal (htop en pequeño), escrito en [raylang](https://github.com/roberto-ayala/raylang). Es la app que sube el escalón que raycode no subía: poseer el terminal ENTERO — pantalla alternativa, cursor oculto, redibujado diferencial línea a línea, resize en vivo — sobre `std/term` + `io.read_timeout` (el patrón tecla-o-plazo probado en raycode), con muestreo vía `std/process` (`ps`, sin `/proc`: igual en macOS y Linux).

```text
$ raytop                      # TUI a 1 s de refresco
$ raytop --interval 250       # más nervioso
$ raytop --once 8 --sort mem  # volcado plano sin TUI y sale

 raytop  load 2.34 2.36 2.09  procs 748  cpu 102.3%  rss 20.4G
    PID▾  CPU%   MEM%     RSS ST  USER       COMMAND
    606   30.1    0.6  204.4M Ss  _windowserver /System/…/WindowServer
  37507    6.0    2.3  835.6M S+  roberto    claude
 q quit  c/m/p sort  / filter  ↑↓ move  r refresh   748/748
```

Teclas: `q` salir · `c`/`m`/`p` ordenar por CPU/MEM/PID (marcador ▾ en la
columna) · `/` filtro incremental (comando, usuario o PID; Enter fija, Esc
borra) · flechas/PgUp/PgDn/Home/End mover selección · `r` refrescar ya.

## Cómo está hecho (el estruje de terminal)

- **Un frame = `rows` strings, cada una rellenada a `cols` celdas**: repintar
  una línea que cambió es un movimiento absoluto de cursor + sobrescritura —
  sin clear-to-EOL, sin parpadeo. El diff compara los strings ya estilizados,
  así que mover la selección repinta exactamente 2 líneas (testeado).
- **Tecla-o-plazo**: `io.read_timeout(64, hasta_el_próximo_refresco)` en un
  solo bucle; los bytes se decodifican con `term.decode` (el decodificador
  puro), un prefijo incompleto espera 25 ms y un ESC suelto es la tecla Esc
  (la disciplina de raycode).
- **Resize por sondeo** de `term.size()` en cada vuelta → repintado completo.
  Aguanta bien a 1 s; no hizo falta SIGWINCH (aunque `signals()` ya lo
  entrega como 28 — queda como mejora).
- **Ancho de celdas propio** (`src/width.ray`): wcwidth mínimo (CJK/kana/
  Hangul/fullwidth/emoji = 2 celdas) para que las columnas no se rompan con
  un comando en japonés — `std/term` no trae esta API (hallazgo).
- Muestreo: `ps -axo pid=,pcpu=,pmem=,rss=,state=,user=,comm=` + `uptime`
  parseados a mano (el último campo conserva sus espacios).

Verificado bajo un pty real (VM y nativo): entra y sale limpio de la pantalla
alternativa, cambia de orden, filtra y solo repinta lo que cambia (187
escrituras posicionadas en una sesión de 5 s con interacción, contra ~400+ de
un repintado ingenuo).

## Rendimiento

749 procesos, 120×40: construir el frame (filtro+sort+layout) cuesta
**0.29 ms en nativo** / 5.2 ms en la VM (18×); el muestreo lo domina `ps`
(~60–70 ms). Cadencias de 100 ms van sobradas en ambos motores.

## Estado actual

| Capacidad | Estado |
|-----------|--------|
| Pantalla alternativa + cursor oculto + restauración garantizada (`term.raw`) | ✅ |
| Redibujado diferencial por línea (selección = 2 líneas repintadas) | ✅ |
| Orden CPU/MEM/PID + filtro incremental + scroll con selección | ✅ |
| Resize en vivo (sondeo de `term.size()`) | ✅ |
| Ancho Unicode de celdas (CJK/emoji) propio | ✅ |
| `--once` (volcado plano, componible en scripts) | ✅ |
| Binario nativo (TUI verificado bajo pty en ambos motores) | ✅ |
| Tests (parser ps/uptime, width, modelo, render/diff puros) | ✅ 13 |
| Barras de CPU/MEM por núcleo, árbol de procesos, kill | 📋 v2 |
| Colores por estado/umbral | 📋 v2 |

## Hallazgos de dogfood (necesidades confirmadas del lenguaje)

Anotados en `raylang/IDEAS.md` §67:

1. **No hay ancho de celdas Unicode en `std/term`** — la predicción del
   catálogo: cualquier TUI de pantalla completa lo necesita; el wcwidth mínimo
   de `src/width.ray` es el candidato natural a `term.width(s)`.
2. **Los literales string no tienen `\x`/`\u`** — todo escape ANSI se
   construye con `char_from_code(27)` (raycode ya lo sufría; segunda app que
   repite el mismo helper → candidato a literal o a `term.style`).
3. **No hay literales hexadecimales** (`0x1F300` no lexea): las tablas de
   rangos Unicode quedan en decimal, ilegibles frente a la spec.
4. **Positivo**: el patrón tecla-o-plazo escala del editor de línea al TUI
   completo sin cambios; `term.raw` restaura SIEMPRE (incluso saliendo por
   `q` con el alt-screen activo); el sondeo de `term.size()` basta a 1 s; y
   el camino `ps` → parse → sort → frame de 749 procesos cabe en 0.3 ms
   nativo — el terminal jamás es el cuello.

## Desarrollo

```sh
ray test                      # 13 tests
ray run src/main.ray --once
ray build --native src/main.ray -o raytop --release
ray run debug/bench.ray       # coste de muestreo y de frame
```

Estructura: `src/main.ray` (CLI) · `sample.ray` (ps/uptime → Snapshot) ·
`model.ray` (orden/filtro/scroll) · `width.ray` (celdas Unicode) ·
`render.ray` (frame puro + diff) · `app.ray` (bucle raw + alt-screen).
