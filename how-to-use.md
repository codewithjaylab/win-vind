# win-vind — Guía completa (how-to-use)

> Versión documentada: **win-vind 5.13.2** (MIT License, por [pit-ray](https://github.com/pit-ray/win-vind)).
> Este documento explica, en orden: **(1) cómo funciona el software**, **(2) cómo configurarlo** y **(3) cómo usarlo**.

---

## Índice

Guía:
1. [Cómo funciona](#1-cómo-funciona)
2. [Cómo configurarlo](#2-cómo-configurarlo)
3. [Cómo usarlo](#3-cómo-usarlo)
4. [Referencia rápida](#4-referencia-rápida)

Referencia completa (autosuficiente, sin volver al repo):
- [Apéndice A — Funciones internas](#apéndice-a--referencia-completa-de-funciones-internas)
- [Apéndice B — Opciones y parámetros](#apéndice-b--referencia-completa-de-opciones-y-parámetros)
- [Apéndice C — Keywords (teclas y prefijos)](#apéndice-c--keywords-todas-las-teclas-y-prefijos-de-modo)
- [Apéndice D — Mapeos por defecto (5 tiers)](#apéndice-d--mapeos-por-defecto-de-todos-los-tiers)
- [Apéndice E — Migración v4 → v5](#apéndice-e--guía-de-migración-v4x--v5x)
- [Apéndice F — Compilar, testear, desarrollar](#apéndice-f--compilar-testear-y-desarrollar)
- [Apéndice G — Dependencias y estructura](#apéndice-g--dependencias-librerías-y-estructura-del-proyecto)

---

# 1. Cómo funciona

## 1.1 ¿Qué es win-vind?

**win-vind** significa **<u>vi</u>m key b<u>ind</u>er for <u>win</u>dows**. Es un sistema híbrido ligero de interfaz CUI + GUI para Windows: una vez instalado, puedes **controlar la GUI de Windows igual que controlas Vim**.

Características clave:

- **Amigable para usuarios de Vim.** Todos los métodos de configuración y el concepto de *modos* derivan de Vim. Un usuario de Vim solo necesita aprender las *macros* y los modos adicionales de win-vind.
- **Muchos comandos internos útiles.** No dependes de scripts externos ni de dependencias (a diferencia de Autohotkey u otras herramientas de key-binding). Puedes crear comandos personalizados combinando comandos internos de bajo nivel ya optimizados.
- **Muy portable y open source.** Es **un único binario pequeño sin dependencias** que corre con **permisos de usuario** (no requiere admin por defecto). También funciona desde la línea de comandos:

  ```sh
  $ win-vind -c "ggyyGp"
  ```

## 1.2 Arquitectura: servidor + clientes

La primera vez que ejecutas win-vind, el programa **queda residente en la bandeja del sistema** (system tray) y se ejecuta como **servidor**. Ese servidor:

- Revisa la **memoria compartida** cada cierto intervalo (opción `listen_interval`).
- Ejecuta los comandos solicitados, además de atender consultas.

**Cualquier instancia nueva de win-vind que se lance mientras ya hay una corriendo se convierte en cliente.** Así puedes pedirle acciones al servidor ya residente desde una terminal:

```sh
# Con win-vind ya en ejecución, esto lanza el cambiar de ventana
$ win-vind -c "<switch_window>"
```

Esto permite invocar funciones concretas (por ejemplo *EasyClick*) desde otras herramientas (AutoHotKey, scripts, etc.).

## 1.3 El concepto de modos

Igual que en Vim hay **normal mode** e **insert mode**. Pero como la GUI tiene **dos tipos de objetivo** (el cursor del ratón y el caret del texto), win-vind tiene **dos modos normal y dos modos visual**, agrupados en dos familias:

- **Grupo GUI**: familia de funciones del cursor del ratón; maneja tanto el cursor como las GUIs manipulables por él (ventanas).
- **Grupo Editor**: familia de funciones especializadas en edición de texto; usa el *caret* para emular Vim y lograr "Vim en todas partes".

Además hay dos modos transversales:

- **Resident mode**: es un modo de "evacuación" que **apaga win-vind** para evitar colisiones con otros programas que usan atajos (Vim, juegos de Steam, etc.). Solo deja los mapeos mínimos necesarios.
- **Command mode**: ejecuta comandos con una **línea de comandos virtual** superpuesta en pantalla, al estilo Vim.

### Tabla de modos

| Modo                   | Bloquea teclas | Rol                                                                                              |
| :--------------------: | :------------: | :----------------------------------------------------------------------------------------------- |
| **GUI Normal**         | ✓              | Controla ventanas y cursor del ratón. Bindings por defecto: `<esc-left>` o `<ctrl-]>`.           |
| **GUI Visual**         | ✓              | Selecciona objetos de la GUI (iconos) manteniendo el botón izquierdo. Se entra con `v` desde GUI Normal. |
| **Edi Normal**         | ✓              | Emula Vim en formularios web, MS Word, etc. Bindings por defecto: `<esc-right>` o `<ctrl-[>`.     |
| **Edi Visual**         | ✓              | Selecciona texto como en Vim. Se entra con `v` desde Edi Normal.                                 |
| **Insert**             | ✘              | Escribe texto como Windows normal, **sin** absorber teclas. Se entra con `i`.                    |
| **Resident**           | ✘              | Modo de evacuación anti-colisiones. Binding por defecto: `<esc-down>`.                           |
| **Command**            | ✓              | Ejecuta comandos con una línea de comandos virtual tipo Vim. Se entra con `:`.                   |

## 1.4 Absorción de teclas (key message blocking)

Este es el mecanismo central: **todos los modos, excepto `insert` y `resident`, bloquean los mensajes de teclado y los enrutan dentro de win-vind.**

Gracias a esa absorción puedes definir mapeos libremente **sin preocuparte por conflictos** con los atajos de Windows o de otras aplicaciones. Esto es lo que diferencia a win-vind de las herramientas de key-binding convencionales.

> Consecuencia: si quieres que una tecla llegue *fuera* de win-vind, debes usar **macros externas** (ver [2.5](#25-macros-externas)).

## 1.5 El sistema de "tiers" (version)

win-vind trae **cinco niveles** de funcionalidad preconfigurada, derivados de Vim. Poniendo la directiva `version` al inicio de tu `.vindrc` eliges qué conjunto de comandos se carga. **Si no pones `version`, se carga `huge`.**

| Tier     | Funciones soportadas                                                       |
| :------- | :------------------------------------------------------------------------- |
| `tiny`   | +mouse +syscmd                                                             |
| `small`  | +mouse +syscmd +window +process                                            |
| `normal` | +mouse +syscmd +window +process +vimemu                                    |
| `big`    | +mouse +syscmd +window +process +vimemu +hotkey +gvmode                    |
| `huge`   | +mouse +syscmd +window +process +vimemu +hotkey +gvmode +experimental      |

- `tiny`: comandos mínimos para mover el ratón y hacer clic desde el teclado (GridMove, EasyClick).
- `small`: añade control flexible de ventanas y lanzamiento de procesos.
- `normal`: añade mapeos de emulación de Vim y permite editar texto en áreas de texto.
- `big`: añade hotkeys que redefinen atajos de Windows, y **GUI Visual Mode** (+gvmode).
- `huge`: features experimentales para operaciones más complejas.

## 1.6 El archivo de configuración `.vindrc`

La configuración es **estilo `.vimrc`**: se llama `.vindrc` y por defecto vive en:

```
C:\Users\<USERNAME>\.win-vind\.vindrc
```

En él puedes: cambiar opciones, ajustar parámetros, remapear teclas de bajo nivel y definir bindings de funciones.

## 1.7 Limitaciones conocidas

- **Solo Windows 10/11 en máquina real.** Puede no funcionar en Windows antiguos ni en entornos virtuales (Wine, VirtualBox).
- win-vind **NO puede operar ventanas con mayor nivel de privilegio** que él mismo. Por ejemplo, si abres *Task Manager* con admin, no podrás mover/hacer clic/hacer scroll en su ventana. Si necesitas operar todo, dale a win-vind permisos de administrador vía **Task Scheduler**.
- **EasyClick** puede fallar en algunas apps en Windows 10 anteriores a 1803 (confirmado OK después de 1909).
- **Windows 10/11 Single Language** puede no poder mapear teclas toggle como `<Capslock>`.
- Para usar movimientos de palabra (`w`, `B`, `e`) en **MS Word**, se recomienda desactivar *"Use smart paragraph selection"*.

---

# 2. Cómo configurarlo

## 2.1 Instalación

Elige un método. **La ruta de instalación y la ruta del archivo de configuración varían según el método.**

### Chocolatey
Instala en el directorio de Chocolatey y **genera config y logs en `C:/Users/*/.win-vind`**.
```sh
$ choco install win-vind
```

### winget
Instala en `C:/Program Files/win-vind` y genera config y logs en `C:/Users/*/.win-vind`.
```sh
$ winget install win-vind
```

### Scoop
Se instala en el directorio de Scoop (añadido a Scoop Extras, con auto-actualización).
```sh
$ scoop bucket add extras
$ scoop install win-vind
```

### Instalador ejecutable
Instala en `C:/Program Files/win-vind` y genera config y logs en `C:/Users/*/.win-vind`.
- `win-vind_5.13.2_32bit_installer.zip`
- `win-vind_5.13.2_64bit_installer.zip`

### Portable (zip)
**No genera archivos externos** salvo el de arranque: la config y los logs se generan **dentro del directorio descomprimido**.
- `win-vind_5.13.2_32bit_portable.zip`
- `win-vind_5.13.2_64bit_portable.zip`

Descargas en: <https://github.com/pit-ray/win-vind/releases>

## 2.2 Dónde vive el `.vindrc`

Según el método de instalación:

| Método                | Configuración / logs                       |
| :-------------------- | :----------------------------------------- |
| Chocolatey            | `C:/Users/<USER>/.win-vind/`               |
| winget                | `C:/Users/<USER>/.win-vind/`               |
| Instalador (.exe)     | `C:/Users/<USER>/.win-vind/`               |
| Portable (zip)        | dentro del directorio descomprimido        |

Ruta por defecto del archivo: **`C:\Users\<USERNAME>\.win-vind\.vindrc`**

## 2.3 Estructura básica del `.vindrc`

```vim
" Elige la versión de {tiny, small, normal, big, huge}.
version normal

" Cambiar parámetros
set shell = cmd
set cmd_fontsize = 14
set cmd_fontname = Consolas
set easyclick_bgcolor=E67E22
set easyclick_fontcolor=34495E

" Mapear capslock a ctrl.
imap <capslock> {<ctrl>}

" Definir atajos útiles
inoremap <ctrl-shift-f> <easyclick><click_left>
inoremap <ctrl-shift-m> <gridmove><click_left>
inoremap <ctrl-shift-s> <switch_window><easyclick><click_left>

" Registrar lanzadores de aplicaciones
noremap <ctrl-1> :! gvim<cr>
noremap <ctrl-2> :e http://example.com<cr>

" Definir macros como en Vim
enoremap t ggyyGp

" Auto-comandos
autocmd AppLeave * <to_insert>
autocmd AppEnter,EdiNormalEnter vim.exe <to_resident>
```

> ⚠️ **Importante:** antes del comando `version` **solo pueden ir comentarios**. Nada más.

```vim
" Solo comentarios aquí.
version tiny
" A partir de aquí cualquier comando.
set shell = cmd
```

## 2.4 Opciones (`set`)

Cambia opciones y parámetros con el comando `set`. La lista completa está en el *cheat sheet* de opciones.

Sintaxis:

| Sintaxis                        | Efecto                                                                                                              |
| :------------------------------ | :------------------------------------------------------------------------------------------------------------------ |
| `set ${OPTION}`                 | Pone la opción a **true**.                                                                                          |
| `set no${OPTION}`               | Pone la opción a **false**.                                                                                         |
| `set ${OPTION} = ${VALUE}`      | Asigna un valor (string o número con float). El string no necesita comillas; los espacios alrededor del `=` se ignoran. |

También puedes probar opciones en vivo desde **Command mode**, sin guardar (se resetea al reiniciar):

```vim
:set cursor_accel=32
```

Ejemplo — ajustar EasyClick:

```vim
set easyclick_bgcolor=E67E22
set easyclick_fontcolor=34495E
set easyclick_fontname=Consolas
set easyclick_fontsize=16
set easyclick_fontweight=600
```

### Opciones más útiles por categoría

| Categoría          | Opciones                                                                              |
| :----------------- | :------------------------------------------------------------------------------------ |
| Sistema            | `initmode`, `listen_interval`, `icon_style`, `tempdir`, `gui_fontname`, `gui_fontsize`, `hintkeys` |
| Línea de comandos  | `vcmdline`, `showcmd`, `cmd_bgcolor`, `cmd_fontcolor`, `cmd_fontname`, `cmd_fontsize`, `cmd_fontweight`, `cmd_fontextra`, `cmd_roughpos`, `cmd_xmargin`, `cmd_ymargin`, `cmd_fadeout`, `cmd_monitor` |
| EasyClick          | `easyclick_bgcolor`, `easyclick_fontcolor`, `easyclick_fontname`, `easyclick_fontsize`, `easyclick_fontweight` |
| GridMove           | `gridmove_bgcolor`, `gridmove_fontcolor`, `gridmove_fontname`, `gridmove_fontsize`, `gridmove_fontweight`, `gridmove_size` |
| Ratón              | `cursor_accel`, `cursor_resolution`, `jump_margin`, `hscroll_pageratio`, `hscroll_speed`, `vscroll_pageratio`, `vscroll_speed`, `keybrd_layout` |
| Ventana            | `arrangewin_ignore`, `window_velocity`, `window_hdelta`, `window_vdelta`, `winresizer_initmode` |
| Caret bloque        | `blockstylecaret`, `blockstylecaret_mode`, `blockstylecaret_width`                    |
| AutoFocus          | `autotrack_popup`                                                                      |
| Caché UIA          | `uiacachebuild`, `uiacachebuild_lifetime`, `uiacachebuild_staybegin`, `uiacachebuild_stayend` |
| Shell              | `shell`, `shell_startupdir`, `shellcmdflag`                                            |
| Emulación Vim      | `charbreak`, `charcache`                                                               |

Ejemplo — cambiar el modo inicial a GUI Normal:

```vim
set initmode = gn
```

## 2.5 Mapeos y macros

win-vind usa configuración **estilo Run Commands**. Si has escrito un `.vimrc`, te resultará natural.

### ¿Qué es un mapeo?
Un binding normal asigna una función a una tecla. El **mapeo** de Vim es una extensión: permite **asignación recursiva** y definición de comandos.

Comandos de mapeo:

| Sintaxis                              | Función                                                              |
| :------------------------------------ | :------------------------------------------------------------------- |
| `${MODE}map ${IN} ${OUT}`             | Mapeo **recursivo**: al pulsar `${IN}` se genera `${OUT}`.           |
| `${MODE}noremap ${IN} ${OUT}`         | Mapeo **no recursivo**: `${OUT}` se interpreta como binding por defecto. |
| `${MODE}unmap ${IN}`                  | Elimina el mapeo de `${IN}`.                                         |
| `${MODE}mapclear`                     | Borra todos los mapeos.                                              |
| `command ${IN} ${OUT}`                | Define un comando que llama a `${OUT}`.                              |
| `delcommand ${IN}`                    | Elimina el comando `${IN}`.                                          |
| `comclear`                            | Borra todos los comandos.                                            |

Los comandos más comunes: `map` (todos los modos), `nmap`/`noremap` (normal), `imap`/`inoremap` (insert).

### Prefijos de modo (`${MODE}`)

| Prefijo | Modo                                              |
| :-----: | :------------------------------------------------ |
| (vacío) | GUI Normal, GUI Visual, Edi Normal, Edi Visual    |
| `g`     | GUI Normal, GUI Visual                            |
| `e`     | Edi Normal, Edi Visual                            |
| `n`     | GUI Normal, Edi Normal                            |
| `v`     | GUI Visual, Edi Visual                            |
| `gn`    | GUI Normal                                        |
| `gv`    | GUI Visual                                        |
| `en`    | Edi Normal                                        |
| `ev`    | Edi Visual                                        |
| `i`     | Insert Mode                                       |
| `r`     | Resident Mode                                     |
| `c`     | Command Mode                                      |

### Sintaxis de teclas

Se usa la misma expresión que en Vim: teclas unidas por `-` entre `<` y `>`. **No hay límite de combinaciones** (p. ej. `<Esc-b-c-a-d>`).

Keywords de sistema comunes: `<cr>`/`<enter>` (Enter), `<esc>`, `<tab>`, `<bs>`, `<space>`, `<left>` `<right>` `<up>` `<down>`, modificadores `<ctrl>`/`<c>`, `<shift>`/`<s>`, `<alt>`/`<a>`, `<win>`, `<capslock>`, `<f1>`…`<f24>`, `<del>`, `<home>`, `<end>`, `<insert>`, `<pageup>`, `<pagedown>`, etc. (todos **case-insensitive**). Ver Keywords del cheat sheet.

### Mapeo recursivo vs. no recursivo

```vim
" Mapping A
nmap b h
nmap o b
nmap p o
```
Se resuelve recursivamente: `b -> h`, `o -> b -> h`, `p -> o -> b -> h` → win-vind **genera automáticamente** un mapeo optimizado equivalente a `nmap o h` / `nmap p h`. Así combinas comandos simples en comandos complejos de forma eficiente.

En cambio `noremap` **no** resuelve recursivamente: el destino se interpreta siempre como el binding por defecto. Mapear a un binding inexistente no hace nada:

```vim
" m no existe por defecto, así que no pasa nada
inoremap g m
```

### Macros externas

Los modos que absorben teclas no las propagan a otras apps. Para eso existe la **macro externa**: se encierra entre `{` y `}`, y emula que el usuario pulsó la tecla realmente. El mapeo tecla-a-tecla (`map a {b}`) es el mapeo de bajo nivel más eficiente.

### Ejemplos de uso

```vim
" 1. Definir cambio de modo
imap <win-]> <to_gui_normal>

" 2. Macro de entrada de texto
nmap mail {win-vind@example.com}

" 3. Lanzador de web
nmap <ctrl-1> :e https://example.com<cr>

" 4. Lanzador de aplicación
nmap <ctrl-2> :! notepad<cr>

" 5. Copiar la línea actual al final (como Vim)
enmap t yyGp
```

## 2.6 Comandos automáticos (`autocmd`)

Ejecuta comandos automáticamente ante eventos y nombres de archivo específicos.

```vim
" Mapeo por defecto para un evento concreto (cualquier aplicación)
autocmd AppLeave * <to_insert>

" Al seleccionar notepad, cambia automáticamente a Editor normal
" (equivale al antiguo dedicate_to_window)
autocmd AppEnter */microsoft*/notepad.exe <to_edi_normal>

" Suprime win-vind en procesos llamados Vim
" (equivale al antiguo suppress_for_vim)
autocmd AppEnter,EdiNormalEnter vim.exe <to_resident>
```

Ver funciones `<autocmd_add>` y `<autocmd_del>` para más detalle.

## 2.7 Cargar `.vindrc` remoto

Inspirado en gestores de plugins de Vim, el comando `source` puede cargar tanto `.vindrc` locales como **`.vindrc` de un repositorio de GitHub** en la forma `user/repo` (desde la raíz del repo):

```vim
" Carga el .vindrc del repo pit-ray/remote_vindrc_demo
source pit-ray/remote_vindrc_demo
```

> ⚠️ `source user/repo` **no verifica la seguridad** del `.vindrc` que descarga: puede ser un agujero de seguridad. Úsalo solo con repositorios de confianza o tus propios dotfiles. Como medida mínima, win-vind solo lee el contenido la **primera vez** que ejecutas `source` (no se actualiza como haría un `git pull`).

## 2.8 Recargar la configuración

Tras editar `.vindrc`, recarga sin reiniciar:

```vim
:source
```
(No necesita argumentos para recargar el archivo actual.)

---

# 3. Cómo usarlo

## 3.1 Iniciar y comprobar que funciona

1. Ejecuta `win-vind.exe` desde la línea de comandos o haz clic en el icono de la app.
2. Si ves el **icono en la bandeja del sistema** (system tray), está funcionando correctamente.

## 3.2 Terminar win-vind

- `:exit` — **forma recomendada** de terminar.
- `<F8> + <F9>` — terminación forzada segura.

## 3.3 Mapa de teclado por defecto (tier `normal`)

### GUI Normal Mode → Editor Normal

| Pulsación                            | Acción                                    |
| :----------------------------------- | :---------------------------------------- |
| `I`, `<esc-right>`, `<ctrl-[>`       | `<click_left>` + `<to_edi_normal>`        |

### Editor Normal Mode

**Transición de modo**

| Pulsación                       | Acción                |
| :------------------------------ | :-------------------- |
| `<Esc-Left>`, `<ctrl-]>`        | `<to_gui_normal>`     |
| `<Esc-Down>`                    | `<to_resident>`       |
| `:`                             | `<to_command>`        |
| `i`                             | `<to_insert>`         |
| `v` / `V`                       | `<to_edi_visual>` / `<to_edi_visual_line>` |

**Scroll**

| Pulsación            | Acción                        |
| :------------------- | :---------------------------- |
| `<C-y>`, `<C-k>`     | `<scroll_up>`                 |
| `<C-j>`, `<C-e>`     | `<scroll_down>`               |
| `<C-u>` / `<C-d>`    | media página arriba / abajo   |
| `<C-b>` / `<C-f>`    | página completa arriba / abajo|
| `zh`, `<C-h>`        | `<scroll_left>`               |
| `zl`, `<C-l>`        | `<scroll_right>`              |
| `zH` / `zL`          | media página izq / der        |

**Atajos**

| Pulsación | Acción                  |
| :-------- | :---------------------- |
| `<C-r>`   | `<redo>`                |
| `u`, `U`  | `<undo>`                |
| `gT`/`gt` | pestaña anterior / siguiente |
| `/`, `?`  | `<search_pattern>`      |

**Movimiento del caret (emulación Vim)**

| Pulsación                                  | Acción                 |
| :----------------------------------------- | :--------------------- |
| `h`, `<Left>`, `<BS>`, `<C-h>`             | izquierda              |
| `l`, `<Space>`, `<Right>`                  | derecha                |
| `k`, `<Up>`, `<C-p>`, `-`, `gk`            | arriba                 |
| `j`, `<Down>`, `<C-n>`, `+`, `<Enter>`, `gj` | abajo                |
| `w` / `b`                                  | palabra adelante / atrás |
| `W` / `B`                                  | BIGWORD adelante / atrás |
| `e` / `E`                                  | fin de palabra / BIGWORD |
| `ge` / `gE`                                | fin de palabra atrás   |

**Saltos del caret**

| Pulsación              | Acción             |
| :--------------------- | :----------------- |
| `0`, `g0`, `<Home>`    | inicio de línea    |
| `$`, `g$`, `<End>`     | fin de línea       |
| `gg`                   | inicio del archivo |
| `G`                    | fin del archivo    |

**Edición (emulación Vim)**

| Pulsación              | Acción                          |
| :--------------------- | :------------------------------ |
| `yy`, `Y`              | `<yank_line>`                   |
| `y`                    | `<yank_with_motion>`            |
| `p` / `P`              | `<put_after>` / `<put_before>`  |
| `dd`                   | borrar línea                    |
| `D`                    | borrar hasta fin de línea       |
| `x`, `<Del>` / `X`     | borrar después / antes          |
| `J`                    | unir con línea siguiente        |
| `r` / `R`              | reemplazar carácter / secuencia |
| `~`                    | cambiar mayúsc/minúsc           |
| `d` / `c`              | delete/change con motion        |
| `S`, `cc`              | change line                     |
| `s` / `C`              | change char / hasta fin de línea|
| `.`                    | repetir último cambio           |

### Insert Mode

| Pulsación                  | Acción            |
| :------------------------- | :---------------- |
| `<ctrl-[>`, `<Esc-Right>`  | `<to_edi_normal>` |

### Resident Mode

| Pulsación       | Acción            |
| :-------------- | :---------------- |
| `<Esc-Right>`   | `<to_edi_normal>` |

### Command Mode

| Pulsación                 | Acción                     |
| :------------------------ | :------------------------- |
| `edinormal`, `en`         | `<to_edi_normal>`          |
| `ev`, `edivisual`         | `<to_edi_visual>`          |
| `evl`, `edivisualline`    | `<to_edi_visual_line>`     |
| `w`                       | `<save>`                   |

> Los mapas completos de cada tier están en el *cheat sheet*: `tiny`, `small`, `normal`, `big`, `huge`.

## 3.4 Funciones internas destacadas

Organizadas por categoría (se usan entre `< >`, encadenables en una macro):

- **Modo**: `<to_command>`, `<to_gui_normal>`, `<to_gui_visual>`, `<to_edi_normal>`, `<to_edi_visual>`, `<to_edi_visual_line>`, `<to_insert>`, `<to_resident>`, `<to_instant_gui_normal>`.
- **Sistema**: `<set>`, `<source>`, `<map>`, `<noremap>`, `<unmap>`, `<mapclear>`, `<command>`, `<delcommand>`, `<comclear>`, `<autocmd_add>`, `<autocmd_del>`.
- **Ratón**: `<click_left>` `<click_right>` `<click_mid>`, `<move_cursor_*>`, `<easyclick>` / `<easyclick_all>`, `<gridmove>`, `<focus_textarea>`, `<jump_cursor_to_*>`, `<scroll_*>`.
- **Ventana**: `<switch_window>`, `<select_{left,right,upper,lower}_window>`, `<move_window_*>`, `<maximize_/minimize_current_window>`, `<resize_window_*>`, `<arrange_windows>`, `<exchange_window_with_nearest>`, `<rotate_windows>`, `<snap_current_window_to_*>`, `<open_new_window>`, `<window_resizer>`.
- **Hotkey**: `<open>`, `<open_startmenu>`, `<hotkey_copy|cut|paste|delete|backspace>`, `<undo>`, `<redo>`, `<save>`, `<select_all>`, `<search_pattern>`, `<start_explorer>`, `<goto_next_page>`, `<goto_prev_page>`, `<forward_ui_navigation>`, `<backward_ui_navigation>`, `<decide_focused_ui_object>`.
- **Escritorio virtual**: `<taskview>`, `<create_new_vdesktop>`, `<close_current_vdesktop>`, `<switch_to_left_vdesktop>`, `<switch_to_right_vdesktop>`.
- **Pestañas**: `<open_new_tab>`, `<close_current_tab>`, `<switch_to_left_tab>`, `<switch_to_right_tab>`.
- **Archivo**: `<makedir>`.
- **Edición de texto**: `<move_*word*>`, `<jump_caret_to_*>` (BOF/BOL/EOF/EOL), `<change_*>`, `<delete_*>`, `<yank_*>`, `<put_*>`, `<replace_*>`, `<join_next_line>`, `<repeat_last_change>`, `<switch_char_case>`.

## 3.5 Tutorial rápido 1 — GUI y ventanas

1. Cambia a **GUI Normal Mode** con `<ctrl-]>`.
2. Pulsa `:!mspaint` para lanzar Microsoft Paint.
3. Llama a **EasyClick** con `<shift-f><shift-f>`.
4. Vuelve a insert mode; selecciona ventanas con `<C-w>h` o `<C-w>l`.
5. Selecciona Microsoft Paint y ciérralo con `:close`.

## 3.6 Tutorial rápido 2 — personalizar

1. Ve a **GUI Normal** mode.
2. Abre tu `.vindrc` con `:e`.
3. Escribe:

   ```vim
   set cmd_fontname = Arial
   inoremap <ctrl-shift-f> <easyclick><click_left>
   inoremap <ctrl-shift-s> <switch_window><easyclick><click_left>
   noremap <ctrl-1> :!notepad<cr>
   ```
4. Recarga con `:source` (sin argumentos).
5. Estando en **GUI Normal**, pulsa **Ctrl + 1** para lanzar Notepad.
6. Pulsa **i** para volver a **Insert** mode.
7. Usa EasyClick con **Ctrl + Shift + f** y cambia de ventana con EasyClick con **Ctrl + Shift + s**.

## 3.7 Uso como comando de automatización (CLI)

Con win-vind ya en ejecución, desde una terminal:

```sh
$ win-vind -c "<switch_window>"
$ win-vind -c "ggyyGp"
```

Solo funciones concretas (p. ej. EasyClick) pueden invocarse así desde otras herramientas.

## 3.8 Truco: usar SOLO EasyClick

Para desactivar todo y usar únicamente EasyClick:

```vim
version tiny
set initmode=i  " Insert mode

" Sobrescribe los bindings por defecto a la tecla dummy <f20>
imap <esc-left> <f20>
imap <ctrl-]> <f20>
imap <f8> <f20>
imap <esc-down> <f20>

" Define tus propias teclas
imap <ctrl-shift-space> <easyclick><click_left>

" Escaneo asíncrono de objetos UI (opcional)
set uiacachebuild
set uiacachebuild_lifetime=5000
set uiacachebuild_staybegin=500
set uiacachebuild_stayend=2000
```

El truco: pones el modo inicial en `insert`, remapeas todos los bindings por defecto a una tecla dummy (`<f20>`) y luego defines EasyClick con las teclas que tú quieras.

---

# 4. Referencia rápida

| Concepto                | Valor / comando                                                              |
| :---------------------- | :--------------------------------------------------------------------------- |
| Archivo de config       | `C:\Users\<USERNAME>\.win-vind\.vindrc`                                       |
| Documentación oficial   | <https://pit-ray.github.io/win-vind/usage/>                                   |
| Repositorio             | <https://github.com/pit-ray/win-vind>                                         |
| Foro de problemas       | <https://github.com/pit-ray/win-vind/issues>                                  |
| Versión estable         | 5.13.2                                                                        |
| Licencia                | MIT                                                                            |
| Terminar                 | `:exit` (recomendado) · `<F8>+<F9>` (forzado)                                 |
| Recargar config          | `:source`                                                                      |
| Editar config            | `:e`                                                                           |
| Modo inicial             | `set initmode = {gn,gv,en,ev,i,r,c}`                                          |
| Todos los keywords       | ver cheat-sheet → Keywords (no case-sensitive)                                |
| Categorías de funciones  | Modo · Sistema · Ratón · Ventana · Hotkey · Escritorio virtual · Pestañas · Archivo · Edición de texto |

> Para la lista exhaustiva de funciones, opciones y mapeos por defecto, consulta el *cheat sheet* oficial y la carpeta `docs/cheat_sheet/` del repositorio.


---


---

# Apéndices (referencia completa)

> Estos apéndices embeben toda la documentación oficial del repo (`docs/` y `CONTRIBUTING.md`) ya limpia de formato Jekyll, para que **no haga falta volver a analizar el repositorio**. El contenido de referencia está en inglés (fuente original); la prosa guía de arriba está en español.

## Apéndice A — Referencia completa de funciones internas

Todas las funciones invocables, agrupadas por categoría. Se usan entre `< >` y son encadenables dentro de una macro.

### Mode

#### **`<to_command>`**
Enter the command mode, which is generally called with `:`.
In the command mode, the typed characters are displayed on the virtual command line.
You can operate the virtual command line as shown in the following table.

|**Key**|**Operation**|
|:---:|:---:|
|`<enter>`|Execute the current command|
|`<bs>`|Delete characters|
|`<up>`|Backward history|
|`<down>`|Forward history|
|`<tab>`|Complete commands|

**Related Options**
- [vcmdline](#vcmdline)
- [cmd_bgcolor](#cmd_bgcolor)
- [cmd_fontcolor](#cmd_fontcolor)
- [cmd_fontname](#cmd_fontname)
- [cmd_fontsize](#cmd_fontsize)
- [cmd_fontweight](#cmd_fontweight)
- [cmd_fontextra](#cmd_fontextra)
- [cmd_roughpos](#cmd_roughpos)
- [cmd_xmargin](#cmd_xmargin)
- [cmd_ymargin](#cmd_ymargin)
- [cmd_fadeout](#cmd_fadeout)
- [cmd_monitor](#cmd_monitor)

**See Also**
- [\<command\>](#command)
- [\<delcommand\>](#delcommand)
- [\<comclear\>](#comclear)

---

#### **`<to_gui_normal>`**
Transition to GUI normal mode. In GUI normal mode, the typed keys are not transmitted to Windows, so you can create any mapping you like without considering shortcut key conflicts.

**See Also**
- [\<to_gui_visual\>](#to_gui_visual)
- [\<to_instant_gui_normal\>](#to_instant_gui_normal)
- [\<gnmap\>](#map)
- [\<gnnoremap\>](#noremap)
- [\<gnunmap\>](#unmap)
- [\<gnmapclear\>](#mapclear)

---

#### **`<to_gui_visual>`**
Enter GUI visual mode. In this mode, the mouse is always in the click state and input is blocked from Windows as in normal mode.

**See Also**
- [\<to_gui_normal\>](#to_gui_normal)
- [\<to_instant_gui_normal\>](#to_instant_gui_normal)
- [\<mgvap\>](#map)
- [\<gvnoremap\>](#noremap)
- [\<gvunmap\>](#unmap)
- [\<gvmapclear\>](#mapclear)

---

#### **`<to_edi_normal>`**
Switch to the editor normal mode. This mode is essentially the same as GUI normal mode, but defines a lot of text-specific mappings that emulate Vim editing in order to achieve Vim everywhere.

**See Also**
- [\<to_edi_visual\>](#to_edi_visual)
- [\<to_edi_visual_line\>](#to_edi_visual_line)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)

---

#### **`<to_edi_visual>`**
Switch to editor visual mode.
The editor visual mode corresponds to the "v" command in Vim, which allows you to make a character-based selection with the keyboard. The typed keys are not propagated to Windows. To select a line, call [\<to_edi_visual_line\>](#to_edi_visual_line) instead. In both ways, the only difference is the initialization, and the transition destination is the editor visual mode.

**See Also**
- [\<to_edi_normal\>](#to_edi_normal)
- [\<to_edi_visual_line\>](#to_edi_visual_line)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)

---

#### **`<to_edi_visual_line>`**
Switch to editor visual mode.
Similar to `<s-v>` of Vim, the selection method is line selection. If you want to do character selection, call [\<to_edi_visual\>](#to_edi_visual) instead.

**See Also**
- [\<to_edi_normal\>](#to_edi_normal)
- [\<to_edi_visual\>](#to_edi_visual)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)

---

#### **`<to_insert>`**
Enters the Insert mode. In this mode, you can directly input and edit text.

**See Also**
- [\<to_resident\>](#to_resident)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)

---

#### **`<to_resident>`**
Enters Resident mode.

**See Also**
- [\<to_insert\>](#to_insert)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)

---

#### **`<to_instant_gui_normal>`**
Temporarily switches to GUI Normal mode and performs matching, which can be used as a map-leader.

**See Also**
- [\<to_gui_normal\>](#to_gui_normal)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)

### System Command

#### **`<set>`**
Change the options and parameters specified using the `set` command in your .vindrc.

|**Syntax**|**Effects**|
|:---|:---|
|`set ${OPTION_NAME}`|Set the value of the option to **true**.|
|`set no${OPTION_NAME}`|Set the value of the option to **false**.|
|`set ${OPTION_NAME} = ${VALUE}`|Set a value of the option. The value can be a string or a number that allows floating points. The string does not need quotation marks, and any character after the non-white character will be handled as the value. White spaces at both ends of the equals sign are ignored.|

**See Also**
- Options
- [\<source\>](#source)
- [\<map\>](#map)
- [\<noremap\>](#noremap)

---

#### **`<source>`**
Load the .vindrc file.

Inspired by many Vim plugin managers such as [vim-plug](https://github.com/junegunn/vim-plug), win-vind has a simple remote .vindrc loading capability using the `source` command.

The `source` command is originally designed to load local .vindrc, but it can also load .vindrc in the form `user/repo` from the root directory of a repository on GitHub.

As a sample, by writing the following in your .vindrc., win-vind loads the .vindrc in [pit-ray/remote_vindrc_demo](https://github.com/pit-ray/remote_vindrc_demo) repository, and `:test_remote` command can be available.

```vim
" Load remote repository .vindrc
source pit-ray/remote_vindrc_demo
```

> **Warning**: `source user/repo` does not verify the safety of the .vindrc it reads, which may be a security hole. Therefore, use it for reliable repositories or your own dotfiles configurations. As a minimum security measure, win-vind only reads the contents of source the first time a `source` command is done, and does not update the contents as `git pull` does.

**See Also**
- [\<set\>](#set)
- [\<map\>](#map)
- [\<noremap\>](#noremap)

---

#### **`<map>`**
Recursively define a map that is invoked by `${IN_CMD}` and generates `${OUT_CMD}`.

```vim
${MODE}map ${IN_CMD} ${OUT_CMD}
```
`${MODE}` is the [Mode Prefix](#mode-prefix).

The `map` allows remapping with user-defined mapping like the following.
```vim
nmap f h  " f --> h
nmap t f  " t --> h
```
The `noremap` performs only the default map.
```vim
nnoremap f h  " f --> h
nnoremap t f  " t --> f
```

**See Also**
- [\<noremap\>](#noremap)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)
- [\<source\>](#source)

---

#### **`<noremap>`**
Non-recursively define a map that is invoked by `${IN_CMD}` and generates `${OUT_CMD}`, which is composed by the default map.

```vim
" Call in .vindrc
${MODE}noremap ${IN_CMD} ${OUT_CMD}
```
`${MODE}` is the [Mode Prefix](#mode-prefix).

**See Also**
- [\<map\>](#map)
- [\<unmap\>](#unmap)
- [\<mapclear\>](#mapclear)
- [\<source\>](#source)

---

#### **`<unmap>`**
Remove the map corresponding to the `${IN_CMD}`.

```vim
${MODE}unmap ${IN_CMD}
```
`${MODE}` is the [Mode Prefix](#mode-prefix).

**See Also**
- [\<mapclear\>](#mapclear)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<source\>](#source)

---

#### **`<mapclear>`**
Delete all maps.

```vim
${MODE}mapclear
```
`${MODE}` is the [Mode Prefix](#mode-prefix).

**See Also**
- [\<unmap\>](#unmap)
- [\<map\>](#map)
- [\<noremap\>](#noremap)
- [\<source\>](#source)

---

#### **`<command>`**
It defines the command to call the `${OUT_CMD}`.

```vim
command ${IN_CMD} ${OUT_CMD}
```
`${MODE}` is the [Mode Prefix](#mode-prefix).

**See Also**
- [\<delcommand\>](#delcommand)
- [\<comclear\>](#comclear)
- [\<map\>](#map)
- [\<noremap\>](#noremap)

---

#### **`<delcommand>`**
Remove the command corresponding to the `{IN_CMD}`.

```vim
delcommand ${IN_CMD}
```
`${MODE}` is the [Mode Prefix](#mode-prefix).

**See Also**
- [\<comclear\>](#comclear)
- [\<command\>](#command)
- [\<map\>](#map)
- [\<noremap\>](#noremap)

---

#### **`<comclear>`**
System Command comclear.
Delete all commands

```vim
comclear
```

**See Also**
- [\<delcommand\>](#delcommand)
- [\<command\>](#command)
- [\<map\>](#map)
- [\<noremap\>](#noremap)

---

#### **`<autocmd_add>`**

**Syntax**
```vim
autocmd {event} {aupat} {cmd}
```

It adds `{cmd}` into autocmd list for `{aupat}`, autocmd pattern, corresponding to `{event}`.
As such as Vim, this function append `{cmd}` into a list rather than overwriting it even if the same `{cmd}` has already existed in a list.
The rule of `{aupat}` is based on the original Vim.
The registered `{cmd}`s will execute in the order added.

**Event**

The following table shows the supported events.
The string of each event is NOT case-sensitive.

|*Event*|*When does it ignite?*|
|:---|:---|
|AppEnter|Select an application|
|AppLeave|Unselect an application|
|GUINormalEnter|Enter to the GUI normal mode|
|GUINormalLeave|Leave from the GUI normal mode|
|GUIVisualEnter|Enter to the GUI visual mode|
|GUIVisualLeave|Leave from the GUI visual mode|
|EdiNormalEnter|Enter to the editor normal mode|
|EdiNormalLeave|Leave from the editor normal mode|
|EdiVisualEnter|Enter to the editor visual mode|
|EdiVisualLeave|Leave from the editor visual mode|
|InsertEnter|Enter to the insert mode|
|InsertLeave|Leave from the insert mode|
|ResidentEnter|Enter to the resident mode|
|ResidentLeave|Leave from the resident mode|
|CmdLineEnter|Enter to the command mode|
|CmdLineLeave|Leave from the command mode|

The event does not allow us to use `*`.
If you want to add a command to multiple events at the same time, `,` without after-space is available.

**Pattern**
If the pattern contains `/`, it matches the absolute path of the executable file of the application which creates each event.
If it does not contain `/`, it is compared against the name of the executable file.
The pattern is NOT case-sensitive.

|Pattern|Interpretation|
|:---|:---|
|`*`|Matches any character|
|`?`|Matches any single character|
|`\?`|Matches the `?` character|
|`.`|Matches the `.` character|
|`~`|Matches the `~` character|

All path delimiters `\` in Windows are treated as `/` in pattern translation.  
If you want to add a command to multiple patterns at the same time, `,` without after-space is available.
All others follow the general regex.

**Examples**

```vim
" Default mapping (match any applications)
autocmd AppLeave * <to_insert>

" Limited mapping (match specific application)
autocmd AppEnter *notepad* <to_edi_normal>
autocmd AppEnter,EdiNormalEnter vim.exe <to_resident>
autocmd AppEnter C:/*/Zotero/zotero.exe <to_edi_normal>
```

**See Also**
- [\<autocmd_del\>](#autocmd_del)

---

#### **`<autocmd_del>`**
**Syntax**
```vim
autocmd! {event} {aupat} {cmd}
```

It remove all autocmd matched `{event}` and `{aupat}`, then register `{cmd}` after delete.

The following syntaxes are available.

```vim
autocmd! {event} {aupat} {cmd}
autocmd! {event} {aupat}
autocmd! * {aupat}
autocmd! {event}
```

Each features are the same as the original Vim.

**Examples**

```vim
autocmd! * *vim*  " Remove all events having the pattern *vim*
autocmd! AppLeave *notepad* <to_insert>  " Remove old events and add a new event
```

**See Also**
- [\<autocmd_add\>](#autocmd_add)

### Mouse

#### **`<click_left>`**
Left button of a mouse click.

**See Also**
- [\<click_right\>](#click_right)
- [\<click_mid\>](#click_mid)
- [\<easyclick\>](#easyclick)
- [\<gridmove\>](#gridmove)

---

#### **`<click_right>`**
Right button of a mouse click.

**See Also**
- [\<click_left\>](#click_left)
- [\<click_mid\>](#click_mid)
- [\<easyclick\>](#easyclick)
- [\<gridmove\>](#gridmove)

---

#### **`<click_mid>`**
Middle button of a mouse click.

**See Also**
- [\<click_left\>](#click_left)
- [\<click_right\>](#click_right)
- [\<easyclick\>](#easyclick)
- [\<gridmove\>](#gridmove)

---

#### **`<move_cursor_left>`**
Move the mouse cursor to the left.

**See Also**
- [\<move_cursor_right\>](#move_cursor_right)
- [\<move_cursor_up\>](#move_cursor_up)
- [\<move_cursor_down\>](#move_cursor_down)

---

#### **`<move_cursor_right>`**
Move the mouse cursor to the right.

**See Also**
- [\<move_cursor_left\>](#move_cursor_left)
- [\<move_cursor_up\>](#move_cursor_up)
- [\<move_cursor_down\>](#move_cursor_down)

---

#### **`<move_cursor_up>`**
Move the mouse cursor up.

**See Also**
- [\<move_cursor_left\>](#move_cursor_left)
- [\<move_cursor_right\>](#move_cursor_right)
- [\<move_cursor_down\>](#move_cursor_down)

---

#### **`<move_cursor_down>`**
Move the mouse cursor down.

**See Also**
- [\<move_cursor_left\>](#move_cursor_left)
- [\<move_cursor_right\>](#move_cursor_right)
- [\<move_cursor_up\>](#move_cursor_up)

---

#### **`<easyclick>`**
Move a cursor using hints on the UI objects without clicking.

> **Note:**
> In versions prior to 5.1.0, there where commands `<easy_click_left>`, `<easy_click_right>`, `<easy_click_mid>`, and `<easy_click_hover>`, which have been merged into `<easyclick>`. For compatibility, the previous commands are automatically replaced by the following.
> <table>
> <tr>
>   <th>Conventional Name</th>
>   <th>Automatically Replaced Name</th>
> </tr>
> <tr>
>   <td><code>&lt;easy_click_left&gt;</code></td>
>   <td><code>&lt;easyclick&gt;&lt;click_left&gt;</code></td>
> </tr>
> <tr>
>   <td><code>&lt;easy_click_right&gt;</code></td>
>   <td><code>&lt;easyclick&gt;&lt;click_right&gt;</code></td>
> </tr>
> <tr>
>   <td><code>&lt;easy_click_mid&gt;</code></td>
>   <td><code>&lt;easyclick&gt;&lt;click_mid&gt;</code></td>
> </tr>
> <tr>
>   <td><code>&lt;easy_click_hover&gt;</code></td>
>   <td><code>&lt;easyclick&gt;</code></td>
> </tr>
> </table>

**See Also**
- [\<gridmove\>](#gridmove)
- [\<click_left\>](#click_left)
- [\<click_right\>](#click_right)
- [\<click_mid\>](#click_mid)
- [\<jump_cursor_to_active_window\>](#jump_cursor_to_active_window)
- [\<jump_cursor_with_keybrd_layout\>](#jump_cursor_with_keybrd_layout)

---

#### **`<easyclick_all>`**
This function is the [\<easyclick\>](#easyclick) to work on all visible applications even if they are not in focus.

**See Also**
- [\<easyclick\>](#easyclick)
- [\<gridmove\>](#gridmove)
- [\<click_left\>](#click_left)
- [\<click_right\>](#click_right)

---

#### **`<gridmove>`**
Move a cursor using tiled hints laid on the entire screen.
To change fonts or colors, you can set the several options, such as `gridmove_bgcolor`, `gridmove_fontcolor`, `gridmove_fontname`, `gridmove_fontsize`, and `gridmove_fontweight`.
In order to change the grid size, set the size with `gridmove_size` option. It assumes a text as its value, such as `12x8` for horizontal 12 cells and vertical 8 cells.

**See Also**
- [\<easyclick\>](#easyclick)
- [\<click_left\>](#click_left)
- [\<click_right\>](#click_right)
- [\<click_mid\>](#click_mid)
- [\<jump_cursor_to_active_window\>](#jump_cursor_to_active_window)
- [\<jump_cursor_with_keybrd_layout\>](#jump_cursor_with_keybrd_layout)

---

#### **`<focus_textarea>`**
Select the text area closest to the cursor and move the mouse cursor over it.
If there are multiple text areas, the selection is based on the minimum Euclidean distance between the mouse cursor and the center point of the bounding box of the text area.

In the previous version of win-vind, this function was attached to the Editor Normal Mode as the `autofocus_textarea` option, but it is now independent. Currently, the `autofocus_textarea` option is deprecated. For compatibility, `autofocus_textarea` defines a mapping such as `autocmd EdiNormalEnter * <focus_textarea>`.

---

#### **`<jump_cursor_to_left>`**
Jump the mouse cursor to the left.

**See Also**
- [\<jump_cursor_to_right\>](#jump_cursor_to_right)
- [\<jump_cursor_to_top\>](#jump_cursor_to_top)
- [\<jump_cursor_to_bottom\>](#jump_cursor_to_bottom)
- [\<jump_cursor_to_hcenter\>](#jump_cursor_to_hcenter)
- [\<jump_cursor_to_vcenter\>](#jump_cursor_to_vcenter)

---

#### **`<jump_cursor_to_right>`**
Jump the mouse cursor to the right.

**See Also**
- [\<jump_cursor_to_left\>](#jump_cursor_to_left)
- [\<jump_cursor_to_top\>](#jump_cursor_to_top)
- [\<jump_cursor_to_bottom\>](#jump_cursor_to_bottom)
- [\<jump_cursor_to_hcenter\>](#jump_cursor_to_hcenter)
- [\<jump_cursor_to_vcenter\>](#jump_cursor_to_vcenter)

---

#### **`<jump_cursor_to_top>`**
Jump the mouse cursor to the top.

**See Also**
- [\<jump_cursor_to_left\>](#jump_cursor_to_left)
- [\<jump_cursor_to_right\>](#jump_cursor_to_right)
- [\<jump_cursor_to_bottom\>](#jump_cursor_to_bottom)
- [\<jump_cursor_to_hcenter\>](#jump_cursor_to_hcenter)
- [\<jump_cursor_to_vcenter\>](#jump_cursor_to_vcenter)

---

#### **`<jump_cursor_to_bottom>`**
Jump the mouse cursor to the bottom.

**See Also**
- [\<jump_cursor_to_left\>](#jump_cursor_to_left)
- [\<jump_cursor_to_right\>](#jump_cursor_to_right)
- [\<jump_cursor_to_top\>](#jump_cursor_to_top)
- [\<jump_cursor_to_hcenter\>](#jump_cursor_to_hcenter)
- [\<jump_cursor_to_vcenter\>](#jump_cursor_to_vcenter)

---

#### **`<jump_cursor_to_hcenter>`**
Jump the mouse cursor to the horizontal center.

**See Also**
- [\<jump_cursor_to_left\>](#jump_cursor_to_left)
- [\<jump_cursor_to_right\>](#jump_cursor_to_right)
- [\<jump_cursor_to_top\>](#jump_cursor_to_top)
- [\<jump_cursor_to_bottom\>](#jump_cursor_to_bottom)
- [\<jump_cursor_to_vcenter\>](#jump_cursor_to_vcenter)

---

#### **`<jump_cursor_to_vcenter>`**
Jump the mouse cursor to the vertical center.

**See Also**
- [\<jump_cursor_to_left\>](#jump_cursor_to_left)
- [\<jump_cursor_to_right\>](#jump_cursor_to_right)
- [\<jump_cursor_to_top\>](#jump_cursor_to_top)
- [\<jump_cursor_to_bottom\>](#jump_cursor_to_bottom)
- [\<jump_cursor_to_hcenter\>](#jump_cursor_to_hcenter)

---

#### **`<jump_cursor_to_active_window>`**
Jump the mouse cursor to the foreground window.

**See Also**
- [\<jump_cursor_with_keybrd_layout\>](#jump_cursor_with_keybrd_layout)
- [\<easyclick\>](#easyclick)
- [\<gridmove\>](#gridmove)

---

#### **`<jump_cursor_with_keybrd_layout>`**
Jump the mouse cursor by keyboard mapping.

**See Also**
- [\<jump_cursor_to_active_window\>](#jump_cursor_to_active_window)
- [\<easyclick\>](#easyclick)
- [\<gridmove\>](#gridmove)

---

#### **`<scroll_up>`**
Scroll the mouse wheel up.

**See Also**
- [\<scroll_up_onepage\>](#scroll_up_onepage)
- [\<scroll_up_halfpage\>](#scroll_up_halfpage)
- [\<scroll_down\>](#scroll_down)

---

#### **`<scroll_up_halfpage>`**
Scroll the mouse wheel up with a half page.

**See Also**
- [\<scroll_up\>](#scroll_up)
- [\<scroll_up_onepage\>](#scroll_up_onepage)
- [\<scroll_down\>](#scroll_down)

---

#### **`<scroll_up_onepage>`**
Scroll the mouse wheel up with a page.

**See Also**
- [\<scroll_up\>](#scroll_up)
- [\<scroll_up_halfpage\>](#scroll_up_halfpage)
- [\<scroll_down\>](#scroll_down)

---

#### **`<scroll_down>`**
Scroll the mouse wheel down.

**See Also**
- [\<scroll_down_onepage\>](#scroll_down_onepage)
- [\<scroll_down_halfpage\>](#scroll_down_halfpage)
- [\<scroll_up\>](#scroll_up)

---

#### **`<scroll_down_halfpage>`**
Scroll the mouse wheel down with a half page.

**See Also**
- [\<scroll_down\>](#scroll_down)
- [\<scroll_down_onepage\>](#scroll_down_onepage)
- [\<scroll_up\>](#scroll_up)

---

#### **`<scroll_down_onepage>`**
Scroll the mouse wheel down with a page.

**See Also**
- [\<scroll_down\>](#scroll_down)
- [\<scroll_down_halfpage\>](#scroll_down_halfpage)
- [\<scroll_up\>](#scroll_up)

---

#### **`<scroll_left>`**
Scroll the mouse wheel left.

**See Also**
- [\<scroll_left_halfpage\>](#scroll_left_halfpage)
- [\<scroll_right\>](#scroll_right)

---

#### **`<scroll_left_halfpage>`**
Scroll the mouse wheel left with a half page.

**See Also**
- [\<scroll_left\>](#scroll_left)
- [\<scroll_right\>](#scroll_right)

---

#### **`<scroll_right>`**
Scroll the mouse wheel right.

**See Also**
- [\<scroll_right_halfpage\>](#scroll_right_halfpage)
- [\<scroll_left\>](#scroll_left)

---

#### **`<scroll_right_halfpage>`**
Scroll the mouse wheel right with a half page.

**See Also**
- [\<scroll_right\>](#scroll_right)
- [\<scroll_left\>](#scroll_left)

### Window

#### **`<window_resizer>`**
Start window resizer. It respects Vim plugin [simeji/winresizer](https://github.com/simeji/winresizer).

**See Also**
- [\<resize_window_width\>](#resize_window_width)
- [\<increase_window_width\>](#increase_window_width)
- [\<arrange_windows\>](#arrange_windows)

---

#### **`<switch_window>`**
Switch a window.

**See Also**
- [\<window_resizer\>](#window_resizer)
- [\<select_left_window\>](#select_left_window)
- [\<select_upper_window\>](#select_upper_window)

---

#### **`<select_left_window>`**
Select the left window.

**See Also**
- [\<select_right_window\>](#select_right_window)
- [\<select_upper_window\>](#select_upper_window)
- [\<select_lower_window\>](#select_lower_window)
- [\<window_resizer\>](#window_resizer)

---

#### **`<select_right_window>`**
Select the right window.

**See Also**
- [\<select_left_window\>](#select_left_window)
- [\<select_upper_window\>](#select_upper_window)
- [\<select_lower_window\>](#select_lower_window)
- [\<window_resizer\>](#window_resizer)

---

#### **`<select_upper_window>`**
Select the upper window.

**See Also**
- [\<select_lower_window\>](#select_lower_window)
- [\<select_left_window\>](#select_left_window)
- [\<select_right_window\>](#select_right_window)
- [\<window_resizer\>](#window_resizer)

---

#### **`<select_lower_window>`**
Select the lower window.

**See Also**
- [\<select_upper_window\>](#select_upper_window)
- [\<select_left_window\>](#select_left_window)
- [\<select_right_window\>](#select_right_window)
- [\<window_resizer\>](#window_resizer)

---

#### **`<move_window_left>`**
Moves the selected widow to the left.
The amount of window movement is based on the [window_velocity](#window_velocity) parameter as in [window_resizer](#window_resizer).
If a number is entered before the command, such as `20<c-w><c-h>`, the window will be moved by `20 * window_velocity`.

The window can only be moved as far as it is visible on the screen.
Therefore, excessive movement will cause the window to stop at the edge of the screen.
If you are using multiple displays, the movement range is determined by the combined resolution of all displays.

**See Also**
- [\<window_resizer\>](#window_resizer)

---

#### **`<move_window_right>`**
Moves the selected widow to the right.
The amount of window movement is based on the [window_velocity](#window_velocity) parameter as in [window_resizer](#window_resizer).
If a number is entered before the command, such as `20<c-w><c-l>`, the window will be moved by `20 * window_velocity`.

The window can only be moved as far as it is visible on the screen.
Therefore, excessive movement will cause the window to stop at the edge of the screen.
If you are using multiple displays, the movement range is determined by the combined resolution of all displays.

**See Also**
- [\<window_resizer\>](#window_resizer)

---

#### **`<move_window_up>`**
Moves the selected widow to the up.
The amount of window movement is based on the [window_velocity](#window_velocity) parameter as in [window_resizer](#window_resizer).
If a number is entered before the command, such as `20<c-w><c-k>`, the window will be moved by `20 * window_velocity`.

The window can only be moved as far as it is visible on the screen.
Therefore, excessive movement will cause the window to stop at the edge of the screen.
If you are using multiple displays, the movement range is determined by the combined resolution of all displays.

**See Also**
- [\<window_resizer\>](#window_resizer)

---

#### **`<move_window_down>`**
Moves the selected widow to the down.
The amount of window movement is based on the [window_velocity](#window_velocity) parameter as in [window_resizer](#window_resizer).
If a number is entered before the command, such as `20<c-w><c-j>`, the window will be moved by `20 * window_velocity`.

The window can only be moved as far as it is visible on the screen.
Therefore, excessive movement will cause the window to stop at the edge of the screen.
If you are using multiple displays, the movement range is determined by the combined resolution of all displays.

**See Also**
- [\<window_resizer\>](#window_resizer)

---

#### **`<maximize_current_window>`**
Maximize the current window.

**See Also**
- [\<minimize_current_window\>](#minimize_current_window)
- [\<resize_window_width\>](#resize_window_width)
- [\<resize_window_height\>](#resize_window_height)
- [\<window_resizer\>](#window_resizer)

---

#### **`<minimize_current_window>`**
Minimize the current window.

**See Also**
- [\<maximize_current_window\>](#maximize_current_window)
- [\<resize_window_width\>](#resize_window_width)
- [\<resize_window_height\>](#resize_window_height)
- [\<window_resizer\>](#window_resizer)

---

#### **`<resize_window_width>`**
Set the width of a window. You have to pass the pixel value as an argument using the command line.

**See Also**
- [\<resize_window_height\>](#resize_window_height)
- [\<increase_window_width\>](#increase_window_width)
- [\<decrease_window_width\>](#decrease_window_width)
- [\<window_resizer\>](#window_resizer)

---

#### **`<increase_window_width>`**
Increase the width of a window.

**See Also**
- [\<decrease_window_width\>](#decrease_window_width)
- [\<increase_window_height\>](#increase_window_height)
- [\<decrease_window_height\>](#decrease_window_height)
- [\<resize_window_width\>](#resize_window_width)

---

#### **`<decrease_window_width>`**
Decrease the width of a window.

**See Also**
- [\<increase_window_width\>](#increase_window_width)
- [\<increase_window_height\>](#increase_window_height)
- [\<decrease_window_height\>](#decrease_window_height)
- [\<resize_window_width\>](#resize_window_width)

---

#### **`<resize_window_height>`**
Set the height of a window. You have to pass the pixel value as an argument using the command line.

**See Also**
- [\<resize_window_width\>](#resize_window_width)
- [\<increase_window_height\>](#increase_window_height)
- [\<decrease_window_height\>](#decrease_window_height)
- [\<window_resizer\>](#window_resizer)

---

#### **`<increase_window_height>`**
Increase the height of a window.

**See Also**
- [\<decrease_window_height\>](#decrease_window_height)
- [\<increase_window_width\>](#increase_window_width)
- [\<decrease_window_width\>](#decrease_window_width)
- [\<resize_window_height\>](#resize_window_height)

---

#### **`<decrease_window_height>`**
Decrease the height of a window.

**See Also**
- [\<increase_window_height\>](#increase_window_height)
- [\<increase_window_width\>](#increase_window_width)
- [\<decrease_window_width\>](#decrease_window_width)
- [\<resize_window_height\>](#resize_window_height)

---

#### **`<arrange_windows>`**
Arrange windows with tile style.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<exchange_window_with_nearest>`**
Exchange a window with the nearest window.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<rotate_windows>`**
Rotate windows in the current monitor.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<rotate_windows_in_reverse>`**
Rotate windows in the current monitor in reverse.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<snap_current_window_to_left>`**
Snap the current window to the left.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<snap_current_window_to_right>`**
Snap the current window to the right.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<snap_current_window_to_top>`**
Snap the current window to the top.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<snap_current_window_to_bottom>`**
Snap the current window to the bottom.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<open_new_window>`**
Open a new window.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<reload_current_window>`**
Reload the current window.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<open_new_window_with_hsplit>`**
Open a new window with a horizontal split.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<open_new_window_with_vsplit>`**
Open a new window with a vertical split.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<close_current_window>`**
Close the current window.

<!--
**See Also**
- [\<a\>](#a)
-->

### Process

#### **`<help>`**
This function is called from the virtual command line as a command and opens the document page matching the arguments. The arguments can be function names, option names, parameter names, or predefined tags.

**Examples**

```vim
" Execute from the virtual command line
:help easyclick      " Function name
:help uiacachebuild  " Option name
:help gridmove_size  " Parameter name
:help usage          " Predefined tag
```

---

#### **`<execute>`**
Open file with the associated application. This is a wrapper for the famous Windows API, **ShellExecute**, which behaves the same as double-clicking in Explorer. Therefore, you can open any format files and URLs. For example, `:e ~/.vimrc` or `:e https://www.google.com`. If there is no argument, it will open .vindrc loaded at initialization.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<exit>`**
Exit win-vind.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<sleep>`**
Sleep win-vind for N seconds.
As the same as Vim, this command is called with commands of the command mode or some bindings.
The duration of time to sleep is specified by the arguments of commands (e.g., `:sleep 10`) or the prefix number of bindings (e.g., `10gs`), as shown in the below examples.
When `m` is included, sleep for N milliseconds.
The default is one seconds.

**Example for command line**

```vim
:sleep       " sleep for one second
:sleep 5     " sleep for five seconds
:sleep 100m  " sleep for 100 milliseconds
```

**Example for .vindrc**

```vim
map <ctrl-1> :sleep 5<cr><easyclick>  " Launch easyclick after 5 seconds.
```

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<start_external>`**
Start an external application. This environment variable is dependent on the application specified in the `shell` option. By appending `;` at the end, it keeps the console window without closing immediately. If the explorer is the foreground window, the current directory of a terminal will be that directory.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<start_shell>`**
Start a terminal. If the explorer is the foreground window, the current directory of a terminal will be that directory.

### Vim Emulation

#### **`<to_insert_BOL>`**
**Vim Emulation:** `I`  
Insert to begin of line.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<to_insert_EOL>`**
**Vim Emulation:** `A`  
Append end of line.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<to_insert_append>`**
**Vim Emulation:** `a`  
Append after a caret.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<to_insert_nlabove>`**
**Vim Emulation:** `O`  
Begin new line above a caret.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<to_insert_nlbelow>`**
**Vim Emulation:** `o`  
Begin new line below a caret.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_caret_left>`**
**Vim Emulation:** `h`  
Move the caret to left.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_caret_down>`**
**Vim Emulation:** `j`  
Move the caret down.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_caret_up>`**
**Vim Emulation:** `k`  
Move the caret up.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_caret_right>`**
**Vim Emulation:** `l`  
Move the caret to right.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_fwd_word>`**
**Vim Emulation:** `w`  
Move words forward for normal mode.

It performs word-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.
Instead, you can [move_fwd_word_simple](#move_fwd_word_simple) for visual mode, which is faster and simpler (of course, it is also available for other modes)

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_fwd_word_simple>`**
**Vim Emulation:** `w`  
Move words forward fast.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_bck_word>`**
**Vim Emulation:** `b`  
Move words backward for normal mode.

It performs word-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.
Instead, you can [move_bck_word_simple](#move_fwd_word_simple) for visual mode, which is faster and simpler (of course, it is also available for other modes)

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_bck_word_simple>`**
**Vim Emulation:** `b`  
Move words backward fast.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_fwd_bigword>`**
**Vim Emulation:** `W`  
Move WORDS forward.

It performs WORD-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.
Instead, you can [move_fwd_word_simple](#move_fwd_word_simple) for visual mode, which is faster and simpler (of course, it is also available for other modes)

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_bck_bigword>`**
**Vim Emulation:** `B`  
Move WORDS backward.

It performs WORD-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.
Instead, you can [move_bck_word_simple](#move_fwd_word_simple) for visual mode, which is faster and simpler (of course, it is also available for other modes)

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_end_word>`**
**Vim Emulation:** `e`  
Forward to the end of words.

It performs word-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_end_bigword>`**
**Vim Emulation:** `E`  
Forward to the end of WORDS.

It performs WORD-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_bckend_word>`**
**Vim Emulation:** `ge`  
Backward to the end of words.

It performs word-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<move_bckend_bigword>`**
**Vim Emulation:** `gE`  
Backward to the end of WORDS.

It performs WORD-motion using an algorithm that is completely identical to Vim.
However, it is not available in visual mode, since the text is selected and copied once and retrieved via the clipboard for text parsing.

The `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.
There is an option [charbreak](#charbreak) to set the criteria for considering a Unicode character as a single character. 

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<jump_caret_to_BOF>`**
**Vim Emulation:** `gg`  
Jump the caret to BOF.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<jump_caret_to_BOL>`**
**Vim Emulation:** `0`  
Jump the caret to begin of line.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<jump_caret_to_EOF>`**
**Vim Emulation:** `G`  
Jump the caret to EOF.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<jump_caret_to_EOL>`**
**Vim Emulation:** `$`  
Jump the caret to end of line.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<change_char>`**
**Vim Emulation:** `s`  
Change Characters.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<change_highlight_text>`**
**Vim Emulation:** `c`, `s`, `S`  
Change highlighted texts.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<change_line>`**
**Vim Emulation:** `cc`, `S`  
Change Lines.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<change_until_EOL>`**
**Vim Emulation:** `C`  
Change until EOL.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<change_with_motion>`**
**Vim Emulation:** `c{motion}`  
Change texts with motion.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<delete_after>`**
**Vim Emulation:** `x`  
Delete chars after the caret.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<delete_before>`**
**Vim Emulation:** `X`  
Delete chars before the caret.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<delete_highlight_text>`**
**Vim Emulation:** `d`, `x`, `X`  
Delete highlighted texts.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<delete_line>`**
**Vim Emulation:** `dd`  
Delete lines.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<delete_line_until_EOL>`**
**Vim Emulation:** `D`  
Delete texts until end of line.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<delete_with_motion>`**
**Vim Emulation:** `d{motion}`  
Delete texts with motion.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<join_next_line>`**
**Vim Emulation:** `J`  
Join a next line.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **``**
**Vim Emulation:** `p`  
Put texts after the caret.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **``**
**Vim Emulation:** `P`  
Put texts before the caret.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<repeat_last_change>`**
**Vim Emulation:** `.`  
Repeat last simple change.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<replace_char>`**
**Vim Emulation:** `r`  
Replace a char.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<replace_sequence>`**
**Vim Emulation:** `R`  
Replace Mode.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<switch_char_case>`**
**Vim Emulation:** `~`  
Switch char case.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<yank_highlight_text>`**
**Vim Emulation:** `y`  
Yank highlighted texts.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<yank_line>`**
**Vim Emulation:** `yy`, `Y`  
Yank lines.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<yank_with_motion>`**
**Vim Emulation:** `y{motion}`  
Yank lines with motion.

<!--
**See Also**
- [\<a\>](#a)
-->

### Hotkey

#### **`<backward_ui_navigation>`**
Backward UI Navigation.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<decide_focused_ui_object>`**
Decide a focused UI object.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<forward_ui_navigation>`**
Forward UI Navigation.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<goto_next_page>`**
Forward to the next page.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<goto_prev_page>`**
Go backward to the previous page.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<hotkey_backspace>`**
Backspace.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<hotkey_copy>`**
Copy.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<hotkey_cut>`**
Cut.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<hotkey_delete>`**
Delete.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<hotkey_paste>`**
Paste.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<open>`**
Open another file.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<open_startmenu>`**
Open the Start Menu.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<redo>`**
Redo.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<save>`**
Save the current file.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<search_pattern>`**
Search Pattern.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<select_all>`**
Select all.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<start_explorer>`**
Start Explorer.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<undo>`**
Undo.

### Virtual Desktop

#### **`<close_current_vdesktop>`**
Close a current virtual desktop.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<create_new_vdesktop>`**
Create a new virtual desktop.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<switch_to_left_vdesktop>`**
Switch to a left virtual desktop.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<switch_to_right_vdesktop>`**
Switch to a right virtual desktop.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<taskview>`**
Task View.

### Tab

#### **`<close_current_tab>`**
Close a current tab.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<open_new_tab>`**
Open a new tab.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<switch_to_left_tab>`**
Switch to a left tab.

<!--
**See Also**
- [\<a\>](#a)
-->

---

#### **`<switch_to_right_tab>`**
Switch to a right tab.

<!--
**See Also**
- [\<a\>](#a)
-->

### File

#### **`<makedir>`**
Create a directory. If you call it with a relative path such as `:mkdir foo`, it creates it in the explorer directory. If no explorer is found, it creates it in `~/Desktop/foo`. If you call it with an absolute path like `:mkdir C:/Users/You/Desktop/bar`, it will create a directory along the path recursively.

<!--
**See Also**
- [\<a\>](#a)
-->

## Apéndice B — Referencia completa de opciones y parámetros

Todas las opciones de `set`, con tipo y valor por defecto.

### System

#### **`initmode`**
**type**: str, **default**: i  
Initial mode of win-vind. The value is the mode prefix.

---

#### **`listen_interval`**
**type**: float, **default**: 1.0  
The time interval in seconds at which the server win-vind will retrieve command requests sent by the client with the `-c` argument.

---

#### **`icon_style`**
**type**: str, **default**: resources/icon32_dark.ico  
Style of the icon to be displayed on the taskbar. By default, **Dark** and **Light** styles are available. The former is `resources/icon32_dark.ico` and the latter is `resouces/icon32_light.ico`. By the way, you can use any tasktray icon you like as long as it is in `.ico` format and **32x32**.

---

#### **`tempdir`**
**type**: str, **default**: ~/Downloads  
Where to download the file with the update checking.

---

#### **`gui_fontname`**
**type**: str, **default**: Segoe UI  
Font name of GUI. If an empty string is passed, the system font will be used.

---

#### **`gui_fontsize`**
**type**: num, **default**: 11  
Font size of GUI

---

#### **`hintkeys`**
**type**: str, **default**: asdghklqwertyuiopzxcvbnmfj  
Specify the characters of hint used for EasyClick and GridMove. It accpets as input a set of non-duplicate characters and assigns them to the hints in order from the first to the last.

### Command Line

#### **`vcmdline`**
**type**: bool, **default**: true  
Show virtual command line

---

#### **`showcmd`**
**type**: bool, **default**: true  
Show the partial command in the virtual command line.
This feature causes some overhead.
If the count of repeats for a command is specified, the command is displayed following the count of repeats.
If you do not enter a repeat count for a command, then the repeat count is denoted as 1.
Unlike Vim, the repeat count is always explicitly displayed to reduce mistakes in the repeat count.

---

#### **`cmd_bgcolor`**
**type**: str, **default**: 323232  
Background color in the virtual command line. (# is optional)

---

#### **`cmd_fontcolor`**
**type**: str, **default**: c8c8c8  
Font color in the virtual command line. (# is optional)

---

#### **`cmd_fontname`**
**type**: str, **default**: Consolas  
Font name for virtual command line. If an empty string is passed, the system font will be used.

---

#### **`cmd_fontsize`**
**type**: num, **default**: 23  
Font size in virtual command line

---

#### **`cmd_fontweight`**
**type**: num, **default**: 400  
Font weight in virtual command line. Its maximum value is 1000.

---

#### **`cmd_fontextra`**
**type**: num, **default**: 1  
Horizontal character spacing in virtual command line.

---

#### **`cmd_roughpos`**
**type**: str, **default**: LowerMid  
Rough position of virtual command line. The choices are `UpperLeft`, `UpperMid`, `UpperRight`, `MidLeft`, `Center`, `MidRight`, `LowerLeft`, `LowerMid`, or `LowerRight`.

---

#### **`cmd_xmargin`**
**type**: num, **default**: 32  
Use `cmd_roughpos` to determine the rough position, and `cmd_xmargin` to determine the detailed horizontal position. The units are in pixels.

---

#### **`cmd_ymargin`**
**type**: num, **default**: 96  
Use `cmd_roughpos` to determine the rough position, and `cmd_ymargin` to determine the detailed vertical position. The units are in pixels.

---

#### **`cmd_fadeout`**
**type**: num, **default**: 5  
Fade-out time in seconds for the virtual command line. If you want the command line to always be visible, make this value large enough.

---

#### **`cmd_monitor`**
**type**: str, **default**: primary  
The monitor on which to draw the command line. The choices are `primary`, `all`, `active`, `${NUMBER}`. The `primary` displays the command line on the primary monitor only. `all` draws command lines on all monitos. `active` displays command lines on the monitor where the selected window is located. `${NUMBER}` shows the command line on `${NUMBER}`th monitor. The `${NUMBER}` is a number starting from 0 and assigned from the left monitor. For example, `set cmd_monitor=1`.

### EasyClick

#### **`easyclick_bgcolor`**
**type**: str, **default**: 323232  
Font background color of hints in EasyClick

---

#### **`easyclick_fontcolor`**
**type**: str, **default**: c8c8c8  
Font color of hints in EasyClick

---

#### **`easyclick_fontname`**
**type**: str, **default**: Consolas  
Font name of hints in EasyClick

---

#### **`easyclick_fontsize`**
**type**: num, **default**: 14  
Font size of hints in EasyClick

---

#### **`easyclick_fontweight`**
**type**: num, **default**: 500  
Font weight of hits in EasyClick. Its maximum value is 1000.

### GridMove

#### **`gridmove_bgcolor`**
**type**: str, **default**: 323232  
Font background color of hints in GridMove

---

#### **`gridmove_fontcolor`**
**type**: str, **default**: c8c8c8  
Font color of hints in GridMove

---

#### **`gridmove_fontname`**
**type**: str, **default**: Consolas  
Font name of hints in GridMove

---

#### **`gridmove_fontsize`**
**type**: num, **default**: 14  
Font size of hints in GridMove

---

#### **`gridmove_fontweight`**
**type**: num, **default**: 500  
Font weight of hits in GridMove. Its maximum value is 1000.

---

#### **`gridmove_size`**
**type**: str, **default**: 12x8  
The grid size in GridMove. It assumes a text as its value, such as `12x8` for horizontal 12 cells and vertical 8 cells.

### Mouse

#### **`cursor_accel`**
**type**: num, **default**: 90  
Pixel-level acceleration in the constatnt acceleration motion of the mouse cursor.

---

#### **`cursor_resolution`**
**type**: num, **default**: 250  
A weight for scaling the time of constant acceleration motion of the mouse cursor.

---

#### **`jump_margin`**
**type**: num, **default**: 10  
A margin in pixels to prevent jumping off the screen when jumping to the edge of the screen using `jump_cursor_to_left`, etc.

---

#### **`hscroll_pageratio`**
**type**: num, **default**: 0.125  
The ratio of one page to the screen width to determine the amount of scrolling movement as a page.

---

#### **`hscroll_speed`**
**type**: num, **default**: 10  
Horizontal scrolling speed of the mouse wheel.

---

#### **`vscroll_pageratio`**
**type**: num, **default**: 0.125  
The ratio of one page to the screen height to determine the amount of scrolling movement as a page.

---

#### **`vscroll_speed`**
**type**: num, **default**: 30  
Vertical scrolling speed of the mouse wheel.

---

#### **`keybrd_layout`**
**type**: str, **default**:   
Keyboard layout kmp file referenced by `jump_cursor_with_keybrd_layout`. By default, only **US (101/102)** or **JP (106/109)** layouts are supported. If your keyboard is not the right one, please create your own kmp file and use its path as the value. If you leave the value empty, the KMP file will be selected automatically.

### Window

#### **`arrangewin_ignore`**
**type**: str, **default**:   
A list of executable filenames to ignore in ArrangeWindows. For example, if you want to remove rainmeter and gvim from the alignment, write `set arrangewin_ignore = rainmeter, gvim`. The name is the name of the executable file without extension.

---

#### **`window_velocity`**
**type**: num, **default**: 100  
Pixel-level velocity in the constatnt acceleration motion of the window in winresizer.

---

#### **`window_hdelta`**
**type**: num, **default**: 100  
Window Width delta for resizing

---

#### **`window_vdelta`**
**type**: num, **default**: 100  
Window height delta for resizing

---

#### **`winresizer_initmode`**
**type**: num, **default**: 0  
Initial mode of window resizer ([0]: Resize, [1]: Move, [2]: Focus)

### Block Style Caret

#### **`blockstylecaret`**
**type**: bool, **default**: false  
Block Style Caret

---

#### **`blockstylecaret_mode`**
**type**: str, **default**: solid  
Mode of block style caret.  There is a `solid` mode with fixed size and a `flex` mode with pseudo blocks by selection.

---

#### **`blockstylecaret_width`**
**type**: num, **default**: 15  
Width of block style caret on solid mode

### AutoFocus

#### **`autotrack_popup`**
**type**: bool, **default**: false  
It is one of standard options on Windows. For example, if shown **Are you sure you want to move this file to the Recycle Bin?**, it automatically moves the cursor to the popup window.

### UIA Cache

#### **`uiacachebuild`**
**type**: bool, **default**: false  
[easyclick](#easyclick) and [focus_textarea](#focus_textarea) are slow because they scan the UI object after being called. If this option is enabled, scanning is done asynchronously and cache is used as a result. Using the cache is 30 times faster than scanning linearly, but the location information, etc. may not always be correct.

---

#### **`uiacachebuild_lifetime`**
**type**: num, **default**: 1500  
Cache lifetime (ms). A high value reduces the computational cost, but decreases the reliability of the cache. A low value increases the computational cost due to frequent cache creation, but guarantees reliability.

---

#### **`uiacachebuild_staybegin`**
**type**: num, **default**: 500  
The time between when the mouse cursor stops moving and when it starts to build a cache. In order to reduce unnecessary computational cost, it is desirable not to create a cache when there is no operation. Therefore, it should be updated only immediately after the mouse stops. The value of this option is the time(ms) that the mouse cursor is considered to be stopped.

---

#### **`uiacachebuild_stayend`**
**type**: num, **default**: 2000  
In order to reduce unnecessary computational cost, it is desirable not to create a cache when there is no operation. The value of this option is the time(ms) between the time the cursor stops moving and the time it stops creating a cache.

### Shell

#### **`shell`**
**type**: str, **default**: cmd  
Name of the shell to use for `:!` commands

---

#### **`shell_startupdir`**
**type**: str, **default**:   
The current directory where commands (e.g. `:shell`, `:terminal`, `:!`) will be executed. For these commands, the current directory is the directory if there is Exeplorer, or the user directory otherwise. If this option is not empty, then the current directory is fixed to a value directory.

---

#### **`shellcmdflag`**
**type**: str, **default**: -c  
Flag passed to the shell to execute `:!` commands

### Vim Emulation

#### **`charbreak`**
**type**: str, **default**: grapheme  
Mode for how to split a single Unicode character. The `grapheme` mode treats a combination character as a single character. The `codepoint` mode processes the combination character for each codepoint.

---

#### **`charcache`**
**type**: bool, **default**: false  
It is a very small cache for one character used by `x` or `X` commands. If it is enabled, the clipboard is opened per once typing. Therefore, you will get the same behavior as the original Vim, whereas the performance maybe drop a litte.

## Apéndice C — Keywords: todas las teclas y prefijos de modo

All keywords in win-vind are not case-sensitive.  

### Mode Prefix

|**Prefix**|**Mode**|
|:---:|:---|
| ` ` |GUI Normal, GUI Visual, Edi Normal, Edi Visual|
|`g` | GUI Normal, GUI Visual|
|`e` | Edi Normal, Edi Visual|
|`n` | GUI Normal, Edi Normal|
|`v` | GUI Visual, Edi Visual|
|`gn` | GUI Normal|
|`gv` | GUI Visual|
|`en` | Edi Normal|
|`ev` | Edi Visual|
|`i` | Insert Mode|
|`r` | Resident Mode|
|`c` | Command Mode|

### Specific Keyword  

|Keyword|Meanings|
|:---:|:---|
|`<num>`|It is a number of any digits. However, you **must not** multiple uses of this keyword per a command.|
|`<any>`|It is an optional string. After this keyword, all characters will be matched.|

### Specific Ascii Keyword  

|Keyword|Meanings|
|:---:|:---|
|`<space>`|Space Key|
|`<hbar>`|Ascii code '-'|
|`<gt>`|Ascii code '&gt;'|
|`<lt>`|Ascii code '&lt;'|

 
### System Keyword  

|Keyword|Meanings|
|:---:|:---|
|`<bs>`|BackSpace Key|
|`<capslock>`|CapsLock Key|
|`<cr>`|Enter Key, Return Key|
|`<enter>`|Enter Key, Return Key|
|`<return>`|Enter Key, Return Key|
|`<ime>`|Key for switching IME|
|`<tab>`|Tab Key|
|`<left>`|Left Key|
|`<right>`|Right Key|
|`<up>`|Up Key|
|`<down>`|Down Key|
|`<shift>`|Left Shift, Right Shift|
|`<s>`|Left Shift, Right Shift|
|`<lshift>`|Left Shift|
|`<ls>`|Left Shift|
|`<rshift>`|Right Shift|
|`<rs>`|Right Shift|
|`<ctrl>`|Left Control, Right Control|
|`<c>`|Left Control, Right Control|
|`<lctrl>`|Left Control|
|`<lc>`|Left Control|
|`<rctrl>`|Right Control|
|`<rc>`|Right Control|
|`<win>`|Left Windows Key, Right Windows Key|
|`<lwin>`|Left Windows Key|
|`<rwin>`|Right Windows Key|
|`<alt>`|Left Alt, Right Alt|
|`<a>`|Left Alt, Right Alt|
|`<lalt>`|Left Alt|
|`<la>`|Left Alt|
|`<ralt>`|Right Alt|
|`<ra>`|Right Alt|
|`<m>`|Left Alt, Right Alt|
|`<lm>`|Left Alt|
|`<rm>`|Right Alt|
|`<app>`|Application Key|
|`<cvt>`|Convert Key|
|`<esc>`|Eescape Key|
|`<kana>`|Kana Key to switch IME mode on Japanese keyboard.|
|`<nocvt>`|No Convert Key|
|`<f1>`|F1|
|`<f2>`|F2|
|`<f3>`|F3|
|`<f4>`|F4|
|`<f5>`|F5|
|`<f6>`|F6|
|`<f7>`|F7|
|`<f8>`|F8|
|`<f9>`|F9|
|`<f10>`|F10|
|`<f11>`|F11|
|`<f12>`|F12|
|`<f13>`|F13|
|`<f14>`|F14|
|`<f15>`|F15|
|`<f16>`|F16|
|`<f17>`|F17|
|`<f18>`|F18|
|`<f19>`|F19|
|`<f20>`|F20|
|`<f21>`|F21|
|`<f22>`|F22|
|`<f23>`|F23|
|`<f24>`|F24|
|`<del>`|Delete Key|
|`<end>`|End Key|
|`<home>`|Home Key|
|`<insert>`|Insert Key|
|`<numlock>`|NumLock Key|
|``|Page Down Key|
|``|Page Up Key|
|``|Pause Key, Break Key|
|`<scroll>`|Scroll Key, Scroll Lock Key|
|`<snapshot>`|Snapshot Key, Print Screen Key, Sys Rq Key|

## Apéndice D — Mapeos por defecto de todos los tiers

**Tier `tiny`**

### GUI Normal Mode

#### Mode Transition

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<esc-left>`, `<ctrl-]>`|[\<to_gui_normal\>](#to_gui_normal)|
|`<esc-down>`|[\<to_resident\>](#to_resident)|
|`:`|[\<to_command\>](#to_command)|
|`i`|[\<click_left\>](#click_left)[\<to_insert\>](#to_insert)|

#### Mouse Movement

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<bs>`, `<left>`, `h`|[\<move_cursor_left\>](#move_cursor_left)|
|`<right>`, `l`, `<space>`|[\<move_cursor_right\>](#move_cursor_right)|
|`<up>`, `-`, `k`|[\<move_cursor_up\>](#move_cursor_up)|
|`+`, `j`, `<down>`|[\<move_cursor_down\>](#move_cursor_down)|

#### Mouse Clicking

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`o`|[\<click_left\>](#click_left)|
|`a`|[\<click_right\>](#click_right)|

#### Mouse Scrolling

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<c-k>`, `<c-y>`|[\<scroll_up\>](#scroll_up)|
|`<c-e>`, `<c-j>`|[\<scroll_down\>](#scroll_down)|
|`<c-u>`|[\<scroll_up_halfpage\>](#scroll_up_halfpage)|
|`<c-d>`|[\<scroll_down_halfpage\>](#scroll_down_halfpage)|
|`<c-b>`|[\<scroll_up_onepage\>](#scroll_up_onepage)|
|`<c-f>`|[\<scroll_down_onepage\>](#scroll_down_onepage)|
|`zh`, `<c-h>`|[\<scroll_left\>](#scroll_left)|
|`zl`, `<c-l>`|[\<scroll_right\>](#scroll_right)|
|`zh`|[\<scroll_left_halfpage\>](#scroll_left_halfpage)|
|`zl`|[\<scroll_right_halfpage\>](#scroll_right_halfpage)|

#### Mouse Jumping

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`0`, `<home>`, `^`|[\<jump_cursor_to_left\>](#jump_cursor_to_left)|
|`$`, `<end>`|[\<jump_cursor_to_right\>](#jump_cursor_to_right)|
|`gg`|[\<jump_cursor_to_top\>](#jump_cursor_to_top)|
|`G`|[\<jump_cursor_to_bottom\>](#jump_cursor_to_bottom)|
|`gm`|[\<jump_cursor_to_hcenter\>](#jump_cursor_to_hcenter)|
|`M`|[\<jump_cursor_to_vcenter\>](#jump_cursor_to_vcenter)|
|`t`|[\<jump_cursor_to_active_window\>](#jump_cursor_to_active_window)|

#### Complex Mouse Controls

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`f`|[\<jump_cursor_with_keybrd_layout\>](#jump_cursor_with_keybrd_layout)|
|`FF`, `Fo`|[\<easyclick\>](#easyclick)[\<click_left\>](#click_left)|
|`Fa`|[\<easyclick\>](#easyclick)[\<click_right\>](#click_right)|
|`Fm`|[\<easyclick\>](#easyclick)[\<click_mid\>](#click_mid)|
|`Fh`|[\<easyclick\>](#easyclick)|
|`Ft`|[\<focus_textarea\>](#focus_textarea)|
|`<ctrl-m>`|[\<gridmove\>](#gridmove)|
|`AA`, `Ao`|[\<easyclick_all\>](#easyclick_all)[\<click_left\>](#click_left)|
|`Aa`|[\<easyclick_all\>](#easyclick_all)[\<click_right\>](#click_right)|
|`Am`|[\<easyclick_all\>](#easyclick_all)[\<click_mid\>](#click_mid)|
|`Ah`|[\<easyclick_all\>](#easyclick_all)|

### Insert Mode

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<Esc-Left>`, `<ctrl-]>`|[\<to_gui_normal\>](#to_gui_normal)|
|`<F8>`|[\<to_instant_gui_normal\>](#to_instant_gui_normal)|
|`<Esc-Down>`|[\<to_resident\>](#to_resident)|

### Resident Mode

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<Esc-Left>`|[\<to_gui_normal\>](#to_gui_normal)|
|`<Esc-Down>`|[\<to_resident\>](#to_resident)|
|`<Esc-up>`|[\<to_insert\>](#to_insert)|

### Command Mode

#### Mode Transition

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`guinormal`, `gn`|[\<to_gui_normal\>](#to_gui_normal)|
|`resident`|[\<to_resident\>](#to_resident)|
|`insert`, `i`|[\<to_insert\>](#to_insert)|

#### System Commands

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`exit`|[\<exit\>](#exit)|
|`sleep`|[\<sleep\>](#sleep)|
|`help`|[\<help\>](#help)|
|`set`|[\<set\>](#set)|
|`{mode}map`|[\<{mode}map\>](#map)|
|`{mode}noremap`|[\<{mode}noremap\>](#noremap)|
|`{mode}unmap`|[\<{mode}unmap\>](#unmap)|
|`{mode}mapclear`|[\<{mode}mapclear\>](#mapclear)|
|`com`, `command`|[\<command\>](#command)|
|`delcommand`, `delc`|[\<delcommand\>](#delcommand)|
|`comc`, `comclear`|[\<comclear\>](#comclear)|
|`source`, `so`|[\<source\>](#source)|
|`autocmd`|[\<autocmd_add\>](#autocmd_add)|
|`autocmd!`|[\<autocmd_del\>](#autocmd_del)|

**Tier `small`**

### GUI Normal Mode

#### Window Open/Close

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-w>n`|[\<open_new_window\>](#open_new_window)|
|`<C-w>q`, `<C-w>c`|[\<close_current_window\>](#close_current_window)|

#### Window Select

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-w>h`|[\<select_left_window\>](#select_left_window)|
|`<C-w>l`|[\<select_right_window\>](#select_right_window)|
|`<C-w>k`|[\<select_upper_window\>](#select_upper_window)|
|`<C-w>j`|[\<select_lower_window\>](#select_lower_window)|
|`<C-w>s`|[\<switch_window\>](#switch_window)|

#### Window Movement

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-w><C-h>`|[\<move_window_left\>](#move_window_left)|
|`<C-w><C-l>`|[\<move_window_right\>](#move_window_right)|
|`<C-w><C-k>`|[\<move_window_up\>](#move_window_up)|
|`<C-w><C-j>`|[\<move_window_down\>](#move_window_down)|

#### Window Resize

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-w>=`|[\<arrange_windows\>](#arrange_windows)|
|`<C-w>r`|[\<rotate_windows\>](#rotate_windows)|
|`<C-w>R`|[\<rotate_windows_in_reverse\>](#rotate_windows_in_reverse)|
|`<C-w>x`|[\<exchange_window_with_nearest\>](#exchange_window_with_nearest)|
|`<C-w><gt>`|[\<increase_window_width\>](#increase_window_width)|
|`<C-w><lt>`|[\<decrease_window_width\>](#decrease_window_width)|
|`<C-w>+`|[\<increase_window_height\>](#increase_window_height)|
|`<C-w>-`|[\<decrease_window_height\>](#decrease_window_height)|
|`<C-w>u`|[\<maximize_current_window\>](#maximize_current_window)|
|`<C-w>d`|[\<minimize_current_window\>](#minimize_current_window)|
|`<C-w><Left>`, `<C-w>H`|[\<snap_current_window_to_left\>](#snap_current_window_to_left)|
|`<C-w>L`, `<C-w><Right>`|[\<snap_current_window_to_right\>](#snap_current_window_to_right)|
|`<C-w>K`|[\<snap_current_window_to_top\>](#snap_current_window_to_top)|
|`<C-w>J`|[\<snap_current_window_to_bottom\>](#snap_current_window_to_bottom)|
|`<C-w>e`|[\<window_resizer\>](#window_resizer)|

### Command Mode

#### Window Open/Close

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`cl`, `close`|[\<close_current_window\>](#close_current_window)|
|`new`|[\<open_new_window\>](#open_new_window)|
|`sp`, `split`|[\<open_new_window_with_hsplit\>](#open_new_window_with_hsplit)|
|`vs`, `vsplit`|[\<open_new_window_with_vsplit\>](#open_new_window_with_vsplit)|

#### Window Select

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`switch`, `sw`|[\<switch_window\>](#switch_window)|

#### Window Resize

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`resizer`, `winresizer`|[\<window_resizer\>](#window_resizer)|
|`only`, `on`, `max`|[\<maximize_current_window\>](#maximize_current_window)|
|`hi`, `min`, `hide`|[\<minimize_current_window\>](#minimize_current_window)|
|`lsp`, `lsplit`|[\<snap_current_window_to_left\>](#snap_current_window_to_left)|
|`rsp`, `rsplit`|[\<snap_current_window_to_right\>](#snap_current_window_to_right)|
|`tsp`, `tsplit`|[\<snap_current_window_to_top\>](#snap_current_window_to_top)|
|`bsp`, `bsplit`|[\<snap_current_window_to_bottom\>](#snap_current_window_to_bottom)|
|`arrange`|[\<arrange_windows\>](#arrange_windows)|
|`reload`|[\<reload_current_window\>](#reload_current_window)|
|`rot`, `rotate`|[\<rotate_windows\>](#rotate_windows)|
|`rerot`, `rerotate`|[\<rotate_windows_in_reverse\>](#rotate_windows_in_reverse)|
|`exchange`|[\<exchange_window_with_nearest\>](#exchange_window_with_nearest)|
|`vertical<space>resize`, `vert<space>res`|[\<resize_window_width\>](#resize_window_width)|
|`vert<space>res<space>+`, `vertical<space>resize<space>+`|[\<increase_window_width\>](#increase_window_width)|
|`vertical<space>resize<space>-`, `vert<space>res<space>-`|[\<decrease_window_width\>](#decrease_window_width)|
|`res`, `resize`|[\<resize_window_height\>](#resize_window_height)|
|`res<space>+`, `resize<space>+`|[\<increase_window_height\>](#increase_window_height)|
|`resize<space>-`, `res<space>-`|[\<decrease_window_height\>](#decrease_window_height)|

#### Process Launcher

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`!`|[\<start_external\>](#start_external)|
|`e`, `edit`, `execute`|[\<execute\>](#execute)|
|`shell`, `sh`, `term`, `terminal`|[\<start_shell\>](#start_shell)|

**Tier `normal`**

### GUI Normal Mode

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`I`, `<esc-right>`, `<ctrl-[>`|[\<click_left\>](#click_left)[\<to_edi_normal\>](#to_edi_normal)|

### Editor Normal Mode

#### Mode Transition

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<Esc-Left>`, `<ctrl-]>`|[\<to_gui_normal\>](#to_gui_normal)|
|`<Esc-Down>`|[\<to_resident\>](#to_resident)|
|`:`|[\<to_command\>](#to_command)|
|`i`|[\<to_insert\>](#to_insert)|
|`v`|[\<to_edi_visual\>](#to_edi_visual)|
|`V`|[\<to_edi_visual_line\>](#to_edi_visual_line)|

#### Scrolling

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-y>`, `<C-k>`|[\<scroll_up\>](#scroll_up)|
|`<C-j>`, `<C-e>`|[\<scroll_down\>](#scroll_down)|
|`<C-u>`|[\<scroll_up_halfpage\>](#scroll_up_halfpage)|
|`<C-d>`|[\<scroll_down_halfpage\>](#scroll_down_halfpage)|
|`<C-b>`|[\<scroll_up_onepage\>](#scroll_up_onepage)|
|`<C-f>`|[\<scroll_down_onepage\>](#scroll_down_onepage)|
|`zh`, `<C-h>`|[\<scroll_left\>](#scroll_left)|
|`zl`, `<C-l>`|[\<scroll_right\>](#scroll_right)|
|`zH`|[\<scroll_left_halfpage\>](#scroll_left_halfpage)|
|`zL`|[\<scroll_right_halfpage\>](#scroll_right_halfpage)|

#### Shortcut

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-r>`|[\<redo\>](#redo)|
|`U`, `u`|[\<undo\>](#undo)|
|`gT`|[\<switch_to_left_tab\>](#switch_to_left_tab)|
|`gt`|[\<switch_to_right_tab\>](#switch_to_right_tab)|
|`/`, `?`|[\<search_pattern\>](#search_pattern)|

#### Mode Transition on Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`I`, `gI`|[\<to_insert_BOL\>](#to_insert_bol)|
|`a`|[\<to_insert_append\>](#to_insert_append)|
|`A`|[\<to_insert_EOL\>](#to_insert_eol)|
|`o`|[\<to_insert_nlbelow\>](#to_insert_nlbelow)|
|`O`|[\<to_insert_nlabove\>](#to_insert_nlabove)|

#### Caret Movement on Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<BS>`, `<C-h>`, `<Left>`, `h`|[\<move_caret_left\>](#move_caret_left)|
|`l`, `<Space>`, `<Right>`|[\<move_caret_right\>](#move_caret_right)|
|`gk`, `<C-p>`, `<Up>`, `-`, `k`|[\<move_caret_up\>](#move_caret_up)|
|`j`, `gj`, `<C-n>`, `<Down>`, `<Enter>`, `+`, `<C-m>`|[\<move_caret_down\>](#move_caret_down)|
|`w`|[\<move_fwd_word\>](#move_fwd_word)|
|`b`|[\<move_bck_word\>](#move_bck_word)|
|`W`|[\<move_fwd_bigword\>](#move_fwd_bigword)|
|`B`|[\<move_bck_bigword\>](#move_bck_bigword)|
|`e`|[\<move_end_word\>](#move_end_word)|
|`E`|[\<move_end_bigword\>](#move_end_bigword)|
|`ge`|[\<move_bckend_word\>](#move_bckend_word)|
|`gE`|[\<move_bckend_bigword\>](#move_bckend_bigword)|

#### Caret Jumping on Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`0`, `g0`, `<Home>`|[\<jump_caret_to_BOL\>](#jump_caret_to_bol)|
|`$`, `g$`, `<End>`|[\<jump_caret_to_EOL\>](#jump_caret_to_eol)|
|`gg`|[\<jump_caret_to_BOF\>](#jump_caret_to_bof)|
|`G`|[\<jump_caret_to_EOF\>](#jump_caret_to_eof)|

#### Edit on Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`yy`, `Y`|[\<yank_line\>](#yank_line)|
|`y`|[\<yank_with_motion\>](#yank_with_motion)|
|`p`|[\](#put_after)|
|`P`|[\](#put_before)|
|`dd`|[\<delete_line\>](#delete_line)|
|`D`|[\<delete_line_until_EOL\>](#delete_line_until_eol)|
|`x`, `<Del>`|[\<delete_after\>](#delete_after)|
|`X`|[\<delete_before\>](#delete_before)|
|`J`|[\<join_next_line\>](#join_next_line)|
|`r`|[\<replace_char\>](#replace_char)|
|`R`|[\<replace_sequence\>](#replace_sequence)|
|`~`|[\<switch_char_case\>](#switch_char_case)|
|`d`|[\<delete_with_motion\>](#delete_with_motion)|
|`c`|[\<change_with_motion\>](#change_with_motion)|
|`S`, `cc`|[\<change_line\>](#change_line)|
|`s`|[\<change_char\>](#change_char)|
|`C`|[\<change_until_EOL\>](#change_until_eol)|
|`.`|[\<repeat_last_change\>](#repeat_last_change)|

### Editor Visual Mode

#### Mode Transisiton

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<Esc-Left>`, `<ctrl-]>`|[\<to_gui_normal\>](#to_gui_normal)|
|`<Esc-Down>`|[\<to_resident\>](#to_resident)|
|`:`|[\<to_command\>](#to_command)|
|`<ctrl-[>`, `<Esc-Right>`|[\<to_edi_normal\>](#to_edi_normal)|

#### Scrolling

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-y>`, `<C-k>`|[\<scroll_up\>](#scroll_up)|
|`<C-j>`, `<C-e>`|[\<scroll_down\>](#scroll_down)|
|`<C-u>`|[\<scroll_up_halfpage\>](#scroll_up_halfpage)|
|`<C-d>`|[\<scroll_down_halfpage\>](#scroll_down_halfpage)|
|`<C-b>`|[\<scroll_up_onepage\>](#scroll_up_onepage)|
|`<C-f>`|[\<scroll_down_onepage\>](#scroll_down_onepage)|
|`zh`, `<C-h>`|[\<scroll_left\>](#scroll_left)|
|`zl`, `<C-l>`|[\<scroll_right\>](#scroll_right)|
|`zH`|[\<scroll_left_halfpage\>](#scroll_left_halfpage)|
|`zL`|[\<scroll_right_halfpage\>](#scroll_right_halfpage)|

#### Caret Movement on Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<BS>`, `<C-h>`, `<Left>`, `h`|[\<move_caret_left\>](#move_caret_left)|
|`l`, `<Space>`, `<Right>`|[\<move_caret_right\>](#move_caret_right)|
|`gk`, `<C-p>`, `<Up>`, `-`, `k`|[\<move_caret_up\>](#move_caret_up)|
|`j`, `gj`, `<C-n>`, `<Down>`, `<Enter>`, `+`, `<C-m>`|[\<move_caret_down\>](#move_caret_down)|
|`w`|[\<move_fwd_word_simple\>](#move_fwd_word_simple)|
|`b`|[\<move_bck_word_simple\>](#move_bck_word_simple)|

#### Caret Jumping on Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`0`, `g0`, `<Home>`|[\<jump_caret_to_BOL\>](#jump_caret_to_bol)|
|`$`, `g$`, `<End>`|[\<jump_caret_to_EOL\>](#jump_caret_to_eol)|
|`gg`|[\<jump_caret_to_BOF\>](#jump_caret_to_bof)|
|`G`|[\<jump_caret_to_EOF\>](#jump_caret_to_eof)|

#### Edit on Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`y`|[\<yank_highlight_text\>](#yank_highlight_text)|
|`X`, `d`, `x`|[\<delete_highlight_text\>](#delete_highlight_text)|
|`c`, `S`, `s`|[\<change_highlight_text\>](#change_highlight_text)|

### Insert Mode

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<ctrl-[>`, `<Esc-Right>`|[\<to_edi_normal\>](#to_edi_normal)|

### Resident Mode

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<Esc-Right>`|[\<to_edi_normal\>](#to_edi_normal)|

### Command Mode

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`edinormal`, `en`|[\<to_edi_normal\>](#to_edi_normal)|
|`ev`, `edivisual`|[\<to_edi_visual\>](#to_edi_visual)|
|`evl`, `edivisualline`|[\<to_edi_visual_line\>](#to_edi_visual_line)|

#### Shortcut

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`w`|[\<save\>](#save)|

#### Vim Emulation

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`-`|[\<move_caret_up\>](#move_caret_up)|
|`+`|[\<move_caret_down\>](#move_caret_down)|
|`<num>`|[\<jump_caret_to_BOF\>](#jump_caret_to_bof)|

**Tier `big`**

### GUI Normal Mode

#### Mode Transition

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`v`|[\<to_gui_visual\>](#to_gui_visual)|

#### Hotkey

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`V`|[\<select_all\>](#select_all)|
|`y`, `yy`, `Y`|[\<hotkey_copy\>](#hotkey_copy)|
|`P`, `p`|[\<hotkey_paste\>](#hotkey_paste)|
|`D`, `dd`|[\<hotkey_cut\>](#hotkey_cut)|
|`x`, `<Del>`|[\<hotkey_delete\>](#hotkey_delete)|
|`X`|[\<hotkey_backspace\>](#hotkey_backspace)|
|`<C-r>`|[\<redo\>](#redo)|
|`U`, `u`|[\<undo\>](#undo)|
|`<C-v>h`|[\<switch_to_left_vdesktop\>](#switch_to_left_vdesktop)|
|`<C-v>l`|[\<switch_to_right_vdesktop\>](#switch_to_right_vdesktop)|
|`<C-v>n`|[\<create_new_vdesktop\>](#create_new_vdesktop)|
|`<C-v>q`|[\<close_current_vdesktop\>](#close_current_vdesktop)|
|`<C-v>s`|[\<taskview\>](#taskview)|
|`gT`|[\<switch_to_left_tab\>](#switch_to_left_tab)|
|`gt`|[\<switch_to_right_tab\>](#switch_to_right_tab)|
|`/`, `?`|[\<search_pattern\>](#search_pattern)|
|`<gt>`|[\<goto_next_page\>](#goto_next_page)|
|`<lt>`|[\<goto_prev_page\>](#goto_prev_page)|
|`<win>`|[\<open_startmenu\>](#open_startmenu)|

### GUI Visual Mode

#### Mode Transition

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<Esc-Left>`, `<ctrl-]>`|[\<to_gui_normal\>](#to_gui_normal)|
|`<Esc-Down>`|[\<to_resident\>](#to_resident)|

#### Mouse Movement

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<BS>`, `<Left>`, `h`|[\<move_cursor_left\>](#move_cursor_left)|
|`l`, `<Space>`, `<Right>`|[\<move_cursor_right\>](#move_cursor_right)|
|`<Up>`, `-`, `k`|[\<move_cursor_up\>](#move_cursor_up)|
|`+`, `j`, `<Down>`|[\<move_cursor_down\>](#move_cursor_down)|

#### Mouse Jumping

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`0`, `<Home>`, `^`|[\<jump_cursor_to_left\>](#jump_cursor_to_left)|
|`$`, `<End>`|[\<jump_cursor_to_right\>](#jump_cursor_to_right)|
|`gg`|[\<jump_cursor_to_top\>](#jump_cursor_to_top)|
|`G`|[\<jump_cursor_to_bottom\>](#jump_cursor_to_bottom)|
|`gm`|[\<jump_cursor_to_hcenter\>](#jump_cursor_to_hcenter)|
|`M`|[\<jump_cursor_to_vcenter\>](#jump_cursor_to_vcenter)|

#### Mouse Scrolling

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`<C-y>`, `<C-k>`|[\<scroll_up\>](#scroll_up)|
|`<C-j>`, `<C-e>`|[\<scroll_down\>](#scroll_down)|
|`<C-u>`|[\<scroll_up_halfpage\>](#scroll_up_halfpage)|
|`<C-d>`|[\<scroll_down_halfpage\>](#scroll_down_halfpage)|
|`<C-b>`|[\<scroll_up_onepage\>](#scroll_up_onepage)|
|`<C-f>`|[\<scroll_down_onepage\>](#scroll_down_onepage)|
|`zh`, `<C-h>`|[\<scroll_left\>](#scroll_left)|
|`zl`, `<C-l>`|[\<scroll_right\>](#scroll_right)|
|`zH`|[\<scroll_left_halfpage\>](#scroll_left_halfpage)|
|`zL`|[\<scroll_right_halfpage\>](#scroll_right_halfpage)|

#### Hotkey

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`y`, `yy`, `Y`|[\<hotkey_copy\>](#hotkey_copy)|
|`P`, `p`|[\<hotkey_paste\>](#hotkey_paste)|
|`D`, `dd`|[\<hotkey_cut\>](#hotkey_cut)|
|`x`, `<Del>`|[\<hotkey_delete\>](#hotkey_delete)|
|`X`|[\<hotkey_backspace\>](#hotkey_backspace)|

### Command Mode

#### Mode Transition

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`gv`, `guivisual`|[\<to_gui_visual\>](#to_gui_visual)|

#### Shortcut

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`vdprev`|[\<switch_to_left_vdesktop\>](#switch_to_left_vdesktop)|
|`vdnext`|[\<switch_to_right_vdesktop\>](#switch_to_right_vdesktop)|
|`closev`|[\<close_current_vdesktop\>](#close_current_vdesktop)|
|`tv`, `taskview`, `vdesktop<space>list`|[\<taskview\>](#taskview)|
|`tabprevious`|[\<switch_to_left_tab\>](#switch_to_left_tab)|
|`tabnext`|[\<switch_to_right_tab\>](#switch_to_right_tab)|
|`tabnew`|[\<open_new_tab\>](#open_new_tab)|
|`tabclose`, `q`, `q!`|[\<close_current_tab\>](#close_current_tab)|
|`ex`, `explorer`|[\<start_explorer\>](#start_explorer)|
|`start`, `win`|[\<open_startmenu\>](#open_startmenu)|
|`find`, `open`|[\<open\>](#open)|
|`forward`|[\<forward_ui_navigation\>](#forward_ui_navigation)|
|`backward`|[\<backward_ui_navigation\>](#backward_ui_navigation)|
|`decide`|[\<decide_focused_ui_object\>](#decide_focused_ui_object)|

**Tier `huge`**

### Command Mode

|**Trigger Commands**|**Called Commands**|
|:---:|:---:|
|`md`, `mkdir`|[\<makedir\>](#makedir)|

## Apéndice E — Guía de migración (v4.x → v5.x)

### from <= 4.4.0 to 5.0.0

> **Where v4 documents go?**
> To keep simplicity, the document pages describes only for latest version.
> Therefore you cannot read web page style documents of conventional win-vind, but you can read old documents in markdown preview style in GitHub.  
> For v4.3.3, please refer to [this documents](https://github.com/pit-ray/win-vind/blob/v4.3.3/docs/cheat_sheet/index.md).

#### 1. Syntax of .vindrc
**Discussed in [#96](https://github.com/pit-ray/win-vind/discussions/96)**
##### The difference between map and noremap
The conventional `map` and `noremap` have different purposes. The map is designed to propagate defined macros to other applications except for win-vind, whereas the noremap effects in win-vind score only.

However, the **NEW** `map` and the **NEW** `noremap` have similar features of Vim and are separated on whether allow remapping like Vim.

Specifically, the `map` allows remapping with user-defined mapping like the following.
```vim
nmap f h  " f --> h
nmap t f  " t --> h
```
The noremap performs only the default map.
```vim
nnoremap f h  " f --> h
nnoremap t f  " t --> f
```

##### Arguments of map/noremap
The command kind is the same, but the way to interpret the arguments.

|**Command** | **Syntax**|
|:--- |:---|
|noremap|`noremap [trigger-cmd] [target-cmd]`|
|map|`map [trigger-cmd] [target-cmd]`|
|unmap|`unmap [trigger-cmd]`|
|mapclear|`mapclear`|
|command|`command [trigger-cmd] [targer-cmd]`|
|delcommand|`delcommand [trigger-cmd] [target-cmd]`|
|comclear|`comclear`|

Note: The syntax is not included `[` and `]`.

The **[trigger-cmd]** is assumed to key typing only and the **[target-cmd]** has three types of intepretation.

|**Type of [target-cmd]**|**Example**|**Notes**|
|:---|:---|:---|
|Function Name|`map FF <easy_click_left>`|Calls the pre-defined functions.|
|Internal Macro|`map XX a<ctrl-f>bcd`|Generate macros inside the internal scope of win-vind. This feature uses to define some shortcuts to a function or some combined mapping consisting of multiple pre-defined functions.|
|External Macro|`map g {This text is inserted}`|Define macros that are propagated outside of win-vind by enclosing them in `{` and `}`. This emulates the action of the user pressing the keyboard itself, and a single key to single key mapping (e.g. `map a {b}`) is the most efficient low-level mapping done.|

These **[target-cmd]** can be incorporated into a single map as follows.
```vim
map g <easy_click_left>b{This text is inserted}<switch_window>hh<cr>
```
The mapping represents a macro that is triggered by `g`, activates easy_click, jumps to the position of the hint in `b`, enters the string "This text is inserted", and then selects the two-left window with switch_window.

This version will preform an optimization process that merges several maps into one map, unless it contains a command to change the mode. For example, the following mappings will be merged into one.

* Raw map
```vim
nmap b h
nmap o b
nmap p o
````
* Optimized map (**THIS VERSION**)
```vim
nmap p h
```

Below are some examples of use.
1. Define mode change mapping
   ```vim
   imap <win-[> <to_edi_normal>
   imap <win-]> <to_gui_normal>
   ``` 
1. Text input macros
   ```vim
   nmap mail {win-vind@example.com}
   ```
1. Web page launcher
   ```vim
   nmap <ctrl-1> :execute https://example.com<cr>
   ```
1. Application launcher
   ```vim
   nmap <ctrl-2> :! notepad<cr>
   ```
1. Copy the current line to the bottom line as in Vim.
   ```vim
   enmap t yyGp
   ```

##### Mode Prefix
**Mentioned in [#91](https://github.com/pit-ray/win-vind/issues/91)**  

We received many requests to register maps across several modes, so we added batch-mapping with the same grouping as in Vim. 

You can mode prefix to specify modes.

|**Prefix**|**Mode**|
|:---:|:---|
| ` ` |GUI Normal, GUI Visual, Edi Normal, Edi Visual|
|`g` | GUI Normal, GUI Visual|
|`e` | Edi Normal, Edi Visual|
|`n` | GUI Normal, Edi Normal|
|`v` | GUI Visual, Edi Visual|
|`gn` | GUI Normal|
|`gv` | GUI Visual|
|`en` | Edi Normal|
|`ev` | Edi Visual|
|`i` | Insert Mode|
|`r` | Resident Mode|
|`c` | Command Mode|

However, external macros in `cmap` and `cnoremap` are not input to other applications and behave the same as internal macros.

#### 2. Self-Mapping
**Discussed in [#123](https://github.com/pit-ray/win-vind/issues/123)**  

You can disable the absorption of some keys and allow them to be input, as in `map <alt> {<alt>}`. However, this is only valid for a single key.
If the target command consists of multiple characters like `map g abcgd` and contains a trigger command, the following warning statement will be printed to log and no mapping will be done.

```

[Warning] Some part of the command generated from mapping `g * :e https://google.com<return>` was ignored to avoid an infinite loop because it was mapped to itself by mapping `g * :e https://google.com<return>`. If you wish to enter the generated command as is, enclose it in `{}`.

```

#### 3. Changed default mapping
**Discussed in [#118](https://github.com/pit-ray/win-vind/discussions/118)**  

Since the mode transition combined with ESC in win-vind was not well received, we adopted the same command as in Vim.

|**Type of map**|**Function ID** |**Conventional trigger of map** |**New trigger of map**|
|:---:|:---:|:---:|:---:|
|imap | `to_gui_normal` | `<Esc-Left>` | `<Ctrl-]>`|
|imap | `to_edi_normal` | `<Esc-Right>` | `<Ctrl-[>`|

#### 4. Renamed  function name
The conventional `<syscmd_*>` function names are renamed to [simple ones](https://github.com/pit-ray/win-vind/blob/9ec52bb02b2e74784dd347ce259abb936b28d9fe/src/bind/mapdefault.cpp#L466-L522).

#### 5. Eliminated options and replacements
The following is the correspondence between the options that were removed and their replacements. The `-` is completely obsolete.

|**Eliminated options** | **Replacements**|
|:---: | :---:|
|`window_accel` | `window_velocity`|
|`window_tweight` | `window_velocity`|
|`window_maxv` | `window_velocity`|
|`cursor_tweight` | `cursor_resolution`|
|`cursor_maxv` | `-`|
|`cmd_maxchar` | `-`|
|`cmd_maxhist` | `-`|

Details of the new options are as follows.

|**New options**|**Notes**|
|:---:|:---|
|`window_velocity` | Pixel-level velocity in the constatnt acceleration motion of the window in winresizer.|
|`cursor_resolution` | A weight for scaling the time of constant acceleration motion of the mouse cursor.|
|`listen_interval`|The time interval in seconds at which the server win-vind will retrieve command requests sent by the client with the `-c` argument in terminal. ([#112](https://github.com/pit-ray/win-vind/issues/112))|

#### 6. New word-motion

Add the following word-motion which behave almost exactly like Vim. ([#57](https://github.com/pit-ray/win-vind/issues/57), [#75](https://github.com/pit-ray/win-vind/pull/75))

  |**ID**|**Feature**|**Emulation**|
  |:---:|:---|:---:|
  |**move_fwd_word**|words forward for normal mode.|`w`|
  |**move_fwd_word_simple**|words forward fast.|`w`|
  |**move_bck_word**|words backward for normal mode.|`b`|
  |**move_bck_word_simple**|words backward fast.|`b`|
  |**move_fwd_bigword**|WORDS forward.|`W`|
  |**move_bck_bigword**|WORDS backward.|`B`|
  |**move_end_word**|Forward to the end of words.|`e`|
  |**move_end_bigword**|Forward to the end of WORDS.|`E`|
  |**move_bckend_word**|Backward to the end of words.|`ge`|
  |**move_bckend_bigword**|Backward to the end of WORDS.|`gE`|

  These functions do not work in visual mode except for `w` and `b`, because they copy the text once and retrieve the text via the clipboard. `iskeyword` option is fixed to the default value of Vim in Windows and cannot change it currently.  
  There is an option `charbreak` to set the criteria for considering a Unicode character as a single character. 

  |ID|Type|Default|Note|
  |:---:|:---:|:---:|:---|
  |`charbreak`|str|grapheme|Mode for how to split a single Unicode character. The `grapheme` mode treats a combination character as a single character. The `codepoint` mode processes the combination character for each codepoint.|

#### 7. New option in terminal
**Discussed in [#101](https://github.com/pit-ray/win-vind/discussions/101), [#97](https://github.com/pit-ray/win-vind/discussions/97)**  
Please see this document.

## Apéndice F — Compilar, testear y desarrollar

#### Quick Start for Build  
If you have already installed **MinGW-w64** or **Visual Studio**, all you need is the next steps.  

##### Visual Studio
  ```bash
  $ cmake -B build -DCMAKE_BUILD_TYPE=Debug -G "Visual Studio 17 2022" -A x64 .
  $ cmake --build build --config Debug
  $ ./build/Debug/win-vind.exe
  ```

##### MinGW-w64 >= 8.2.0
  ```bash
  $ cmake -B build -DCMAKE_BUILD_TYPE=Debug -G "MinGW Makefiles" .
  $ cmake --build build --config Debug
  $ ./build/win-vind.exe
  ```

#### Run Test 
See [here](tests/README.md) for unit tests and runtime test.

#### Make Installer
```bash
$ ./tools/create_assets.bat 1.0.0 -msvc 64
```

## Dependencies

### Softwares
I recommend to install follow softwares.

|Name|Recommended Version|Download Link|
|:---:|:---:|:---:|
|CMake|3.14.4|<a href="https://cmake.org/download/">Download - CMake</a>|
|NSIS|3.06.1|<a href="https://nsis.sourceforge.io/Download">Download - NSIS</a>|
|Windows10 SDK|10.0.19041.0|<a href="https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk/">Microsoft Windows10 SDK - Windows app development</a>|

### Libraries
These libraries are bundled in the libs directory.

|**Name**|**What is**|**Purpose**|**License**|
|:---:|:---:|:---:|:---:|
|[fluent-tray](https://github.com/pit-ray/fluent-tray)|GUI framework|Create GUI for the system tray or popups.|[MIT License](https://github.com/pit-ray/fluent-tray/blob/main/LICENSE.txt)|
|[argparse](https://github.com/p-ranav/argparse)|Argument Parser|Parse arguments in the command line.|[MIT License](https://github.com/p-ranav/argparse/blob/master/LICENSE)|
|[doctest](https://github.com/onqtam/doctest)|Unit test framework|For basic unit test|[MIT License](https://github.com/onqtam/doctest/blob/master/LICENSE.txt)|
|[fff](https://github.com/meekrosoft/fff)|Macro-based fake function framework|To mock Windows API|[MIT License](https://github.com/meekrosoft/fff/blob/master/LICENSE)|
|[pydirectinput](https://github.com/learncodebygaming/pydirectinput)|Mouse and keyboard automation for Windows|To emulate inputs for runtime tests|[MIT License](https://github.com/learncodebygaming/pydirectinput/blob/master/LICENSE.txt)|

### Tests

## win-vind Test
**In this documents, we assume executing commands in tests directory not in the project root.**

### Unit Test
It are run using CTest at compile time. This is based on branch coverage.

#### Visual Studio 2019
```bash
$ cmake -B build_msvc -G "Visual Studio 17 2022" unit
$ cmake --build build_msvc
$ ctest -C Debug --test-dir build_msvc --output-on-failure
```

#### MinGW-w64 >= GCC 11.2.0
```bash
$ cmake -B build_mingw -G "MinGW Makefiles" unit
$ cmake --build build_mingw
$ ctest -C Debug --test-dir build_mingw --output-on-failure
```

### Runtime Test
This tool uses to test for integration.
Specify the path of executable file after built win-vind, and then this tools generate proper keystrokes to check the win-vind behavior. However, It enumlates user inputs actually, so may destroy other applications that are currently running. For this, it should be run in a virtual environment such as VirtualBox.

#### Requirements
The runtime test is implemented in python scripts.

#### Run Test
```bash
$ python runtime/test.py "../bin_64/win-vind/win-vind.exe"
```

## Apéndice G — Dependencias, librerías y estructura del proyecto

### Dependencies

#### Softwares
I recommend to install follow softwares.

|Name|Recommended Version|Download Link|
|:---:|:---:|:---:|
|CMake|3.14.4|<a href="https://cmake.org/download/">Download - CMake</a>|
|NSIS|3.06.1|<a href="https://nsis.sourceforge.io/Download">Download - NSIS</a>|
|Windows10 SDK|10.0.19041.0|<a href="https://developer.microsoft.com/en-us/windows/downloads/windows-10-sdk/">Microsoft Windows10 SDK - Windows app development</a>|

#### Libraries
These libraries are bundled in the libs directory.

|**Name**|**What is**|**Purpose**|**License**|
|:---:|:---:|:---:|:---:|
|[fluent-tray](https://github.com/pit-ray/fluent-tray)|GUI framework|Create GUI for the system tray or popups.|[MIT License](https://github.com/pit-ray/fluent-tray/blob/main/LICENSE.txt)|
|[argparse](https://github.com/p-ranav/argparse)|Argument Parser|Parse arguments in the command line.|[MIT License](https://github.com/p-ranav/argparse/blob/master/LICENSE)|
|[doctest](https://github.com/onqtam/doctest)|Unit test framework|For basic unit test|[MIT License](https://github.com/onqtam/doctest/blob/master/LICENSE.txt)|
|[fff](https://github.com/meekrosoft/fff)|Macro-based fake function framework|To mock Windows API|[MIT License](https://github.com/meekrosoft/fff/blob/master/LICENSE)|
|[pydirectinput](https://github.com/learncodebygaming/pydirectinput)|Mouse and keyboard automation for Windows|To emulate inputs for runtime tests|[MIT License](https://github.com/learncodebygaming/pydirectinput/blob/master/LICENSE.txt)|

### Estructura del proyecto

```
win-vind/
├─ CMakeLists.txt          # definición de build (project VERSION 5.13.2)
├─ CONTRIBUTING.md         # guía de contribución + build
├─ README.md               # presentación
├─ docs/                   # sitio de documentación (Jekyll)
│  ├─ usage/               #   guía de uso (fuente de Parte 1-3 de este doc)
│  ├─ cheat_sheet/         #   referencia: functions/ options/ keywords/ defaults/
│  ├─ defaults/            #   mapeos por defecto por tier
│  ├─ migration/           #   guía de migración v4 -> v5
│  ├─ faq/  downloads/  ja/ (traducción japonés)
│  └─ _config.yml, _layouts, _sass, assets, imgs
├─ libs/                   # librerías empaquetadas (header-only)
│  ├─ argparse/            #   parser de argumentos CLI
│  ├─ doctest/             #   framework de unit tests
│  ├─ fluent_tray/         #   GUI para system tray / popups
│  └─ meekrosoft/          #   fff: mocking de la API de Windows
├─ res/                    # recursos y assets de build
│  ├─ build_assets/  installer/  resources/
│  └─ icon.png
├─ src/                    # código fuente C++
│  ├─ main.cpp             #   entrada (parseo CLI, arranque servidor/cliente)
│  ├─ bind/                #   bindings, mapeos por defecto, autocmd, macros
│  ├─ core/                #   núcleo (modos, input, emulación Vim, ventanas)
│  ├─ opt/                 #   opciones de configuración (set ...)
│  └─ util/                #   utilidades (winwrap, strings, etc.)
├─ tests/                  # tests
│  ├─ unit/                #   tests unitarios (CTest, coverage de ramas)
│  └─ runtime/             #   tests de integración en Python (emula input real)
├─ tools/                  # scripts de build/empaquetado (.bat) y utilidades Python
│  ├─ build.bat  create_bin.bat  create_assets.bat
│  ├─ generate_cheatsheet.py  translate.py  json2rc.py
│  └─ install_openssl.bat  push_chocolatey_package.bat ...
└─ how-to-use.md           # ESTE documento
```

| Directorio / archivo | Qué es |
| :--- | :--- |
| `src/bind/` | Definición de mapas, mapeos por defecto, `autocmd`, resolución de macros. |
| `src/core/` | Núcleo: máquina de modos, absorción de input, emulación Vim, control de ventanas/ratón. |
| `src/opt/` | Implementación de todas las opciones accesibles con `set`. |
| `src/util/` | Wrappers de la API de Windows y utilidades (tests incluidos). |
| `libs/` | Dependencias header-only empaquetadas (no requieren instalación aparte). |
| `docs/` | Fuente del sitio web; **este documento se derivó de aquí**. |
| `tools/` | Scripts `.bat` de build/instaladores y utilidades Python (cheatsheet, traducciones). |
| `tests/unit/` | CTest basado en cobertura de ramas. |
| `tests/runtime/` | Integración que emula pulsaciones reales (¡ejecutar en VM!). |
