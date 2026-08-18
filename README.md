# bd3-Repro-nosql
# Sistema de Gestión de Telemetría y Reprogramación Automotriz

## 1. Definición del Problema

**Contexto:**
Los talleres especializados en potenciación automotriz (reprogramación de ECU Stage 1, Stage 2, etc.) enfrentan un desafío importante al registrar el historial de modificaciones de un vehículo y medir el impacto de estas en el rendimiento. 

**El Problema:**
Actualmente, los datos de rendimiento (telemetría) y las configuraciones de hardware varían drásticamente entre marcas y modelos (por ejemplo, los sensores de un vehículo viejo no reportan la misma estructura de datos que de un vehículo más moderno). Un modelo relacional tradicional requeriría tablas con esquemas rígidos y múltiples campos nulos para acomodar los distintos tipos de sensores y modificaciones.

**Rol de la Base de Datos:**
MongoDB actuará como el núcleo del sistema, permitiendo:
1. Almacenar configuraciones vehiculares con esquemas flexibles (cada auto tiene componentes distintos).
2. Registrar un historial detallado de las intervenciones mecánicas y de software.
3. Procesar de forma eficiente grandes volúmenes de datos de telemetría estructurados como series temporales generados durante las pruebas de rendimiento (Dyno tests o pruebas en calle).

## 2. Modelado Conceptual Orientado a Documentos y Esquema de Colecciones

El sistema se estructura en tres colecciones principales, diseñadas según los patrones de acceso de la aplicación:

### Colección: `vehiculos`
Almacena la información del auto, su estado actual y datos básicos del propietario utilizando el patrón *Extended Reference* para evitar *Joins* innecesarios al listar los autos en el taller.

```json
{
  "_id": "ObjectId('64e5...1')",
  "dominio": "AB123CD",
  "marca": "Peugeot",
  "modelo": "307",
  "motor": "2.0 HDI",
  "propietario": {
    "cliente_id": "ObjectId('64e5...9')",
    "nombre": "Carlos Gómez",
    "telefono": "+54 9 11 1234-5678"
  },
  "estado_actual": {
    "nivel_stage": 1,
    "potencia_estimada_hp": 130
  }
}

```
### Coleccion: `intervenciones`
Registra cada trabajo realizado en el vehículo. Las tareas específicas (modificaciones de hardware o software) se modelan como documentos anidados dentro de un array.
```json
{
  "_id": "ObjectId('64e5...2')",
  "vehiculo_id": "ObjectId('64e5...1')",
  "fecha": "2026-08-18T10:30:00Z",
  "tipo_trabajo": "Reprogramación y Mantenimiento",
  "descripcion_general": "Carga de mapa Stage 1 y revisión eléctrica",
  "modificaciones": [
    { 
      "componente": "ECU", 
      "accion": "Reprogramación de mapa de inyección", 
      "detalle": "Stage 1 - Aumento de presión de turbo" 
    },
    { 
      "componente": "Alternador", 
      "accion": "Reemplazo y testeo", 
      "detalle": "Prueba de salida estabilizada en 14V" 
    }
  ],
  "mecanico_responsable": "Ignacio"
}

```
### Coleccion: `telemetria`
Almacena las lecturas de los sensores en tiempo real durante las pruebas de rendimiento.

```json
{
  "timestamp": "2026-08-18T11:05:22.000Z",
  "metadata": {
    "vehiculo_id": "ObjectId('64e5...1')",
    "intervencion_id": "ObjectId('64e5...2')",
    "tipo_prueba": "Prueba en rodillo - Post Repro"
  },
  "mediciones": {
    "rpm": 4500,
    "presion_turbo_bar": 1.2,
    "temperatura_aceite_c": 95,
    "voltaje_alternador": 14.1
  }
}

```
## 3. Fundamentación de la lógica no relacional

El diseño de las colecciones se basó estrictamente en cómo la aplicación leerá y escribirá los datos, tomando las siguientes decisiones clave entre anidar y referenciar:

**Uso de Anidamiento en intervenciones:

Decisión: Los detalles de las modificaciones (ej. reemplazo de alternador, flasheo de ECU) se guardan en un array de subdocumentos dentro de la intervención, en lugar de crear una tabla relacional de "Detalle_Intervencion".

--Justificación: Patrón de "Datos que se consultan juntos, se guardan juntos". Cuando un usuario abre el detalle de una orden de trabajo, necesita ver todas las tareas realizadas inmediatamente. Como el número de tareas por intervención es acotado (no sufrirá de crecimiento ilimitado), anidarlo optimiza la lectura al requerir un solo viaje a la base de datos.

**Uso de Referencias entre vehículos e intervenciones:

Decisión: Las intervenciones no se anidan dentro del documento del vehículo, sino que viven en su propia colección referenciando el vehiculo_id.

--Justificación: Un vehículo puede tener múltiples intervenciones a lo largo de los años. Si anidamos todo en el vehículo, el documento crecería de forma desproporcionada, afectando el rendimiento y acercándose al límite de 16MB de BSON. Además, los patrones de acceso del taller requerirán frecuentemente consultas como "Listar todas las intervenciones realizadas en la última semana", lo cual es mucho más eficiente consultando una colección dedicada.

##Patrón Extended Reference en Propietarios:

Decisión: Se incluyó el nombre y teléfono del dueño directamente en el documento del vehículo.

--Justificación: En la pantalla principal del sistema de gestión, se listan los autos ingresados junto a su dueño. Para evitar un $lookup constante hacia una colección de clientes, se duplica esta información inmutable (o de muy baja mutabilidad) optimizando radicalmente el tiempo de lectura.

##Colección de Series Temporales para telemetría:

Decisión: Separar las lecturas de los sensores en una colección optimizada para tiempo en lugar de un documento normal.

--Justificación: La ingestión de datos de sensores automotrices se da a muy alta frecuencia (múltiples registros por segundo). MongoDB maneja este volumen de forma altamente eficiente mediante colecciones Time Series, que reducen el espacio en disco y aceleran las agregaciones basadas en rangos de tiempo (ej. calcular el pico máximo de RPM en una franja de 2 minutos).
