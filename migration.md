# migration.md — Migración de win-vind a un stack propio

> Documento de decisión técnica. Objetivo: tener una herramienta tipo *key binder
> Vim para Windows* funcionando en una laptop corporativa, eligiendo lenguaje,
> arquitectura y "spec kit" (stack + scaffolding) antes de escribir código.

Fecha: 2026-10-09
Base analizada: `pit-ray/win-vind` v5.13.2 (fork local `C:\workspace\win-vim\win-vind`)

---

## 0. TL;DR (la decisión)

| Pregunta | Respuesta corta |
|---|---|
| ¿Electron? | **No para el core.** Sirve solo como UI de configuración opcional. |
| Lenguaje más apropiado | **C# / .NET 8+ (WinUI 3 o WPF)** — mejor balance esfuerzo/paridad. |
| Alternativa si quieres 1 binario real sin runtime | **Rust + `windows-rs`**. |
| Alternativa si ya dominas C++ | **C++ + WinUI 3 / Qt** (es casi reescribir win-vind). |
| Por qué NO Electron | El core necesita hooks Win32 de bajo nivel + COM/UIAutomation + `SendInput`; Node no puede hacerlo sin módulos nativos (C++), y ahí pierdes el punto de usar Electron. |
| Riesgo #1 en laptop corporativa | WDAC / Device Guard / EDR bloquean binarios sin firmar y marcan hooks globales de teclado como keylogger. |

---

## 1. Contexto: qué es y cómo está hecho win-vind hoy

win-vind es un **único binario C++17** (sin runtime, sin instalador pesado) que
engancha el teclado del sistema y traduce teclas a comandos sobre el GUI de Windows.

Stack real verificado en el repo:

- **Lenguaje / build**: C++17, CMake 3.6+, compila con MSVC (`/W4 /std:c++17 /MT`) y MinGW g++.
- **Tamaño**: ~26.5k LOC en `src/` (`src/core` 42 archivos, `src/bind` 138, `src/util` 40, `src/opt` 16).
- **Librerías enlazadas** (`src/CMakeLists.txt`): `psapi` (procesos), `dwmapi` (composición de ventanas), `userenv` (variables de entorno), `icuuc` (Unicode). Vendored: `argparse`, `doctest`, `fluent_tray`, `meekrosoft`.
- **Captura de teclado**: hook global de bajo nivel `SetWindowsHookEx(WH_KEYBOARD_LL, ...)` en `src/core/inputgate.cpp` (singleton `InputGate`). Es el corazón de todo.
- **Detección de elementos clicables (EasyClick)**: **UI Automation (COM)** — `IUIAutomation`, `IUIAutomationCacheRequest`, `IUIAutomationElementArray` en `src/util/uia.cpp`, `cuia.cpp`, `uiwalker.cpp`; etiquetas de pistas en `src/core/hintassign.cpp`.
- **Entrada sintética / mouse**: `SendInput` en `src/util/mouse.cpp`.
- **Overlay / HUD** (línea de comandos, cabecera, hints): **GDI** `TextOutW` en `src/util/screen_textrender.cpp`.
- **Tray**: `libs/fluent_tray` (Win32).
- **Modelo de modo / interpretación**: `src/core/inputhub.cpp`, `mode.cpp`, `mapsolver.cpp`, `rcparser.cpp` (parsea `.vindrc`), `background.cpp` (hilo de fondo), `keystroke_repeater`, `interval_timer`.
- **Modos**: GUI Normal/Visual, Edi Normal/Visual, Insert, Resident, Command.

### Consecuencia de diseño
Todo lo que hace "interesante" a win-vind vive **fuera** del alcance del JavaScript
de navegador/Electron: hooks de sistema, COM, `SendInput`, Win32 windos. Cualquier
stack que elijas para el **core** tiene que hablar Win32 nativo. La decisión del
lenguaje es, en la práctica, "¿en qué lenguaje escribo código Win32 con la menor
fricción?".

---

## 2. Los 4 problemas que definen la elección

1. **Hook global de teclado (bloqueante, tiempo real).** `WH_KEYBOARD_LL` debe
   responder en < 1 ms o el sistema descarta el hook. Necesita callback nativo.
2. **Automatización de UI.** Enumerar controles de otra app = COM/UIAutomation
   (o MSAA). Necesita interop COM.
3. **Inyección de entrada.** `SendInput` para simular teclas/mouse.
4. **Overlay sobre el escritorio.** HUD de comandos y hints. Se hace con GDI o
   Direct2D/DirectWrite, o con una ventana transparente top-most.

Además, requisitos de producto que el lenguaje debe permitir:
- **Binario único / portable** (fácil de copiar a la laptop corporativa).
- **Sin admin** para lo básico; firma opcional.
- **Arranque rápido**, poca RAM (es un daemon de fondo).

---

## 3. Comparativa de stacks

Puntuación: 🟢 excelente · 🟡 viable · 🔴 no recomendado para el core.

| Stack | Hook WH_KEYBOARD_LL | UIAutomation (COM) | SendInput | Overlay | Binario único | Esfuerzo total | Veredicto |
|---|---|---|---|---|---|---|---|
| **C# / .NET 8 + WinUI3/WPF** | 🟢 P/Invoke | 🟢 COM interop nativo (`UIAutomationClient`) | 🟢 P/Invoke | 🟢 WPF/Direct2D o `TextOutW` | 🟢 `PublishSingleFile`+`SelfContained` (≈60–80 MB) | **Bajo-medio** | ✅ **Recomendado** |
| **Rust + `windows-rs`** | 🟢 | 🟢 (crate `windows`) | 🟢 | 🟡 Direct2D o `tiny-skia` | 🟢 real (~5–15 MB, sin runtime) | Medio-alto | ✅ Alternativa sólida |
| **C++ + WinUI3 / Qt** | 🟢 | 🟢 | 🟢 | 🟢 | 🟢 | Alto (es win-vind otra vez) | 🟡 Si ya dominas C++ |
| **Go (`golang.org/x/sys/windows`)** | 🟡 cgo o `syscall` crudo, GC en callback del hook | 🟡 interop COM manual y verboso | 🟢 | 🟡 | 🟢 | Medio | 🟡 OK, COM doloroso |
| **Electron / Node** | 🔴 `globalShortcut` no es un hook; necesitas addon C++ | 🔴 requiere addon nativo (node-ffi/edge-js) | 🔴 addon | 🟢 (es su fuerte) | 🔴 ~150 MB+ | Alto + fragilidad ABI | ❌ No para el core |
| **Python (pywin32/cffi)** | 🟡 funciona pero GIL + latencia en el hook | 🟡 `comtypes`/`uiautomation` | 🟢 | 🟡 Tk/Qt | 🔴 PyInstaller grande | Bajo al inicio, frágil | 🟡 Solo prototipo |
| **AutoHotkey v2** | 🟢 (es su especialidad) | 🔴 UIA muy limitado | 🟢 | 🟡 GUI básica | 🟢 `.ahk`+runtime | Muy bajo | 🟡 Prototipo rápido, no producto |
| **Tauri (Rust + webview)** | 🔴 igual que Electron (necesita core nativo) | 🔴 | 🔴 | 🟢 webview | 🟡 | — | ❌ El webview es solo cosmético |

### Por qué Electron / Tauri no ganan aquí
Electron brilla cuando el 90% del valor es UI. En un key binder el 90% del valor
es **integración con el SO**, justo lo que Electron no puede hacer. Terminarías
escribiendo un addon C++ para el hook + COM para UIA, es decir, **igual harías el
core en nativo** pero cargando además un Chromium de 150 MB y un puente IPC/N-API
que es una fuente constante de bugs. Electron queda como capa **opcional** para
un panel de settings moderno (ver §6).

---

## 4. Recomendación principal: C# / .NET 8+ (WinUI 3 o WPF)

**Por qué C# gana para tu caso:**
- Interop Win32 y COM **de primera clase** (`DllImport`, `[ComImport]`,
  `LibraryImport` source-generated): cubre los 4 problemas sin C++.
- **UIAutomation** viene expuesta por .NET (`System.Windows.Automation` y
  `UIAutomationClient`/`UIAutomationTypes`), o vía `Interop.UIAutomationClient`.
- Daemon + tray + overlay con muy poco código; WPF dibuja el HUD y las hints
  con `TextBlock` en vez de pelear con GDI.
- **Productividad**: reproduces en semanas lo que en C++/Rust toma meses.
- Portabilidad: `dotnet publish -r win-x64 --self-contained -p:PublishSingleFile=true`
  → un `.exe` que copias a la laptop corporativa.

**Desventajas que debes aceptar:**
- No es "un binario de 2 MB": self-contained ≈ 60–80 MB (o más con Trim). Con
  framework-dependent cae a ~5 MB pero requiere el runtime .NET instalado
  (a veces bloqueado en laptops corporativas → usa self-contained).
- El callback del hook debe ser **estático y no asignar en el hot path**; mantén
  el trabajo fuera del callback (encola y procesa en tu hilo).
- P/Invoke mal hecho filtra handles/GC pressure; usa `[LibraryImport]` y firmas
  correctas.

### Cómo se ve cada problema en C#

```csharp
// 1) Hook global de teclado
[DllImport("user32.dll", SetLastError = true)]
static extern IntPtr SetWindowsHookEx(int idHook, LowLevelKeyboardProc lpfn, IntPtr hMod, uint dwThreadId);
const int WH_KEYBOARD_LL = 13;

// 2) UIAutomation — usar el assembly de .NET o Interop.UIAutomationClient
//    System.Windows.Automation.AutomationElement.RootElement.FindAll(...)
//    con TreeScope.Descendants y Condition por ControlType.IsInvokable

// 3) Entrada sintética
[DllImport("user32.dll")]
static extern uint SendInput(uint nInputs, INPUT[] pInputs, int cbSize);

// 4) Overlay: ventana WPF AllowsTransparency + Topmost, dibuja hints/command line
```

---

## 5. Alternativa: Rust + `windows-rs`

Si el criterio #1 es **un binario pequeño, sin runtime y sin admin**, Rust es la
mejor opción "de verdad":

- **Pro**: un solo `.exe` de ~5–15 MB sin runtime; seguridad de memoria en el
  callback del hook; crates `windows` (Win32 + COM + UIAutomation generados desde
  los metadatos oficiales de Microsoft), `windows-hooks`/`rdev` para referencia,
  `tray-icon`, `myshell`.
- **Contra**: más código para UIAutomation (COM manual), curva del borrow checker
  en estructuras de estado global (usa `OnceCell`/canales), y para el overlay
  tendrás que usar Direct2D o `tiny-skia` en vez de un toolkit de alto nivel.
- **Cuándo elegirlo**: si la laptop corporativa prohíbe instalar el .NET runtime
  y WDAC bloquea apps self-contained grandes; o si te importa el tamaño/portabilidad.

---

## 6. Arquitectura propuesta (aplicable a C# o Rust)

Proceso único, todo en memoria, sin red. Capas:

```
┌───────────────────────────────────────────────────────────┐
│  UI / Settings (WinUI 3 o Electron opcional, proceso aparte)│
│  edita config.json/.vindrc y llama al core por IPC local      │
└───────────────────────────▲───────────────────────────────┘
                            │ named pipe / local socket / CLI -c
┌───────────────────────────┴───────────────────────────────┐
│  Core (daemon, un binario)                                  │
│                                                             │
│  ┌─ Input Layer ────────────────────────────────────────┐  │
│  │ global keyboard hook (WH_KEYBOARD_LL) + mouse hook   │  │
│  │ enqueue KeyEvent (lock-free ring buffer)             │  │
│  └───────────────────────────┬──────────────────────────┘  │
│  ┌─ Engine / Interpreter ─────▼──────────────────────────┐  │
│  │ Parser (.vindrc) → Modes → MapSolver → Command tree   │  │
│  │ autocmd, repeat (. / counts), macro recording         │  │
│  └───────────────────────────┬──────────────────────────┘  │
│  ┌─ Platform Services ────────▼──────────────────────────┐  │
│  │ UIAutomation client (EasyClick)                       │  │
│  │ SendInput (keyboard/mouse)  ·  Window mgmt (DWM/user32)│  │
│  │ Process/window enumeration (psapi)                    │  │
│  └───────────────────────────┬──────────────────────────┘  │
│  ┌─ Output / Overlay ─────────▼──────────────────────────┐  │
│  │ HUD transparente top-most: command line, hints, mode  │  │
│  └───────────────────────────────────────────────────────┘  │
│  Tray icon + single-instance guard + autostart (HKCU Run)   │
└─────────────────────────────────────────────────────────────┘
```

### Modelo de hilos (crítico)
- **Hilo del hook**: solo captura y encola. NUNCA hace UIAutomation, I/O ni
  allocations pesadas (bloquearía el input del sistema).
- **Hilo del engine**: desencola `KeyEvent`, resuelve el mapa, despacha comandos.
- **Hilo UI (STA)**: COM/UIAutomation y el overlay **deben** correr en un hilo STA
  (como en win-vind con COM). Serializa ahí.
- **Guard de instancia única**: mutex nombrado; el segundo lanzamiento actúa como
  cliente (`mybind -c "<cmd>"`), igual que win-vind.

### Modelo de datos de configuración
- Migrar el `.vindrc` es lo que da continuidad: parser propio (PEG/regex) que
  produzca un AST de comandos. Guarda los **nombres de funciones tal cual**
  (`<switch_window>`, `<easyclick>`, `<click_left>`...) para no romper los
  configs del usuario — ver `atajos.md` y `how-to-use.md` en este repo.

---

## 7. Spec kit / Starter kit recomendado

### 7.1 Opción C# (.NET 8+)
- **Runtime/SDK**: .NET 8 (LTS) o .NET 9.
- **UI**: WinUI 3 (Windows App SDK) para settings; **para el overlay** un WPF
  `Window` con `AllowsTransparency=true`, `WindowStyle=None`, `Topmost=true`
  (más simple y probado que WinUI para HUD sin chrome).
- **Tray**: `H.NotifyIcon.WinUI` (o `Microsoft.Windows.SDK` `Shell_NotifyIcon`).
- **Autostart**: clave `HKCU\Software\Microsoft\Windows\CurrentVersion\Run`
  (sin admin) o Programador de tareas.
- **Interop**: `[LibraryImport]` (source generators) + `Interop.UIAutomationClient`
  (NuGet) para los tipos COM de UIA.
- **Single instance / IPC**: `Mutex` nombrado + `NamedPipeServerStream`.
- **Config**: `System.Text.Json` con `JsonSerializerContext` (AOT-friendly).
- **Tests**: `xUnit` + `FlaUI` (para pruebas de UIA) o `MSTest`.
- **Empaquetado**: `dotnet publish -c Release -r win-x64 --self-contained
  -p:PublishSingleFile=true -p:PublishTrimmed=true`.
- **Firma**: `signtool` con certificado EV/OV (necesario para evitar SmartScreen
  y para pasar políticas corporativas).

**Scaffolding:**
```bash
dotnet new winui3-app -n MyVindApp        # o: dotnet new wpf -n MyVindApp
# estructura:
MyVindApp/
  src/Core/            # daemon: hooks, engine, platform
    Input/             #   keyboards hook, ring buffer
    Engine/            #   parser, modes, mapsolver
    Platform/          #   uia, sendinput, window, process, tray
    Overlay/           #   HUD
  src/Settings/        # WinUI/WPF config panel (opcional, proceso aparte)
  tests/               # xUnit + FlaUI
  res/                 # iconos, assets, .vindrc de ejemplo
  MyVindApp.sln
```

### 7.2 Opción Rust
- **Crates**: `windows` (Win32 + COM + UIAutomation), `windows-sys` (FFI crudo),
  `tray-icon`, `tao`/`winit` (ventana overlay), `tiny-skia` o `windows::Win32::Graphics::Direct2D`
  (dibujo HUD), `serde`+`serde_json`, `notify` (hot-reload config), `thiserror`, `anyhow`.
- **Scaffolding**: `cargo new myvind --bin`, workspace con crates
  `core` (lib) → `daemon` (bin) → `settings` (bin, opcional) → `cli` (bin `-c`).
- **Empaquetado**: build `--release` (MSVC toolchain), luego `cargo-wix` o
  Inno Setup para el instalador.
- **Firma**: igual que C#, `signtool`.

### 7.3 Componentes transversales (los dos casos)
- **Cheatsheet / docs de configuración** versionados junto al binario.
- **CI**: GitHub Actions `windows-latest` → build + firma opcional + release.
- **Telemetría**: ninguna (privacidad; y suele prohibirse en corporativo).

---

## 8. Migración de `.vindrc` (paridad funcional)

Orden para mantener a los usuarios (tú) productivos durante el cambio:

1. **Parser** de `.vindrc`: `version`, `set`, `map`/`noremap` (prefijos de modo),
   `unmap`, `autocmd`, `source`, `command`. Referencia exacta: `how-to-use.md`.
2. **Modos base**: GUI Normal, Insert, Command (+ transiciones `to_gui_normal`,
   `to_insert`, `to_command`, `to_ed`).
3. **Comandos más usados primero**: mover ventana/foco, EasyClick, scroll, click,
   `:! , :<` (comandos externos), `open`/`close window`.
4. **Nice-to-have**: registro, macros (`. / q`?), counts, `keyset`.

> Nota: win-vind **no tiene `mapleader`**. Un "leader" se simula con un prefijo
> (p. ej. `<ctrl-win>`) duplicado con `inoremap`. Respeta esa semántica o los
> configs existentes se romperán.

---

## 9. Roadmap por fases

| Fase | Entregable | Criterio de "hecho" |
|---|---|---|
| 0 | Spike | Hook global captura y reinyecta una tecla en 1 día. |
| 1 | MVP | Daemon + tray + `noremap`/`inoremap` + GUI Normal/Insert + config JSON. |
| 2 | Paridad de comandos | 20 comandos top + parser `.vindrc` + modo Command. |
| 3 | EasyClick | UIAutomation + overlay de hints + `hintassign`. |
| 4 | Pulido | Single binary firmado + autostart + hot-reload + tests. |
| 5 (opcional) | Settings UI | Panel WinUI 3 o Electron que edita la config y recarga el daemon. |

---

## 10. Riesgos en laptop corporativa (léelo antes de codear)

1. **WDAC / Device Guard**: bloquea binarios no firmados o no en allowlist.
   Un `.exe` self-contained propio puede quedar **silenciosamente bloqueado**
   (te vas a topar con esto igual que con `hermes.exe` en esta máquina).
   → Plan B: compilar **framework-dependent** si el runtime ya está permitido, o
   pedir allowlist del hash con el equipo de IT.
2. **EDR / Antivirus**: un `WH_KEYBOARD_LL` + `SendInput` se parece **exactamente**
   a un keylogger. Puede ser cuarentenado o generar alertas.
   → Firma el binario, documenta su propósito, y prueba en la máquina corporativa
   ANTES de invertir semanas de desarrollo.
3. **Permisos**: el hook global funciona sin admin; **no** puede interactuar con
   ventanas elevadas (Task Manager admin, UAC). Igual que win-vind.
4. **SSO / instalación**: si no puedes instalar el .NET runtime, ve a Rust o a
   self-contained; si no puedes ejecutar nada sin firma, el proyecto no arranca
   hasta que IT lo apruebe.
5. **Política aceptable de uso**: revisa que un remapeador de teclado global esté
   permitido antes de desplegarlo.

---

## 11. Alternativa "no migrar"

Si el único objetivo es *"tenerlo en la laptop corporativa"* y no *"reconstruirlo"*,
la ruta más barata es: instalar **win-vind** ya compilado (`winget`/`scoop`/`.exe`
portable) o compilar el fork local, y que IT permita el binario. Migrar solo tiene
sentido si necesitas: (a) algo que win-vind no hace, (b) control total del código,
o (c) un stack que puedas mantener tú a largo plazo. **Decide esto primero**: es el
punto que cambia todo el resto del documento.

---

## 12. Decisión sugerida

1. Corre el **spike de la Fase 0 en C#** (1 día) para confirmar que .NET te da el
   hook + UIA en la laptop corporativa sin bloqueos.
2. Si pasa → sigue **C# + WPF (daemon) + WinUI 3 (settings opcional)**.
3. Si WDAC/bloqueo impide .NET → salta a **Rust + `windows-rs`**.
4. Electron **solo** si en algún momento quieres un panel de ajustes bonito; nunca
   como core.

---

### Apéndice: mapa de referencia win-vind → tu implementación

| win-vind (C++) | Equivalente C# | Equivalente Rust |
|---|---|---|
| `src/core/inputgate.cpp` (hook) | `SetWindowsHookEx` P/Invoke | `SetWindowsHookExW` (`windows` crate) |
| `src/core/inputhub.cpp` (engine) | cola + `Channel<T>` | `crossbeam` + thread |
| `src/core/mapsolver.cpp` | diccionario de mapas | `HashMap` + match |
| `src/core/rcparser.cpp` | parser a AST | `nom`/`pest` |
| `src/util/uia.cpp` (EasyClick) | `System.Windows.Automation` | `windows::Win32::UI::Accessibility` |
| `src/util/mouse.cpp` (`SendInput`) | `SendInput` P/Invoke | `SendInput` (`windows`) |
| `src/util/screen_textrender.cpp` (GDI HUD) | WPF overlay | Direct2D / `tiny-skia` |
| `libs/fluent_tray` | `H.NotifyIcon` | `tray-icon` |
| `src/core/background.cpp` | `Task`/`Thread` | `std::thread` |
