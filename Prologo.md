# LEMA DEL PROYECTO: **RAILGUARD‑V**

## 1. DESCRIPCIÓN DEL PROYECTO

**RAILGUARD‑V** es una solución embarcada de **geolocalización resiliente** para **un tren** que mantiene una estimación **continua y con integridad** de su posición sobre la vía en escenarios de **conectividad degradada** y **GNSS degradado o atacado** (jamming/spoofing).  

La propuesta integra en una única arquitectura:

1) **GNSS con detección de jamming/spoofing** y evaluación de integridad (solo se usa cuando es consistente).  
2) **Localización visual** con **cámara frontal** + **mapa visual ligero offline**, sin dependencia de red.  
3) **Odometría disponible** (distancia/velocidad por eje), usada como columna vertebral en GNSS‑denied (túneles) y como referencia dinámica para detectar GNSS anómalo.  
4) **Posicionamiento colaborativo tren–tren (V2V)** con intercambio mínimo, tolerante a cortes y fusión conservadora.

**Salida principal** (para explotación/operación y analítica):
- Posición “sobre vía”: **coordenada curvilínea** $$s$$ (equivalente a PK/distance‑along‑track) + **identificador de rama/vía** (track ID) en zonas con varias vías.
- Velocidad longitudinal $$\dot{s}$$.
- **Incertidumbre** (intervalos 95% y/o covarianza).
- Estado GNSS (OK/degradado/jamming sospechado/spoofing sospechado) y registro de evidencias.

---

## 2. CONTEXTO FERROVIARIO (PARA INGENIEROS NO FERROVIARIOS)

Un tren está **confinado** a una red de vías (topología tipo grafo). Esto permite estimar posición en coordenadas sobre la infraestructura:

- $$s$$: distancia acumulada sobre el eje de la vía (1D sobre una rama).
- $$k$$: rama/vía activa (discreto) especialmente en estaciones, apartaderos y bifurcaciones.

En entornos como túneles o estaciones cubiertas, el **GNSS** puede:
- perderse (insuficientes satélites), o
- parecer “bueno” pero ser falso (spoofing), o
- volverse ruidoso (jamming).

La **odometría** ferroviaria (por ejes) aporta continuidad en esos tramos, pero:
- puede degradarse con patinaje/deslizamiento (especialmente en aceleración/frenado y condiciones adversas),
- requiere calibración (diámetro efectivo rueda, etc.).

La visión frontal aporta “anclas” adicionales al reconocer elementos del entorno. Y el V2V permite que trenes con buena estimación ayuden a otros cuando hay ventanas de comunicación.

---

## 3. ARQUITECTURA COMPLETA (EMBARCADA + MAPAS + V2V)

### 3.1 Diagrama lógico (texto)
**Sensores → Extracción de evidencias → Fusión con integridad → Salidas + logging**
- GNSS → métricas GNSS + posición/velocidad
- Odometría → $$\Delta s$$, $$\dot{s}$$
- Cámara frontal → landmarks + calidad; (opcional) odometría visual
- V2V → estimaciones de otros trenes + incertidumbre + eventos

Todo converge en un **Motor de Fusión** con:
- **Map‑matching** a red ferroviaria (geometría/topología),
- **gestión de incertidumbre** y rechazo de outliers,
- **detección de GNSS anómalo** (jamming/spoofing),
- **fusión conservadora** de V2V (evitar sobreconfianza).

### 3.2 Componentes embarcados (tren piloto)
**Entradas**
- GNSS multi‑constelación (integración con receptor existente).
- **Odometría disponible** (señal de velocidad/distancia; resolución y frecuencia según tren).
- Cámara frontal (1080p; WDR deseable).
- (Recomendable) IMU compacta: mejora dinámica e integridad, pero el diseño base se apoya en odometría+visión.

**Módulos**
1. **GIM – GNSS Integrity Monitor**  
2. **VLM – Visual Localization Module** (landmarks + asociación a mapa)  
3. **ODM – Odometer Monitor** (calidad, detección de patinaje/deslizamiento)  
4. **CMM – Cooperative Messaging Module (V2V)**  
5. **FUS – Fusion Engine** (estimación $$s,k,\dot{s}$$ con incertidumbre)  
6. **Logger / Replay** para validación y mejora iterativa.

### 3.3 Mapas (offline, ligeros)
- **Mapa geométrico de vía**: ejes, PK, topología (bifurcaciones), “corredor” de tolerancia.  
- **Mapa visual ligero**: conjunto de landmarks por tramo (con descriptores compactos y PK aproximada).  
  - Se descarga/actualiza cuando hay conectividad, pero **opera 100% offline**.

---

## 4. FUENTES DE DATOS Y OBSERVACIONES (DETALLE)

### 4.1 GNSS (posición/velocidad + calidad)
**Observación**: $$z_{gnss} = (x,y,v,\dots)$$ + indicadores (C/N0, DOP, #sat, flags).  
**Uso**: ancla cuando supera tests de integridad; si no, se descarta o se de‑pondera.

### 4.2 Odometría (disponible)
**Observación**:
- incremento longitudinal $$\Delta s_{odo}$$ y/o velocidad $$\dot{s}_{odo}$$ a alta frecuencia.

**Monitor de calidad (ODM)**
- Detecta intervalos de posible patinaje/deslizamiento mediante:
  - inconsistencias con visión (cuando hay),
  - dinámica longitudinal (cambios bruscos improbables),
  - reglas ferroviarias (aceleración típica máxima, etc., parametrizable).
- Ajusta el modelo de ruido de la odometría (covarianza) en el filtro.

**Rol en GNSS‑denied**
- Es la fuente principal de “propagación” del estado $$s$$ en túneles.
- Se corrige periódicamente por visión y por re‑anclajes GNSS válidos.

### 4.3 Visión: landmarks + (opcional) odometría visual
**Landmarking (principal)**
- Detecta y describe hitos: señales, pórticos, postes, entradas de túnel, estructuras singulares.
- Hace asociación al mapa visual con “gating” por:
  - hipótesis de tramo $$s$$ (del filtro),
  - secuencia temporal (no solo detección aislada),
  - plausibilidad topológica (rama/vía).

**Odómetro visual (opcional)**
- Estima movimiento relativo; ayuda cuando hay pocos landmarks, pero su calidad depende de textura/iluminación.

### 4.4 V2V (colaboración con mensajes mínimos)
**Mensaje mínimo**
- timestamp, $$\hat{s}$$, $$\hat{k}$$, $$\dot{\hat{s}}$$
- incertidumbre (covarianza comprimida o intervalos)
- eventos (opcional): spoofing sospechado, paso por landmark fuerte.

**Tolerancia a cortes**
- Envío oportunista + almacenamiento local.
- La fusión trata mensajes retrasados con incertidumbre inflada.

---

## 5. ALGORITMO (FILTRO) Y GESTIÓN DE INCERTIDUMBRE

### 5.1 Estado y representación
- Estado continuo: $$\mathbf{x} = (s, \dot{s}, b_{odo})$$  
  donde $$b_{odo}$$ representa sesgo/escala (calibración efectiva de odometría) estimable online.  
- Estado discreto: $$k$$ (rama/vía). Se mantiene como multi‑hipótesis cuando hay ambigüedad.

### 5.2 Propagación (predicción)
En cada paso:
- $$s_{t+1} = s_t + (\Delta s_{odo} \cdot \alpha) + w_s$$  
- $$\dot{s}_{t+1} = \dot{s}_{odo} + w_v$$  
- $$\alpha$$ (o $$b_{odo}$$) se ajusta lentamente para capturar variaciones (diámetro efectivo, etc.).  
El ruido $$w_s, w_v$$ se hace **adaptativo** según el monitor ODM (más incertidumbre si patinaje probable).

### 5.3 Actualización con observaciones (correcciones)
- **GNSS**: se proyecta a la red ferroviaria (map‑matching) y corrige $$s,k$$ si pasa integridad.  
- **Visión (landmarks)**: cada match aporta una observación de $$s$$ (y a veces de $$k$$) con covarianza basada en calidad.  
- **V2V**: se combina estimación externa con **fusión conservadora**.

### 5.4 Detección de jamming/spoofing (GIM)
Se ejecutan tests combinados:

**Indicadores de jamming**
- caída sostenida de C/N0, pérdida de satélites, DOP alto, aumento de varianza.

**Indicadores de spoofing**
- GNSS consistente internamente pero **inconsistente** con:
  - odometría (saltos de $$s$$ no explicables por $$\Delta s_{odo}$$),
  - visión (landmarks incompatibles),
  - mapa (fuera del corredor, cambio de vía imposible).
- análisis temporal (saltos abruptos o “teleport”).

**Decisión**
- Modo GNSS‑OK: GNSS entra con su covarianza nominal.
- Modo GNSS‑degradado/jamming: GNSS entra con covarianza inflada o se ignora.
- Modo spoofing sospechado: GNSS se **rechaza**; se registra evento; navegación continúa con odometría+visión (+V2V).

### 5.5 Fusión V2V (conservadora)
Para evitar sobreconfianza (correlaciones desconocidas), se aplica **Covariance Intersection (CI)** o equivalente:
- combina $$\hat{s}$$ y covarianzas sin suponer independencia,
- garantiza que la incertidumbre resultante no sea artificialmente baja.

---

## 6. TRL OBJETIVO

**TRL objetivo del piloto**: **TRL 6**  
Prototipo integrado (GNSS‑integridad + odometría + visión + V2V) validado en entorno relevante Renfe, con métricas cuantitativas de precisión, continuidad e integridad.

**Evolución esperada**
- Tras piloto y endurecimiento (industrialización, monitorización, procesos de actualización de mapa): TRL 7.

---

## 7. PILOTO EN RENFE (1 TREN): ALCANCE, HITOS, MÉTRICAS

### 7.1 Alcance
- Instalación en **un tren** (unidad piloto) usando **odometría existente** + cámara frontal + GNSS.
- Selección de 1–2 corredores con:
  - túneles (GNSS‑denied),
  - estaciones complejas (multi‑vía),
  - urbano (multipath) y zonas abiertas (baseline).

**Nota de validación**: para medir precisión se requiere una “verdad terreno” durante pruebas (p. ej. solución de referencia temporal, mediciones puntuales o dataset de comparación). Se define conjuntamente en el arranque.

### 7.2 Hitos (orientativo, ajustable a calendario de Renfe)
**H0 – Definición (Semanas 1–2)**
- Línea(s) piloto, permisos, inventario de señales de odometría y GNSS, montaje cámara.
- Criterios de éxito y plan de seguridad/ciberseguridad de pruebas.

**H1 – Integración base (Semanas 3–7)**
- Integración odometría + motor de fusión $$s,k$$ con mapa geométrico.
- Integridad GNSS (detección jamming/spoofing) integrada en el filtro.
- Primeras pruebas con GNSS nominal.

**H2 – Mapa visual + visión (Semanas 6–10, paralelo)**
- Captura de pasadas para construir mapa visual ligero.
- Integración VLM (landmarks) y validación de tasas de matching.

**H3 – V2V mínimo (Semanas 9–12)**
- Definición de mensaje mínimo, almacenamiento y reenvío.
- Ensayos con conectividad intermitente (simulada si es necesario).

**H4 – Campaña de pruebas (Semanas 13–20)**
- Pruebas diurnas/nocturnas, distintas velocidades.
- Evaluación en túneles (GNSS‑denied), urbano, estaciones.
- Informe final + plan de escalado.

### 7.3 Métricas (propuestas)
**Precisión longitudinal sobre vía**
- $$P95$$ y $$P99$$ del error en $$s$$ (m).
- Error máximo sostenido en túnel (m) vs longitud de túnel.

**Continuidad**
- % del trayecto con solución válida (sin dropouts).
- Tiempo máximo sin GNSS manteniendo $$P95$$ < umbral acordado.

**Integridad GNSS**
- Tasa de detección (TPR) de anomalías y falsa alarma (FPR).
- Tiempo de detección ante spoofing/jamming (TTD, s).
- Tasa de “rechazo correcto” de GNSS falso sin degradar navegación.

**Contribución de visión**
- nº de re‑anclajes por km y su calidad.
- reducción del error acumulado en GNSS‑denied respecto a “solo odometría”.

**Contribución V2V**
- mejora (conservadora) de incertidumbre/precisión cuando hay intercambio.
- robustez a latencia: rendimiento con mensajes retrasados.

**Operación**
- coste de mantenimiento del mapa visual (frecuencia de actualización por km/mes).
- carga computacional embarcada (CPU/GPU, W) y memoria.

---

## 8. RIESGOS Y MITIGACIONES (ENFOQUE CON ODOMETRÍA)

### R1) Error de odometría por patinaje/deslizamiento
**Riesgo**: en frenado fuerte o baja adherencia, $$\Delta s_{odo}$$ puede sesgarse.  
**Mitigación**
- Monitor ODM para detectar intervalos de baja confianza y aumentar covarianza.
- Ajuste online de $$b_{odo}$$ (escala/sesgo).
- Uso de visión como ancla frecuente en zonas con landmarks; GNSS cuando sea íntegro.

### R2) Variabilidad visual (noche/lluvia/suciedad)
**Mitigación**
- Cámara WDR/HDR + procedimientos de limpieza/inspección en piloto.
- Control de calidad visual: si baja, no “forzar” matches; aumentar incertidumbre.

### R3) Ambigüedad multi‑vía
**Mitigación**
- Estado discreto multi‑hipótesis $$k$$ con poda por probabilidad.
- Secuencias de landmarks y restricciones topológicas para desambiguar.

### R4) Spoofing “sofisticado” (sin saltos evidentes)
**Mitigación**
- Tests cruzados GNSS vs odometría+visión+mapa.
- Modelos de consistencia temporal y dinámica (no solo umbrales simples).
- Registro de evidencias para mejora iterativa y análisis forense.

### R5) Dependencia de V2V
**Mitigación**
- V2V es incremental: el sistema cumple requisitos sin él.
- Mensajes mínimos, store‑and‑forward, fusión CI.

### R6) Mantenimiento del mapa visual
**Mitigación**
- Mapa ligero versionado con caducidad.
- Detección de “mapa envejecido” (caída de matching) y remapeo localizado.

---

## 9. VALOR PARA RENFE (RESULTADOS ESPERADOS)

- Continuidad en túneles y zonas GNSS difíciles gracias a **odometría + visión**.
- Mayor ciberresiliencia: **detección y mitigación** de GNSS jamming/spoofing.
- Estimación con **incertidumbre cuantificada** para integrar en sistemas operativos.
- Capa V2V que mejora resiliencia sin exigir conectividad continua.
- Arquitectura replicable a otros trenes y líneas con coste contenido (mapa visual ligero).

---

## 10. RESUMEN EJECUTIVO (PARA CIERRE DE PÁGINA)

RAILGUARD‑V integra GNSS con monitor de integridad (jamming/spoofing), odometría ferroviaria disponible, localización visual con cámara frontal y mapa visual ligero offline, y colaboración V2V con mensajes mínimos. Mediante un filtro híbrido (estado continuo $$s,\dot{s}$$ + selección discreta de vía) y gestión adaptativa de incertidumbre, proporciona una posición sobre vía continua, robusta y trazable. El piloto en un tren de Renfe busca alcanzar TRL 6 con métricas objetivas de precisión, continuidad e integridad en escenarios reales (túneles, urbano y estaciones complejas), incluyendo un plan de mitigación de riesgos centrado en odometría y condiciones visuales adversas.
