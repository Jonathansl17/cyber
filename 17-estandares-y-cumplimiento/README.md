# Estándares y cumplimiento

## Conceptos previos

- Control de seguridad: medida que reduce un riesgo; técnica (MFA), administrativa (política) o física (cerradura).
- Riesgo: pérdida esperada que combina amenaza, vulnerabilidad e impacto; cálculo y respuestas en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/).
- Tríada CIA: confidencialidad, integridad y disponibilidad; también en [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/).
- Estándar: documento publicado por un organismo reconocido que fija requisitos o buenas prácticas para que distintas organizaciones hagan algo de la misma manera.
- Marco (framework): estructura organizada de objetivos y prácticas que guía un programa completo; suele ser voluntario y adaptable.
- Política: documento aprobado por la dirección que dice qué se debe hacer (por ejemplo, "toda cuenta con acceso remoto usa MFA").
- Evidencia: prueba verificable de que un control existe y funciona: un log, una captura de configuración, un registro firmado, un ticket.
- Vulnerabilidad: debilidad de un sistema que una amenaza puede aprovechar.
- Parche: actualización del fabricante que corrige un fallo.
- Hardening (endurecimiento): reducir la superficie de ataque de un sistema quitando servicios, cerrando puertos y ajustando configuraciones; detalle en [`14-defensa-y-hardening`](../14-defensa-y-hardening/).

## Understand Common Standards

Los estándares de seguridad son documentos de referencia que dicen qué debe proteger una organización y cómo demostrarlo; existen porque sin ellos cada empresa inventaría su propio programa, sería imposible comparar dos proveedores y un regulador o un cliente no tendría con qué medir. Los cinco que pide el roadmap vienen de tres organismos: ISO (Organización Internacional de Normalización, con la IEC), NIST (Instituto Nacional de Estándares y Tecnología de EE. UU.) y CIS (Center for Internet Security, una organización sin ánimo de lucro).

### ISO

**ISO/IEC 27001** es una norma internacional certificable que fija los requisitos para establecer, implantar, mantener y mejorar un ISMS (Information Security Management System, sistema de gestión de seguridad de la información); **ISO/IEC 27002** es la norma complementaria que explica cómo aplicar cada uno de los controles de seguridad que 27001 enumera.

Existe porque las empresas necesitan demostrar a clientes y socios de cualquier país que gestionan la seguridad de forma seria, sin que cada cliente tenga que auditarlas por su cuenta. Un certificado ISO 27001 emitido por un organismo acreditado sirve en todo el mundo.

Analogía: 27001 es el reglamento de un examen de conducir (qué tienes que demostrar para obtener la licencia); 27002 es el manual del conductor (cómo se hace cada maniobra). Te examinan contra el reglamento, no contra el manual.

La clave de 27001 es que no es una lista de productos sino un sistema de gestión: un ciclo de mejora continua PDCA (Plan, Do, Check, Act, es decir, planificar, hacer, verificar, actuar) basado en riesgo. La versión vigente es la de 2022. Su estructura:

- Cláusulas 4 a 10, obligatorias para certificarse: 4 Contexto de la organización (alcance, partes interesadas), 5 Liderazgo (compromiso de la dirección, política), 6 Planificación (evaluación y tratamiento de riesgos, objetivos), 7 Soporte (recursos, competencia, concienciación, documentación), 8 Operación (ejecutar el tratamiento de riesgos), 9 Evaluación del desempeño (medición, auditoría interna, revisión por la dirección), 10 Mejora (no conformidades y acciones correctivas).
- Anexo A: 93 controles agrupados en 4 temas: organizacionales (37), de personas (8), físicos (14) y tecnológicos (34).
- SoA (Statement of Applicability, declaración de aplicabilidad): el documento donde la organización dice cuáles de los 93 controles aplica, cuáles excluye y por qué. Un control solo se excluye si el análisis de riesgos lo justifica.

ISO 27002:2022 tiene los mismos 93 controles, cada uno con su propósito y guía de implantación, y les añade atributos (por ejemplo, si el control es preventivo, detectivo o correctivo, y qué propiedad de la tríada CIA protege). 27002 no es certificable.

Ciclo de certificación: auditoría de etapa 1 (revisión de la documentación), auditoría de etapa 2 (comprobar que el ISMS funciona en la práctica), certificado válido 3 años con auditorías de seguimiento anuales y una recertificación al final.

Ejemplo: una empresa de software de 50 personas quiere vender a bancos europeos, que exigen ISO 27001. Define el alcance (desarrollo y operación de su plataforma), hace su análisis de riesgos, decide aplicar 85 de los 93 controles (excluye, por ejemplo, controles de desarrollo subcontratado porque no subcontrata) y lo justifica en la SoA. Tras 9 meses de funcionamiento con evidencias, pasa la etapa 2 con 2 no conformidades menores y obtiene el certificado.

```
ISO 27001 → requisitos del ISMS; certificable; cláusulas 4-10 + Anexo A.
ISO 27002 → guía de implantación de los 93 controles; no certificable.
Anexo A → 93 controles en 4 temas: organizacionales 37, personas 8, físicos 14, tecnológicos 34.
SoA → qué controles se aplican, cuáles no y por qué.
```

### RMF

El **RMF** (Risk Management Framework, definido en NIST SP 800-37 Rev. 2) es un proceso de siete pasos que integra la seguridad y la privacidad en todo el ciclo de vida de un sistema de información, desde su preparación hasta su monitoreo continuo, y termina con una decisión formal de un responsable sobre si el riesgo del sistema es aceptable.

Existe porque en el gobierno federal de EE. UU. cada sistema necesita que alguien con autoridad acepte por escrito su riesgo antes de ponerlo en producción, y hacía falta un proceso común para llegar a esa decisión con evidencias. Es obligatorio para agencias federales y sus contratistas, y muchas empresas lo usan como modelo.

Analogía: el proceso para que un edificio nuevo obtenga el permiso de ocupación. Se clasifica el edificio (vivienda, hospital), se eligen las normas que le aplican, se construye, un inspector verifica, la autoridad firma el permiso y luego hay inspecciones periódicas.

Los siete pasos, en orden:

1. Prepare (preparar): fijar contexto, roles, estrategia de riesgo de la organización e inventario de activos.
2. Categorize (categorizar): clasificar el sistema según el impacto que tendría perder la confidencialidad, integridad o disponibilidad de su información: bajo, moderado o alto (FIPS 199). El sistema toma el nivel más alto de los tres (high-water mark).
3. Select (seleccionar): elegir los controles de SP 800-53 partiendo de la línea base correspondiente a la categoría (baja, moderada o alta, definidas en SP 800-53B) y ajustarlos.
4. Implement (implementar): poner los controles en marcha y documentarlos en el plan de seguridad del sistema.
5. Assess (evaluar): un evaluador independiente comprueba que los controles están bien implantados y funcionan (procedimientos de SP 800-53A).
6. Authorize (autorizar): el AO (Authorizing Official, funcionario responsable) revisa el riesgo residual y emite o niega la ATO (Authority to Operate, autorización para operar).
7. Monitor (monitorear): vigilar de forma continua los controles y los cambios, y reevaluar cuando algo cambia.

```
Prepare -> Categorize -> Select -> Implement -> Assess -> Authorize -> Monitor
                ^                                                       |
                +-------------------- cambios significativos -----------+
```

Ejemplo: una agencia lanza un portal de citas que guarda nombres y teléfonos. Confidencialidad moderada, integridad moderada, disponibilidad baja: el sistema es "moderado". Se parte de la línea base moderada de 800-53, se implantan los controles, un evaluador encuentra 4 debilidades, se registran en un plan de acción (POA&M) con fechas, el AO acepta el riesgo residual y firma una ATO.

```
RMF (SP 800-37) → 7 pasos: Prepare, Categorize, Select, Implement, Assess, Authorize, Monitor.
FIPS 199 → categoriza bajo/moderado/alto por C, I y A; manda el más alto.
ATO → decisión formal del AO de que el riesgo residual es aceptable.
```

### NIST

**NIST SP 800-53** (Rev. 5) es un catálogo de controles de seguridad y privacidad que describe, con un identificador único cada uno, qué debe hacer un sistema u organización para protegerse; es la biblioteca de la que el RMF saca los controles en su paso Select.

Existe porque "proteger el sistema" necesita traducirse a requisitos concretos y verificables. Cada control tiene un identificador de familia y número, un enunciado, una discusión y posibles mejoras (enhancements), y se puede evaluar con los procedimientos de SP 800-53A.

Analogía: el catálogo de piezas de un fabricante de coches. No dice qué coche construir; dice qué piezas existen, con su código, y cada modelo (cada línea base) elige las suyas.

Las 20 familias de la Rev. 5, con su sigla:

- AC Access Control (control de acceso)
- AT Awareness and Training (concienciación y formación)
- AU Audit and Accountability (auditoría y registro)
- CA Assessment, Authorization, and Monitoring (evaluación, autorización y monitoreo)
- CM Configuration Management (gestión de configuración)
- CP Contingency Planning (planes de contingencia)
- IA Identification and Authentication (identificación y autenticación)
- IR Incident Response (respuesta a incidentes)
- MA Maintenance (mantenimiento)
- MP Media Protection (protección de soportes)
- PE Physical and Environmental Protection (protección física y ambiental)
- PL Planning (planificación)
- PM Program Management (gestión del programa)
- PS Personnel Security (seguridad del personal)
- PT PII Processing and Transparency (tratamiento de datos personales y transparencia)
- RA Risk Assessment (evaluación de riesgos)
- SA System and Services Acquisition (adquisición de sistemas y servicios)
- SC System and Communications Protection (protección de sistemas y comunicaciones)
- SI System and Information Integrity (integridad de sistemas e información)
- SR Supply Chain Risk Management (riesgo de la cadena de suministro)

Ejemplo de lectura de controles: `AC-2` es Account Management (altas, bajas y revisión de cuentas); `IA-2(1)` es la mejora 1 de IA-2: MFA para cuentas privilegiadas; `RA-5` es Vulnerability Monitoring and Scanning; `SI-2` es Flaw Remediation (parcheo). Un auditor que revisa `AC-2` pedirá, por ejemplo, la lista de cuentas dadas de baja el último trimestre y comprobará que todas las salidas de personal se desactivaron en menos del plazo fijado por la política.

Mención aparte, **NIST SP 800-61** es la guía de NIST para gestionar incidentes de seguridad. La Rev. 2 (2012) definía el ciclo de cuatro fases (Preparación; Detección y análisis; Contención, erradicación y recuperación; Actividad posterior al incidente); la Rev. 3 (abril de 2025) la sustituye y reorganiza la respuesta según las funciones de CSF 2.0. Se desarrolla en [`16-respuesta-a-incidentes-y-forense`](../16-respuesta-a-incidentes-y-forense/).

```
SP 800-53 → catálogo de controles; 20 familias; ID familia-número(mejora), p. ej. IA-2(1).
SP 800-53B → líneas base baja, moderada y alta.
SP 800-53A → cómo evaluar cada control.
SP 800-61 → gestión de incidentes; Rev. 3 alineada con CSF 2.0.
```

### CIS

Los **CIS Critical Security Controls** (versión 8, con la actualización 8.1 de 2024) son un conjunto priorizado de 18 controles y 153 salvaguardas que indican qué hacer primero para frenar los ataques más comunes; los **CIS Benchmarks** son guías de configuración segura, paso a paso, para productos concretos (Windows, Ubuntu, Apache, AWS, Kubernetes y más de cien tecnologías).

Existen porque marcos como 800-53 son enormes y una pyme no sabe por dónde empezar. CIS parte de datos de ataques reales y ordena: si solo puedes hacer 56 cosas, haz estas.

Analogía: los Controls son la lista de "qué revisar antes de un viaje largo" (frenos, neumáticos, aceite, en ese orden); los Benchmarks son el manual de taller de tu modelo exacto de coche, con el par de apriete de cada tornillo.

Los 18 controles de la v8, en orden:

1. Inventory and Control of Enterprise Assets (inventario de activos)
2. Inventory and Control of Software Assets (inventario de software)
3. Data Protection (protección de datos)
4. Secure Configuration of Enterprise Assets and Software (configuración segura)
5. Account Management (gestión de cuentas)
6. Access Control Management (gestión de accesos)
7. Continuous Vulnerability Management (gestión continua de vulnerabilidades)
8. Audit Log Management (gestión de logs)
9. Email and Web Browser Protections (protección de correo y navegador)
10. Malware Defenses (defensa contra malware)
11. Data Recovery (recuperación de datos)
12. Network Infrastructure Management (gestión de la infraestructura de red)
13. Network Monitoring and Defense (monitoreo y defensa de red)
14. Security Awareness and Skills Training (concienciación y formación)
15. Service Provider Management (gestión de proveedores)
16. Application Software Security (seguridad del software de aplicación)
17. Incident Response Management (gestión de respuesta a incidentes)
18. Penetration Testing (pruebas de penetración)

El orden no es casual: no puedes proteger lo que no sabes que tienes, por eso los controles 1 y 2 van primero. Las salvaguardas se agrupan en tres Implementation Groups:

- IG1: 56 salvaguardas, la "higiene básica" que toda organización debería cumplir, pensada para una pyme con poco personal de IT.
- IG2: IG1 + 74 = 130, para organizaciones con datos sensibles y un equipo de IT dedicado.
- IG3: IG2 + 23 = las 153, para organizaciones con datos muy sensibles y equipos de seguridad especializados.

Los Benchmarks tienen dos perfiles: Level 1 (ajustes básicos que casi no afectan al funcionamiento) y Level 2 (defensa en profundidad para entornos muy sensibles, que puede romper alguna funcionalidad). Se pueden comprobar automáticamente con herramientas como CIS-CAT o, de forma aproximada, con auditores libres como Lynis.

Ejemplo: una recomendación típica del benchmark de Ubuntu es "asegurar que el acceso SSH como root está deshabilitado". Comprobarla:

```
$ sudo sshd -T | grep permitrootlogin
permitrootlogin yes          # no cumple
# tras poner "PermitRootLogin no" en /etc/ssh/sshd_config y recargar:
permitrootlogin no           # cumple
```

```
CIS Controls v8 → 18 controles, 153 salvaguardas, priorizados por ataques reales.
IG1 / IG2 / IG3 → 56 / 130 / 153 salvaguardas según madurez y sensibilidad.
CIS Benchmarks → guías de configuración segura por producto; Level 1 básico, Level 2 estricto.
```

### CSF

El **NIST CSF 2.0** (Cybersecurity Framework, publicado el 26 de febrero de 2024) es un marco voluntario que organiza los resultados de ciberseguridad que cualquier organización debería alcanzar en seis funciones, sin imponer cómo lograrlos.

Existe porque las directivas y los equipos técnicos no hablaban el mismo idioma. El CSF da un vocabulario común de alto nivel para que la dirección entienda el programa, que se pueda medir la situación actual frente a la deseada y que se mapee a catálogos más detallados (800-53, ISO 27001, CIS). La versión 1.1 se pensó para infraestructura crítica; la 2.0 está pensada para organizaciones de cualquier tamaño y sector y añade la función Govern.

Analogía: un plan de salud personal con seis áreas (decidir objetivos y presupuesto, conocerte, prevenir, detectar síntomas, tratar y recuperarte). No dice qué medicamento tomar; dice qué tienes que tener cubierto.

Las seis funciones, en orden:

1. Govern (GV): establecer y supervisar la estrategia, expectativas, roles, políticas y la gestión del riesgo de ciberseguridad, incluida la cadena de suministro. Rodea a las otras cinco.
2. Identify (ID): entender los activos, los riesgos actuales y las oportunidades de mejora.
3. Protect (PR): aplicar salvaguardas: identidad y acceso, formación, seguridad de datos y de plataformas, resiliencia de la infraestructura.
4. Detect (DE): encontrar y analizar posibles ataques y compromisos: monitoreo continuo y análisis de eventos.
5. Respond (RS): actuar ante un incidente detectado: gestión, análisis, comunicación y mitigación.
6. Recover (RC): restaurar activos y operaciones afectados y comunicar la recuperación.

```
                 +-------------------------+
                 |        GOVERN (GV)      |
                 |  +-------------------+  |
                 |  | IDENTIFY  PROTECT |  |
                 |  | RECOVER   DETECT  |  |
                 |  |     RESPOND       |  |
                 |  +-------------------+  |
                 +-------------------------+
```

Estructura: cada función se divide en categorías (22 en total) y estas en subcategorías (106), que son resultados concretos; por ejemplo, PR.AA cubre gestión de identidad, autenticación y control de acceso. Dos herramientas de uso:

- Perfiles (Profiles): el perfil actual describe qué resultados se logran hoy; el perfil objetivo, cuáles se quieren lograr. La diferencia es el plan de trabajo.
- Niveles (Tiers): describen el rigor de la gestión del riesgo, del 1 al 4: Partial (parcial), Risk Informed (informado por el riesgo), Repeatable (repetible), Adaptive (adaptativo). No son una nota de madurez obligatoria; son una referencia para fijar la meta.

Ejemplo: una clínica con 30 empleados evalúa su perfil actual y ve que en Recover no tiene backups probados (RC sin cubrir) y en Govern nadie es responsable de la seguridad. Su perfil objetivo para el año próximo fija: un responsable nombrado (GV.RR), backups 3-2-1 con prueba trimestral de restauración (PR y RC) y pasar de Tier 1 a Tier 2.

```
CSF 2.0 → marco voluntario de resultados; 6 funciones, 22 categorías, 106 subcategorías.
Govern / Identify / Protect / Detect / Respond / Recover → GV, ID, PR, DE, RS, RC.
Perfil actual vs objetivo → dónde estoy y a dónde voy.
Tiers 1-4 → Partial, Risk Informed, Repeatable, Adaptive.
```

### Los estándares se complementan: uno dice qué lograr, otro qué controles y otro cómo configurarlos

> [!IMPORTANT]
> ISO 27001, RMF, SP 800-53, CIS y CSF no compiten: responden a preguntas distintas del mismo programa de seguridad, y una organización madura usa varios a la vez mapeados entre sí.

La complementariedad de los estándares es una idea práctica que dice que cada documento ocupa una capa: el CSF fija qué resultados perseguir, ISO 27001 y RMF fijan cómo gestionar el programa y demostrarlo, SP 800-53 y CIS Controls dicen qué controles concretos poner y los CIS Benchmarks dicen cómo dejar configurado cada producto.

```
Qué lograr (resultados)        → NIST CSF 2.0
Cómo gestionar y demostrarlo   → ISO 27001 (certificado) / RMF (ATO)
Qué controles                  → SP 800-53, ISO 27002, CIS Controls
Cómo configurar cada producto  → CIS Benchmarks
```

```
                 ISO 27001     RMF            800-53        CIS Controls   CSF 2.0
Organismo        ISO/IEC       NIST           NIST          CIS            NIST
Tipo             sistema de    proceso de     catálogo de   lista          marco de
                 gestión       7 pasos        controles     priorizada     resultados
¿Obligatorio?    voluntario,   sí, gobierno   sí, gobierno  voluntario     voluntario
                 lo exigen     federal EE.UU. federal EE.UU.
                 clientes
Resultado        certificado   ATO            controles     IG cumplido    perfil actual
formal           3 años                       evaluados                    vs objetivo
```

La fila "Resultado formal" es distinta en cada columna: ahí está la idea entera. Solo ISO 27001 da un certificado de un tercero; RMF da una autorización interna; los demás son guías con las que medirse.

Ejemplo: una empresa usa el CSF para presentar a su consejo el estado por funciones, mantiene ISO 27001 porque se lo piden sus clientes, toma los controles concretos de CIS IG2 y aplica los CIS Benchmarks Level 1 a todos sus servidores. NIST publica mapeos oficiales (Informative References) entre CSF 2.0 y 800-53, ISO y CIS, así que una misma evidencia sirve para varios marcos.

Límite: mapear no significa equivalencia exacta; un control de CIS puede cubrir solo parte de una subcategoría del CSF.

```
CSF → qué lograr; ISO 27001 / RMF → cómo gestionarlo y demostrarlo.
800-53 / ISO 27002 / CIS Controls → qué controles; CIS Benchmarks → cómo configurar.
```

## Roles of Compliance and Auditors

El **cumplimiento** (compliance) es la función de una organización que asegura que se cumplen las obligaciones externas e internas que le aplican (leyes, regulaciones, contratos, estándares y políticas propias) y que puede demostrarlo con evidencias; el **auditor** es la persona o entidad independiente que examina esas evidencias y emite una opinión formal sobre si los controles existen y funcionan.

Existen porque una organización no puede ser juez de sí misma. Los reguladores, clientes y accionistas necesitan confiar en que lo que la empresa dice hacer es cierto, y esa confianza viene de una separación de funciones: unos operan, otros vigilan que se cumpla y otros, independientes, verifican.

Analogía: un restaurante. La cocina opera; el encargado de calidad comprueba cada día que se cumplen las normas de higiene y guarda los registros de temperatura de las neveras; el inspector de sanidad llega sin depender del restaurante, revisa los registros y la cocina y emite un informe que puede cerrar el local.

Obligaciones típicas que gestiona cumplimiento:

- Leyes de privacidad: GDPR en la Unión Europea (multas de hasta 20 millones de euros o el 4 % de la facturación anual mundial, lo que sea mayor), HIPAA para datos de salud en EE. UU.
- Estándares contractuales: PCI DSS (versión 4.0) para quien procesa tarjetas de pago; SOC 2 para proveedores de servicios en la nube.
- Regulaciones sectoriales y financieras: SOX para la información financiera de empresas cotizadas en EE. UU.
- Estándares voluntarios exigidos por clientes: ISO 27001.

Qué hace el equipo de cumplimiento:

- Inventariar las obligaciones aplicables y traducirlas a controles y políticas (un mismo control, como MFA, suele cubrir varias obligaciones a la vez).
- Asignar dueños a cada control y recoger evidencias de forma continua.
- Vigilar cambios regulatorios, formar al personal y preparar las auditorías.
- Gestionar las no conformidades y los planes de remediación.

Qué hace el auditor:

- Interno: empleado de la organización pero independiente de las áreas que audita; reporta al comité de auditoría del consejo, no a quien dirige la operación.
- Externo: tercero independiente: un organismo certificador para ISO 27001, una firma de contadores (CPA) para SOC 2, un QSA (Qualified Security Assessor) para PCI DSS, o el propio regulador.

El modelo de las tres líneas (del Institute of Internal Auditors) ordena estos roles:

```
1.ª línea  Operación y gestión (IT, desarrollo)   → posee el riesgo y opera los controles
2.ª línea  Riesgo y cumplimiento                  → define marcos, supervisa, asesora
3.ª línea  Auditoría interna                      → verifica de forma independiente
           Auditoría externa y reguladores        → verificación desde fuera
```

Proceso típico de una auditoría:

1. Planificación: alcance, criterios (contra qué estándar), calendario.
2. Trabajo de campo: el auditor obtiene evidencia por entrevista, inspección de documentos, observación y reejecución (repetir el control él mismo), normalmente sobre una muestra.
3. Hallazgos: cada desviación se clasifica (en ISO, no conformidad mayor o menor; además, observaciones y oportunidades de mejora).
4. Informe: opinión formal y hallazgos.
5. Remediación y seguimiento: la organización presenta un plan de acción con fechas y el auditor verifica que se cerró.

Ejemplo: en una auditoría SOC 2, el control dice "los accesos de los empleados que se van se revocan en menos de 24 horas". El auditor pide la lista de 40 bajas del año, toma una muestra de 25 y compara la fecha de salida en RR. HH. con la fecha de desactivación de la cuenta. Encuentra 2 cuentas desactivadas a los 5 días: es una excepción que aparece en el informe, y la empresa automatiza la baja desde el sistema de RR. HH.

SOC 2 tiene dos tipos que se confunden a menudo: Type I evalúa si los controles están bien diseñados en una fecha concreta; Type II evalúa si funcionaron de verdad durante un periodo (normalmente de 3 a 12 meses). Los clientes serios piden Type II.

```
Cumplimiento → asegura y demuestra que se cumplen leyes, contratos y estándares.
Auditor interno → independiente de la operación; reporta al comité de auditoría.
Auditor externo → tercero: certificador ISO, CPA para SOC 2, QSA para PCI DSS.
Tres líneas → operación posee el riesgo; riesgo y cumplimiento supervisa; auditoría verifica.
SOC 2 Type I → diseño en una fecha; Type II → funcionamiento durante un periodo.
No conformidad mayor / menor → fallo sistémico o ausencia de control / fallo puntual.
```

### Cumplir no es lo mismo que estar seguro

> [!IMPORTANT]
> El cumplimiento demuestra que se satisface un conjunto mínimo de requisitos en el momento auditado; no garantiza que la organización resista un ataque. Hay empresas certificadas que sufren brechas graves.

Esta distinción es una idea de gobierno que dice que el cumplimiento es un suelo y no un techo: los estándares van por detrás de las amenazas, la auditoría mira una muestra y una fecha, y un control puede existir en papel y funcionar mal en la práctica.

```
Seguridad    → ¿resistimos, detectamos y nos recuperamos de un ataque real?
Cumplimiento → ¿satisfacemos los requisitos que nos aplican y podemos demostrarlo?
```

```
                       Organización A        Organización B
Certificado ISO        sí                    no
Parches críticos       90 días de media      7 días de media
Pruebas de restauración en papel             mensuales
Resistencia real       baja                  alta
```

La última fila es opuesta a la primera: el certificado no predice la resistencia. Ejemplo cotidiano: tener la revisión técnica del coche al día no impide que los frenos fallen tres meses después si nadie los revisa.

Límite: el cumplimiento sí aporta algo real: obliga a un mínimo, a documentar y a revisar periódicamente. La regla es usar los requisitos como punto de partida y medir la seguridad con pruebas reales (red team, simulacros de restauración, métricas de detección, ver [`09-conceptos-de-seguridad`](../09-conceptos-de-seguridad/)).

```
Cumplimiento → suelo mínimo demostrable; seguridad → resistencia real.
```

## Basics of Vulnerability Management

La **gestión de vulnerabilidades** es un proceso continuo y cíclico que descubre los activos de una organización, identifica sus vulnerabilidades, las prioriza según el riesgo, las corrige o mitiga y verifica la corrección, para reducir de forma sostenida la superficie de ataque.

Existe porque cada año se publican decenas de miles de vulnerabilidades nuevas (más de 40 000 CVE en 2024) y ninguna organización puede parchearlo todo de inmediato. Sin un proceso, se parchea lo que hace ruido en las noticias y se olvida lo que de verdad está expuesto. Es el control 7 de CIS (Continuous Vulnerability Management) y el `RA-5` de 800-53.

Analogía: el mantenimiento de una flota de 100 furgonetas. Primero se sabe cuántas hay y dónde están, se revisan periódicamente, se arregla primero lo que puede causar un accidente en las que más circulan, se comprueba que el arreglo quedó bien y se repite.

### Ciclo de gestión de vulnerabilidades

El ciclo es una secuencia de seis fases que se repite sin fin:

```
   +--> 1. Descubrir activos ---> 2. Identificar (escanear) ---+
   |                                                         |
   6. Informar y mejorar                         3. Evaluar y priorizar
   |                                                         |
   +--- 5. Verificar (reescanear) <--- 4. Remediar -----------+
```

1. Descubrir: inventario de activos (servidores, equipos, nube, contenedores, aplicaciones), con dueño y criticidad. No se puede escanear lo que no se sabe que existe.
2. Identificar: escaneo con herramientas y otras fuentes (avisos de fabricantes, inventario de software contra bases de datos de vulnerabilidades).
3. Evaluar y priorizar: confirmar hallazgos, descartar falsos positivos y ordenar por riesgo.
4. Remediar: elegir entre corregir (parche, cambio de configuración), mitigar (control compensatorio: bloquear el puerto, regla de WAF, aislar) o aceptar formalmente el riesgo con fecha de revisión.
5. Verificar: volver a escanear para confirmar que la corrección funcionó.
6. Informar y mejorar: métricas (tiempo medio de remediación, porcentaje dentro de plazo, vulnerabilidades críticas abiertas) y ajuste del proceso.

Las políticas suelen fijar plazos de remediación por severidad; un ejemplo habitual es 15 días para críticas, 30 para altas, 90 para medias y 180 para bajas. La guía de NIST para el parcheo es SP 800-40 Rev. 4.

```
Corregir → parche o cambio de configuración: elimina la vulnerabilidad.
Mitigar → control compensatorio: la vulnerabilidad sigue, pero no se puede explotar o duele menos.
Aceptar → decisión formal y firmada, con fecha de revisión.
```

### CVE

**CVE** (Common Vulnerabilities and Exposures) es un sistema de identificadores públicos y únicos para vulnerabilidades conocidas, que permite que todas las herramientas, fabricantes y analistas se refieran al mismo fallo con el mismo nombre.

Existe porque antes de 1999 cada fabricante y cada escáner llamaba distinto a la misma vulnerabilidad, y era imposible saber si dos informes hablaban del mismo problema. El programa lo opera MITRE con financiación de CISA, y los identificadores los asignan las CNA (CVE Numbering Authorities): fabricantes como Microsoft o Red Hat, investigadores y CERT.

Formato: `CVE-AAAA-NNNN`, donde AAAA es el año de asignación y NNNN un número de 4 dígitos o más. Ejemplo: `CVE-2021-44228` es Log4Shell, la ejecución remota de código en la librería Log4j.

Términos vecinos:

- NVD (National Vulnerability Database, de NIST): enriquece cada CVE con puntuación CVSS, productos afectados (CPE) y tipo de fallo (CWE).
- CWE (Common Weakness Enumeration): el tipo de error, no el caso concreto. CWE-79 es XSS; CVE-2021-44228 es una instancia concreta que NVD clasifica, entre otras, como CWE-917 (inyección de expresiones).
- CPE: nombre estandarizado de un producto y versión, como `cpe:2.3:a:apache:log4j:2.14.1`.

Analogía: el CVE es la matrícula de un coche concreto; el CWE es el modelo de coche; el CPE es la ficha técnica de qué coches afecta.

```
CVE → identificador único de una vulnerabilidad concreta (CVE-AAAA-NNNN).
CWE → tipo de debilidad (la categoría de error).
CPE → nombre estandarizado de producto y versión afectados.
NVD → base de NIST que enriquece los CVE con CVSS, CPE y CWE.
```

### CVSS

**CVSS** (Common Vulnerability Scoring System, mantenido por FIRST) es un sistema de puntuación de 0,0 a 10,0 que mide la gravedad técnica de una vulnerabilidad a partir de cómo se explota y qué daño produce.

Existe para dar un lenguaje numérico común: un "es grave" no se puede comparar, un 9,8 sí. Las versiones en uso son la 3.1 (la más extendida) y la 4.0 (publicada en noviembre de 2023).

Rangos de severidad (iguales en 3.1 y 4.0):

```
0,0          → None (ninguna)
0,1 - 3,9    → Low (baja)
4,0 - 6,9    → Medium (media)
7,0 - 8,9    → High (alta)
9,0 - 10,0   → Critical (crítica)
```

Grupos de métricas: en 3.1, Base (propiedades intrínsecas del fallo), Temporal (madurez del exploit, existencia de parche) y Environmental (ajustes según tu entorno); en 4.0, Base, Threat, Environmental y Supplemental. El valor que publica NVD es casi siempre solo el Base.

Métricas Base de CVSS 3.1, cada una con sus valores:

- AV, Attack Vector: N (red), A (red adyacente), L (local), P (físico). Cuanto más lejos puede estar el atacante, más alta la nota.
- AC, Attack Complexity: L (baja), H (alta, requiere condiciones fuera del control del atacante).
- PR, Privileges Required: N (ninguno), L (bajos), H (altos).
- UI, User Interaction: N (ninguna), R (requiere que la víctima haga algo, como abrir un archivo).
- S, Scope: U (sin cambio), C (changed: el fallo en un componente afecta a otro, como escapar de un sandbox).
- C, I, A: impacto en confidencialidad, integridad y disponibilidad: H (alto), L (bajo), N (ninguno).

Ejemplo de lectura del vector de Log4Shell:

```
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:H   →  10,0 Critical
  AV:N  explotable por red
  AC:L  sin condiciones especiales
  PR:N  sin cuenta previa
  UI:N  sin que la víctima haga nada
  S:C   afecta más allá del componente vulnerable
  C/I/A:H  control total
```

Si el mismo fallo exigiera estar logueado (PR:L) y que un usuario abriera un archivo (UI:R), la nota bajaría a la franja alta: el vector explica de dónde sale cada décima. Las calculadoras oficiales de FIRST permiten cambiar cada métrica y ver el efecto.

```
CVSS → gravedad técnica de 0,0 a 10,0; mantenido por FIRST.
Rangos → 0 None; 0,1-3,9 Low; 4,0-6,9 Medium; 7,0-8,9 High; 9,0-10,0 Critical.
Base / Temporal (Threat en 4.0) / Environmental → intrínseco / estado del exploit / tu entorno.
Vector → AV, AC, PR, UI, S, C, I, A: cada letra explica la nota.
```

### CVSS mide gravedad, no riesgo

> [!IMPORTANT]
> Un CVSS de 9,8 dice lo malo que sería el fallo en el peor caso genérico, no lo probable que es que te ataquen por él ni lo que te costaría; priorizar solo por CVSS lleva a parchear lo ruidoso y dejar abierto lo que de verdad se está explotando.

La diferencia entre gravedad y riesgo es una regla de priorización que dice que la nota Base de CVSS solo cubre el factor "impacto técnico" del riesgo; faltan la probabilidad de explotación y el valor y exposición del activo concreto.

```
Prioridad ≈ gravedad (CVSS) × probabilidad de explotación (EPSS, KEV) × criticidad y exposición del activo
```

Las fuentes que completan la imagen:

- CISA KEV (Known Exploited Vulnerabilities): catálogo de vulnerabilidades con explotación confirmada en el mundo real. Si está en KEV, va primero, sea cual sea su CVSS.
- EPSS (Exploit Prediction Scoring System, de FIRST): probabilidad de 0 a 1 de que una vulnerabilidad sea explotada en los próximos 30 días.
- Contexto propio: ¿el activo está expuesto a Internet?, ¿guarda datos sensibles?, ¿hay un control compensatorio?
- SSVC (Stakeholder-Specific Vulnerability Categorization, usado por CISA): árbol de decisión que termina en Track, Track*, Attend o Act.

```
                        Vuln. 1                  Vuln. 2                Vuln. 3
CVSS Base               9,8 Critical             7,5 High               9,1 Critical
En CISA KEV             no                       sí                     no
EPSS                    0,01                     0,95                   0,02
Activo                  servidor interno de      web pública con        equipo de laboratorio
                        pruebas, sin datos       datos de clientes      aislado
Prioridad real          media                    máxima                 baja
```

La fila de prioridad real no sigue a la fila de CVSS: la vulnerabilidad con la nota más baja es la primera que hay que parchear. Ejemplo cotidiano: una ventana sin cerrar en el piso 20 es un fallo "grave", pero la puerta de la calle sin llave en un barrio con robos esa misma semana es lo urgente.

Límite: CVSS sigue siendo útil como primer filtro y para comunicar; el error es usarlo como única variable.

```
CVSS → gravedad técnica genérica; no riesgo.
CISA KEV → explotación confirmada: va primero.
EPSS → probabilidad (0 a 1) de explotación en 30 días.
Prioridad → CVSS × probabilidad × criticidad y exposición del activo.
```

### Escáneres de vulnerabilidades

Un **escáner de vulnerabilidades** es una herramienta automatizada que examina sistemas, redes o aplicaciones, identifica versiones de software y configuraciones, y las compara con una base de datos de vulnerabilidades conocidas para producir una lista de hallazgos con su CVE y severidad.

Existen porque revisar a mano miles de equipos contra decenas de miles de CVE es imposible. Herramientas conocidas: Nessus (Tenable), OpenVAS / Greenbone (libre), Qualys, Rapid7 InsightVM; para aplicaciones web, OWASP ZAP y Burp Suite; para contenedores e imágenes, Trivy.

Tipos y decisiones:

- Sin credenciales (unauthenticated): ve el equipo como lo vería un atacante desde la red: puertos, banners de servicios. Rápido, pero se basa en versiones anunciadas y genera más falsos positivos.
- Con credenciales (authenticated): inicia sesión y lee los paquetes instalados y la configuración. Mucho más preciso; es lo recomendado para el inventario interno.
- Por agente: un pequeño programa instalado en cada equipo informa aunque el portátil esté fuera de la red corporativa.
- Interno vs externo: desde dentro de la red ve todo; desde Internet ve lo que ve un atacante externo.
- Escaneo de vulnerabilidades vs pentest: el escáner encuentra y lista fallos conocidos automáticamente; el pentest los explota manualmente para demostrar el impacto real.

Analogía: un control médico con análisis de sangre. El escáner sin credenciales es mirar al paciente desde fuera; el escáner con credenciales es el análisis de sangre: más molesto de montar, mucho más preciso.

Ejemplo contra una máquina virtual propia de laboratorio (nunca contra sistemas ajenos sin autorización escrita):

```
$ nmap -sV --script vulners -p 22,80 192.168.56.10
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| vulners:
|   cpe:/a:openbsd:openssh:7.4:
|       CVE-2023-38408   9.8
80/tcp open  http    Apache httpd 2.4.49
| vulners:
|   cpe:/a:apache:http_server:2.4.49:
|       CVE-2021-42013   9.8
|       CVE-2021-41773   7.5
```

Lectura con lo visto: los tres son críticos o altos por CVSS, pero las dos de Apache 2.4.49 (recorrido de rutas y ejecución remota) están en CISA KEV y el puerto 80 es el expuesto, así que se actualiza Apache primero; luego OpenSSH.

> [!WARNING]
> Un hallazgo basado solo en el número de versión puede ser un falso positivo: distribuciones como Red Hat o Debian aplican el parche sin cambiar la versión principal (backporting). Confirma con un escaneo con credenciales o con el aviso del fabricante antes de abrir el ticket.

```
Escáner de vulnerabilidades → compara versiones y configuración con una base de CVE.
Sin credenciales → visión del atacante; menos preciso, más falsos positivos.
Con credenciales → lee paquetes y configuración; mucho más preciso.
Agente → informa desde cada equipo, también fuera de la red.
Escaneo → lista fallos conocidos; pentest → los explota para demostrar impacto.
Backporting → parche aplicado sin cambiar la versión: causa falsos positivos.
```

## Recursos para aprender y practicar

### Videos

- [Security Standards - CompTIA Security+ SY0-701 - 5.1](https://www.youtube.com/watch?v=jBvdRpXaomk) — Professor Messer; panorama de estándares de seguridad que pide el examen. Nodo: Understand Common Standards.
- [What is ISO 27001? Simple Explanation with Examples](https://www.youtube.com/watch?v=Ga4drOX7w3w) — Dejan Kosutic (Advisera); ISMS, cláusulas y Anexo A en lenguaje llano. Nodo: ISO.
- [CertMike Explains NIST Risk Management Framework](https://www.youtube.com/watch?v=d8Z1RyhmRXg) — Mike Chapple; los siete pasos del RMF. Nodo: RMF.
- [The NIST Cybersecurity Framework (CSF) 2.0](https://www.youtube.com/watch?v=pPPiaGU12Og) — NIST (canal oficial); qué cambia en 2.0 y la función Govern. Nodo: CSF.
- [Strengthen Cybersecurity Posture with the CIS Critical Security Controls](https://www.youtube.com/watch?v=6OqnCxdncX4) — CIS (canal oficial); los controles y su priorización. Nodo: CIS.
- [Compliance - CompTIA Security+ SY0-701 - 5.4](https://www.youtube.com/watch?v=IjJf4jLtONQ) — Professor Messer; informes de cumplimiento, consecuencias del incumplimiento. Nodo: Roles of Compliance and Auditors.
- [Audits and Assessments - CompTIA Security+ SY0-701 - 5.5](https://www.youtube.com/watch?v=uo2Yw720mv4) — Professor Messer; auditoría interna, externa, atestación. Nodo: Roles of Compliance and Auditors.
- [CVE and CVSS explained | Security Detail](https://www.youtube.com/watch?v=oSyEGkX6sX0) — Red Hat; qué es un CVE, cómo se lee CVSS y por qué no basta para decidir. Nodo: Basics of Vulnerability Management.
- [CertMike Explains CVSS](https://www.youtube.com/watch?v=2zIQSzf7hQk) — Mike Chapple; métricas del vector CVSS. Nodo: Basics of Vulnerability Management.
- [Vulnerability Scanning - CompTIA Security+ SY0-701 - 4.3](https://www.youtube.com/watch?v=9B0mtWk_AM0) — Professor Messer; tipos de escaneo, con y sin credenciales. Nodo: Basics of Vulnerability Management.

### Lectura y documentación

- [ISO/IEC 27001:2022](https://www.iso.org/standard/27001) y [ISO/IEC 27002:2022](https://www.iso.org/standard/75652.html) — fichas oficiales de ISO con el resumen y el alcance de cada norma.
- [NIST SP 800-37 Rev. 2, Risk Management Framework](https://csrc.nist.gov/pubs/sp/800/37/r2/final) y [About the RMF](https://csrc.nist.gov/projects/risk-management/about-rmf) — los siete pasos y sus tareas.
- [NIST SP 800-53 Rev. 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final) — catálogo completo de controles y sus 20 familias.
- [NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final) — recomendaciones de respuesta a incidentes alineadas con CSF 2.0.
- [CIS Critical Security Controls v8](https://www.cisecurity.org/controls/v8) y [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks) — los 18 controles, los IG y las guías de configuración descargables.
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework) y [el documento CSF 2.0 (CSWP 29)](https://nvlpubs.nist.gov/nistpubs/CSWP/NIST.CSWP.29.pdf) — funciones, categorías, perfiles y niveles.
- [CVSS v3.1 Specification](https://www.first.org/cvss/v3.1/specification-document) y [CVSS v4.0 Specification](https://www.first.org/cvss/v4.0/specification-document) — definición oficial de cada métrica y rango.
- [CISA Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) y [FIRST EPSS](https://www.first.org/epss/) — las dos fuentes de probabilidad real de explotación.
- [NIST SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final) — el parcheo como mantenimiento preventivo planificado.

### Práctica

- [TryHackMe: Governance & Regulation](https://tryhackme.com/room/cybergovernanceregulation) — GRC, ISO 27001, SP 800-53 y su papel en la organización. Nodos: Understand Common Standards y Roles of Compliance and Auditors.
- [TryHackMe: Vulnerabilities 101](https://tryhackme.com/room/vulnerabilities101) — qué es una vulnerabilidad, cómo se puntúa y bases de datos como NVD. Nodo: CVE y CVSS.
- [TryHackMe: Vulnerability Scanning Tools](https://tryhackme.com/room/vulnerabilityscanningtools), [OpenVAS](https://tryhackme.com/room/openvas) y [Understanding Vulnerability Databases](https://tryhackme.com/room/understandingvulnerabilitydatabases) — gratis; escanear, leer CVE/CVSS y priorizar. Nodo: Basics of Vulnerability Management.
- [TryHackMe: OpenVAS](https://tryhackme.com/room/openvas) — montar y usar un escáner de vulnerabilidades. Nodo: Escáneres.
- [Ejercicio 1: Recalcula el vector CVSS de tres CVE](ejercicios.md#ejercicio-1-recalcula-el-vector-cvss-de-tres-cve) — reconstruye el vector 3.1 de 3 CVE de NVD y compara tu franja de severidad con la oficial.
- [Ejercicio 2: Escaneo con y sin credenciales usando Greenbone](ejercicios.md#ejercicio-2-escaneo-con-y-sin-credenciales-usando-greenbone) — escanea una VM propia sin y con credenciales y prioriza los 5 primeros con KEV y EPSS.
- [Ejercicio 3: Decide corregir, mitigar o aceptar con Nmap vulners](ejercicios.md#ejercicio-3-decide-corregir-mitigar-o-aceptar-con-nmap-vulners) — reproduce la salida de `vulners` contra tu VM y escribe una decisión de tratamiento por hallazgo.
- [Ejercicio 4: Audita tu Linux con Lynis y contrástalo con el CIS Benchmark](ejercicios.md#ejercicio-4-audita-tu-linux-con-lynis-y-contrástalo-con-el-cis-benchmark) — mira tu "hardening index" y mapea 5 sugerencias a controles CIS.
- [Ejercicio 5: Perfil CSF 2.0 actual y objetivo de tu casa](ejercicios.md#ejercicio-5-perfil-csf-20-actual-y-objetivo-de-tu-casa) — construye perfil actual y objetivo de tus dispositivos y deduce el plan de trabajo.

## Cuadro resumen

Todo lo visto, en una línea por término.

Understand Common Standards

```
ISO 27001 → requisitos del ISMS; certificable; cláusulas 4-10 + Anexo A.
ISO 27002 → guía de implantación de los 93 controles; no certificable.
Anexo A → 93 controles: organizacionales 37, personas 8, físicos 14, tecnológicos 34.
SoA → qué controles se aplican, cuáles no y por qué.
Certificación ISO → etapa 1 documental, etapa 2 práctica; 3 años con seguimiento anual.
RMF (SP 800-37) → 7 pasos: Prepare, Categorize, Select, Implement, Assess, Authorize, Monitor.
FIPS 199 → categoriza bajo/moderado/alto por C, I y A; manda el más alto.
ATO → decisión formal del AO de que el riesgo residual es aceptable.
SP 800-53 → catálogo de controles; 20 familias; ID familia-número(mejora), p. ej. IA-2(1).
SP 800-53B / 800-53A → líneas base baja-moderada-alta / cómo evaluar cada control.
SP 800-61 → gestión de incidentes; Rev. 3 alineada con CSF 2.0.
CIS Controls v8 → 18 controles, 153 salvaguardas, priorizados por ataques reales.
IG1 / IG2 / IG3 → 56 / 130 / 153 salvaguardas según madurez y sensibilidad.
CIS Benchmarks → guías de configuración segura por producto; Level 1 básico, Level 2 estricto.
CSF 2.0 → marco voluntario de resultados; 6 funciones, 22 categorías, 106 subcategorías.
Govern / Identify / Protect / Detect / Respond / Recover → GV, ID, PR, DE, RS, RC.
Perfil actual vs objetivo → dónde estoy y a dónde voy.
Tiers 1-4 → Partial, Risk Informed, Repeatable, Adaptive.
CSF → qué lograr; ISO 27001 / RMF → cómo gestionarlo y demostrarlo.
800-53 / ISO 27002 / CIS Controls → qué controles; CIS Benchmarks → cómo configurar.
```

Roles of Compliance and Auditors

```
Cumplimiento → asegura y demuestra que se cumplen leyes, contratos y estándares.
Auditor interno → independiente de la operación; reporta al comité de auditoría.
Auditor externo → tercero: certificador ISO, CPA para SOC 2, QSA para PCI DSS.
Tres líneas → operación posee el riesgo; riesgo y cumplimiento supervisa; auditoría verifica.
Auditoría → planificar, trabajo de campo con muestras, hallazgos, informe, seguimiento.
SOC 2 Type I → diseño en una fecha; Type II → funcionamiento durante un periodo.
No conformidad mayor / menor → fallo sistémico o ausencia de control / fallo puntual.
GDPR → hasta 20 M€ o 4 % de la facturación mundial.
Cumplimiento → suelo mínimo demostrable; seguridad → resistencia real.
```

Basics of Vulnerability Management

```
Gestión de vulnerabilidades → ciclo continuo: descubrir, identificar, priorizar, remediar, verificar, informar.
Corregir / Mitigar / Aceptar → parche / control compensatorio / decisión formal con fecha.
Plazos típicos → crítica 15 días, alta 30, media 90, baja 180 (según política).
CVE → identificador único de una vulnerabilidad concreta (CVE-AAAA-NNNN).
CWE → tipo de debilidad; CPE → producto y versión; NVD → base de NIST que los enriquece.
CVSS → gravedad técnica de 0,0 a 10,0; mantenido por FIRST.
Rangos CVSS → 0 None; 0,1-3,9 Low; 4,0-6,9 Medium; 7,0-8,9 High; 9,0-10,0 Critical.
Vector CVSS 3.1 → AV, AC, PR, UI, S, C, I, A.
CISA KEV → explotación confirmada: va primero.
EPSS → probabilidad (0 a 1) de explotación en 30 días.
Prioridad → CVSS × probabilidad × criticidad y exposición del activo.
Escáner sin credenciales → visión del atacante; con credenciales → mucho más preciso.
Escaneo → lista fallos conocidos; pentest → los explota para demostrar impacto.
Backporting → parche sin cambio de versión: causa falsos positivos.
```
