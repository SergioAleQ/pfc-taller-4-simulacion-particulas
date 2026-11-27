# Informe de corrección

Taller 4 — Simulación de partículas en paralelo.
Fundamentos de Programación Funcional y Concurrente, Universidad del Valle.

Este informe argumenta que cada función del taller hace lo que el enunciado pide. Se usan dos técnicas: **inducción** para los programas recursivos y **estado, invariante y transformación** para los que recorren el arreglo acumulando (`foldLeft` y la recursión de cola `acumular`). Se demuestran de forma explícita las tres cosas que exige la rúbrica:

1. La notación matemática respalda cada argumento.
2. Partir el sistema en segmentos **no pierde ni repite** partículas (Lema 1, sección 1.4, y Lemas 3 y 4).
3. La operación con que se combinan los resultados parciales es **asociativa**, y por eso la versión paralela da el mismo resultado que la secuencial (Teoremas 1, 3 y 4).

## 0. Notación y supuestos

- Un sistema es un arreglo $s$ de $n$ partículas, con índices $0, 1, \ldots, n-1$. La partícula $i$ tiene posición $p_i$, velocidad $v_i$ y masa $m_i \geq 0$.
- Un **tramo** es un intervalo semiabierto de índices $[d, h)$ con $0 \leq d \leq h \leq n$. Tiene $k = h - d$ elementos y contiene los índices $d, d+1, \ldots, h-1$.
- La energía cinética de una partícula es $e_i = \tfrac{1}{2}\, m_i\, \lVert v_i \rVert^2$.
- **Cuándo se deja de partir.** Un tramo de $k$ elementos se resuelve sin partir si $k \leq \text{umbral}$ o $k \leq 1$. La segunda condición garantiza que la recursión termina con **cualquier** umbral: con umbral $\leq 0$, un tramo de un elemento no dejaría de partirse (la mitad de 1 es 0 y el tramo no se reduce).
- **Punto medio.** $m = d + \lfloor k/2 \rfloor$. El tramo izquierdo $[d, m)$ tiene $\lfloor k/2 \rfloor$ elementos y el derecho $[m, h)$ tiene $\lceil k/2 \rceil$.
- `parallel(a, b)` evalúa `a` y `b` posiblemente a la vez y devuelve el par de resultados. Todas las funciones de este taller son **puras**: leen el arreglo `s` y nunca escriben en él. Cada llamado construye arreglos o valores nuevos que no comparte con el otro. Por eso `parallel(a, b)` devuelve el mismo par que devolvería evaluar `a` y luego `b`: no hay condiciones de carrera ni dependencia del orden de ejecución.
- Los argumentos trabajan con números reales. La sección 3.5 dice qué cambia con los `Double` de punto flotante.

El código analizado es el de `SimulacionParticulas.scala`, sin `var`, `while` ni `return`.

---

## 1. Punto 1: un paso de la simulación

### 1.1 Programas

```scala
def mover(p: Particula, dt: Double): Particula =
  p.copy(pos = p.pos + p.vel * dt)

def pasoSecuencial(s: Sistema, dt: Double): Sistema =
  pasoSegmento(s, dt, 0, s.length)

def pasoSegmento(s: Sistema, dt: Double, desde: Int, hasta: Int): Sistema =
  Array.tabulate(hasta - desde)(i => mover(s(desde + i), dt))

def pasoParalelo(s: Sistema, dt: Double, umbral: Int): Sistema = {
  def avanzar(desde: Int, hasta: Int): Sistema =
    if (hasta - desde <= umbral || hasta - desde <= 1)
      pasoSegmento(s, dt, desde, hasta)
    else {
      val mitad = desde + (hasta - desde) / 2
      val (izquierda, derecha) = parallel(
        avanzar(desde, mitad),
        avanzar(mitad, hasta))
      izquierda ++ derecha
    }

  avanzar(0, s.length)
}
```

### 1.2 Especificación

Sea $f_{\text{paso}}(s, dt)$ el sistema $s'$ de $n$ partículas tal que, para todo $0 \leq i < n$:

```math
s'_i = (\,p_i + v_i \cdot dt,\; v_i,\; m_i\,)
```

La posición avanza $v_i \cdot dt$ y la velocidad y la masa no cambian. Con `mover` como arriba, $s'_i = \text{mover}(s_i, dt)$. Se debe demostrar que, para todo sistema $s$, todo $dt$ y todo umbral:

```math
\text{pasoParalelo}(s, dt, \text{umbral}) = \text{pasoSecuencial}(s, dt) = f_{\text{paso}}(s, dt)
```

### 1.3 Corrección de `pasoSegmento` y `pasoSecuencial`

`Array.tabulate(len)(g)` construye un arreglo nuevo de longitud `len` cuyo elemento $j$ es $g(j)$. Por lo tanto, para $0 \leq d \leq h \leq n$:

- La longitud de $\text{pasoSegmento}(s, dt, d, h)$ es $h - d$.
- Para $0 \leq j < h - d$, el elemento $j$ es $\text{mover}(s_{d+j}, dt)$.

Con $d = 0$ y $h = n$ se obtiene $\text{pasoSecuencial}(s, dt)_i = \text{mover}(s_i, dt) = s'_i$ para todo $0 \leq i < n$, así que $\text{pasoSecuencial} = f_{\text{paso}}$.

Con los ejemplos del enunciado: $(0,0) + (1,2)\cdot 2 = (2,4)$, y con $dt = 0$ resulta $(0,0)$.

### 1.4 Lema 1 (la partición no pierde ni repite elementos)

> **Lema 1.** Para todo tramo $[d, h)$, $\text{avanzar}(d, h)$ devuelve un arreglo $r$ de longitud $h - d$ tal que $r_j = \text{mover}(s_{d+j}, dt)$ para $0 \leq j < h - d$.

**Demostración**, por inducción fuerte sobre $k = h - d$.

**Caso base:** $k \leq \text{umbral}$ o $k \leq 1$. Entonces `avanzar` devuelve $\text{pasoSegmento}(s, dt, d, h)$, y por la sección 1.3 cumple lo pedido. Esto incluye el tramo vacío ($k = 0$), que devuelve el arreglo vacío.

**Caso de inducción:** $k > \text{umbral}$ y $k \geq 2$. Sea $m = d + \lfloor k/2 \rfloor$.

1. *Los dos tramos son más pequeños y no vacíos.* Como $k \geq 2$, $1 \leq \lfloor k/2 \rfloor < k$ y $\lceil k/2 \rceil < k$, de modo que $d < m < h$. Los dos tramos tienen menos de $k$ elementos, y se puede aplicar la hipótesis de inducción (HI).
2. *Por la HI*, $\text{izq} = \text{avanzar}(d, m)$ tiene longitud $a = \lfloor k/2 \rfloor$ y $\text{izq}_j = \text{mover}(s_{d+j}, dt)$ para $0 \leq j < a$. Igualmente $\text{der} = \text{avanzar}(m, h)$ tiene longitud $b = \lceil k/2 \rceil$ y $\text{der}_j = \text{mover}(s_{m+j}, dt)$ para $0 \leq j < b$.
3. *La concatenación.* $\text{izq} \mathbin{+\!\!+} \text{der}$ tiene longitud $a + b = k$. Su elemento $j$ es $\text{izq}_j$ si $j < a$ y $\text{der}_{j-a}$ si $j \geq a$. En el segundo caso:

```math
\text{der}_{j-a} = \text{mover}(s_{m + j - a}, dt) = \text{mover}(s_{d + j}, dt)
```

porque $m - a = d$. Entonces, para **todo** $0 \leq j < k$, el elemento $j$ del resultado es $\text{mover}(s_{d+j}, dt)$. $\blacksquare$

**Por qué esto significa que no se pierde ni se repite nada.** Los dos tramos $[d, m)$ y $[m, h)$ cumplen:

```math
[d, m) \cap [m, h) = \emptyset \qquad\text{y}\qquad [d, m) \cup [m, h) = [d, h)
```

Es decir, forman una **partición** de $[d, h)$: cada índice cae en exactamente un tramo. Aplicada de forma recursiva, la partición de $[0, n)$ termina en tramos pequeños que son disjuntos entre sí y cuya unión es $[0, n)$: cada partícula se procesa **exactamente una vez**. La concatenación $\mathbin{+\!\!+}$ coloca el resultado de cada tramo en su lugar, en el orden de los índices, y eso se demostró en el punto 3 de la inducción.

Como ejemplo, con $n = 5$ y umbral $1$ la recursión produce este árbol. Las hojas son $[0,1), [1,2), [2,3), [3,4), [4,5)$: cinco tramos, cada índice exactamente una vez.

```mermaid
graph TD
    A["[0,5)"] --> B["[0,2)"]
    A --> C["[2,5)"]
    B --> D["[0,1)"]
    B --> E["[1,2)"]
    C --> F["[2,3)"]
    C --> G["[3,5)"]
    G --> H["[3,4)"]
    G --> I["[4,5)"]
```

**Terminación.** En cada llamado recursivo el tamaño del tramo disminuye estrictamente (paso 1 del caso de inducción), y el caso base se alcanza cuando el tamaño es a lo sumo el umbral o a lo sumo 1. Por lo tanto la recursión termina para todo umbral.

### 1.5 Teorema 1 (la versión paralela da lo mismo que la secuencial)

> **Teorema 1.** Para todo sistema $s$, todo $dt$ y todo umbral: $\text{pasoParalelo}(s, dt, \text{umbral}) = \text{pasoSecuencial}(s, dt)$.

**Demostración.** `pasoParalelo` devuelve $\text{avanzar}(0, n)$. Por el Lema 1 con $d = 0$ y $h = n$, ese arreglo tiene longitud $n$ y su elemento $j$ es $\text{mover}(s_j, dt)$. Por la sección 1.3, el arreglo de `pasoSecuencial` tiene la misma longitud y los mismos elementos. Dos arreglos con la misma longitud y los mismos elementos en cada posición son iguales. Esto vale para cualquier umbral, porque el Lema 1 no depende de él. $\blacksquare$

En este punto la igualdad es **exacta**, también con `Double`: cada elemento se calcula con la misma expresión $p_i + v_i \cdot dt$ sin importar cómo se agrupe el trabajo, y no hay sumas entre partículas que cambien de orden.

**Caso de control.** Si $\text{umbral} \geq n$, entonces $n \leq \text{umbral}$ y `avanzar(0, n)` cae en el caso base: $\text{pasoParalelo}(s, dt, n) = \text{pasoSegmento}(s, dt, 0, n)$, que es literalmente el cuerpo de `pasoSecuencial`. Las dos versiones ejecutan la misma función, sin ningún `parallel`, y por eso la aceleración medida con ese umbral debe dar $1$ (sección 5).

---

## 2. Punto 2: magnitudes globales en secuencial

### 2.1 Especificación

```math
E(s) = \sum_{i=0}^{n-1} e_i = \sum_{i=0}^{n-1} \tfrac{1}{2}\, m_i \lVert v_i \rVert^2
\qquad
M(s) = \sum_{i=0}^{n-1} m_i
\qquad
S(s) = \sum_{i=0}^{n-1} m_i\, p_i
```

```math
CM(s) = \begin{cases} S(s) / M(s) & \text{si } M(s) \neq 0 \\ (0, 0) & \text{si } M(s) = 0 \end{cases}
```

Una suma sin términos vale $0$ (en el caso de $S$, el vector $(0,0)$). Por eso $E(\text{Array}()) = 0$.

### 2.2 Cómo se argumenta un `foldLeft`

`s.foldLeft(z)(f)` recorre $s_0, s_1, \ldots, s_{n-1}$ en ese orden: parte del valor inicial $z$ y, con cada elemento, aplica $f$ al acumulador y a ese elemento. Es un programa iterativo y se argumenta con la técnica de estado, invariante y transformación:

- **Estado:** $(i, ac)$, donde $i$ es cuántos elementos se han procesado y $ac$ es el acumulador.
- **Estado inicial:** $(0, z)$.
- **Transformación:** $(i, ac) \mapsto (i + 1,\; f(ac, s_i))$.
- **Estado final:** $i = n$.
- **Terminación:** $n - i$ baja en 1 en cada paso y es no negativo.

### 2.3 `energiaSecuencial`

```scala
def energiaSecuencial(s: Sistema): Double =
  s.foldLeft(0.0) { (total, p) =>
    total + 0.5 * p.masa * p.vel.norma * p.vel.norma
  }
```

Aquí $z = 0$ y $f(ac, s_i) = ac + e_i$, porque $\tfrac12\, m\, \lVert v \rVert\, \lVert v \rVert = e$.

- **Invariante:** $\text{Inv}(i, ac) \equiv 0 \leq i \leq n \;\land\; ac = \sum_{k=0}^{i-1} e_k$.

**1. $\text{Inv}$ vale al inicio.** $0 \leq 0 \leq n$ y $ac = 0$ es la suma vacía.

**2. La invariante se conserva.** Suponga $i < n$ y $\text{Inv}(i, ac)$. El nuevo estado es $(i+1, ac + e_i)$. Se cumple $0 \leq i+1 \leq n$, y

```math
ac + e_i = \sum_{k=0}^{i-1} e_k + e_i = \sum_{k=0}^{i} e_k = \sum_{k=0}^{(i+1)-1} e_k
```

**3. Del estado final al resultado.** Con $i = n$ la invariante da $ac = \sum_{k=0}^{n-1} e_k = E(s)$.

Por lo tanto $\text{energiaSecuencial}(s) = E(s)$. Con el ejemplo del enunciado: $e_0 = \tfrac12 \cdot 2 \cdot 5^2 = 25$ y $e_1 = 0$, así que $E = 25$.

### 2.4 `masaTotal`

```scala
def masaTotal(s: Sistema): Double =
  s.foldLeft(0.0)((total, p) => total + p.masa)
```

Es el mismo argumento con $f(ac, s_i) = ac + m_i$ y la invariante $ac = \sum_{k < i} m_k$. Al terminar, $ac = M(s)$.

### 2.5 `centroDeMasaSecuencial`

```scala
def centroDeMasaSecuencial(s: Sistema): Vector2 = {
  val (posicionPonderada, masa) = s.foldLeft((Vector2(0.0, 0.0), 0.0)) {
    case ((suma, masaAcumulada), p) =>
      (suma + p.pos * p.masa, masaAcumulada + p.masa)
  }
  if (masa == 0.0) Vector2(0.0, 0.0)
  else posicionPonderada * (1.0 / masa)
}
```

- **Estado:** $(i, sx, ms)$, con estado inicial $(0, (0,0), 0)$.
- **Invariante:** $\text{Inv}(i, sx, ms) \equiv 0 \leq i \leq n \;\land\; sx = \sum_{k < i} m_k p_k \;\land\; ms = \sum_{k < i} m_k$.
- **Transformación:** $(i, sx, ms) \mapsto (i+1,\; sx + m_i p_i,\; ms + m_i)$.

La demostración es la de la sección 2.3 aplicada a las dos componentes a la vez. En el estado final, $sx = S(s)$ y $ms = M(s)$. Después se divide **una sola vez**: si $M(s) = 0$ el resultado es el origen, y si no es $S(s) \cdot (1 / M(s)) = S(s) / M(s)$. Eso es $CM(s)$.

Con el ejemplo del enunciado: $S = 2 \cdot (0,0) + 2 \cdot (4,0) = (8, 0)$, $M = 4$ y $CM = (2, 0)$.

---

## 3. Punto 3: las mismas magnitudes en paralelo

### 3.1 Programas

```scala
def energiaParalela(s: Sistema, umbral: Int): Double = {
  def sumar(desde: Int, hasta: Int): Double =
    if (hasta - desde <= umbral || hasta - desde <= 1) {
      def acumular(i: Int, total: Double): Double =
        if (i >= hasta) total
        else {
          val v = s(i).vel.norma
          acumular(i + 1, total + 0.5 * s(i).masa * v * v)
        }
      acumular(desde, 0.0)
    }
    else {
      val mitad = desde + (hasta - desde) / 2
      val (izquierda, derecha) = parallel(
        sumar(desde, mitad),
        sumar(mitad, hasta))
      izquierda + derecha
    }

  sumar(0, s.length)
}

def centroDeMasaParalelo(s: Sistema, umbral: Int): Vector2 = {
  def resumir(desde: Int, hasta: Int): (Vector2, Double) =
    if (hasta - desde <= umbral || hasta - desde <= 1)
      s.slice(desde, hasta).foldLeft((Vector2(0.0, 0.0), 0.0)) {
        case ((suma, masaAcumulada), p) =>
          (suma + p.pos * p.masa, masaAcumulada + p.masa)
      }
    else {
      val mitad = desde + (hasta - desde) / 2
      val (izquierda, derecha) = parallel(
        resumir(desde, mitad),
        resumir(mitad, hasta))
      (izquierda._1 + derecha._1, izquierda._2 + derecha._2)
    }

  val (posicionPonderada, masa) = resumir(0, s.length)
  if (masa == 0.0) Vector2(0.0, 0.0)
  else posicionPonderada * (1.0 / masa)
}
```

### 3.2 La operación de combinación es asociativa

Cada hoja de la recursión calcula una **magnitud parcial** de su tramo, y cada nodo interno combina las de sus dos hijos.

- Para la energía, las magnitudes parciales son números reales y la combinación es la suma $+$.
- Para el centro de masa, son pares $(S, M) \in \mathbb{R}^2 \times \mathbb{R}$ (suma ponderada de posiciones y masa) y la combinación es la suma componente a componente:

```math
(S_1, M_1) \oplus (S_2, M_2) = (S_1 + S_2,\; M_1 + M_2)
```

**Proposición 2.** $\oplus$ es **asociativa**, **conmutativa** y tiene elemento neutro $(\vec{0}, 0)$. Lo mismo vale para la suma de reales.

**Demostración.** Se calcula componente a componente, y cada componente es una suma de reales (o de vectores de $\mathbb{R}^2$), que es asociativa y conmutativa y tiene a $0$ como neutro:

```math
\big((S_1, M_1) \oplus (S_2, M_2)\big) \oplus (S_3, M_3) = (S_1 + S_2 + S_3,\; M_1 + M_2 + M_3) = (S_1, M_1) \oplus \big((S_2, M_2) \oplus (S_3, M_3)\big)
```

El neutro es $(\vec{0}, 0)$ porque $(S, M) \oplus (\vec{0}, 0) = (S, M)$. $\blacksquare$

**Por qué importa la asociatividad.** El recorrido secuencial suma de izquierda a derecha: $(\cdots((e_d + e_{d+1}) + e_{d+2}) + \cdots)$. La recursión paralela combina en forma de **árbol**, con otros paréntesis, por ejemplo $(e_d + e_{d+1}) + (e_{d+2} + e_{d+3})$. Que ambas formas den el mismo valor es exactamente la asociatividad. La conmutatividad no hace falta, porque la recursión conserva el orden de los índices. El neutro $0$ cubre el tramo vacío ($n = 0$).

### 3.3 Teorema 3 (energía)

> **Teorema 3.** Para todo sistema $s$ y todo umbral: $\text{energiaParalela}(s, \text{umbral}) = E(s)$ (en $\mathbb{R}$).

**Lema 3.** Para todo tramo $[d, h)$, $\text{sumar}(d, h) = \sum_{k=d}^{h-1} e_k$.

**Demostración**, por inducción fuerte sobre $k = h - d$.

**Caso base:** $k \leq \text{umbral}$ o $k \leq 1$. Entonces `sumar` devuelve $\text{acumular}(d, 0)$. Esa función es iterativa con estado $(i, ac)$, inicial $(d, 0)$, final $i \geq h$ y $\text{Inv}(i, ac) \equiv d \leq i \leq h \land ac = \sum_{k=d}^{i-1} e_k$. La demostración es la de la sección 2.3 cambiando los límites: el estado inicial cumple la invariante (suma vacía), cada transformación $(i, ac) \mapsto (i+1, ac + e_i)$ la conserva, el estado final tiene $i = h$ y $ac = \sum_{k=d}^{h-1} e_k$, y la terminación está garantizada porque $h - i$ baja en 1 en cada paso.

**Caso de inducción:** $k > \text{umbral}$ y $k \geq 2$. Como en el Lema 1, $d < m < h$ y los dos tramos son más pequeños, así que vale la HI: $a = \sum_{k=d}^{m-1} e_k$ y $b = \sum_{k=m}^{h-1} e_k$. Entonces

```math
a + b = \sum_{k=d}^{m-1} e_k + \sum_{k=m}^{h-1} e_k = \sum_{k=d}^{h-1} e_k
```

La última igualdad es la que garantiza que no se pierde ni se repite ninguna partícula: $[d, m)$ y $[m, h)$ son una partición de $[d, h)$ (Lema 1), así que cada $e_k$ aparece exactamente una vez. $\blacksquare$

**Demostración del Teorema 3.** Por el Lema 3 con $d = 0$ y $h = n$, $\text{energiaParalela}(s, \text{umbral}) = \sum_{k=0}^{n-1} e_k = E(s)$, que es lo que calcula `energiaSecuencial` (sección 2.3). $\blacksquare$

### 3.4 Teorema 4 (centro de masa) y por qué los promedios no se promedian

> **Teorema 4.** Para todo sistema $s$ y todo umbral: $\text{centroDeMasaParalelo}(s, \text{umbral}) = CM(s)$ (en $\mathbb{R}$).

**Lema 4.** Para todo tramo $[d, h)$:

```math
\text{resumir}(d, h) = \Big( \sum_{k=d}^{h-1} m_k p_k,\; \sum_{k=d}^{h-1} m_k \Big)
```

**Demostración.** Es la del Lema 3 para las dos componentes a la vez.

- *Caso base.* `s.slice(d, h)` es el arreglo con los elementos $s_d, \ldots, s_{h-1}$, en ese orden. El `foldLeft` sobre él es el de la sección 2.5 restringido a ese tramo, con la misma invariante, y devuelve el par de sumas sobre $[d, h)$.
- *Caso de inducción.* La HI da los pares de $[d, m)$ y $[m, h)$, y $\oplus$ los suma componente a componente. Como $[d, m)$ y $[m, h)$ son una partición de $[d, h)$, la suma de las dos mitades es la suma de todo el tramo. $\blacksquare$

**Demostración del Teorema 4.** Por el Lema 4, `resumir(0, n)` devuelve $(S(s), M(s))$. Después se divide una sola vez, igual que en `centroDeMasaSecuencial`: origen si $M(s) = 0$ y $S(s)/M(s)$ si no. $\blacksquare$

**Por qué la suma es válida para repartir y el promedio no.**

- La **suma** es asociativa (Proposición 2) y tiene neutro. Eso permite agrupar el trabajo como se quiera y obtener lo mismo, y un tramo sin partículas simplemente aporta $(\vec{0}, 0)$.
- El **promedio no se puede aplicar directamente a los resultados parciales**, por dos razones.
  1. *Ignora cuánto pesa cada mitad.* Sea la mitad $A$ con una partícula de masa $1$ en $x = 0$, y la mitad $B$ con una partícula de masa $3$ en $x = 4$. Los centros de masa parciales son $0$ y $4$, y su promedio es $2$. Pero el centro de masa verdadero es $\dfrac{1 \cdot 0 + 3 \cdot 4}{1 + 3} = 3$.
  2. *No es asociativa.* Con $\text{prom}(a, b) = (a + b)/2$:

```math
\text{prom}(\text{prom}(0, 4), 8) = \tfrac{2 + 8}{2} = 5
\qquad\neq\qquad
\text{prom}(0, \text{prom}(4, 8)) = \tfrac{0 + 6}{2} = 3
```

  El resultado dependería de cómo se agruparan los tramos, es decir, de cómo se repartió el trabajo.
- Por eso se lleva el par $(S, M)$, cuya combinación sí es asociativa, y la división $S/M$ se hace **una sola vez, al final**.

### 3.5 Punto flotante

Los Teoremas 3 y 4 son exactos en $\mathbb{R}$. Con `Double` (IEEE 754) la suma **no es exactamente asociativa**: cambiar los paréntesis cambia el redondeo. Entonces la versión paralela y la secuencial no siempre coinciden bit a bit, sino con una diferencia muy pequeña. Para la energía, que suma términos no negativos, el error relativo de cualquier orden de suma es del orden de $(n-1)\, 2^{-53}$, y por eso la diferencia entre las dos versiones también es diminuta. Por esta razón las pruebas comparan con una **tolerancia** y no con igualdad exacta, como pide el enunciado. Esto es distinto del punto 1, donde la igualdad sí es exacta porque no hay sumas entre partículas.

---

## 4. Punto 4: trabajo y profundidad

### 4.1 Programas

```scala
def trabajo(n: Int): Int = n

def profundidad(n: Int, umbral: Int): Int =
  if (n <= 0) 0
  else if (n <= umbral || n <= 1) n
  else {
    val mitad = n / 2
    1 + math.max(profundidad(mitad, umbral), profundidad(n - mitad, umbral))
  }

def tiempoEstimado(n: Int, p: Int, umbral: Int): Double =
  profundidad(n, umbral) + trabajo(n).toDouble / p
```

### 4.2 Especificación

- **Trabajo:** una operación por partícula, y repartir no lo reduce: $W(n) = n$.
- **Profundidad:** el número de veces que se parte hasta llegar al umbral, más el trabajo del tramo que queda sin partir. Como `parallel` termina cuando acaba la rama más lenta, se toma el máximo de las dos mitades:

```math
D(n) = \begin{cases}
0 & \text{si } n \leq 0 \\
n & \text{si } n \geq 1 \text{ y } (n \leq \text{umbral} \text{ o } n \leq 1) \\
1 + \max\big(D(\lfloor n/2 \rfloor),\, D(\lceil n/2 \rceil)\big) & \text{en otro caso}
\end{cases}
```

- **Tiempo estimado:** $T(n, p) = D(n) + W(n)/p$.

### 4.3 Terminación y significado de `profundidad`

**Terminación.** En el caso recursivo $n \geq 2$, y $1 \leq \lfloor n/2 \rfloor \leq \lceil n/2 \rceil < n$. Cada llamado recibe un entero estrictamente menor y siempre $\geq 1$, así que se llega al caso base.

> **Proposición 5.** Considere el árbol de llamados de $D(n)$, donde cada hoja es un tramo que no se parte. Si una hoja está a profundidad $t$ (partió $t$ veces para llegar a ella) y tiene tamaño $q$, entonces
>
> ```math
> D(n) = \max_{\text{hojas}} \,(t + q)
> ```

**Demostración**, por inducción fuerte sobre $n$.

**Caso base:** $n \leq 0$, o $n \leq \text{umbral}$, o $n \leq 1$. El árbol es una sola hoja con $t = 0$ y $q = n$, y $D(n) = n = 0 + n$ (con $D(n) = 0$ si $n \leq 0$).

**Caso de inducción:** $n \geq 2$ y $n > \text{umbral}$. Cada hoja del árbol de $n$ es una hoja de uno de los dos subárboles, con una partición más: su profundidad aumenta en 1. Por la HI, el máximo de $t' + q$ sobre las hojas de cada subárbol es $D(\lfloor n/2 \rfloor)$ y $D(\lceil n/2 \rceil)$, y entonces

```math
\max_{\text{hojas}} (t + q) = 1 + \max\big(D(\lfloor n/2 \rfloor),\, D(\lceil n/2 \rceil)\big) = D(n) \qquad \blacksquare
```

**Corolario (forma cerrada en un caso típico).** Si $n = 2^j c$ con $c \geq 1$ y el umbral es exactamente $c$, todos los tramos de un nivel tienen el mismo tamaño $n/2^t$. Los tramos se parten mientras superan $c$, es decir, hasta $t = j$, y todas las hojas tienen $t = j$ y $q = c$. Entonces $D(n) = j + c$.

**Observación.** $D$ no es monótona en $n$: con umbral 1000, $D(1000) = 1000$ pero $D(1001) = 1 + \max(D(500), D(501)) = 1 + 501 = 502$. Por eso se necesita el máximo sobre **las dos** mitades: con `profundidad(2001, 1000)` las mitades son $1000$ y $1001$, y la de $1000$ (que no se parte) es la que más pesa: $D(2001) = 1 + \max(1000, 502) = 1001$.

### 4.4 Los ejemplos del enunciado

| Llamado | Cálculo | Resultado |
|---|---|---|
| `profundidad(1024, 1024)` | $j = 0$, $c = 1024$: $0 + 1024$ | 1024 |
| `profundidad(1024, 512)` | $j = 1$, $c = 512$: $1 + 512$ | 513 |
| `profundidad(1024, 1)` | $j = 10$, $c = 1$: $10 + 1$ | 11 |
| `tiempoEstimado(1024, 4, 1)` | $11 + 1024/4$ | 267.0 |
| `trabajo(1024)` | $n$ | 1024 |
| `profundidad(8, 2)` | $j = 2$, $c = 2$: $2 + 2$ | 4 |
| `profundidad(10, 3)` | $1 + \max(D(5), D(5))$, con $D(5) = 1 + \max(D(2), D(3)) = 1 + 3 = 4$ | 5 |

### 4.5 Alcance de la estimación

`profundidad` es una **estimación del camino crítico** de `pasoParalelo`: el trabajo del tramo secuencial más grande, más las particiones necesarias para llegar a él. No incluye el costo de crear tareas ni de concatenar arreglos. Es una cota idealizada que se compara con lo medido, no una medida exacta (se discute en el informe de paralelización).

---

## 5. Punto 5: medir la aceleración

### 5.1 Programas

```scala
def aceleracion(tSecuencial: Double, tParalelo: Double): Double =
  tSecuencial / tParalelo

def umbralConveniente(s: Sistema, dt: Double, umbrales: List[Int]): Int =
  umbrales match {
    case Nil => s.length
    case primero :: resto =>
      val tiempoPrimero = medir(3) { pasoParalelo(s, dt, primero) }
      resto.foldLeft((primero, tiempoPrimero)) {
        case ((mejorUmbral, mejorTiempo), candidato) =>
          val tiempo = medir(3) { pasoParalelo(s, dt, candidato) }
          if (tiempo < mejorTiempo) (candidato, tiempo)
          else (mejorUmbral, mejorTiempo)
      }._1
  }
```

`medir` es la función que trae el repositorio: ejecuta el bloque varias veces con `withWarmer(new Warmer.Default) measure`, que calienta la máquina virtual antes de medir, y devuelve la mediana de las mediciones en milisegundos.

### 5.2 `aceleracion`

La especificación es el cociente $A = t_{\text{sec}} / t_{\text{par}}$, con $t_{\text{par}} > 0$ (un tiempo medido nunca es cero), y el programa es literalmente ese cociente. Los ejemplos: $100/25 = 4$ y $100/100 = 1$.

**Caso de control.** Por la sección 1.5, con umbral igual al tamaño del sistema `pasoParalelo` ejecuta el mismo código que `pasoSecuencial`. Los tiempos son dos mediciones de la misma computación, así que $A = t/t' \approx 1$, con la única diferencia del ruido de la medición. Si el control da un valor muy distinto de $1$, la comparación no es justa o la medición está contaminada.

### 5.3 `umbralConveniente`

**Especificación.** Sea $L = (u_1, \ldots, u_r)$ la lista de umbrales y $t_k$ el tiempo que `medir` devuelve para $u_k$ (una medición por candidato). Si $L$ es vacía, el resultado es $n$, que equivale a no repartir. Si no, el resultado es $u_j$, donde $j$ es el **primer** índice con $t_j = \min_k t_k$.

**Argumento.** Si la lista es vacía, el programa devuelve `s.length` $= n$. Si no, el `foldLeft` es un programa iterativo:

- **Estado:** $(k, (u^*, t^*))$, donde $k$ es cuántos candidatos se han procesado.
- **Estado inicial:** $(1, (u_1, t_1))$, porque el primer candidato se mide antes del `foldLeft`.
- **Invariante:** $\text{Inv}(k, (u^*, t^*)) \equiv (u^*, t^*) = (u_j, t_j)$, con $j$ el primer índice de $\{1, \ldots, k\}$ que minimiza $t$.
- **Transformación:** al recibir el candidato $u_{k+1}$ con tiempo $t_{k+1}$, el par se reemplaza si y solo si $t_{k+1} < t^*$.

**Se conserva.** Si $t_{k+1} < t^*$, entonces $t_{k+1}$ es estrictamente menor que el mínimo de los $k$ primeros y el nuevo primer mínimo es $k+1$. Si $t_{k+1} \geq t^*$, el primer mínimo no cambia (en un empate se conserva el anterior, por la desigualdad estricta). En el estado final $k = r$, y `._1` entrega $u_j$.

**Límite de esta garantía.** A diferencia de las otras funciones, el resultado depende de **mediciones** y no solo de los datos de entrada. La función es correcta respecto a los tiempos medidos en esa ejecución; con otra máquina o con otra carga, otro umbral podría ganar si los tiempos son muy parecidos. Por eso `medir` toma la mediana de varias corridas, que reduce el efecto de una corrida atípica.

---

## 6. Resumen

| Función | Técnica | Qué se demuestra |
|---|---|---|
| `mover`, `pasoSegmento`, `pasoSecuencial` | Definición de `tabulate` | Cada elemento es la partícula $i$ avanzada $dt$ |
| `pasoParalelo` | Inducción fuerte (Lema 1) | La partición no pierde ni repite; es igual a `pasoSecuencial` (Teorema 1) |
| `energiaSecuencial`, `masaTotal`, `centroDeMasaSecuencial` | Invariante de `foldLeft` | El estado final contiene la suma pedida |
| `energiaParalela` | Inducción fuerte (Lema 3) y asociatividad de $+$ | Igual a $E(s)$ (Teorema 3) |
| `centroDeMasaParalelo` | Inducción fuerte (Lema 4) y asociatividad de $\oplus$ | Igual a $CM(s)$ (Teorema 4); el promedio no sirve como combinación |
| `trabajo`, `profundidad`, `tiempoEstimado` | Inducción fuerte (Proposición 5) | $D(n)$ es el máximo de profundidad más tamaño de hoja; ejemplos del enunciado |
| `aceleracion`, `umbralConveniente` | Definición e invariante de `foldLeft` | Cociente de tiempos; el umbral devuelto es el de menor tiempo medido |

Correspondencia con la rúbrica:

- **Paralelismo, criterio 2** (la división en segmentos no pierde ni repite): Lema 1 (sección 1.4) y Lema 3 (sección 3.3).
- **Paralelismo, criterio 3** (la operación que combina es asociativa): Proposición 2 (sección 3.2).
- **Corrección, criterio 3** (la versión paralela da lo mismo que la secuencial): Teoremas 1, 3 y 4 (secciones 1.5, 3.3 y 3.4), con la salvedad de punto flotante de la sección 3.5.
