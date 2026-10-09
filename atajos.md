# Atajos personalizados de win-vind

> Documento de **tu** configuración personal (no de los atajos por defecto de win-vind).
> Para los atajos y funciones por defecto del programa, ver `how-to-use.md` en el repo.

- Config: `C:\Users\e4dev\.win-vind\.vindrc`
- Backup: `C:\Users\e4dev\.win-vind\.vindrc.bak`
- Recargar tras editar: abrir con `:e` y recargar con `:source` (o reiniciar win-vind).

---

## Leader key: `Ctrl+Win`

Pulsa **Ctrl+Win** (los dos juntos), suéltalos, y luego la **tecla de la acción**.

Funciona en los modos **GUI Normal / GUI Visual** y **Edi Normal / Edi Visual**
(los modos donde win-vind absorbe el teclado). **No** funciona mientras escribes en **Insert**.

| Leader + tecla | Acción                          | Mapeo en `.vindrc`                          |
| :------------: | :------------------------------ | :------------------------------------------ |
| `Ctrl+Win f`   | EasyClick (clic por hints)      | `noremap <ctrl-win>f <easyclick><click_left>` |
| `Ctrl+Win g`   | GridMove (cursor por cuadrícula)| `noremap <ctrl-win>g <gridmove>`            |
| `Ctrl+Win w`   | Cambiar de ventana              | `noremap <ctrl-win>w <switch_window>`       |
| `Ctrl+Win m`   | Maximizar ventana               | `noremap <ctrl-win>m <maximize_current_window>` |
| `Ctrl+Win n`   | Minimizar ventana               | `noremap <ctrl-win>n <minimize_current_window>` |
| `Ctrl+Win q`   | Cerrar ventana (Alt+F4)         | `noremap <ctrl-win>q <close_current_window>` |
| `Ctrl+Win y`   | Seleccionar todo + copiar       | `noremap <ctrl-win>y <select_all><hotkey_copy>` |
| `Ctrl+Win p`   | Pegar                           | `noremap <ctrl-win>p <hotkey_paste>`        |
| `Ctrl+Win x`   | Cortar                          | `noremap <ctrl-win>x <hotkey_cut>`          |
| `Ctrl+Win l`   | Abrir gvim                       | `noremap <ctrl-win>l :! gvim<cr>`           |
| `Ctrl+Win e`   | Abrir web (Google)              | `noremap <ctrl-win>e :e https://www.google.com<cr>` |
| `Ctrl+Win t`   | Abrir Explorador                | `noremap <ctrl-win>t <start_explorer>`      |
| `Ctrl+Win r`   | Pausar win-vind (resident)      | `noremap <ctrl-win>r <to_resident>`         |

---

### Cómo se usa EasyClick (`Ctrl+Win f`)

Al pulsarlo aparecen **etiquetas de LETRAS** sobre los elementos clicables de la ventana activa.

- **Escribe la letra (o letras) que ves** encima del objetivo → el cursor salta ahí y hace clic.
- Si hay **muchos** elementos, las etiquetas son de **2 letras**: escribe las dos (una detrás de otra).
- `Backspace` borra la última letra escrita.
- **`Enter` y `Esc` NO confirman: CANCELAN** los hints. ← por eso "no se abre" si pulsas Enter.
- Las letras disponibles salen de la opción `hintkeys` (por defecto: `asdghklqwertyuiopzxcvbnmfj`).

---

## Otros atajos configurados

| Atajo                        | Modo         | Acción                                  |
| :--------------------------- | :----------- | :-------------------------------------- |
| `CapsLock`                   | Insert       | Se comporta como `Ctrl`                 |
| `Ctrl+Shift+f`               | Insert       | EasyClick + clic izquierdo              |
| `Ctrl+Shift+m`               | Insert       | GridMove + clic izquierdo               |
| `Ctrl+Shift+s`               | Insert       | Cambiar ventana + EasyClick + clic izq. |
| `Ctrl+1`                     | Normal/Visual| Abrir gvim                              |
| `Ctrl+2`                     | Normal/Visual| Abrir `http://example.com`              |
| `t`                          | Edi Normal   | Copiar la línea actual al final (`ggyyGp`) |

---

## Hacer clic con el puntero

Tras mover el puntero (con `Ctrl+Win g` = GridMove, o con `h/j/k/l`), así se hace clic.
En win-vind **no hay un "Enter que abre" aparte**: el clic ES la acción que abre/activa.

| Tecla (en GUI Normal) | Acción                                             |
| :-------------------: | :------------------------------------------------- |
| `Enter` (`<cr>`)      | Clic izquierdo  ← lo añadimos para que sea intuitivo |
| `o`                   | Clic izquierdo (por defecto)                       |
| `a`                   | Clic derecho (por defecto)                         |
| `i` / `I`             | Clic izquierdo + entrar a editar el campo de texto |

> Los iconos del escritorio necesitan **doble clic** para abrirse. Si quieres, cambio
> `Enter` a `<click_left><click_left>` (doble clic) o le pongo una tecla aparte.

---

## Modos: cómo entrar / salir

| Acción                    | Teclas                          |
| :------------------------ | :------------------------------ |
| GUI Normal                | `<ctrl-]>` o `<esc-left>`       |
| Edi Normal                | `<ctrl-[>` o `<esc-right>`      |
| Insert (escribir)         | `i`                             |
| Resident (pausar)         | `<esc-down>` o `Ctrl+Win r`     |
| Comando (`:`)             | `:`                             |
| Salir de win-vind         | `:exit` (o `<F8>+<F9>` forzado) |
| Salir de Command mode     | `<esc>` / `<ctrl-[>`            |

---

## Comandos automáticos (autocmd)

| Al...                                          | Hace              |
| :--------------------------------------------- | :---------------- |
| Salir de una aplicación (`AppLeave *`)         | Ir a Insert       |
| Entrar en `vim.exe` (`AppEnter,EdiNormalEnter`)| Ir a Resident     |

---

## Opciones activas

```vim
version normal
set shell = cmd
set cmd_fontsize = 14
set cmd_fontname = Consolas
set easyclick_bgcolor=E67E22
set easyclick_fontcolor=34495E
```

---

## Notas

- **Insert opcional:** si quieres usar el leader mientras escribes, descomenta en el `.vindrc`:
  `inoremap <ctrl-win> <to_instant_gui_normal>`
  Con eso `Ctrl+Win` entra un instante en GUI Normal para **una** tecla y vuelve a Insert.
  Ojo: esa tecla sigue los mapas de GUI Normal, **no** los `<ctrl-win>X`.
- **Reservado por Windows:** Ctrl+Win+D (escritorio nuevo), Ctrl+Win+←/→ (cambiar escritorio),
  Ctrl+Win+F4 (cerrar escritorio). Tus teclas (f, g, w, m, n, q, y, p, x, l, e, t, r) no chocan.
- **Sintaxis:** en `.vindrc`, todo lo que va tras una `"` sin cerrar es comentario. Un combo se
  escribe como `<ctrl-win>`, y una secuencia como `<ctrl-win>f` = "Ctrl+Win y luego f".
- Para añadir un atajo nuevo: añade una línea `noremap <ctrl-win>K <funcion>` en el bloque leader.

