# usecaseOperaciones

# ANÁLISIS FUNCIONAL - OPACT111 (Sistema de Gestión de Líneas Diarias de Operaciones)

## 1. FLUJO GENERAL DEL PROCESO

### 1.1 Objetivo Principal
Sistema legacy de gestión operacional de líneas de transporte (autobuses/buses). Administra el ciclo de vida completo de una línea de viaje diaria: desde su apertura, despacho, confirmación, hasta cierre y reintegro de recursos.

### 1.2 Flujo de Ejecución General
```
INICIO DEL FORMULARIO
    ↓
[1] INGRESO DE FECHA Y HORA BASE
    ├─ Validación de rango de fechas (±15 días de hoy)
    └─ Carga de rutas disponibles
    ↓
[2] SELECCIÓN DE LÍNEA DE TRANSPORTE
    ├─ Consulta de buses disponibles
    ├─ Consulta de conductores
    └─ Validación de tipo de servicio (regular vs especial)
    ↓
[3] GESTIÓN DE ESTADO DE LÍNEA
    ├─ OPCIÓN A: ABRIR LÍNEA CERRADA
    │   ├─ Validación de permiso por agencia
    │   ├─ Verificación de anticipos
    │   └─ Decisión sobre anular reintegro anterior
    │
    ├─ OPCIÓN B: CERRAR LÍNEA ABIERTA
    │   ├─ Validación de pasajeros mínimos
    │   ├─ Determinación del motivo de cierre
    │   └─ Cálculo de reintegros
    │
    └─ OPCIÓN C: CAMBIAR A OTRO BUS (Desvío)
        ├─ Selección de nuevo bus
        └─ Reprogramación de ruta
    ↓
[4] PROCESOS ASOCIADOS
    ├─ Reintegros (Expresos, Motivos, Valores)
    ├─ Novedades de Conductor
    ├─ Contingencias
    └─ Revisiones de Vehículos
    ↓
[5] GENERACIÓN DE COMPROBANTES
    ├─ Órdenes de combustible
    └─ Comprobantes de ingreso
    ↓
FIN DEL PROCESO
```

### 1.3 Decisiones Críticas
- **Validación de rango temporal**: Líneas no pueden abrirse después de 3 días
- **Validación de agencia**: Solo la agencia que cerró puede abrir en tránsito
- **Validación de pasajeros**: Mínimo establecido para no cerrar
- **Validación de tipo de servicio**: Diferencia entre regular (P) y especial (C)
- **Validación de revisiones**: Verificación de auditoría técnica pendiente

---

## 2. CASOS DE USO POR PROCESO

### PROCESO 1: VALIDACIÓN Y CARGA DE INFORMACIÓN BASE

**Nombre**: Inicialización de Parámetros de Control

**Objetivo**: Establecer la fecha de trabajo, hora base y cargar todas las rutas disponibles para esa fecha.

**Descripción Funcional**: 
- El usuario ingresa la fecha de la línea a actualizar
- El sistema valida que la fecha esté dentro del rango permitido (±15 días de hoy)
- Se carga la lista de rutas y horas disponibles
- Se limpia el bloque de detalles de línea anterior

**Evento Inicial**: `Control_Fecha_KeyNextField` - Presionar Tab en campo Fecha

**Precondiciones**:
- Fecha no debe ser nula
- Empresa y Agencia deben estar configuradas
- Usuario debe tener permiso de acceso

**Flujo Principal**:
1. Usuario ingresa fecha (dd/MM/yyyy)
2. Sistema convierte a formato sql.Date
3. Ejecuta validación de rango (fecha ± 15 días)
4. Si válido: limpia bloque St_Linea_Diaria y carga rutas via `llenarRutas()`
5. Posiciona cursor en campo Hora_B

**Flujos Alternativos**:
- **Si fecha > hoy+15 días**: Muestra mensaje, resetea a fecha actual, lanza excepción
- **Si fecha < hoy-15 días**: Muestra mensaje, resetea a fecha actual, lanza excepción

**Validaciones Realizadas**:
- Rango de fecha ±15 días
- Formato de fecha válido
- Fecha no nula

**Métodos Involucrados**:
- `Control_Fecha_KeyNextField.fireEvent()`
- `Control_Fecha_WhenValidateField.fireEvent()`
- `facade.clearBlock("St_Linea_Diaria")`
- `facade.llenarRutas()`
- Queries SQL: St_Linea_Diaria, Rutas, Tipo_Bus

**Datos de Entrada**:
- Fecha de la línea
- Empresa (desde contexto)
- Agencia (desde contexto)

**Datos de Salida**:
- Lista de rutas disponibles
- Fecha validada en facade.getDatos().setFecha_Trabajo()

**Resultado Esperado**: Formulario listo para seleccionar ruta/hora con rutas disponibles precargadas

---

### PROCESO 2: MANEJO DE ESTADO DE LÍNEA (Apertura/Cierre)

**Nombre**: Gestión del Ciclo de Vida de Línea Diaria

**Objetivo**: Cambiar el estado de una línea entre Abierto (A), Cerrado (C) o Despachado (D)

**Descripción Funcional**:
- Control centralizado mediante campo `Est_despacho` en St_Linea_Diaria
- Validaciones complejas según origen/destino de la línea
- Decisiones sobre destino final del bus (Taller, Disponibles, Inactivos)
- Cálculo automático de reintegros

**Evento Inicial**: `St_linea_diaria_Est_despacho_WhenRadioChanged`

**Precondiciones**:
- Línea debe estar previamente cargada
- Línea origen debe coincidir o ser tránsito
- Permisos de la agencia validados

**Flujo Principal - APERTURA DE LÍNEA (A)**:
1. Validar no sea línea desde agencia origen si es tránsito
2. Si es origen: permitir apertura
3. Si es tránsito: validar solo agencia que cerró puede abrir
4. Verificar que fecha no sea anterior a 3 días
5. Buscar línea con mismo bus en otras rutas
6. Si existe: rechazar (bus ocupado)
7. Buscar anticipos pendientes
8. Si existen: preguntar si anular reintegro anterior
9. Si sí: marcar `AnulaReintegro="S"`
10. Si no: proceder normal
11. Marcar `Actualizar="S"` y `ActualizaLinea="S"`

**Flujo Principal - CIERRE DE LÍNEA (C)**:
1. Validar que línea esté confirmada (Num_Bus no nulo)
2. Contar pasajeros mínimos vendidos
3. Si < mínimo: oferta reducida
   - Mostrar opción de cerrar por demanda baja
   - Si sí: calcular reintegros
4. Si >= mínimo: preguntar problema técnico
   - Si sí (problema técnico): Novedad="03", Proceso="AL2"
   - Si no: preguntar si hay tiquetes vendidos
      - Si > mínimo: advertencia
      - Proceder a cierre normal: Novedad="27", Proceso="AL1"
5. Mostrar ventana de "Motivos_Cierre"

**Flujos Alternativos**:
- **Línea ya abierta**: "La línea ya se encuentra abierta" → abort
- **Línea ya cerrada**: "La línea ya se encuentra cerrada" → abort
- **Intento de estado 'D' (Despachado)**: "No se puede colocar en estado despachado" → reset
- **Falta confirmación**: "Esta línea no puede ser cerrada porque no ha sido confirmada" → abort
- **Falta despacho**: "Esta línea no puede ser cerrada porque no ha sido despachada" → abort
- **Revisión pendiente**: Muestra mensaje de revisión pendiente pero continúa

**Validaciones Realizadas**:
- Estado previo vs nuevo
- Permisos por agencia
- Rango de fechas
- Presencia de confirmación y despacho
- Revisiones pendientes
- Anticipos pendientes

**Métodos Involucrados**:
- `facade.getSt_linea_diaria().setEst_despacho()`
- `facade.getSt_linea_diaria().setActualizar()`
- `facade.calcularreintegro(outMensaje)` → calcula si hay excedentes
- `facade.showWindow("Motivos_Cierre")`
- `facade.buscarbus()`
- Queries: St_Linea_Diaria, St_Anticipo_Bus, Op_Revision, Agencia

**Datos de Entrada**:
- Estado destino (A, C, D)
- Información de línea actual
- Anticipo pendiente (si existe)

**Datos de Salida**:
- Estado actualizado
- Flags: Actualizar, ActualizaLinea, AnulaReintegro, PasarA*, etc.
- Ventana modal abierta (si aplica)

**Resultado Esperado**: 
- Apertura: Línea abierta para continuación de venta
- Cierre: Diálogo de motivos y cálculo de reintegro

---

### PROCESO 3: CÁLCULO Y GENERACIÓN DE REINTEGROS

**Nombre**: Procesamiento de Reintegro de Expresos

**Objetivo**: Calcular montos a reintegrar por viáticos, peajes, combustible, etc. cuando una línea no se realiza completamente.

**Descripción Funcional**:
- Consulta anticipos generados para el bus
- Valida datos de autorización o cálculo previo
- Permite ingreso manual de valores a reintegrar
- Genera comprobante de ingreso

**Evento Inicial**: `MotivosReintegro_Aceptar_WhenMouseClick`

**Precondiciones**:
- Bus debe tener anticipo vigente
- Anticipo debe estar en estado "GR" (Grabado)
- Fecha de anticipo dentro de rango permitido (últimos 5 días)

**Flujo Principal**:
1. Obtener información del bus (placa, empresa)
2. Buscar anticipos más recientes (últimos 5 días)
3. Si no hay anticipo: preguntar si quitar bus del expreso
4. Si hay anticipo:
   a. Copiar valores de viáticos/peajes/combustible
   b. Si anticipo tipo "C" (Cálculo):
      - Buscar datos en tabla Op_Bus_Expreso
      - Buscar fecha del viaje en Op_Autoriza_Ant_Exp
   c. Si anticipo tipo "A" (Autorización):
      - Cargar valores de Op_Autoriza_Ant_Exp
   d. Calcular galones excedentes:
      - Si reintegro completo: sum(gln_origen+gln_transito+gln_adic) - sum(gln_usados)
      - Si parcial: sum(gln_origen+gln_transito+gln_adic) - (tiempo_parcial * consumo)
5. Mostrar ventana ReintegroExpresos con valores pre-llenados
6. Usuario ingresa valores a reintegrar
7. Validación: valor_reintegro <= valor_entregado
8. Cálculo total: suma de todos los reintegros
9. Confirmación del usuario
10. Generación de comprobante de ingreso
11. Actualización de estado del bus

**Validaciones Realizadas**:
- Anticipo debe existir
- Anticipo debe estar en estado GR
- Valores a reintegrar no pueden exceder valores entregados
- Tiempo parcial no puede exceder tiempo total del viaje
- Peajes validados contra tabla de rutas
- Comprobante de ingreso validado

**Métodos Involucrados**:
- `facade.getReintegroExpresos()` - Container de datos
- `facade.getPAnticipo()` - Información de anticipo
- `facade.getBuses()` - Información del bus
- `facade.getReintegroExpresosDet()` - Detalle del reintegro
- `facade.actualizaexcedentesexp()`
- `facade.generacpteingresoexp()`
- `facade.pasarataller()`, `facade.pasaradisponibles()`, `facade.pasarainactivos()`
- Stored Procedures: `Quitabusexpreso`

**Datos de Entrada**:
- Número de bus
- Motivo de reintegro (P, V, A, R)
- Valores opcionales a reintegrar

**Datos de Salida**:
- Valores de reintegro calculados
- Comprobante generado
- Estado del bus actualizado

**Resultado Esperado**: 
- Comprobante de ingreso generado
- Bus movido a estado final (Taller, Disponibles o Inactivos)
- Montos reintegrados registrados

---

### PROCESO 4: GESTIÓN DE NOVEDADES DE CONDUCTOR

**Nombre**: Registro de Novedades/Infracciones del Conductor

**Objetivo**: Registrar incidentes o cambios en la conducta del conductor durante el viaje.

**Evento Inicial**: Cuando se selecciona buscar nuevo conductor después de cambio de bus

**Descripción Funcional**:
- Captura cédula del nuevo conductor
- Valida existencia de contrato vigente
- Registra fecha y hora del cambio
- Almacena observaciones

**Precondiciones**:
- Bus debe estar identificado
- Línea debe tener cambio de conductor

**Flujo**:
1. Mostrar ventana "Novedad"
2. Capturar cédula del conductor
3. Buscar en tabla Contrato (estado='V')
4. Si no encuentra: buscar en Contrato_Temp
5. Si encuentra: cargar nombre del conductor
6. Si no encuentra: mostrar error
7. Generar número secuencial
8. Grabar registro en Op_Nov_Conductor

**Métodos**: `OpNovConductor_Grabar_WhenMouseClick`, `OpNovConductor_Regresar_WhenMouseClick`

**Datos de Salida**: Registro generado en Op_Nov_Conductor

---

### PROCESO 5: GESTIÓN DE DESVÍOS Y CONTINGENCIAS

**Nombre**: Manejo de Cambios de Ruta y Contingencias

**Objetivo**: Permitir cambiar a un bus alterno cuando el programado no está disponible o tiene problemas.

**Descripción Funcional**:
- Selecciona ruta alternativa
- Valida disponibilidad de bus alterno
- Valida restricciones internacionales
- Valida que viaje sea futuro
- Genera orden de reprogramación

**Eventos**: `Contingencia_Enturnar_WhenMouseClick`

**Flujo**:
1. Validar fecha del desvío sea futura
2. Validar ruta destino
3. Si ruta es internacional: bus debe ser internacional
4. Si tipo especial: solo permite dentro de misma familia
5. Calcular si hay excedentes a reintegrar
6. Si hay: preguntar si anular o continuar
7. Grabar desvío en tabla Desvio
8. Programar línea alternativa

**Resultado**: Nueva línea programada, bus anterior desvinculado

---

## 3. CASOS DE USO POR MÉTODO

### MÉTODO 1: `Control_Fecha_KeyNextField.fireEvent()`

**Objetivo**: Transición desde campo Fecha a campo Hora_B con carga de datos

**Quién lo invoca**: Sistema al presionar Tab en campo Fecha

**Métodos que invoca**:
- `facade.getDatos().setFecha_Trabajo()` - Copia fecha a contenedor Datos
- `facade.clearBlock("St_Linea_Diaria")` - Limpia datos previos
- `facade.goItem("Control.Hora_B")` - Posiciona cursor

**Eventos relacionados**: WhenValidateField del mismo campo

**Entradas**: Fecha ingresada

**Salidas**: 
- Fecha de trabajo establecida
- Bloque de línea limpio
- Cursor posicionado

**Validaciones**: Ninguna (se hace en WhenValidateField)

**Lógica paso a paso**:
```java
// 1. Copia fecha a contenedor Datos
facade.getDatos().setFecha_Trabajo(facade.getControl().getFecha());

// 2. Limpia detalle de línea
facade.clearBlock("St_Linea_Diaria");

// 3. Posiciona en siguiente campo
facade.goItem("Control.Hora_B");
```

**Impacto**: Prepara sistema para ingreso de hora base

---

### MÉTODO 2: `Control_Fecha_WhenValidateField.fireEvent()`

**Objetivo**: Validar que fecha esté dentro del rango permitido

**Quién lo invoca**: Motor de validación de campos al salir del campo

**Métodos que invoca**:
- `facade.getSql().executeSelectInto()` - Consulta fechas base
- `isNull()`, `isGreater()`, `isLower()` - Comparaciones
- `sic_abi.messageBox()` - Muestra mensaje
- `facade.getControl().setFecha()` - Resetea fecha
- `facade.clearBlock()` - Limpia datos

**Eventos relacionados**: KeyNextField

**Entradas**: Fecha ingresada

**Salidas**: Validación pass/fail

**Validaciones**:
1. Fecha no nula
2. Fecha no > hoy+15 días
3. Fecha no < hoy-15 días

**Lógica paso a paso**:
```java
// 1. Consultar fecha actual y limites
SELECT 
  to_date(sysdate,'dd/MM/yyyy') AS fact,
  To_Date(...(Sysdate + 15)...) AS fp,
  To_Date(...(Sysdate - 15)...) AS fant
FROM Dual

// 2. Si fecha es nula
IF (isNull(fecha)) THEN
  messageBox("Debe Especificar la Fecha")
  throw JFormsTriggerFailure
END IF

// 3. Si fecha es posterior al límite
IF (isGreater(fecha, fechaPosterior)) THEN
  messageBox("La Fecha No Puede Exceder Más de 15 Días")
  setFecha(fechaActual)
  throw JFormsTriggerFailure
END IF

// 4. Si fecha es anterior al límite
IF (isLower(fecha, fechaAnterior)) THEN
  messageBox("La Fecha No Puede Ser Más de 15 Días Antes")
  setFecha(fechaActual)
  throw JFormsTriggerFailure
END IF

// 5. Limpiar bloque si todo válido
facade.clearBlock("St_Linea_Diaria")
```

**Impacto**: Garantiza que solo se trabajen fechas válidas dentro del rango operacional

---

### MÉTODO 3: `Control_Hora_b_KeyNextField.fireEvent()`

**Objetivo**: Validar hora en formato HHMI y cargar rutas disponibles

**Quién lo invoca**: KeyNextField al salir del campo Hora_B

**Métodos que invoca**:
- `StringUtils.length()` - Valida longitud
- `facade.clearBlock()` - Limpia líneas
- `facade.goBlock()` - Posiciona en bloque
- `facade.llenarRutas()` - Carga rutas disponibles
- `StringUtils.toChar()` - Convierte formato de hora

**Entradas**: Hora ingresada

**Salidas**: 
- Validación pass/fail
- Rutas cargadas
- Hora en formato HH:MI AM

**Validaciones**:
1. Hora no nula
2. Longitud <= 4 caracteres
3. Formato HHMI válido

**Lógica paso a paso**:
```java
// 1. Validar longitud
IF (length(hora) > 4) THEN
  messageBox("La Hora debe estar en el Formato HHMI")
  throw JFormsTriggerFailure
END IF

// 2. Si fecha está establecida
IF (isNotNull(fecha)) THEN
  clearBlock("St_Linea_Diaria")
  goBlock("St_Linea_Diaria")
  clearBlock(noValidate)
  llenarRutas()  // Carga rutas para esa fecha/hora
ELSE
  messageBox("Debe especificar la Fecha")
  throw JFormsTriggerFailure
END IF

// 3. Convertir hora a formato display
horaFormato = toChar(toDate(hora, "HH24MI"), "HH:MI AM")
setHora_b(horaFormato)

// 4. Pasar al siguiente campo
nextItem()
```

**Impacto**: Activa carga de rutas disponibles y convierte formato de entrada

---

### MÉTODO 4: `St_linea_diaria_Est_despacho_WhenRadioChanged.fireEvent()`

**Objetivo**: Procesar cambio de estado de la línea

**Quién lo invoca**: Radio button cuando usuario selecciona nuevo estado

**Métodos que invoca**: [Ver Proceso 2 - Gestión de Estado de Línea]

**Impacto**: Más crítico - dispara múltiples procesos y validaciones

---

### MÉTODO 5: `Datos_Aceptar_WhenMouseClick.fireEvent()`

**Objetivo**: Procesar aceptación de motivo de cierre y confirmar cambios

**Quién lo invoca**: Click en botón "Aceptar" en ventana Motivos_Cierre

**Métodos que invoca**:
- `facade.origendespachada()` - Consulta si línea es origen despachada
- `facade.calcularreintegro()` - Calcula reintegros necesarios
- `futils.preguntar()` - Interfaz con usuario
- `facade.procesodedesvio()` - Si es desvío
- `facade.hideWindow()` - Cierra ventana

**Lógica - INACTIVOS (I)**:
1. Marcar Proceso="AL1", Novedad=motivo, PasarAInactivos="S"
2. Consultar si línea es origen despachada
3. Si sí: mostrar ventana Reintegro
4. Si no: calcular reintegro automático
5. Si hay excedentes: preguntar si hacer reintegro

**Lógica - TALLER (T)**:
1. Similar a INACTIVOS
2. Marcar Proceso="AL2", PasarATaller="S"

**Lógica - DISPONIBLES (D)**:
1. Similar
2. Marcar Proceso="AL3", PasarADisponibles="S"

**Lógica - DESVÍO (V)**:
1. Validar no sea servicio especial
2. Llamar `facade.procesodedesvio()`
3. No marcar como cerrada

**Impacto**: Define destino final del bus después del cierre

---

### MÉTODO 6: `ReintegroExpresos_Aceptar_WhenMouseClick.fireEvent()`

**Objetivo**: Confirmar valores de reintegro y generar comprobantes

**Quién lo invoca**: Click en "Aceptar" en ventana ReintegroExpresos

**Métodos que invoca**:
- Cálculo de totales
- `facade.actualizaexcedentesexp()` - Actualiza tabla de excedentes
- `facade.generacpteingresoexp()` - Genera comprobante
- `Quitabusexpreso.executeStoredProcedure()` - Actualiza estado en BD
- `facade.pasarataller()`, `pasaradisponibles()`, `pasarainactivos()` - Destino final
- `facade.commitTransaction()` - Graba cambios

**Flujo**:
```
Validar valores
  ↓
Mostrar confirmación de monto
  ↓
Si Aceptar:
  ├─ Actualizar excedentes
  ├─ Generar comprobante ingreso
  ├─ Quitar bus de expreso (si corresponde)
  ├─ Pasar a destino final
  ├─ Commit transacción
  └─ Imprimir (si configurado)
  ↓
Si Cancelar:
  ├─ Rollback
  └─ Mostrar mensaje de cancelación
```

**Impacto**: Genera registros contables y operacionales finales

---

### MÉTODO 7: `Datos_Motivo_WhenValidateField.fireEvent()`

**Objetivo**: Validar que código de motivo exista y sea válido para cierre de línea

**Quién lo invoca**: Validación de campo Motivo

**Métodos que invoca**:
- `facade.getSql().executeSelectInto()` - Consulta St_Nov_Lineas
- `sic_abi.messageBox()` - Error si no válido

**Lógica**:
```java
IF (motivo IS NOT NULL) THEN
  SELECT Des_Nov FROM St_Nov_Lineas
  WHERE Cod_Nov = motivo AND Cierre_Linea = 'S'
  
  IF NOT FOUND THEN
    messageBox("El motivo digitado no es válido")
    CLEAR motivo
    throw JFormsTriggerFailure
  END IF
END IF
```

**Impacto**: Asegura que solo se usen motivos de cierre válidos

---

## 4. MAPA DE LLAMADAS (Jerarquía)

```
EVENTO: St_linea_diaria_Est_despacho_WhenRadioChanged
  ├─ facade.limpiadata()                    [Resetea flags]
  ├─ facade.limpiaact()                     [Limpia acciones]
  │
  ├─ [SI ESTADO = 'A' - APERTURA]
  │  ├─ facade.getSql().executeSelectInto() [Consulta línea]
  │  ├─ [Validar línea no tenga otro bus]
  │  ├─ facade.getVariables()               [Consulta anticipos]
  │  ├─ facade.calcularreintegro()          [Calcula excedentes]
  │  │  └─ RETORNA: mensaje (si hay excedentes)
  │  └─ futils.preguntar()                  [Si hay excedentes]
  │
  ├─ [SI ESTADO = 'C' - CIERRE]
  │  ├─ Contar pasajeros vendidos
  │  │  └─ facade.getSql().executeSelectInto() [St_Tiquete]
  │  │
  │  ├─ [SI pasajeros < mínimo]
  │  │  ├─ facade.calcularreintegro()
  │  │  └─ Si excedentes: futils.preguntar()
  │  │     ├─ SI: facade.showWindow("Datos")
  │  │     │   └─ Datos_Aceptar_WhenMouseClick()
  │  │     │      ├─ facade.origendespachada()
  │  │     │      ├─ facade.calcularreintegro()
  │  │     │      ├─ facade.showWindow("Reintegro")
  │  │     │      └─ Reintegro_Aceptar_WhenMouseClick()
  │  │     └─ NO: facade.getSt_linea_diaria().setPasarAInactivos("S")
  │  │
  │  ├─ [SI agencia = origen]
  │  │  ├─ Validar confirmada (Num_Bus != null)
  │  │  ├─ Mostrar ventana "Motivos_Cierre"
  │  │  └─ Datos_Aceptar_WhenMouseClick()
  │  │
  │  └─ [SI agencia != origen - TRANSITO]
  │     ├─ Validar despacho anterior
  │     └─ Contar pasajeros
  │        ├─ Si >= mínimo: Motivos_Cierre
  │        └─ Si < mínimo: Similar flujo de reintegro
  │
  └─ futils.enable("Placa_Nueva")           [Habilita cambio de bus]

EVENTO: MotivosReintegro_Aceptar_WhenMouseClick
  ├─ Validar bus existe
  ├─ facade.getSql().executeSelectInto() [Buscar anticipos]
  │  └─ Si no hay: preguntar quitar bus
  │
  ├─ Cargar información del anticipo
  │  ├─ Galones excedentes
  │  ├─ Valores de viáticos
  │  └─ Información del chasis/bus
  │
  ├─ [SI anticipo tipo 'C' - Cálculo]
  │  └─ Buscar en Op_Bus_Expreso, Op_Autoriza_Ant_Exp
  │
  ├─ [SI anticipo tipo 'A' - Autorización]
  │  └─ Buscar valores en Op_Autoriza_Ant_Exp
  │
  ├─ facade.showWindow("ReintegroExpresos")
  └─ [Usuario ingresa valores]
     └─ ReintegroExpresos_Aceptar_WhenMouseClick()
        ├─ Calcular total reintegro
        ├─ futils.preguntar() [Confirmar monto]
        ├─ facade.actualizaexcedentesexp()
        ├─ facade.generacpteingresoexp()
        ├─ Quitabusexpreso [Stored Procedure]
        ├─ facade.pasarataller/disponibles/inactivos()
        ├─ facade.commitTransaction()
        └─ [Opcionalmente] facade.imprimir()
```

---

## 5. DIAGRAMA DE FLUJO TEXTUAL

```
INICIO DE SESIÓN
│
├─ FORMULARIO OPACT111 CARGADO
│  ├─ Mostrar Control Block
│  │  ├─ Campo Fecha (date)
│  │  ├─ Campo Hora_B (time)
│  │  └─ Botones de acción
│  │
│  └─ Mostrar St_Linea_Diaria Block
│     ├─ Tabla/Grid de líneas
│     └─ Controles de estado
│
├─ USUARIO INGRESA FECHA
│  │
│  ├─ VALIDACIÓN AUTOMÁTICA
│  │  ├─ ¿Fecha es nula?
│  │  │  ├─ SÍ: Error → Reset a fecha hoy
│  │  │  └─ NO: Continuar
│  │  │
│  │  ├─ ¿Fecha > hoy + 15 días?
│  │  │  ├─ SÍ: Error → Reset a fecha hoy
│  │  │  └─ NO: Continuar
│  │  │
│  │  └─ ¿Fecha < hoy - 15 días?
│  │     ├─ SÍ: Error → Reset a fecha hoy
│  │     └─ NO: Continuar → ÉXITO
│  │
│  └─ CARGA DE RUTAS
│     └─ Ejecutar llenarRutas()
│        └─ Consultar rutas disponibles para esa fecha
│
├─ USUARIO INGRESA HORA
│  │
│  ├─ VALIDACIÓN AUTOMÁTICA
│  │  ├─ ¿Hora es nula?
│  │  │  ├─ SÍ: Error
│  │  │  └─ NO: Continuar
│  │  │
│  │  └─ ¿Longitud (hora) > 4?
│  │     ├─ SÍ: Error "Formato HHMI"
│  │     └─ NO: Continuar → Convertir formato
│  │
│  └─ SE CARGA LÍNEA
│     └─ Select St_Linea_Diaria para esa fecha/hora/ruta
│
├─ USUARIO VE LÍNEA DISPONIBLE
│  │
│  ├─ ¿Línea está ABIERTA (A)?
│  │  │
│  │  └─ Opción 1: CERRAR LÍNEA
│  │     │
│  │     ├─ ¿Línea confirmada?
│  │     │  ├─ NO: Error "No está confirmada"
│  │     │  └─ SÍ: Continuar
│  │     │
│  │     ├─ Contar pasajeros vendidos
│  │     │
│  │     ├─ ¿Pasajeros >= mínimo (Vble_Minimo)?
│  │     │  │
│  │     │  ├─ NO: DEMANDA BAJA
│  │     │  │  ├─ Novedad = "05"
│  │     │  │  ├─ Proceso = "AL3"
│  │     │  │  ├─ Ir a ventana Reintegro (si origen despachada)
│  │     │  │  └─ Sino: Calcular reintegro automático
│  │     │  │
│  │     │  └─ SÍ: Preguntar "¿Problema técnico?"
│  │     │     │
│  │     │     ├─ SI - PROBLEMA TÉCNICO
│  │     │     │  ├─ Novedad = "03"
│  │     │     │  ├─ Proceso = "AL2"
│  │     │     │  ├─ Calcular reintegro
│  │     │     │  └─ PasarATaller = "S" (o config)
│  │     │     │
│  │     │     └─ NO - CIERRE NORMAL
│  │     │        ├─ Preguntar "¿Hay tiquetes > mínimo?"
│  │     │        ├─ Novedad = "27"
│  │     │        ├─ Proceso = "AL1"
│  │     │        └─ Ir a ventana Motivos_Cierre
│  │     │
│  │     └─ [En ventana Motivos_Cierre]
│  │        ├─ Usuario selecciona motivo (P, V, A, R, etc.)
│  │        │
│  │        ├─ Click Aceptar:
│  │        │  ├─ Validar motivo válido
│  │        │  ├─ Solicitar observación
│  │        │  │
│  │        │  ├─ ¿Motivo = "I" (Inactivos)?
│  │        │  │  ├─ SÍ: PasarAInactivos = "S"
│  │        │  │  └─ Reintegro si: GeneraCpteIngreso = "S"
│  │        │  │
│  │        │  ├─ ¿Motivo = "T" (Taller)?
│  │        │  │  ├─ SÍ: PasarATaller = "S"
│  │        │  │  └─ Reintegro si aplica
│  │        │  │
│  │        │  ├─ ¿Motivo = "D" (Disponibles)?
│  │        │  │  ├─ SÍ: PasarADisponibles = "S"
│  │        │  │  └─ Reintegro si aplica
│  │        │  │
│  │        │  ├─ ¿Motivo = "V" (Desvío)?
│  │        │  │  ├─ SÍ: procesodedesvio()
│  │        │  │  └─ NO CIERRA línea
│  │        │  │
│  │        │  └─ Finalizar: ActualizaLinea = "S"
│  │        │
│  │        └─ Click Cancelar: Abort, sin cambios
│  │
│  ├─ ¿Línea está CERRADA (C)?
│  │  │
│  │  └─ Opción 2: ABRIR LÍNEA CERRADA
│  │     │
│  │     ├─ ¿Agencia es origen?
│  │     │  ├─ SÍ: Permitir apertura
│  │     │  └─ NO: ¿Agencia = que cerró la línea?
│  │     │     ├─ NO: Error "Solo agencia que cerró puede abrir"
│  │     │     └─ SÍ: Permitir apertura
│  │     │
│  │     ├─ ¿Fecha de apertura - fecha de cierre > 3 días?
│  │     │  ├─ SÍ: Error "No puede abrir después de 3 días"
│  │     │  └─ NO: Continuar
│  │     │
│  │     ├─ Buscar anticipos pendientes
│  │     │
│  │     ├─ ¿Hay anticipo pendiente?
│  │     │  ├─ SÍ: Preguntar "¿Anular reintegro?"
│  │     │  │  ├─ SÍ: AnulaReintegro = "S"
│  │     │  │  └─ NO: Continuar normal
│  │     │  └─ NO: Continuar
│  │     │
│  │     └─ Actualizar: Actualizar = "S", Estado = "A"
│  │
│  └─ ¿Línea está DESPACHADA (D)?
│     └─ Error: "No se puede colocar en estado despachado"
│
├─ USUARIO SELECCIONA NUEVO BUS (Desvío)
│  │
│  ├─ Habilitar campos Placa_Nueva, Nuevo_Bus
│  │
│  ├─ Usuario ingresa nueva placa
│  │
│  ├─ EVENTO: St_linea_diaria_Placa_nueva_KeyNextField
│  │  └─ facade.placanewPostchange()
│  │     ├─ Validar placa existe
│  │     ├─ Cargar info del nuevo bus
│  │     └─ Validar tipo de servicio compatible
│  │
│  ├─ Mostrar ventana Desvío
│  │
│  ├─ Usuario selecciona destino del bus anterior
│  │
│  └─ Click Aceptar:
│     ├─ Marcar EnturnarContingencia = "S"
│     ├─ Marcar QuitarBusLinea = "S"
│     └─ Programar línea alternativa
│
├─ EJECUCIÓN DE CAMBIOS
│  │
│  └─ Click GUARDAR en formulario principal
│     │
│     ├─ PreCommit: Validaciones finales
│     │
│     ├─ Insert/Update de St_Linea_Diaria
│     │
│     ├─ Si Actualizar = "S": Update estado línea
│     │
│     ├─ Si ActualizaNuevoBus = "S": Insert nuevo registro
│     │
│     ├─ Si GeneraCpteIngreso = "S": Generar comprobante
│     │
│     ├─ Si PasarATaller = "S": Insert en tabla de Taller
│     │
│     ├─ Si PasarADisponibles = "S": Insert en Disponibles
│     │
│     ├─ Si PasarAInactivos = "S": Insert en Inactivos
│     │
│     ├─ Si ImprimeOrden = "S": Generar orden combustible
│     │
│     └─ COMMIT TRANSACTION
│        └─ Éxito: Limpiar formulario, ir a inicio
│
└─ FIN
```

---

## 6. DEPENDENCIAS

### 6.1 Métodos Compartidos (Reutilizables)
- `facade.clearBlock(String blockName)` - Limpia datos de un bloque
- `facade.goBlock(String blockName)` - Posiciona en un bloque
- `facade.goItem(String itemName)` - Posiciona en un campo
- `facade.showWindow(String windowName)` - Abre ventana modal
- `facade.hideWindow(String windowName)` - Cierra ventana modal
- `facade.synchronize()` - Sincroniza datos en memoria
- `facade.rollbackForm()` - Revierte cambios
- `facade.commitTransaction()` - Confirma cambios

### 6.2 Métodos Utilitarios
- `StringUtils.length()`, `StringUtils.toChar()` - Manipulación de strings
- `futils.preguntar()` - Diálogo de confirmación
- `futils.enable()` - Habilita/deshabilita campos
- `sic_abi.messageBox()` - Muestra mensajes
- `isNull()`, `isGreater()`, `isLower()`, `isDifferent()` - Funciones lógicas
- `add()`, `substract()`, `multiply()` - Operaciones matemáticas
- `nvl()` - NVL de Oracle (null coalescing)

### 6.3 Eventos del Sistema
- **KeyNextField**: Navegación entre campos
- **WhenValidateField**: Validación al salir del campo
- **WhenRadioChanged**: Cambio en radio buttons
- **WhenMouseClick**: Click en botón
- **PreInsert**: Antes de insertar registro
- **WhenNewDataInstance**: Nueva instancia de datos

### 6.4 Acceso a Base de Datos
- **Tablas principales**:
  - `St_Linea_Diaria` - Líneas de transporte diarias
  - `St_Anticipo_Bus` - Anticipos por bus
  - `St_Nov_Lineas` - Motivos de cierre
  - `Op_Nov_Conductor` - Novedades de conductor
  - `Op_Revision` - Revisiones técnicas
  - `Buses` - Información de buses
  - `Contrato`, `Contrato_Temp` - Datos de conductores
  - `Rutas` - Definición de rutas
  - `Agencia` - Sucursales/agencias
  - `Op_Bus_Expreso` - Registro de expresos
  - `Op_Autoriza_Ant_Exp` - Autorizaciones de expresos
  - `Op_Tab_Disponibles` - Buses en disponibles
  - `Tipo_Bus` - Clasificación de buses

- **Stored Procedures**:
  - `Quitabusexpreso` - Cambia estado del bus
  - `Validarevisiones` - Valida revisiones
  - `Pasartaller`, `Pasaradisponibles` - Cambian estado

### 6.5 Servicios Externos
- **Web Services**:
  - `WSSincronizacionBrasilia` - Sincronización Geotech
  - `NaviBO` - Información de navegación

- **Procesos Externos**:
  - `Imprimir` - Generación de reportes
  - `EnviarCorreo` - Envío de emails

### 6.6 Variables Globales/Contexto
- `Control.Empresa` - Empresa actual
- `Control.Agencia` - Agencia/sucursal actual
- `Control.Usuario` - Usuario actual
- `Control.CocheTour` - Indica si es servicio especial
- `facade.getGlobalVariable("empresa")` - Variable global empresa

### 6.7 Objetos Compartidos (Containers)
- `Control` - Parámetros de control
- `St_linea_diaria` - Datos de línea
- `Datos` - Datos de cierre/motivo
- `Variables` - Variables temp del proceso
- `PAnticipo` - Datos de anticipo
- `Buses` - Información del bus
- `ReintegroExpresos` - Reintegros
- `MotivosReintegro` - Motivos

---

## 7. RESUMEN FUNCIONAL

### ¿QUÉ HACE ESTE ARCHIVO?

El archivo `OPACT111Facade.java` implementa la lógica de negocio principal de un **Sistema de Gestión de Líneas Diarias de Operaciones de Transporte** (posiblemente de una empresa de transporte interurbano o de pasajeros).

### ¿QUÉ PROBLEMA RESUELVE?

1. **Gestión de Líneas**: Permite abrir, cerrar, modificar y actualizar líneas de transporte diarias
2. **Control de Recursos**: Administra asignación de buses y conductores
3. **Operaciones Financieras**: Calcula y registra reintegros de gastos operacionales
4. **Auditoría**: Registra novedades, cambios y decisiones
5. **Reportería**: Genera comprobantes y órdenes

### ¿QUÉ OPERACIONES REALIZA?

1. **Validaciones**: Rango de fechas, existencia de registros, permisos, estado previo
2. **Consultas**: Líneas, buses, conductores, anticipos, rutas
3. **Inserciones**: Registros de novedades, cambios de estado, reintegros
4. **Actualizaciones**: Estados de línea, datos de bus, información de conductor
5. **Cálculos**: Reintegros, galones excedentes, totales de reintegro
6. **Generación**: Comprobantes, órdenes de combustible
7. **Integración**: Llamadas a stored procedures, web services

### ¿QUÉ INFORMACIÓN MODIFICA?

- `St_Linea_Diaria` - Estado, destino final, flags de proceso
- `St_Anticipo_Bus` - Estados de anticipo
- `Op_Nov_Conductor` - Registro de novedades
- `Op_Bus_Expreso` - Estados de expresos
- `Op_Tab_Disponibles`, `Op_Tab_Taller`, `Op_Tab_Inactivos` - Destino del bus

### ¿QUÉ INFORMACIÓN CONSULTA?

- Líneas disponibles para una fecha/hora
- Buses disponibles
- Conductores activos (contrato vigente)
- Rutas y horarios
- Anticipos generados
- Revisiones técnicas pendientes
- Información de agencias
- Motivos de cierre
- Históricos de transacciones

---

## 8. OBSERVACIONES

### 8.1 Código Duplicado
- **Validación de fecha/hora**: Se repite en múltiples eventos
  - `Control_Fecha_WhenValidateField`
  - `Control_Hora_b_PreTextField`
  - Ambas tienen lógica similar de consulta de fechas límite

- **Consultas de anticipos**: Se repite en 3+ lugares
  - `MotivosReintegro_Aceptar_WhenMouseClick`
  - `ReintegroExpresos_Cedconductor_KeyNextField`
  - Estructura similar de SELECT

- **Pasar a estados finales**: 3 métodos prácticamente idénticos
  - `facade.pasarataller()`
  - `facade.pasaradisponibles()`
  - `facade.pasarainactivos()`

### 8.2 Métodos Muy Acoplados
- **`St_linea_diaria_Est_despacho_WhenRadioChanged`**: 
  - Método monolítico de > 800 líneas
  - Maneja: Validaciones, cálculos, decisiones, navegación, almacenamiento
  - Debería decomponerse en: Validador, Calculador, Decisor, Navegador

- **`ReintegroExpresos_Aceptar_WhenMouseClick`**:
  - Maneja: Cálculos, transacciones, stored procedures, UI
  - Débil separación de responsabilidades

### 8.3 Posibles Refactorizaciones
1. **Extraer validadores**:
   - `DateValidator`, `HourValidator`, `AnticipValidator`

2. **Extraer calculadores**:
   - `ReintegroCalculator`, `GasolineCalculator`, `MinimumCalculator`

3. **Extraer decisores**:
   - `LineStateDecider`, `BusDestinationDecider`, `ProcessDecider`

4. **Aplicar patrón Strategy**:
   - Para diferentes tipos de cierre (I, T, D, V)
   - Para diferentes tipos de anticipo (C, A)

5. **Usar Builder pattern**:
   - Para construcción compleja de St_Linea_Diaria

6. **Event-driven architecture**:
   - En lugar de métodos acoplados

### 8.4 Riesgos Funcionales
1. **Pérdida de datos**:
   - Si rollback no se ejecuta correctamente, datos inconsistentes
   - Falta de transaccionalidad explícita en algunos flujos

2. **Validaciones insuficientes**:
   - Falta validación de permisos granulares
   - No hay validación de cupos de capacidad de buses

3. **Lógica de negocio hardcodeada**:
   - Rangos de fecha (±15 días) - debería ser configurable
   - Códigos de novedad (03, 05, 11, 27) - debería venir de tabla
   - Mínimo de pasajeros - debería ser configurable

4. **Recursión posible**:
   - Si `procesodedesvio()` llama nuevamente a eventos de línea

5. **State machine incompleto**:
   - No todos los estados válidos están cubiertos
   - Falta validación de transiciones de estado inválidas

6. **Escalabilidad**:
   - Método monolítico puede tener timeout en muchos registros
   - Consultas sin índices pueden ser lentas

### 8.5 Métodos Muertos o No Utilizados
(Basado en el análisis del código legible)
- Posibles métodos no invocados en archivos no visibles
- Recomendación: Hacer análisis estático del código completo

### 8.6 Mejoras Sugeridas
1. Implementar logging detallado
2. Añadir timeouts en consultas
3. Usar connection pooling
4. Implementar caché de consultas frecuentes
5. Añadir auditoría completa de cambios
6. Separar validaciones de lógica de negocio
7. Implementar compensating transactions
8. Crear reportes de actividad
9. Implementar soft-delete en lugar de updates
10. Versionamiento de registros críticos

