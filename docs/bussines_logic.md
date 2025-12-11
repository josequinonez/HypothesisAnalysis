# Lógica de Negocio

## Reglas de Promociones

### Definición de Promoción Completa vs Incompleta

#### Promoción Simple Completa
Una promoción simple se considera **COMPLETA** cuando:
1. El producto cumple la condición de cantidad exacta
2. La promoción está vigente en la fecha del evento
3. El evento ocurre en la etapa evaluada (add_cart, checkout, purchase)
```python
def es_completa_simple(cantidad, condicion, fecha_evento, vigencia):
    cumple = cumple_patron(cantidad, condicion.tipo, condicion.cant_inicial)
    activa = vigencia.inicio <= fecha_evento <= vigencia.cierre
    return cumple and activa
```

#### Promoción Simple Incompleta (Near Miss)
Una promoción simple se considera **INCOMPLETA** cuando:
- La promoción está activa
- NO cumple el patrón completo
- La cantidad es exactamente N-1 (un boleto menos del requerido)
```python
def es_incompleta_simple(cantidad, cant_inicial):
    return cantidad == cant_inicial - 1
```

#### Promoción Combinada Completa
Una promoción combinada está **COMPLETA** cuando:
1. TODOS los productos requeridos están en la sesión
2. CADA producto cumple su cantidad mínima requerida
3. La promoción está vigente
```python
def evaluar_promocion_combinada(sesion, requisitos):
    for requisito in requisitos:
        producto = requisito['clave_edicion_producto']
        cantidad_requerida = requisito['cantidad_inicial']
        cantidad_actual = sesion[producto].sum()
        
        if cantidad_actual < cantidad_requerida:
            return False
    return True
```

### Tipos de Condiciones de Cantidad

| Tipo | Descripción | Evaluación |
|------|-------------|------------|
| 1 | Exactamente N | `cantidad == N` |
| 2 | Mínimo N | `cantidad >= N` |
| 3 | Máximo N | `cantidad <= N` |
| 4 | Entre N y M | `N <= cantidad <= M` |
| 5 | Acumula N | `cantidad >= N` (acumulativo) |
| 6 | Por cada N | `cantidad >= N AND cantidad % N == 0` |
| 7 | Múltiplo de N | `cantidad % N == 0` |

## Categorización del Timing de Login

### Objetivo
Entender en qué momento del funnel el usuario se autentica y cómo esto impacta la conversión.

### Categorías Principales
```python
# Jerarquía de evaluación (orden importa)
categorias_login = [
    'SIN LOGIN EN SESIÓN',
    'LOGIN YA INICIADO',  # Sesión con logout pero sin login
    'LOGIN ANTES DE VIEW_ITEM_LIST',
    'LOGIN ANTES DE SELECT_ITEM',
    'LOGIN ANTES DE ADD_TO_CART',
    'LOGIN ENTRE ADD_TO_CART Y BEGIN_CHECKOUT',
    'LOGIN ENTRE BEGIN_CHECKOUT Y PURCHASE',
    'LOGIN DESPUÉS DE BEGIN_CHECKOUT',
    'CON LOGIN SIN EVENTOS DE FUNNEL'
]
```

### Casos Especiales

#### "Login Ya Iniciado"
Detecta sesiones donde el usuario ya estaba autenticado:
```python
if (has_purchase and 
    login_time is NULL and 
    logout_time is not NULL and 
    logout_time > purchase_time):
    return 'LOGIN YA INICIADO'
```

#### "Login Sin Registrar"
Usuarios que compran sin haberse registrado/logueado:
```python
es_sin_registro = (
    categoria_login == 'SIN LOGIN EN SESIÓN' and
    has_purchase == True
)
```

## Cálculo de Métricas

### Montos por Etapa
```python
# Precio unitario inferido
precio = MONTO_PURCHASE / qty_purchase  # Si hay compra
precio = catalogo[item].precio_unitario  # Si no hay compra

# Montos
MONTO_ADD_TO_CART = precio * qty_add_to_cart
MONTO_BEGIN_CHECKOUT = precio * qty_begin_checkout
MONTO_PURCHASE = precio * qty_purchase
```

### KPIs de Conversión

#### Tasa de Conversión por Etapa (no implementado)
```python
conversion_rate = {
    'cart_to_checkout': sesiones_con_checkout / sesiones_con_cart,
    'checkout_to_purchase': sesiones_con_purchase / sesiones_con_checkout,
    'overall': sesiones_con_purchase / total_sesiones
}
```

#### Impacto de Promociones (no implementado)
```python
impact = {
    'con_patron': conversion_con_patron_completo,
    'sin_patron': conversion_sin_patron,
    'lift': (conversion_con_patron / conversion_sin_patron) - 1
}
```

#### Oportunidad Perdida (no implementado)
```python
oportunidad_perdida = (
    sesiones_con_patron_incompleto * 
    valor_promedio_compra * 
    tasa_conversion_esperada
)
```

### KPI principal para la H2 (implementado)
```python
KPI_sesiones_resumen = (
    sesiones_sin_login_sin_purchase_con_patron_bc_monto_begin_checkout-((sesiones_con_login_sin_purchase_con_patron_bc_monto_begin_checkout /
        sesiones_con_login_monto_begin_checkout)
    * sesiones_sin_login_sin_purchase_con_patron_bc_monto_begin_checkout)
)
```


## Procesamiento por Sesión

### Algoritmo Principal
```python
def procesar_sesiones(df_eventos):
    resultados = []
    
    for (user, sesion), df_sesion in df_eventos.groupby(['USER', 'SESION']):
        # 1. Evaluar promociones combinadas para toda la sesión
        promos_completas = evaluar_promociones_sesion(df_sesion)
        
        # 2. Procesar cada producto en la sesión
        for _, producto in df_sesion.iterrows():
            # Detectar patrones simples
            patrones_simples = detectar_simples(producto)
            
            # Verificar si combinadas están completas
            patrones_combinados = verificar_combinadas(
                producto, promos_completas
            )
            
            # Consolidar resultados
            resultado = merge_patrones(patrones_simples, patrones_combinados)
            resultados.append(resultado)
    
    return pd.DataFrame(resultados)
```

## Validaciones de Negocio

### Vigencia de Promociones
```python
def promocion_activa(fecha_evento, fecha_inicio, fecha_cierre):
    if pd.isna(fecha_inicio) or pd.isna(fecha_cierre):
        return False
    return fecha_inicio <= fecha_evento <= fecha_cierre
```

### Consistencia de Cantidades
```python
# Validación: cart >= checkout >= purchase
assert qty_add_to_cart >= qty_begin_checkout
assert qty_begin_checkout >= qty_purchase
```

### Integridad de Montos
```python
# Montos deben ser consistentes con cantidades
assert MONTO_PURCHASE == qty_purchase * precio_unitario
```

## Casos de Uso de Negocio (algunas mejoras posibles)

### 1. Optimización de Promociones
- Identificar promociones con alta tasa de incompletitud
- Ajustar umbrales (ej: de 5 a 4 boletos)
- Timing óptimo para mostrar promociones

### 2. Reducción de Abandono
- Usuarios con patrones incompletos → candidatos para remarketing
- Categorías de login tardío → fricción en checkout

### 3. Personalización
- Usuarios frecuentes con patrones completos → VIP
- Usuarios con near-misses frecuentes → incentivos adicionales
