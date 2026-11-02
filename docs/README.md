# Taller 4: Fluid Attacks con Juice Shop

Desarrollo de Sistemas Seguros
Manuela Jiménez Arredondo

Para este trabajo usé OWASP Juice Shop. Hice un fork en mi cuenta ([manujimeneza/juice-shop](https://github.com/manujimeneza/juice-shop)) y sobre ese fork configuré los escáneres de Fluid Attacks (SAST, SCA y Secret Scan), primero en local con Docker y después en GitHub Actions.

## Enlaces de los commits

- Paso 2: https://github.com/manujimeneza/juice-shop/commit/2e97565ea0365690114641dd433bb72a7fa25973
- Paso 3: https://github.com/manujimeneza/juice-shop/commit/a6eb166f7aca74f9618b837f1f23c34e3fab5224
- Paso 4: https://github.com/manujimeneza/juice-shop/commit/641acd04afaf9879a29bfc29e5a618a331bfcfe9

| Paso | Lo que pide el enunciado | Cómo lo cumplí | Resultado en Actions |
|---|---|---|---|
| 1 | CLI local, pantallazo del comando y de la ejecución | Imagen Docker de Fluid Attacks en mi Mac | Capturas 1 a 6 |
| 2 | 3 escáneres, archivo de configuración, informativo, sin CSV, sin romper | `strict: false`, salida SARIF | [Verde](https://github.com/manujimeneza/juice-shop/actions/runs/41357862214) |
| 3 | 3 escáneres, archivo de configuración, CSV, romper el build | `strict: true`, `format: ALL` / `CSV` | [Rojo en los tres jobs](https://github.com/manujimeneza/juice-shop/actions/runs/41359285248) |
| 4 | 3 escáneres, archivo de configuración, CSV, romper solo sobre un umbral | `strictness_threshold: high` | [Rojo solo en SCA](https://github.com/manujimeneza/juice-shop/actions/runs/41359304451) |

## Alcance y limitaciones de la revisión

Antes de los resultados quiero dejar claro qué cubre este análisis, para que las conclusiones no se generalicen más de lo que la evidencia permite:

- **Qué se analizó:** el código fuente del repositorio Juice Shop en la rama `master` del fork (commit `717f5ef`), incluyendo los archivos de infraestructura que están dentro del repositorio (Dockerfiles y Terraform).
- **Qué se excluyó a propósito:** `node_modules`, `build`, `dist`, `frontend/dist` y archivos `*.min.js`. Son código generado o de terceros; las dependencias de terceros se revisan con SCA a partir de `package.json`, no leyendo su código.
- **Qué no cubre:** la aplicación en ejecución (eso sería DAST, que no se pedía), la configuración real del servidor donde se despliegue, variables de entorno, servicios externos ni secretos guardados fuera del repositorio. Si un hallazgo no aparece, solo puedo afirmar que no está en el código analizado, no que no exista en un despliegue real de la aplicación.
- **Herramienta:** se usaron las actions oficiales `fluidattacks/sast-action`, `sca-action` y `ss-action` (versión `main` al 1 de noviembre de 2026) sin token de la plataforma de Fluid Attacks, es decir, con las reglas gratuitas del escáner. Los hallazgos son los que detectan esas reglas; un análisis manual o con otras herramientas podría encontrar más.

## Paso 1. Ejecución en local

Primero cloné el repositorio original de Juice Shop en mi computador y descargué la imagen de Docker del CLI.

```zsh
cd ~/Desktop/Universidad
git clone https://github.com/juice-shop/juice-shop.git
docker pull fluidattacks/sast:latest
```

La primera vez me salió el error `Cannot connect to the Docker daemon` porque no tenía Docker Desktop abierto. Lo abrí y volví a correr el comando.

![Clon y primer intento de docker pull](capturas/01-clon-y-docker-pull.png)

![Descarga de la imagen](capturas/02-docker-pull-descarga.png)

Cuando terminó la descarga ejecuté el escaneo montando la carpeta actual en el contenedor:

```zsh
docker run --rm -v .:/my-dir fluidattacks/sast:latest sast scan /my-dir
```

![Inicio del escaneo](capturas/03-docker-pull-ok-y-scan.png)

El escaneo tardó 27 segundos y reportó 109 vulnerabilidades. En la terminal se ve mi usuario (`manu@192 juice-shop`), o sea que se está ejecutando en mi máquina y no en un servidor.

![Resumen SAST local](capturas/04-sast-local-resumen.png)

Uno de los hallazgos es un XSS basado en DOM en `frontend/src/app/search-result/search-result.component.ts`, línea 113, porque se usa `bypassSecurityTrustHtml` sobre la descripción de los productos (CWE-79).

![Hallazgos SAST](capturas/05-sast-local-hallazgos.png)

También probé el SCA, que revisa las dependencias declaradas en `package.json`. Reportó 50 vulnerabilidades, por ejemplo en `js-yaml` y `zod`.

```zsh
docker pull fluidattacks/sca:latest
docker run --rm -v .:/my-dir fluidattacks/sca:latest sca scan /my-dir
```

![Resumen SCA local](capturas/06-sca-local-resumen.png)

Después lo corrí con un archivo de configuración ([config-local.yaml](evidencias/config-local.yaml)) para exportar los resultados a CSV:

```zsh
docker run --rm -v .:/my-dir fluidattacks/sast:latest sast scan /my-dir/config-local.yaml
```

Con la configuración salieron 107 hallazgos en lugar de 109, porque el archivo excluye `node_modules`, `build`, `dist` y los `*.min.js`. El CSV está en [evidencias/fluid-local-resultados.csv](evidencias/fluid-local-resultados.csv).

## Antes de los pasos 2, 3 y 4

Hice el fork de Juice Shop y activé GitHub Actions en el fork (viene desactivado por defecto). Después lo cloné en `juice-shop-fork`. La rama principal es `master`.

Creé una rama distinta para cada paso, todas a partir de `master`, así cada commit queda separado y el resultado de un paso no afecta al otro. En cada rama solo agregué archivos dentro de `.github`, sin tocar el código de la aplicación:

```
.github/
├── fluid/
│   ├── sast.yaml
│   ├── sca.yaml
│   └── ss.yaml
└── workflows/
    └── fluid-pasoN.yml
```

Decisiones que tomé en los workflows y por qué:

- **`fetch-depth: 0`** en el checkout: las actions ejecutan `git remote show origin` para saber cuál es la rama principal, y con el checkout por defecto (solo el último commit) eso falla.
- **`scanner_mode: full`**: en ramas que no son la principal, SAST y Secret Scan pasan solos a modo diferencial y solo revisan los archivos que cambiaron. Como mis commits solo agregan archivos en `.github`, el escaneo diferencial no habría revisado nada del código. Con `full` se analiza todo el repositorio.
- **Artefactos en lugar de la pestaña Security:** subir el SARIF a la pestaña Security depende de GitHub Advanced Security. Guardar los resultados como artefactos del workflow funciona en cualquier repositorio y permite descargar los CSV como evidencia.

## Paso 2. Modo informativo

En este paso los tres escáneres solo reportan y no rompen el build. En los archivos de SAST y SCA dejé:

```yaml
strict: false
output:
  format: SARIF
```

Usé SARIF y no CSV porque el enunciado pide explícitamente no exportar a CSV, y SARIF es el formato estándar que leen las herramientas de análisis.

El archivo de Secret Scan es más corto porque esa action solo lee las claves `ss` y `output`; si se le agregan `strict` o `namespace` las ignora.

![Commit del paso 2](capturas/07-paso2-commit-push.png)

**Resultado:** los tres jobs en verde. Los escáneres sí encontraron hallazgos (son los mismos de los pasos 3 y 4), pero en modo no estricto terminan con código 0 y la salida `vulnerabilities_found` queda en `false`, que es lo que se esperaba.

## Paso 3. Modo estricto y exportación a CSV

Aquí había que exportar a CSV y hacer que el build falle. En SAST y SCA cambié a:

```yaml
strict: true
output:
  format: ALL
```

Elegí `ALL` porque genera el CSV que pide el enunciado y además el SARIF, así queda la misma evidencia que en el paso 2. En Secret Scan puse `format: CSV`, porque esa action no acepta `ALL`.

![Commit del paso 3](capturas/08-paso3-commit-push.png)

### Problema que encontré

La primera vez que lo subí (commit `94789af`) el workflow salió en verde, aunque tenía `strict: true`. Me pareció raro, así que revisé el código de la action ([generate-config.sh](https://github.com/fluidattacks/sast-action/blob/main/generate-config.sh)) y vi esto:

```bash
docker run ... sast scan "${CONFIG_FILE}" || exit_code=$?
if [[ ${exit_code} -eq 0 ]]; then
  echo "vulnerabilities_found=false" >> "${GITHUB_OUTPUT}"
elif [[ ${exit_code} -eq 1 ]]; then
  echo "vulnerabilities_found=true" >> "${GITHUB_OUTPUT}"
```

El escáner sí termina con error, pero la action lo atrapa y solo deja una variable `vulnerabilities_found=true`, sin hacer fallar el job. Para solucionarlo agregué un paso al final de cada job que revisa esa variable:

```yaml
- name: Romper el build (modo estricto)
  if: steps.scan.outputs.vulnerabilities_found == 'true'
  run: exit 1
```

Escogí esta solución y no otras (por ejemplo, ejecutar el CLI directamente con `docker run` sin la action) porque el enunciado pide usar las actions de Fluid Attacks, y así la decisión de bloquear queda visible en el workflow como un paso con nombre propio.

En Secret Scan además cuento las filas del CSV. Ese escáner terminó con código 0 aunque encontró 10 secretos, así que la variable sola no era suficiente.

También noté que con `format: ALL` el archivo se llamaba `fluid-sast-estricto.sarif.csv` y no `fluid-sast-estricto.csv`, por eso no se estaba guardando. Cambié la ruta a `fluid-sast-estricto*`.

**Resultado** (commit `a6eb166`): los tres jobs en rojo, que es lo correcto porque Juice Shop está hecho para ser vulnerable y en modo estricto cualquier hallazgo bloquea.

| Escáner | Hallazgos en el CSV | Severidad |
|---|---|---|
| SAST | 107 | 101 Low, 6 Medium |
| SCA | 50 | 26 Low, 23 Medium, 1 High |
| Secret Scan | 10 | No asigna severidad |

Los CSV están en [evidencias/paso3](evidencias/paso3/).

## Paso 4. Umbral de severidad

En este paso el build solo debe fallar si hay vulnerabilidades graves. En SAST y SCA agregué:

```yaml
strict: true
strictness_threshold: high
output:
  format: ALL
```

**Por qué `high`:** los valores posibles son `none`, `low`, `medium`, `high` y `critical`, siguiendo los rangos de CVSS v4 (high es un puntaje de 7.0 o más). Escogí `high` porque:

- Con `low` o `medium` el resultado sería prácticamente igual al paso 3, porque casi todos los hallazgos son de esas severidades, y no se vería el efecto del umbral.
- Con `critical` no fallaría nada, porque no hubo ningún hallazgo crítico; el control no estaría probando nada.
- `high` corresponde a vulnerabilidades explotables por red con impacto alto en confidencialidad, integridad o disponibilidad, que es lo que un equipo normalmente no dejaría pasar a producción.

El escáner sigue reportando todo en el CSV, pero solo falla si encuentra algo high o critical. En el workflow dejé el mismo paso de corrección que en el paso 3.

![Commit del paso 4](capturas/09-paso4-commit-push.png)

En la captura se ve que `git checkout -b paso4-umbral` me dio error porque la rama ya existía, y el commit quedó en `master`. Lo corregí moviendo el commit a la rama `paso4-umbral` y dejando `master` como estaba en GitHub.

### Secret Scan en el paso 4: por qué queda informativo

El enunciado pide los tres escáneres con umbral, pero Secret Scan no lo permite:

- La documentación de Fluid Attacks indica que `strictness_threshold` solo existe para SAST, SCA y DAST.
- Revisando la action, Secret Scan solo lee las claves `ss` y `output` del archivo de configuración. Si se le pone `strictness_threshold` lo ignora sin dar error.
- En el CSV del Secret Scan no hay una columna de severidad (las columnas son `file_path`, `line`, `description`, `cvss_v4`, `finding`, `title`, `kind`, `method` y `verified`), así que tampoco se podía implementar el umbral en el workflow leyendo el CSV.

Tenía dos opciones: que bloqueara siempre (igual que en el paso 3) o dejarlo informativo. Lo dejé informativo para que el paso 4 muestre solo el efecto del umbral, y escribí la justificación en un comentario de `ss.yaml`. Aun así, sigue exportando el CSV que pide el enunciado.

**Resultado:**

| Job | Resultado | Por qué |
|---|---|---|
| SAST | Verde | Los 107 hallazgos son Low (101) o Medium (6); ninguno llega a high |
| SCA | Rojo | Hay 1 hallazgo High: `sequelize ^6.37.3` con CVE-2026-69240 (inyección SQL, CVSS 7.2) |
| Secret Scan | Verde | Informativo, ver justificación arriba |

Con el mismo código, SAST falla en el paso 3 y pasa en el paso 4; ahí se nota el efecto del umbral. Y de las 50 vulnerabilidades de SCA, una sola es suficiente para bloquear.

Los CSV están en [evidencias/paso4](evidencias/paso4/).

## Análisis de los hallazgos

**SCA - el que rompe el build en el paso 4.** La dependencia `sequelize` (el ORM con el que Juice Shop habla con la base de datos) está en `^6.37.3` y tiene CVE-2026-69240, una inyección SQL con CVSS 7.2. Es dependencia directa y el escáner indica que la versión corregida es la `6.37.4`.

- *Corrección necesaria para que el paso 4 pase:* actualizar `sequelize` a `6.37.4` en `package.json` (línea 148) y regenerar `package-lock.json`. Es la corrección mínima porque el escáner la marca como cambio de versión menor sin cambios incompatibles, así que no obliga a modificar el código de la aplicación.
- *Recomendación adicional (no necesaria para pasar el umbral):* activar Dependabot para que las dependencias vulnerables se actualicen con PRs automáticos. Lo separo porque no corrige este hallazgo, solo ayuda a que no se vuelva a acumular.

**SAST.** Los 6 hallazgos Medium están en archivos de infraestructura y no en el código de la aplicación: 5 son de Terraform (balanceador de carga sin protección contra borrado) y 1 de un Dockerfile de pruebas (`test/smoke/Dockerfile`) que corre como root. Los 101 Low incluyen hallazgos en el código, como el XSS de `search-result.component.ts`. Me llamó la atención que una aplicación diseñada para tener vulnerabilidades como XSS e inyección SQL no tenga ningún hallazgo SAST high. Mi interpretación es que el escáner califica la severidad con CVSS sin conocer el contexto del reto, y que muchas de las vulnerabilidades de Juice Shop dependen de la lógica de la aplicación, algo que un análisis estático no ve completo. Por eso el umbral `high` en SAST, aplicado a ciegas, dejaría pasar código que sabemos que es vulnerable.

**Secret Scan.** Los 10 secretos están en archivos de prueba (`*.spec.ts`, `test/api`, `test/server`, `test/cypress`), por ejemplo `forgedJwt.spec.ts`. Todos aparecen como "Generic pattern, provider unknown", es decir, el escáner detectó algo con forma de token o clave pero no pudo verificar que sea una credencial real. Por los nombres de los archivos parecen tokens JWT de prueba. No puedo confirmar desde el código si alguno funciona en un entorno real, así que lo dejo como limitación: habría que revisarlos uno por uno antes de descartarlos como falsos positivos.

## Ejecuciones en GitHub Actions

![Ejecuciones en Actions](capturas/10-actions-ejecuciones.png)

Esta captura la tomé antes de corregir el paso 3, por eso todavía aparece en verde. Los resultados finales están en los enlaces del principio.

## Conclusiones

1. **Un pipeline en verde no demuestra que el control funcione.** Mi primer paso 3 tenía `strict: true` y aun así pasaba. Si no hubiera revisado por qué, habría entregado un control que no bloqueaba nada. En DevSecOps hay que comprobar que la puerta de seguridad falla cuando debe fallar, no solo que el workflow se ejecuta.

2. **El umbral es útil, pero depende de cómo la herramienta califica la severidad.** En el paso 4 el umbral `high` bloqueó la única dependencia con riesgo alto y dejó pasar 157 hallazgos de menor severidad, que es lo que se busca para no frenar el desarrollo. Pero en SAST dejó pasar un XSS real porque el escáner lo calificó como Low. Por eso creo que el umbral sirve bien para SCA, donde la severidad viene de CVEs públicos, y en SAST tendría que acompañarse de una revisión manual de los hallazgos Low y Medium en vez de confiar solo en el número.
