# Reductores, `GROUP BY` y `HAVING`

# Índice

- [1 Funciones de agregado](#1-funciones-de-agregado)
  - [1.1 Qué son](#11-qué-son)
  - [1.2 Ejemplos](#12-ejemplos)
  - [1.3 Tipos](#13-tipos)
  - [1.4 ¿De dónde salen las `List(a)`?](#14-de-dónde-salen-las-lista)
  - [1.5 Sobre qué se ejecutan](#15-sobre-qué-se-ejecutan)
  - [1.6 Y `WHERE` qué](#16-y-where-qué)
  - [1.7 Gotchas](#17-gotchas)
    - [1.7.1 Gotcha 1: `NULL`](#171-gotcha-1-null)
    - [1.7.2 Gotcha 2: `List(a)` is empty](#172-gotcha-2-lista-is-empty)
- [2 Agrupamientos de tuplas con `GROUP BY`](#2-agrupamientos-de-tuplas-con-group-by)
  - [2.1 Qué hace `GROUP BY`](#21-qué-hace-group-by)
  - [2.2 Criterio de agrupamiento](#22-criterio-de-agrupamiento)
  - [2.2 ¿Y el `SELECT`?](#22-y-el-select)
  - [2.3 Gotchas](#23-gotchas)
    - [2.3.1 Gotcha 1: `NULL`](#231-gotcha-1-null)
    - [2.3.2 Gotcha 2: Columnas, _reductores_ y `SELECT`](#232-gotcha-2-columnas-reductores-y-select)
      - [2.3.2.1 Problema](#2321-problema)
      - [2.3.2.2 Solución 1](#2322-solución-1)
      - [2.3.2.3 Solución 2](#2323-solución-2)
      - [2.3.2.4 Corolario](#2324-corolario)
- [3 `HAVING`](#3-having)
  - [3.1 `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `SELECT`](#31-from-where-group-by-having-select)

# 1 Funciones de agregado

## 1.1 Qué son

Es como se suele llamar a las funciones que **reducen** una colección de valores a uno solo. En la literatura también aparecen como **funciones de columnas o colectivas**.

Tal y como vimos en clase, en realidad se trata de funciones de tipo `reduce` o `fold`, que de forma **muy** simplificada podemos asimilar a

```
reduce :: List(a) -> b
```

donde el tipo `a` puede o no ser igual al tipo `b`.

## 1.2 Ejemplos

En ANSI/SQL podemos encontrarnos con:

- `AVG`: reduce a la media.
- `MAX`: reduce al máximo.
- `MIN`: reduce al mínimo.
- `SUM`: reduce a la suma.
- `COUNT`: reduce al número de elementos.

Podemos considerar `DISTINCT` como una función de agregado que reduce la duplicidad, si bien también hace más cosas.

## 1.3 Tipos

```
AVG :: List(Number) -> Number
MAX :: List(Number) -> Number
MIN :: List(Number) -> Number
SUM :: List(Number) -> Number
COUNT :: List(a)    -> Number
```

En `AVG`, `MAX`, `MIN` y `SUM` el tipo `a` es igual al tipo `b` y debe, además, ser númerico. En `COUNT`, sin embargo, `a` puede ser cualquier tipo válido en SQL, mientras que `b` debe ser numérico.

En el caso de `DISTINCT`, `a` y `b` pueden ser cualquier tipo válido en SQL pero tienen que ser el mismo (`a = b`).

```
DISTINCT :: List(a) -> a
```

## 1.4 ¿De dónde salen las `List(a)`?

De los valores de las columnas de una tabla. Por ejemplo, supongamos que una columna contiene las `menciones` de un `hashtag`.

| id | hashtag          | menciones |
|----|------------------|-----------|
| 1  | `#AquiSufriendo` |  35       |
| 2  | `#XoveYO!`       |  140      |
| 3  | `#ImperativeSQL` |  3        |
| 4  | `#HaskellFTW`    |  1337     |

Si quisieramos saber qué `hashtag` tiene más menciones podríamos aplicar `MAX` a la columna `menciones`:

```sql
SELECT MAX(menciones) AS "menciones"
FROM HashtagTable;
```

## 1.5 Sobre qué se ejecutan

Se ejecutan sobre columnas de **grupos de tuplas**. **Por defecto**, si no aparece ninguna cláusula `GROUP BY`, **toda la tabla** es un **único grupo de tuplas**. Sin embargo, si aparece un `GROUP BY`, se usará ese criterio para hacer diferentes agrupaciones de tuplas y posteriormente aplicar el _reductor_ a cada uno de los grupos.

Por lo tanto si sólo existe un único grupo, el resultado será una tabla con una sola tupla, mientras que si contamos con `n` grupos, obtendremos `n` tuplas.

## 1.6 Y `WHERE` qué

`WHERE` se ejecuta **antes** que el _reductor_. Por tanto si _filtramos_ (recordad que `WHERE` es en realidad `filter`) acorde a un **predicado**:

1. Aplicamos el filtro (`WHERE`).
2. Sobre el resultado ya filtrado aplicamos el _reductor_.
3. El resultado será una tabla con una sola fila.

## 1.7 Gotchas

### 1.7.1 Gotcha 1: `NULL`

Los valores `NULL` son eliminados automáticamente antes de aplicar un _reductor_.

### 1.7.2 Gotcha 2: `List(a)` is empty

Si la lista (`List(a)`) sobre la que queremos aplicar un _reductor_ está vacia:

- `COUNT` produce `0`.
- `AVG`, `MAX`, `MIN` y `SUM` producen `NULL`.
- **ToDo**: ¿`DISTINCT` produce `NULL`?

# 2 Agrupamientos de tuplas con `GROUP BY`

## 2.1 Qué hace `GROUP BY`

Tal y como comentábamos con anterioridad

> **Por defecto**, si no aparece ninguna cláusula `GROUP BY`, **toda la tabla** es un **único grupo de tuplas**.

Por lo tanto, `GROUP BY` permite agrupar tuplas acorde a un criterio. Estos grupos de tuplas **pueden ser pensados** como **nuevas subtablas**.

## 2.2 Criterio de agrupamiento

Se usa una o varias columnas para agrupar, de forma que todas aquellas tuplas que tengan el **mismo valor para esa(s) columnas** formarán parte del mismo grupo/subtabla.

Pensemos en una tabla con todo el alumnado del Illa de Sarón. Si agrupamos por la columna `curso`, todo el alumnado que pertenezca a un mismo curso (**mismo valor para la columna `curso`**) formarán parte del mismo grupo/subtabla.

## 2.2 ¿Y el `SELECT`?

El `SELECT` se ejecuta sobre cada uno de los grupos/subtablas. Por tanto si tenemos 8 grupos como resultado de un `GROUP BY`, tendremos 8 tuplas en el resultado final tras la ejecución del `SELECT`.

## 2.3 Gotchas

### 2.3.1 Gotcha 1: `NULL`

Los `NULL` se consideran iguales entre sí y por lo tanto van todos a un mismo grupo/subtabla.

### 2.3.2 Gotcha 2: Columnas, _reductores_ y `SELECT`

#### 2.3.2.1 Problema

¿Qué pasa con este tipo de consultas?

```sql
SELECT
  name,
  classroom,
  MAX(grade) AS 'Nota máxima'
FROM IllaDeSaron
GROUP BY classroom;
```

Suponiendo que hay:

- 100 alumnos/as en el Illa de Sarón
- 15 clases en el Illa de Sarón

analicemos el número de tuplas del resultado esperado por cada argumento del `SELECT`,

- `SELECT name` debería devolver tantas tuplas como alumnos hay en la tabla `IllaDeSaron`, es decir 100 tuplas.
- `SELECT classroom ... GROUP BY classroom` debería devolver una tupla por cada clase, es decir 15 tuplas.
- `SELECT MAX(grade) AS 'Nota máxima' ... GROUP BY classroom` debería devolver una tupla por cada clase, es decir 15 tuplas.

¿Vemos el problema? ¿Cómo hacemos para que **SQL devuelva 100 tuplas y 15 tuplas a la vez**?

**NO PODEMOS**. De ahí el error.

#### 2.3.2.2 Solución 1

Eliminamos `name` de la consulta

```sql
SELECT
  classroom,
  MAX(grade) AS 'Nota máxima'
FROM IllaDeSaron
GROUP BY classroom;
```

Devolvemos **15 tuplas** (tantas como clases hay en el Illa de Sarón).

#### 2.3.2.3 Solución 2

Añadimos `name` a `GROUP BY`

```sql
SELECT
  name,
  classroom,
  MAX(grade) AS 'Nota máxima'
FROM IllaDeSaron
GROUP BY classroom, name;
```

👀 Probablemente devolvamos **casi 100 tuplas**, tantas como alumnos 👀

#### 2.3.2.4 Corolario

Si aparece una columna en un `SELECT ... GROUP BY` y no forma parte del `GROUP BY`, tenemos un problema. Pensemos en `name` en el ejemplo anterior.

La excepción a esta regla es que esa columna sea el argumento de un _reductor_ ya que entonces sí sería correcto. Pensemos en `MAX(grade) AS 'Nota máxima'` en el ejemplo anterior, ya que `grade` aparece en el `SELECT` y no aparece en `GROUP BY`.

# 3 `HAVING`

Tras haber formado los **grupos/subtablas** con `GROUP BY`, con `HAVING` descartamos aquellos grupos (**subtablas**) que **no cumplan el predicado**.

## 3.1 `FROM`, `WHERE`, `GROUP BY`, `HAVING`, `SELECT`

Este es precísamente el orden de ejecución.

1. Seleccionamos la tabla/s del `FROM`
2. Ejecutamos el predicado del `WHERE` **en cada una de las tuplas**, filtrando (**eliminado**) aquellas que no lo cumplan.
3. Ejecutamos el `GROUP BY` **en cada una de las tuplas**, haciendo grupos/**subtablas** según el criterio especificado.
4. Ejecutamos el `HAVING` **una vez por cada grupo/subtabla**, filtrando (**eliminando**) aquellos grupos (**subtablas**) que no lo cumplan.
5. Finalmente ejecutamos el `SELECT` **una vez por cada grupo/subtabla** resultante, tras el filtrado de `HAVING`.
