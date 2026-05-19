# Propuesta IoT Solution — FIRSTstudent Smart Fuel Consumption Monitor

---

## Pregunta 1. Definition of System Requirements

### System Requirement — Especificación de requisitos

**Capacidades de suministro de energía (Power Supply)**

- Alimentación desde el sistema eléctrico del bus (12V Tipo D / 24V Tipo C)
- Regulador DC-DC step-down con protección contra sobrevoltaje e inversión de polaridad (norma ISO 7637-2)
- ESP32 opera a 3.3V, consumo activo ~240mA, consumo en deep sleep <10µA
- Sensores YF-S201 y HC-SR04 operan a 5V DC, ~15mA cada uno durante medición
- Raspberry Pi (Gateway) alimentado a 5V / 3A desde regulador independiente
- Supercapacitor de respaldo (≥1F) para escritura segura ante corte súbito de energía
- Modo deep sleep del ESP32 en bus apagado: lectura de nivel cada 5 minutos, consumo <1mA

**Restricciones de Time-Delay**

- Lectura YF-S201 → ESP32: < 100 ms
- Lectura HC-SR04 → ESP32: < 200 ms
- ESP32 → Raspberry Pi (REST local): < 500 ms
- Raspberry Pi → Servidor del Distrito (4G/HTTPS): < 2 segundos
- Servidor Distrito → HALO Cloud (sincronización): < 10 segundos
- Alertas críticas (nivel bajo / fuga detectada): < 1 segundo end-to-end nodo → Gateway
- Actualización First View Dashboard (padres / admin): < 3 segundos
- Frecuencia de muestreo YF-S201: cada 1 segundo (motor encendido)
- Frecuencia de muestreo HC-SR04: cada 30 segundos (operación), cada 5 minutos (reposo)

---

## Pregunta 2. Definition of Physical Layer Requirements

### Physical Layer Requirement — Especificación de requisitos

**Número y tipos de nodos sensores y actuadores**

- Sensores (por bus):
  - YF-S201: sensor de flujo de combustible (caudal volumétrico)
  - HC-SR04: sensor ultrasónico de nivel de tanque
  - GPS Module: posición georreferenciada en tiempo real
  - Cámara IA (HALO Driver Monitor): monitoreo de conductor
  - RFID Reader: asistencia de estudiantes
- Actuadores (por bus):
  - DriverHub Tablet: alertas visuales y sonoras al conductor, navegación
- Nodos Gateway/Fog/Cloud:
  - Raspberry Pi 4 (In-Bus Gateway): concentrador local, REST server, Internet Gateway 4G
  - Servidor de Distrito (Fog Node): análisis predictivo, optimización de rutas
  - HALO Cloud: gestión global de flota

**Target Uncertainty (Incertidumbre objetivo)**

- YF-S201 (flujo de combustible):
  - Rango: 1 – 30 L/min
  - Incertidumbre: ±3% del fondo de escala → ±0.9 L/min máximo
  - Nota: requiere calibración del factor K para diésel (viscosidad diferente al agua)
- HC-SR04 (nivel de tanque):
  - Rango: 2 cm – 400 cm
  - Incertidumbre: ±3 mm en distancia → ±0.5 galones en volumen (según geometría del tanque)

**Target Accuracy and Precision de los actuadores**

- DriverHub Tablet (alerta nivel bajo):
  - Disparo correcto ≥ 99% de eventos reales
  - Falsos positivos < 1%
  - Latencia detección → alerta en tablet: < 1 segundo
- Sistema de recomendación de rutas (Fog AI):
  - Reducción de consumo ≥ 5% vs. ruta base
  - Variación entre ejecuciones del mismo escenario < 2%
- Servicio de notificaciones HALO Cloud:
  - Tasa de entrega exitosa > 99.5% en ventana de 60 segundos

**Processing Power (Esfuerzo computacional)**

- Edge Node (ESP32 — 240 MHz, dual core, 520KB SRAM):
  - Conteo de pulsos YF-S201 por interrupción hardware: O(1), carga <1%
  - Cálculo de caudal instantáneo (aritmética simple): carga <2%
  - Conversión distancia → volumen HC-SR04 (tabla de lookup): carga <2%
  - Detección de anomalías por umbral: carga <1%
  - Serialización JSON y REST POST al Gateway: carga ~10%
  - Uso total estimado de CPU en operación normal: < 15%
- Fog Node (Servidor del Distrito):
  - CPU mínimo: 8 núcleos
  - RAM mínimo: 32 GB
  - Almacenamiento: SSD NVMe
  - GPU opcional para inferencia de modelos ML
  - Algoritmos: Random Forest / LSTM para mantenimiento predictivo, heurísticas de ruteo

---

## Pregunta 3. Definition of Information Layer Requirements

### Information Layer Requirement — Especificación de requisitos

**Definición de usuarios finales**

- Conductor del bus: usuario operativo, interactúa solo con DriverHub Tablet en ruta
- Despachador del distrito: usuario técnico, supervisa múltiples buses desde sala de operaciones
- Administrador del distrito escolar: perfil gerencial, accede desde oficina vía web
- Padre / Tutor del estudiante: usuario no técnico, accede desde app móvil First View
- Analista de operaciones FIRSTstudent: técnico avanzado, accede al HALO Dashboard global
- Técnico de mantenimiento: usuario técnico de campo, consulta alertas de vehículos

**Definición de número y tipos de servicios por usuario final**

- Conductor:
  - S1: Alerta de nivel de combustible bajo (Tablet)
  - S4: Aviso simplificado de anomalía de consumo (Tablet)
  - S5: Recomendación de ruta eficiente en combustible (Tablet)
  - S10: Notificación de repostaje programado (Tablet)
- Despachador:
  - S1: Alerta de nivel bajo (Dashboard)
  - S2: Monitoreo de consumo en tiempo real
  - S3: Historial de consumo por viaje/ruta
  - S4: Alerta de anomalía de consumo
  - S5: Supervisión de recomendaciones de ruta
  - S6: Reporte de eficiencia de flota por distrito
  - S8: Seguimiento GPS del bus
  - S10: Notificación de repostaje programado
- Administrador del distrito:
  - S1, S2, S3, S4, S6, S7, S8, S9, S10
- Padre / Tutor:
  - S8: Seguimiento GPS del bus (First View App)
  - Notificaciones de cambios de servicio y retrasos
- Analista HALO:
  - Todos los servicios (S1 al S10) con visibilidad global de flota
- Técnico de mantenimiento:
  - S1: Alerta de nivel bajo
  - S3: Historial de consumo
  - S4: Alerta de anomalía
  - S7: Predicción de mantenimiento
  - S10: Notificación de repostaje

**Definición de necesidades de información integrada por servicio**

- S1 – Alerta nivel bajo:
  - Datos: nivel actual del tanque (HC-SR04) + capacidad del tanque (TankConfig) + umbral configurado
  - Fuente: ESP32 → Raspberry Pi → Tablet y Dashboard
- S2 – Consumo en tiempo real:
  - Datos: caudal instantáneo (YF-S201) + distancia recorrida (GPS) + velocidad + ID de viaje
  - Fuente: ESP32 + GPS del Gateway → Servidor del Distrito
- S3 – Historial por viaje:
  - Datos: serie temporal de consumo + ID ruta + ID conductor + timestamp + distancia
  - Fuente: District Database → HALO Cloud
- S4 – Anomalía de consumo:
  - Datos: consumo actual + baseline histórico por ruta/bus + modelo de detección de outliers
  - Fuente: AI Engine del Servidor del Distrito
- S5 – Ruta eficiente:
  - Datos: mapa de rutas + consumo histórico por segmento + tráfico + número de estudiantes
  - Fuente: Route Recommendation Service (Fog)
- S6 – Reporte de flota:
  - Datos: consumo agregado por bus/ruta/conductor/período + costos estimados
  - Fuente: District Database → HALO Cloud BI
- S7 – Predicción de mantenimiento:
  - Datos: serie histórica de consumo del vehículo + kilometraje + registro de mantenimientos
  - Fuente: AI Engine del Servidor del Distrito (ML model)
- S8 – Seguimiento GPS:
  - Datos: posición GPS en tiempo real + estado del viaje + ID de bus
  - Fuente: GPS Module → Raspberry Pi → First View / HALO Dashboard
- S9 – KPIs globales:
  - Datos: consumo total de flota + eficiencia promedio + CO2 estimado + costo combustible
  - Fuente: HALO Cloud Data Lake + BI
- S10 – Repostaje programado:
  - Datos: nivel actual + consumo estimado × km restantes de ruta
  - Fuente: ESP32 + Route Service (Fog) → Notificaciones

---

## Pregunta 4. C4 Model Container Diagram — Código PlantUML

```plantuml
@startuml FIRSTstudent_C4_Container

!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

LAYOUT_WITH_LEGEND()

title C4 Container Diagram — FIRSTstudent Smart Fuel Consumption Monitor

' ===== PERSONAS =====
Person(conductor, "Conductor", "Opera el bus. Recibe alertas y navegación via DriverHub Tablet.")
Person(despachador, "Despachador", "Supervisa la flota del distrito en tiempo real.")
Person(admin, "Administrador de Distrito", "Gestiona reportes de eficiencia y mantenimiento.")
Person(padre, "Padre / Tutor", "Rastrea el bus de su hijo via First View App.")
Person(analista, "Analista HALO", "Accede a KPIs globales de flota desde HALO Dashboard.")
Person(tecnico, "Técnico de Mantenimiento", "Consulta alertas predictivas de vehículos.")

' ===== EDGE: IN-BUS SYSTEM =====
System_Boundary(edge, "In-Bus Edge System (por bus — ~44,500 unidades)") {

    Container(fuelMonitor, "Smart Fuel Monitor Node", "ESP32 + YF-S201 + HC-SR04 / C++ Arduino", "Mide flujo de combustible y nivel del tanque. Detecta anomalías locales. Envía payload JSON al Gateway.")

    Container(rfidNode, "RFID Reader Node", "Módulo RFID / Firmware embebido", "Registra asistencia de estudiantes al abordar y descender.")

    Container(cameraNode, "HALO Driver Monitor", "Cámara IA / SDK propietario", "Monitorea comportamiento del conductor con visión por computadora.")

    Container(gpsModule, "GPS Module", "Hardware GPS / NMEA", "Provee posición georreferenciada en tiempo real al Gateway.")

    Container(gateway, "In-Bus Gateway Module", "Raspberry Pi 4 / Python + REST", "Concentra datos de todos los nodos. REST server local. Internet Gateway 4G hacia el exterior.")

    Container(tablet, "DriverHub Tablet", "Android / iOS App", "Entrega navegación, alertas de combustible y recomendaciones de ruta al conductor.")
}

' ===== FOG: DISTRICT STATION =====
System_Boundary(fog, "School District Station — Fog Node") {

    Container(aggregator, "Data Aggregator Service", "Python / MQTT Broker / REST API", "Recibe y normaliza telemetría de todos los buses del distrito.")

    Container(districtDB, "District Database", "TimescaleDB / PostgreSQL", "Almacena series temporales de consumo, rutas, alertas y eventos de la flota.")

    Container(aiEngine, "Predictive AI Engine", "Python / scikit-learn / TensorFlow", "Ejecuta modelos de mantenimiento predictivo y detección de anomalías de consumo.")

    Container(routeService, "Route Recommendation Service", "Python / REST API", "Genera rutas eficientes combinando consumo histórico, tráfico y ocupación del bus.")
}

' ===== CLOUD: HALO PLATFORM =====
System_Boundary(cloud, "HALO Cloud Platform — Global") {

    Container(haloGateway, "HALO API Gateway", "REST / gRPC / API Management", "Punto de entrada único para sincronización de datos de todos los distritos.")

    Container(haloCoreServices, "HALO Core Services", "Microservicios / Kubernetes", "Dispatch, routing global, training y gestión de flota a escala de Norteamérica.")

    Container(dataLake, "Cloud Data Lake", "AWS S3 + Redshift / BigQuery", "KPIs globales, históricos de flota y datos para Business Intelligence.")

    Container(firstViewApp, "First View App", "iOS / Android App", "Permite a padres rastrear el bus en tiempo real y recibir alertas de servicio.")

    Container(haloDashboard, "HALO Dashboard", "Web App / React", "KPIs, alertas y mapa de flota para despachadores, administradores y analistas.")

    Container(notificationService, "Notification Service", "Push / SMS / Email", "Envía alertas críticas y notificaciones programadas a todos los perfiles de usuario.")
}

' ===== RELACIONES EDGE INTERNAS =====
Rel(fuelMonitor, gateway, "FuelDataPayload JSON", "HTTP REST / WiFi local")
Rel(rfidNode, gateway, "Eventos de asistencia", "HTTP REST / Serial")
Rel(cameraNode, gateway, "Alertas de conductor", "HTTP REST / local")
Rel(gpsModule, gateway, "Posición NMEA", "Serial / UART")
Rel(gateway, tablet, "Navegación y alertas", "HTTP REST / WiFi local")

' ===== RELACIONES EDGE → FOG =====
Rel(gateway, aggregator, "Telemetría consolidada del bus", "HTTPS / 4G")

' ===== RELACIONES FOG INTERNAS =====
Rel(aggregator, districtDB, "Persiste telemetría y eventos", "SQL")
Rel(districtDB, aiEngine, "Datos históricos para modelos", "SQL / API")
Rel(aiEngine, routeService, "Consumo estimado por segmento", "REST interna")
Rel(routeService, gateway, "Recomendación de ruta al bus", "HTTPS / 4G")

' ===== RELACIONES FOG → CLOUD =====
Rel(aggregator, haloGateway, "Sincronización de datos del distrito", "HTTPS / Internet")

' ===== RELACIONES CLOUD INTERNAS =====
Rel(haloGateway, haloCoreServices, "Operaciones de flota", "gRPC interno")
Rel(haloCoreServices, dataLake, "KPIs y datos analíticos", "API / ETL")
Rel(haloCoreServices, notificationService, "Disparo de alertas", "Event Bus")

' ===== RELACIONES USUARIOS =====
Rel(conductor, tablet, "Consulta navegación y recibe alertas", "Touch / Visual")
Rel(despachador, haloDashboard, "Supervisa flota en tiempo real", "HTTPS / Browser")
Rel(admin, haloDashboard, "Consulta reportes y eficiencia", "HTTPS / Browser")
Rel(padre, firstViewApp, "Rastrea el bus de su hijo", "HTTPS / Mobile")
Rel(analista, haloDashboard, "Analiza KPIs globales", "HTTPS / Browser")
Rel(tecnico, haloDashboard, "Consulta alertas de mantenimiento", "HTTPS / Browser")
Rel(notificationService, padre, "Alertas de servicio", "Push / SMS")
Rel(notificationService, conductor, "Repostaje programado", "Push / App")
Rel(notificationService, despachador, "Anomalías críticas", "Push / SMS / Email")

@enduml
```

**Criterios y sustento de decisión:**

- La separación en tres capas (Edge / Fog / Cloud) garantiza los mejores tiempos de respuesta locales sin depender de conectividad a internet
- El Raspberry Pi como único punto de salida 4G por bus evita multiplicar conexiones activas en los 44,500 vehículos
- TimescaleDB en el Fog es la elección óptima para series temporales de consumo de combustible
- El HALO API Gateway centraliza autenticación y versionado sin acoplar el distrito directamente con los microservicios internos
- El AI Engine vive en el Fog y no en el Cloud para reducir latencia en recomendaciones de ruta que llegan al conductor

---

## Pregunta 5. Class Diagram — OO Embedded Application (Smart Fuel Consumption Monitor)

**Jerarquía base ModestIoT Framework:**

- `IoTNode` (clase abstracta raíz):
  - Atributos: `nodeId: String`, `location: String`
  - Métodos: `setup(): void`, `loop(): void {abstract}`

- `SensorNode` hereda de `IoTNode` (abstracta):
  - Atributos: `samplingRate: uint16_t`, `lastReadMs: unsigned long`
  - Métodos: `readSensor(): float {abstract}`, `isReady(): bool`

- `ActuatorNode` hereda de `IoTNode` (abstracta):
  - Atributos: `isActive: bool`
  - Métodos: `activate(): void {abstract}`, `deactivate(): void {abstract}`, `getStatus(): bool`

**Clases concretas de sensores:**

- `FlowSensor` hereda de `SensorNode` — implementa YF-S201:
  - Atributos: `pin: uint8_t`, `pulseCount: volatile uint32_t`, `kFactor: float`, `flowRateLPM: float`
  - Métodos: `readSensor(): float`, `IRAM_ATTR onPulse(): void` (ISR)
  - Decisión: `pulseCount` es `volatile` porque se modifica desde una ISR (interrupción hardware)

- `UltrasonicSensor` hereda de `SensorNode` — implementa HC-SR04:
  - Atributos: `trigPin: uint8_t`, `echoPin: uint8_t`, `distanceCm: float`, `tankGeometry: TankConfig*`
  - Métodos: `readSensor(): float`
  - Decisión: delega la conversión distancia → volumen a `TankConfig` (principio Open-Closed)

**Clases concretas de actuadores:**

- `AlertActuator` hereda de `ActuatorNode`:
  - Atributos: `ledPin: uint8_t`, `buzzerPin: uint8_t`, `alertLevel: AlertLevel`
  - Métodos: `activate(): void`, `deactivate(): void`, `triggerAlert(lvl: AlertLevel): void`

**Tipos de datos (Value Objects / DTOs):**

- `TankConfig` — Value Object:
  - Atributos: `capacityGallons: float`, `tankType: BusType`, `sensorOffsetCm: float`
  - Métodos: `toVolume(distanceCm: float): float`
  - Decisión: encapsula geometría del tanque separada del sensor; permite cambiar Tipo C/D sin modificar `UltrasonicSensor`

- `FuelDataPayload` — struct DTO:
  - Atributos: `timestamp: uint32_t`, `flowRateLPM: float`, `tankLevelPct: float`, `totalConsumedL: float`, `busId: String`
  - Decisión: struct plano facilita serialización a JSON con ArduinoJson sin overhead de herencia

**Clase de infraestructura:**

- `GatewayClient`:
  - Atributos: `serverUrl: String`, `wifiSSID: String`, `wifiPassword: String`
  - Métodos: `connect(): bool`, `send(payload: FuelDataPayload): bool`
  - Decisión: encapsula toda la comunicación WiFi/HTTP; cambiar a BLE solo requiere modificar esta clase

**Clase raíz del firmware:**

- `FuelMonitorController` hereda de `IoTNode` — compone todos los demás objetos:
  - Atributos: `flowSensor: FlowSensor`, `ultrasonicSensor: UltrasonicSensor`, `alertActuator: AlertActuator`, `gateway: GatewayClient`
  - Métodos: `setup(): void`, `loop(): void`
  - Decisión: corresponde directamente al archivo `.ino`; `setup()` y `loop()` son el punto de entrada de Arduino

**Estructura de archivos resultante para el equipo de implementación:**

- `FuelMonitorController.ino` — punto de entrada Arduino
- `IoTNode.h`, `SensorNode.h`, `ActuatorNode.h` — jerarquía base (solo .h)
- `FlowSensor.h` / `FlowSensor.cpp`
- `UltrasonicSensor.h` / `UltrasonicSensor.cpp`
- `AlertActuator.h` / `AlertActuator.cpp`
- `GatewayClient.h` / `GatewayClient.cpp`
- `FuelDataPayload.h`, `TankConfig.h` — tipos de datos (solo .h)

---

*FIRSTstudent, Inc. — Propuesta IoT Solution bajo marco de 12 pasos IoT System Design, Domain-Driven Design y OO Software Design*