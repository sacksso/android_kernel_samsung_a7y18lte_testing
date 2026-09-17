# Handoff — Kernel custom LineageOS 18.1, Galaxy A7 2018 (a7y18lte)

Contexto para continuar el trabajo con otra IA (con acceso directo a los
repos). Escrito desde cero, sin asumir que la otra IA vio nada de esta
conversación.

## Dispositivo y repos

- Samsung Galaxy A7 2018 (a7y18lte, SM-A750F/FN/G), Exynos7885, Linux 4.4.302,
  GCC 6.4.1.
- **Base/upstream que compila el workflow**: `exynos7885-dev/kernel_samsung_exynos7885`,
  rama `lineage-18.1`. Es el repo que el workflow clona como `kernel/` — es
  literalmente el árbol que termina en el teléfono.
- **Fork propio**: `sacksso/sack_kernel_samsung_exynos7885_testing`, rama
  `lineage-18.1`. **IMPORTANTE: el workflow NO clona este repo.** Hoy es solo
  el repo donde vive el workflow (`.github/workflows/*.yml`) y donde se hizo
  un experimento viejo de OC directo en `exynos-acme.c` que nunca llega a
  compilarse por este pipeline (ver sección "Limpieza del fork" abajo).
- Donante para BFQ/WireGuard (mismo hardware): `prashantpaddune/android_kernel_samsung_a7y18lte`,
  clonado en cada build como `donor/`.
- AnyKernel3 para empaquetar: `sacksso/AnyKernel3-a7y18lte`.
- Driver táctil real: Imagis IST40XX (`CONFIG_TOUCHSCREEN_IST40XX=y`) — NO
  Zinitix (código muerto en el árbol, ya confirmado, no repetir el error).
- RAM del dispositivo: 4GB.

## Workflow activo

`16_Compile_kernel_testing_final.yml` (en `sack_kernel_samsung_exynos7885_testing`).
Patrón de trabajo establecido y que hay que seguir respetando:

1. Diagnóstico no bloqueante (grep/find sin `exit 1`) para confirmar contra el
   log real del CI, nunca asumir por nombre de archivo o por analogía con
   otro kernel/dispositivo.
2. Recién con el log real en mano, escribir el parche, siempre con
   verificación (`grep ... || { echo ERROR...; exit 1; }`).
3. Validar siempre el YAML completo con `yaml.safe_load` + `bash -n` de cada
   bloque `run:` antes de dar por bueno un cambio.
4. Decisiones de riesgo real (flasheo, DTB, OC) se confirman explícito con el
   usuario — nunca default silencioso.

## Línea activa: overclock de CPU (estado actual)

### Mecanismo implementado (opt-in, ya validado en log de CI real)

En el step `Add CPU OC opt-in (cpu_max_c1/c2 boot params, default = no-op)`
del workflow, se parchea `drivers/cpufreq/exynos-acme.c` (de la base, en
runner, vía `perl -0777 -pi`) insertando:

```c
static unsigned long arg_cpu_max_c1;   /* default 0 = sin cambios */
static int __init cpufreq_read_cpu_max_c1(char *str) { ... }
__setup("cpu_max_c1=", cpufreq_read_cpu_max_c1);

static unsigned long arg_cpu_max_c2;   /* default 0 = sin cambios */
static int __init cpufreq_read_cpu_max_c2(char *str) { ... }
__setup("cpu_max_c2=", cpufreq_read_cpu_max_c2);
```

Y dentro de `init_domain()`, justo después del clamp de `min-freq`:

```c
if (domain->id == 0 && arg_cpu_max_c1)
    domain->max_freq = arg_cpu_max_c1;
else if (domain->id == 1 && arg_cpu_max_c2)
    domain->max_freq = arg_cpu_max_c2;
```

**Diseño a propósito, distinto del kernel donante del que se portó la idea**
(que traía `arg_cpu_max_c1/c2` hardcodeados a 1690000/2184000, OC activo por
defecto): acá el default es `0` = sin cambios, opt-in explícito vía cmdline,
consistente con la política de "cero overclock por defecto" del proyecto.

Confirmado en el log real de CI (`0_build.txt`, run del 2026-09-17): el step
se aplica limpio, diff visible, verificación pasa. **Pero el override nunca
se dispara en el dispositivo real**, porque no existe ningún step en el
workflow que agregue `cpu_max_c1=<khz> cpu_max_c2=<khz>` a la cmdline real del
kernel (se buscó `bootargs`/`chosen`/`cpu_max_c1=[0-9]` en todo el log, cero
resultados). Por eso "el OC no se aplicó" — es el comportamiento esperado del
diseño, no un bug.

### Mapeo de dominios confirmado (real, de este árbol, no asumido)

```
domain@0 (ACME, exynos7885.dtsi) -> sibling-cpus "0-5" -> cal-id ACPM_DVFS_CPUCL1 -> LITTLE (A53) -> cpu_max_c1
domain@1 (ACME, exynos7885.dtsi) -> sibling-cpus "6-7" -> cal-id ACPM_DVFS_CPUCL0 -> BIG    (A73) -> cpu_max_c2
```

### Techos reales confirmados por LUT (código real, `cmucal-vclk.c` del fork)

```c
struct vclk_lut vdd_cpucl0_lut[] = {   // BIG
    {2192666, ...sod...},              // techo real BIG
    {1698666, ...od...},
    {1300000, ...nm...},
    {747500,  ...ud...},
    {476666,  ...sud...},
};
struct vclk_lut vdd_cpucl1_lut[] = {   // LITTLE
    {1599000, ...sod...},              // techo real LITTLE
    {1352000, ...od...},
    {1001000, ...nm...},
    {598000,  ...ud...},
    {385125,  ...sud...},
};
```

- **LITTLE**: techo real `1599000` kHz. El valor `1690000` que se venía usando
  como default histórico (y que sigue en la tabla `mif-perf` del DT como
  primer escalón) **excede el techo real por 91000 kHz** — no hay corner ASV
  calibrado ahí. Ya se probó en otra sesión/kernel y causó un cuelgue
  reproducible (atribuido en su momento a un bug de la app HKT Tweaks, no
  descartado del todo como causa real).
- **BIG**: techo real `2192666` kHz, stock ya asentado en `2184000` (el
  escalón de tabla inmediatamente por debajo). `2288000` (que aparecía
  "vestido" en la tabla `mif-perf` de `domain@1` y en `user-default-qos` de
  `cpufreq-ufc`) **no existe en ningún LUT real** — cerrado como no viable,
  mismo patrón que ya se cerró para GPU (ver abajo).

### GPU — investigación CERRADA, no reabrir sin nueva evidencia

- Techo ASV real G3D: `1196000` kHz (`vdd_g3d_lut[]`). DT actual:
  `gpu_max_clock = <1100000>` (`exynos7885-mali.dtsi`). Los niveles
  `1300000`/`1200000` de `gpu_dvfs_table` están vestidos pero por encima del
  techo real — igual patrón que el caso `2288000` de CPU BIG.
- Se probó implementar `gpu_max_clock=1200000` en el workflow, el usuario
  reportó "no funcionó" y se removió por completo del workflow. **No
  reabrir sin evidencia nueva y explícita del usuario.**

### Limpieza del fork (hecha en esta sesión, pendiente de subir a GitHub)

`sacksso/sack_kernel_samsung_exynos7885_testing` tenía en su propio
`drivers/cpufreq/exynos-acme.c` un experimento viejo (de una sesión anterior,
con menos experiencia del usuario) que:

- hardcodeaba `domain->max_freq = 2288000` para BIG en **tres puntos
  distintos** dentro de `init_domain()` (con un bloque `if/else` duplicado
  literalmente dos veces seguidas);
- forzaba `policy->max`/`policy->cpuinfo.max_freq = 2288000` en
  `exynos_cpufreq_driver_init()`;
- en `exynos_cpufreq_verify()`, para `policy->cpu >= 6`, devolvía `0` **sin
  pasar por `cpufreq_frequency_table_verify()`** — bypasseaba la validación
  normal de tabla para todo el cluster BIG;
- en `__exynos_cpufreq_target()`, forzaba `target_freq = 2288000` con
  `CPUFREQ_RELATION_L` en **cada** pedido de frecuencia del BIG (no solo en
  el máximo — todo pedido, incluido el governor bajando en idle). Como
  `2288000` no existe en ningún LUT real, esto muy probablemente hacía
  fallar `cpufreq_frequency_table_target()` en cada transición del cluster
  BIG;
- agregaba un notifier (`exynos_thermal_override_notifier`, `late_initcall`)
  que volvía a pisar `policy->max` a `2288000` en cada evento
  `CPUFREQ_POLICY_INIT`/`UPDATE` para CPU >= 6;
- agregaba un segundo `late_initcall` (`exynos_force_big_max_freq`) que
  recorría las CPUs 6-7 y volvía a pisar la policy una vez más.

**El usuario confirmó que ese código es descarte, no le importa perderlo.**
Se limpió el archivo completo quitando los 4 puntos de forzado + los dos
`late_initcall` agregados, y se reemplazó `init_domain()` por el mismo
mecanismo opt-in `arg_cpu_max_c1`/`arg_cpu_max_c2` ya validado en el CI (texto
idéntico, para que el fork y el parche de CI digan lo mismo). Verificado: cero
apariciones de `2288000` o comentarios "Forzar" en el resultado, llaves `{}`
balanceadas (102/102).

**Pendiente**: subir ese `exynos-acme.c` limpio al fork (reemplazando el
actual en `drivers/cpufreq/exynos-acme.c`, rama `lineage-18.1`). El archivo
resultante se entregó al usuario en esta sesión — debería tenerlo a mano para
pasarlo o pedirle que lo suba.

### Pendiente inmediato para activar el OC de verdad

Falta un step de workflow que inyecte `cpu_max_c1=<khz> cpu_max_c2=<khz>` en
la cmdline real del kernel. Dos rutas posibles, sin confirmar todavía cuál
aplica a este dispositivo/ROM:

1. **DTS `/chosen/bootargs`**: se revisó `exynos7885.dtsi` (el `.dtsi` a
   nivel SoC) y **no tiene nodo `chosen`** — es esperable, ese nodo suele
   vivir en el `.dts` de placa (algo como `exynos7885-a7y18lte*.dts`), no en
   el `.dtsi` común. Falta ese archivo.
2. **`BOARD_KERNEL_CMDLINE`** en `BoardConfig.mk` del device tree de
   LineageOS (más típico en builds Samsung/Exynos, donde `mkbootimg` hornea
   la cmdline directo en el `boot.img`). Falta ese archivo si es el caso.

Para saber cuál de las dos rutas es la real sin adivinar, lo más directo es
que el usuario corra esto en el dispositivo actual y comparta la salida:

```bash
cat /proc/cmdline
```

Con eso se identifica el mecanismo real y se escribe el step correspondiente
(mismo patrón que los demás: diagnóstico → log real → parche con
verificación).

### Valores sugeridos para la primera prueba (conservadores, no el máximo)

Sobre los techos reales confirmados arriba, y siguiendo el criterio de subir
en pasos pequeños:

```
cpu_max_c1=1599000   (techo real LITTLE, no 1690000 que está fuera de LUT)
cpu_max_c2=2184000   (mismo valor stock — probar primero que el mecanismo
                       funciona sin cambiar nada, después subir)
```

No usar `1690000` para LITTLE ni `2288000`/valores por encima de `2192666`
para BIG — ninguno de los dos existe en el LUT real de este árbol.

## Otras líneas de trabajo del proyecto (no relacionadas al OC, contexto general)

El foco general del proyecto (decisión explícita del usuario) es optimizar
sin OC — el hilo de OC de arriba es una reapertura puntual para testing, no
un cambio de rumbo del proyecto completo. Pendientes activos en paralelo:

1. **DTB de revisión de placa**: 3 candidatos (`_00`/`_01`/`_03`), sin
   verificar byte a byte. Falta confirmación del usuario.
2. **UKSM**: desactivado, anchor real confirmado en `mm/mmap.c` línea
   939-942, falta escribir el parche real (nunca hecho). KSM legacy activo
   mientras tanto.
3. **LMK**: array de minfree ajustado a 6 niveles (antes 4) para el perfil de
   4GB RAM, implementado en workflow, probado solo contra réplica sintética,
   **nunca corrido contra el archivo real todavía**.
4. **F2FS**: se agregó un step de diagnóstico no bloqueante para comparar
   versión/features (compresión, ATGC, extent cache) — pendiente que el
   usuario corra el workflow y pase el log antes de proponer cualquier
   cambio real.
5. **KernelSU**: mencionado como interés, sin gap de continuidad resuelto —
   puede haber contexto de sesiones anteriores (`CONTEXTO_kernel_a7y18lte_v3`
   a `v6`) no consolidado.
6. **SELinux**: confirmado por el usuario que YA está enforcing (kernel base
   y propio) — no requiere trabajo.
7. **zram writeback**, **thermal/IPA (curva de throttling)**: mencionados
   como interesantes, no descartados, no empezados.

## Reglas del proyecto que hay que seguir respetando

- No asumir nada por coincidencia de nombre entre repos/kernels/dispositivos
  — ya pasó con Zinitix/Imagis, tMIx/Bifrost, exynos7884/7885, y en esta
  misma sesión con la confusión entre `exynos7885-dev/kernel_samsung_exynos7885`
  (lo que realmente compila) y `sack_kernel_samsung_exynos7885_testing` (el
  fork del usuario, que el workflow no toca).
- Cuando otra IA (incluida vos) describe código o comportamiento de un
  archivo real, verificarlo contra el archivo real antes de actuar — no dar
  por buena la descripción sola. En esta sesión se verificó línea por línea
  el `exynos-acme.c` del fork contra lo que había reportado otra IA de
  GitHub, y coincidió 100% — pero el criterio es siempre verificar, no
  asumir que coincide.
- Cero OC/cambios de riesgo por defecto — todo opt-in, con verificación
  explícita, y builds separadas (base / mejoras genéricas / tuning / DVFS /
  OC al final) para poder atribuir cualquier problema a una sola causa.
- SELinux se mantiene enforcing siempre; no usar
  `androidboot.selinux=permissive` para "arreglar" nada.
