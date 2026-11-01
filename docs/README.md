# Informe de ejecución — Fluid Attacks sobre OWASP Juice Shop

**Curso:** 0453_2666 Desarrollo de Sistemas Seguros — Semana 4 (DevSecOps)
**Autora:** Manuela Jiménez
**Repositorio analizado:** fork de [juice-shop/juice-shop](https://github.com/juice-shop/juice-shop) → [manujimeneza/juice-shop](https://github.com/manujimeneza/juice-shop)
**Herramientas:** Fluid Attacks SAST, SCA y Secret Scan (CLI en Docker y GitHub Actions)

---

## Resumen de entregables

| Paso | Qué pide | Rama | Commit (enlace para el informe) | Resultado del pipeline |
|---|---|---|---|---|
| 1 | CLI local | — | — | Escaneo local: SAST 109 hallazgos, SCA 50 |
| 2 | 3 scanners, informativo, sin CSV, sin romper | `paso2-informativo` | [2e97565](https://github.com/manujimeneza/juice-shop/commit/2e97565ea0365690114641dd433bb72a7fa25973) | ✅ [verde](https://github.com/manujimeneza/juice-shop/actions/runs/41357862214) |
| 3 | 3 scanners, CSV, romper el build (estricto) | `paso3-estricto` | [a6eb166](https://github.com/manujimeneza/juice-shop/commit/a6eb166f7aca74f9618b837f1f23c34e3fab5224) | ❌ [rojo en los 3 jobs](https://github.com/manujimeneza/juice-shop/actions/runs/41359285248) |
| 4 | 3 scanners, CSV, romper solo sobre umbral | `paso4-umbral` | [641acd0](https://github.com/manujimeneza/juice-shop/commit/641acd04afaf9879a29bfc29e5a618a331bfcfe9) | ❌ [rojo solo en SCA](https://github.com/manujimeneza/juice-shop/actions/runs/41359304451) |

Enlaces completos:

- Paso 2: https://github.com/manujimeneza/juice-shop/commit/2e97565ea0365690114641dd433bb72a7fa25973
- Paso 3: https://github.com/manujimeneza/juice-shop/commit/a6eb166f7aca74f9618b837f1f23c34e3fab5224
- Paso 4: https://github.com/manujimeneza/juice-shop/commit/641acd04afaf9879a29bfc29e5a618a331bfcfe9

---

## Paso 1 — Ejecución local del CLI

El CLI de Fluid Attacks se distribuye como imagen Docker. Se ejecutó en macOS (zsh) sobre un clon local de Juice Shop, montando el directorio actual en el contenedor (`-v .:/my-dir`).

### 1.1 Clonar el repositorio y descargar el CLI

```zsh
cd ~/Desktop/Universidad
git clone https://github.com/juice-shop/juice-shop.git
docker pull fluidattacks/sast:latest
```

El primer `docker pull` falló porque Docker Desktop no estaba abierto (`Cannot connect to the Docker daemon`). Tras iniciarlo, se repitió el comando.

![Clon del repositorio y primer docker pull](capturas/01-clon-y-docker-pull.png)

![Descarga de la imagen fluidattacks/sast](capturas/02-docker-pull-descarga.png)

### 1.2 Ejecutar el escaneo SAST

```zsh
docker run --rm -v .:/my-dir fluidattacks/sast:latest sast scan /my-dir
```

![Imagen descargada e inicio del escaneo](capturas/03-docker-pull-ok-y-scan.png)

El escaneo terminó en 27 segundos con **109 vulnerabilidades**. El prompt `manu@192 juice-shop %` evidencia que corre en la máquina local:

![Resumen del escaneo SAST local](capturas/04-sast-local-resumen.png)

Ejemplo de hallazgo: *371. DOM-Based cross-site scripting (XSS)* en `frontend/src/app/search-result/search-result.component.ts`, CWE-79.

![Detalle de hallazgos SAST](capturas/05-sast-local-hallazgos.png)

### 1.3 Escaneo SCA local (opcional)

```zsh
docker pull fluidattacks/sca:latest
docker run --rm -v .:/my-dir fluidattacks/sca:latest sca scan /my-dir
```

Resultado: **50 vulnerabilidades** en dependencias, por ejemplo `js-yaml ^3.14.0` y `zod ^4.4.3`.

![Resumen del escaneo SCA local](capturas/06-sca-local-resumen.png)

### 1.4 Ejecución con archivo de configuración

Configuración usada: [`evidencias/config-local.yaml`](evidencias/config-local.yaml)

```zsh
docker run --rm -v .:/my-dir fluidattacks/sast:latest sast scan /my-dir/config-local.yaml
```

Salida: [`evidencias/fluid-local-resultados.csv`](evidencias/fluid-local-resultados.csv)

---

## Preparación para GitHub Actions

1. Fork de `juice-shop/juice-shop` a `manujimeneza/juice-shop` (público).
2. Activación de Actions en el fork (los forks lo traen desactivado).
3. Clon local en `~/Desktop/Universidad/juice-shop-fork`. La rama por defecto detectada es `master`.
4. Una rama por paso, creada desde `master`, para que cada enlace de commit sea independiente.

Cada rama agrega solo archivos nuevos bajo `.github/`:

```
.github/
├── fluid/
│   ├── sast.yaml   # configuración SAST
│   ├── sca.yaml    # configuración SCA
│   └── ss.yaml     # configuración Secret Scan
└── workflows/
    └── fluid-pasoN.yml
```

Decisiones comunes a los tres workflows:

- `fetch-depth: 0`: las actions consultan el historial para detectar la rama por defecto.
- `scanner_mode: full`: fuera de la rama por defecto, SAST y Secret Scan pasan a modo diferencial. Como solo cambian archivos de `.github/`, el escaneo diferencial no analizaría nada.
- La evidencia se sube como **artefacto** del workflow (`actions/upload-artifact`), sin depender de GitHub Advanced Security.

---

## Paso 2 — Modo informativo

**Rama:** `paso2-informativo` · **Commit:** [2e97565](https://github.com/manujimeneza/juice-shop/commit/2e97565ea0365690114641dd433bb72a7fa25973)

Configuración clave (SAST y SCA):

```yaml
strict: false          # reporta, pero no falla el pipeline
output:
  format: SARIF        # sin CSV
```

Secret Scan solo lee las claves `ss` y `output`, así que su configuración se limita a `include`/`exclude` y salida SARIF.

![Commit y push del Paso 2](capturas/07-paso2-commit-push.png)

**Resultado:** los 3 jobs en verde. Cada job imprime `vulnerabilities_found = false` porque en modo no estricto el escáner termina con código 0.

---

## Paso 3 — Modo estricto con CSV

**Rama:** `paso3-estricto` · **Commit:** [a6eb166](https://github.com/manujimeneza/juice-shop/commit/a6eb166f7aca74f9618b837f1f23c34e3fab5224)

Configuración clave (SAST y SCA):

```yaml
strict: true           # código de salida 1 ante cualquier hallazgo
output:
  format: ALL          # genera .csv y .sarif
```

Secret Scan: `format: CSV` (no admite `ALL` ni `strict`).

![Commit y push del Paso 3](capturas/08-paso3-commit-push.png)

### Hallazgo durante la ejecución: el primer intento salió en verde

La primera versión del workflow (commit `94789af`) terminó **en verde**, aunque la configuración tenía `strict: true`. Al revisar el código de la action ([`generate-config.sh`](https://github.com/fluidattacks/sast-action/blob/main/generate-config.sh)):

```bash
docker run ... sast scan "${CONFIG_FILE}" || exit_code=$?
if [[ ${exit_code} -eq 0 ]]; then
  echo "vulnerabilities_found=false" >> "${GITHUB_OUTPUT}"
elif [[ ${exit_code} -eq 1 ]]; then
  echo "vulnerabilities_found=true" >> "${GITHUB_OUTPUT}"
```

La action **captura** el código 1 del modo estricto y lo convierte en la salida `vulnerabilities_found=true`, sin fallar el job. La decisión de romper el build queda en manos del pipeline. La corrección (commit `a6eb166`) agrega a cada job un paso explícito:

```yaml
- name: Romper el build (modo estricto)
  if: steps.scan.outputs.vulnerabilities_found == 'true'
  run: exit 1
```

En Secret Scan, el binario `ss` termina con código 0 aunque encuentre secretos. Por eso el quiebre también cuenta las filas del CSV.

Hubo un segundo problema: con `format: ALL` el escáner agrega la extensión al `file_path` y genera `fluid-sast-estricto.sarif.csv`. La ruta del artefacto se cambió a `fluid-sast-estricto*`.

**Resultado:** ❌ los 3 jobs en rojo (SAST, SCA y Secret Scan), que es lo esperado sobre una aplicación deliberadamente vulnerable.

CSV exportados: [`evidencias/paso3/`](evidencias/paso3/)

---

## Paso 4 — Quiebre condicionado por `strictness_threshold`

**Rama:** `paso4-umbral` · **Commit:** [641acd0](https://github.com/manujimeneza/juice-shop/commit/641acd04afaf9879a29bfc29e5a618a331bfcfe9)

Configuración clave (SAST y SCA):

```yaml
strict: true
strictness_threshold: high   # solo high y critical rompen el build
output:
  format: ALL
```

El escáner **sigue reportando** todas las severidades en el CSV, pero solo termina con código 1 si alguna es `high` o `critical`. El workflow aplica la misma corrección del Paso 3.

**Secret Scan:** según la documentación, solo SAST, SCA y DAST soportan `strictness_threshold`. Por eso en este paso Secret Scan queda informativo, con la justificación escrita en `ss.yaml`.

![Commit y push del Paso 4](capturas/09-paso4-commit-push.png)

> En la captura, `git checkout -b paso4-umbral` falló porque la rama ya existía, y el commit quedó en `master` local. Se movió a `paso4-umbral` (`git branch -f`) y `master` se restauró a `origin/master` antes del push definitivo.

**Resultado:**

| Job | Resultado | Interpretación |
|---|---|---|
| SAST (umbral high) | ✅ verde | Ningún hallazgo SAST alcanza high/critical. Todos quedan reportados en el CSV |
| SCA (umbral high) | ❌ rojo | Hay dependencias con CVE de severidad high o superior |
| Secret Scan | ✅ verde | Informativo por diseño |

Esto muestra el efecto del umbral: con el mismo código, el Paso 3 falla en SAST y el Paso 4 no.

CSV exportados: [`evidencias/paso4/`](evidencias/paso4/)

---

## Vista general de las ejecuciones en GitHub Actions

![Listado de workflows en Actions](capturas/10-actions-ejecuciones.png)

> Esta captura es anterior a las correcciones: muestra el primer intento del Paso 3 en verde. Los runs definitivos están enlazados en la tabla de resumen.

---

## Conclusiones

1. **Local frente a pipeline:** el mismo motor detecta los mismos hallazgos en local (Docker) y en GitHub Actions. La diferencia es que en el pipeline el análisis corre solo en cada push.
2. **Informativo → estricto → umbral** es una adopción gradual: primero se ve el estado real, después se bloquea todo y al final se bloquea solo lo grave, para no frenar el desarrollo por hallazgos de baja severidad.
3. **Hay que verificar que la puerta de calidad realmente bloquee.** `strict: true` en la configuración no bastó: la action absorbe el código de salida. Un pipeline en verde no prueba que el control funcione, así que hay que comprobarlo con un caso que deba fallar.
