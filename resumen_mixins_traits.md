# Resumen Exhaustivo: Mixins, Traits y Stateful Traits

---

## Paper 1: Mixin-based Inheritance
**Autores:** Gilad Bracha y William Cook  
**Publicado en:** Proceedings of OOPSLA/ECOOP 1990

---

### 1.1 Contexto y motivación

El paper parte de la observación de que Smalltalk, Beta y CLOS representan tres mecanismos de herencia superficialmente distintos, pero que en realidad comparten una estructura subyacente común. El objetivo es identificar esa estructura y generalizarla en un modelo unificado basado en la **composición de mixins**.

El mecanismo subyacente es un operador binario no asociativo `⊲` que combina un conjunto de atributos (delta) con un padre:

```
C = Δ(P) ⊕ P
```

donde `Δ` es el conjunto de cambios y `P` es la especificación del padre.

---

### 1.2 Herencia en Smalltalk

En Smalltalk, la subclase está en control: puede **reemplazar** métodos del padre. El subclase referencia el padre mediante `super`.

```
∆ ⊲ P = ∆(P) ⊕ P    (Smalltalk)
```

Los nuevos atributos tienen precedencia sobre los heredados. La subclase puede reescribir completamente cualquier método del padre.

**Ejemplo conceptual:**
- `Person` define `display` que muestra el nombre.
- `Graduate` extiende `display` invocando `super display` y luego mostrando el grado académico.

**Limitación:** no es posible definir un delta independientemente del padre, por lo que no hay reutilización real del delta.

---

### 1.3 Herencia en Beta

En Beta, el **padre está en control** (prefixing). El padre usa el comando `inner` para invocar el comportamiento extendido del subpatrón. Los métodos del padre no pueden ser reemplazados, solo extendidos.

```
C'(∅) = P' ⊲ ∆'(∅)    (Beta)
```

Los atributos originales tienen precedencia sobre las extensiones. Beta y Smalltalk tienen jerarquías **invertidas**: la relación subclase/superclase de Smalltalk es análoga a la relación superpatrón/subpatrón en Beta.

**Limitación:** si se quiere una familia de clases mezcladas, hay que copiar la clase base completa.

---

### 1.4 Herencia múltiple en CLOS y Mixins

CLOS soporta herencia múltiple mediante **linearización** del grafo de ancestros. El grafo se convierte en una lista lineal donde cada ancestro aparece exactamente una vez. Esto resuelve el problema de la herencia duplicada (diamond problem).

```
C = ∆₁ ⊲ (∆₂ ⊲ (... ⊲ (∆ₙ ⊲ ∅)))
```

**Crítica:** la linearización viola encapsulamiento porque puede alterar las relaciones padre-hijo entre clases.

Un **mixin en CLOS** es una subclase abstracta diseñada para ser aplicada a diferentes padres, agregando comportamiento parcial. Los mixins dependen de la linearización para ubicarse en la cadena de herencia. Formalmente, un mixin es una **subclase abstracta parametrizada por su padre**.

---

### 1.5 Herencia como composición de mixins

La propuesta central del paper es tomar los mixins como construcción definitoria primaria. La herencia se formula como **composición explícita de mixins**.

El operador de composición de mixins `?` se define como:

```
M₁ ? M₂ = fun(i) M₁(M₂(i) ⊕ i) ⊕ M₂(i)
```

- `M₁ ? M₂` es asociativo.
- La composición es **explícita**: no hay linearización implícita.
- Una clase ordinaria es un mixin degenerado (no usa el parámetro inner/super).
- Un mixin es **completo** si no referencia su parámetro padre y define todos sus campos.

**Ventaja clave:** como la composición es explícita, no puede violar el encapsulamiento. El programador elige explícitamente el orden de todos los componentes.

---

### 1.6 Aplicación a Modula-3

Se extiende Modula-3 generalizando los tipos objeto a mixins. La sintaxis resultante permite:

```
type GraduateMixin =
  object degree: string
  methods display := displayGraduateMixin
  super display() := No_Op
  end;

type Graduate = Person GraduateMixin;
```

Las reglas de tipado establecen que `T₁ = T₂ T₃` implica `T₁ <: T₂` y `T₁ <: T₃`. Todas las reglas se pueden verificar estáticamente, lo que garantiza seguridad y eficiencia.

---

### 1.7 Conclusiones del paper

- Smalltalk, Beta y CLOS son variaciones de un mismo mecanismo subyacente de composición.
- Los mixins hacen explícita la abstracción que CLOS implementa implícitamente mediante linearización.
- La composición explícita elimina los problemas de encapsulamiento de la linearización.
- La extensión a Modula-3 muestra que el enfoque es aplicable a lenguajes con tipado estático.

---
---

## Paper 2: Traits: Composable Units of Behaviour
**Autores:** Nathanael Scharli, Stéphane Ducasse, Oscar Nierstrasz, Andrew P. Black  
**Publicado en:** ECOOP 2003, LNCS 2743, pp. 248–274

---

### 2.1 Motivación y problemas identificados

El paper argumenta que ninguna de las variantes de herencia existentes (simple, múltiple, por mixins) es suficientemente expresiva ni libre de problemas conceptuales.

#### Problemas con herencia simple
- No es suficientemente expresiva para factorizar todas las características comunes.
- Puede forzar duplicación de código en jerarquías complejas.
- Las interfaces de Java resuelven el subtyping pero no el code sharing.

#### Problemas con herencia múltiple
1. **Conflictos de características:** El "diamond problem" es especialmente problemático para el estado. Los conflictos de variables de instancia son más difíciles de resolver que los de métodos.
2. **Acceso a características sobreescritas:** `super` no es suficiente cuando hay múltiples padres. C++ y Eiffel requieren nombrar explícitamente la superclase, lo que hace el código frágil.
3. **Wrappers genéricos:** No es posible escribir una entidad reutilizable que envuelva métodos de clases aún desconocidas. El ejemplo del `SyncReadWrite` ilustra que los `super`-sends se resuelven estáticamente.

#### Problemas con mixins
1. **Ordenamiento total:** La composición de mixins es lineal; puede no existir un orden total adecuado para resolver ciertos conflictos.
2. **Dispersión del glue code:** El código de conexión queda hardcodeado en clases intermediarias generadas al aplicar los mixins uno a uno. La entidad compuesta no tiene control total sobre cómo se combinan los mixins.
3. **Jerarquías frágiles:** Agregar un método a un mixin puede silenciosamente sobreescribir un método de un mixin anterior en la cadena. Puede ser imposible restablecer el comportamiento original sin modificar varios mixins.

---

### 2.2 Definición de Traits

Un **trait** es un conjunto de métodos que implementa comportamiento. Sus propiedades fundamentales son:

- **Provee** un conjunto de métodos (comportamiento).
- **Requiere** un conjunto de métodos que sirven como parámetros del comportamiento provisto.
- **No especifica variables de estado:** los métodos nunca acceden directamente a variables de instancia.
- La **composición es conmutativa y asociativa:** el orden es irrelevante.
- Los conflictos deben ser resueltos **explícitamente**.
- **Propiedad de aplanamiento (flattening property):** la semántica de una clase definida con traits es idéntica a la de una clase donde todos los métodos de los traits estuvieran definidos directamente en ella.

La ecuación fundamental es:

```
Clase = Superclase + Estado + Traits + Glue
```

---

### 2.3 Operadores de composición

| Operador | Descripción |
|----------|-------------|
| **Suma (+)** | Combina dos traits; los métodos en conflicto se marcan como conflictivos |
| **Exclusión (−)** | Elimina un método de un trait para evitar conflictos |
| **Alias (@)** | Introduce un nombre alternativo para un método de un trait |
| **Sobreescritura** | Los métodos de la clase tienen precedencia sobre los métodos del trait |

---

### 2.4 Reglas de precedencia

1. Los **métodos de la clase** tienen precedencia sobre los métodos del trait.
2. Los **métodos del trait** tienen precedencia sobre los métodos de la superclase (consecuencia del flattening).

---

### 2.5 Ejemplo de uso (Smalltalk/Squeak)

```smalltalk
"Definición de un trait"
Trait named: #TDrawing uses: {}
  draw
    ↑ self drawOn: World canvas
  refresh
    ↑ self refreshOn: World canvas
  refreshOn: aCanvas
    aCanvas form deferUpdatesIn: self bounds while: [self drawOn: aCanvas]

"Composición de una clase"
Object subclass: #Circle
  instanceVariableNames: 'center radius'
  uses: { TCircle . TDrawing }
  initialize
    center := 0@0. radius := 50
  drawOn: aCanvas
    aCanvas fillOval: self bounds color: Color black
```

---

### 2.6 Traits compuestos

Los traits pueden componerse de otros traits. Las propiedades de los subtraits se propagan:
- Los **métodos provistos** de los subtraits se propagan al trait compuesto.
- Los **requerimientos no satisfechos** de los subtraits se convierten en requerimientos del trait compuesto.
- La **propiedad de aplanamiento** se mantiene en múltiples niveles de composición.

---

### 2.7 Resolución de conflictos

Un conflicto ocurre **si y solo si** dos traits proveen métodos con el mismo nombre que **no se originan del mismo trait**. Si el mismo método se obtiene múltiples veces por diferentes caminos pero proviene del mismo trait, **no hay conflicto** (same-operation exception).

Los conflictos se resuelven:
1. Definiendo un método en la clase o trait compuesto que sobreescriba los conflictivos.
2. Usando **aliases** para acceder a los métodos conflictivos sin duplicarlos.
3. Usando **exclusión** para descartar uno de los métodos conflictivos.

```smalltalk
"Resolución con aliases"
Object subclass: #Circle
  uses: { TCircle @ {#circleHash -> #hash. #circleEqual: -> #=} .
          TColor  @ {#colorHash  -> #hash. #colorEqual:  -> #=} }
  hash
    ↑ self circleHash bitXor: self colorHash
  = anObject
    ↑ (self circleEqual: anObject) and: [self colorEqual: anObject]
```

---

### 2.8 Diseño: aliasing vs. renaming

El paper elige **aliasing** (en lugar de renaming como en Eiffel) porque:
- El renaming invalida el flattening property, ya que cambia el nombre original.
- El aliasing simplemente agrega un nombre alternativo sin afectar referencias existentes.
- El aliasing no requiere modificar el código fuente de los métodos que referencian el nombre original.

---

### 2.9 Evaluación contra los problemas identificados

| Problema | Solución en Traits |
|----------|-------------------|
| Conflictos de estado | No aplica: traits no tienen estado |
| Conflictos de métodos | Resolución explícita obligatoria en la clase |
| Acceso a características sobreescritas | Via aliases; sin referencias a nombres de traits en el código fuente |
| Wrappers genéricos | `super` en un trait referencia al padre de la clase que usa el trait |
| Ordenamiento total | La composición es simétrica; el orden es irrelevante |
| Dispersal del glue code | El glue siempre está en la entidad combinadora |
| Jerarquías frágiles | Los cambios directos pueden generar conflictos, pero son locales y resolubles sin efecto cascada |

---

### 2.10 Implementación en Squeak

- Se extiende la implementación de clases con una variable adicional para el composition clause.
- Los métodos de un trait se copian al diccionario de métodos de la clase (excepto los sobreescritos).
- Los métodos con `super` requieren una copia especial porque almacenan referencia explícita a la superclase.
- **No se requieren cambios en la VM** de Squeak.
- **Sin penalización de performance:** equivalente a una implementación de herencia simple donde los métodos del trait estuvieran definidos directamente en la clase.

---

### 2.11 Caso de estudio: jerarquía de colecciones de Smalltalk-80

Se refactorizó la jerarquía de colecciones de Squeak 3.2:
- Se crearon **48 traits** diferentes.
- Se implementaron **567 métodos** (10% menos que la implementación original).
- El código es **12% más pequeño**.
- Adicionalmente, el 9% de los métodos originales estaban ubicados demasiado alto en la jerarquía (para compartir código), un problema que los traits eliminan por completo.

---

### 2.12 Conclusiones del paper

- Los traits son una extensión completamente compatible con herencia simple.
- La propiedad de aplanamiento garantiza comprensibilidad óptima.
- Los traits son ideales para lenguajes de herencia simple como Smalltalk.
- Como trabajo futuro se plantea: namespaces, traits con estado, reemplazar herencia con composición, traits para instancias individuales, y sistemas de tipos para traits.

---
---

## Paper 3: Stateful Traits
**Autores:** Alexandre Bergel, Stéphane Ducasse, Oscar Nierstrasz, Roel Wuyts  
**Publicado en:** 14th International Smalltalk Conference (ISC 2006), LNCS vol. 4406

---

### 3.1 Motivación: limitaciones de los traits sin estado

Los **stateless traits** (traits del Paper 2) son puramente grupos de métodos sin variables de instancia. El acceso al estado se logra a través de **required methods** (accessors). Aunque esto funciona razonablemente bien en la práctica, genera cuatro limitaciones conceptuales:

#### (i) Reusabilidad limitada
La interfaz requerida de un trait queda saturada de accessors sin interés conceptual. La atención del programador debería enfocarse en los métodos hook que sí son responsabilidad del cliente, no en los accessors triviales.

#### (ii) Boilerplate glue code
Cada clase cliente debe generar variables de instancia, accessors y código de inicialización duplicado. En la jerarquía de colecciones refactorizada con stateless traits, el **24% de las clases (7 de 29) eran "shell classes"**: clases vacías que solo declaran variables y definen sus accessors.

#### (iii) Propagación de accessors requeridos
Si la implementación de un trait evoluciona y necesita una nueva variable, todos los clientes se ven impactados aunque la interfaz pública no cambie. Los required accessors se propagan y acumulan de trait en trait.

#### (iv) Violación de encapsulamiento
En Smalltalk, los métodos son públicos. Si un trait requiere accessors, las clases cliente deben exponer públicamente información que quizás debería ser privada.

---

### 3.2 Solución: Stateful Traits

Los stateful traits extienden los stateless traits permitiendo que los traits tengan **variables de instancia privadas**. El principio guía se mantiene: **el cliente retiene el control de la composición**.

**Tres aspectos centrales del modelo:**

1. Las variables de instancia son **privadas al scope del trait** que las define.
2. El cliente (clase o trait compuesto) puede **acceder a variables seleccionadas** de un trait usado, mapeándolas a nuevos nombres privados al scope del cliente.
3. El cliente puede **mergear variables de múltiples traits** mapeándolas a un nombre común.

---

### 3.3 Operador de acceso a variables (`@@`)

El operador `@@` es la única adición sintáctica/semántica al modelo original.

**Sintaxis:**
```
T @@ { nuevoNombre → nombreOriginal }
```

Si `T` define una variable privada `x`, entonces `T @@ {y → x}` representa un nuevo trait en el que la variable `x` es accesible desde el scope del cliente bajo el nombre `y`. El nombre `x` sigue siendo privado al trait.

**Ejemplo:**
```smalltalk
Trait named: #TSyncReadWrite
  uses: {}
  instVarNames: 'lock'

Object subclass: #SyncStream
  uses: TSyncReadWrite @ {#hashFromSync → #hash}
        @@ {syncLock → lock}
       + TStream @ {#hashFromStream → #hash}
  instVarNames: ''
```

---

### 3.4 Tres escenarios de composición de variables

#### Escenario 1: Variables privadas (sin `@@`)
Cada trait mantiene su variable `x` completamente privada. No hay conflictos posibles. Corresponde a composición **black-box** (similar a Jigsaw).

```smalltalk
"T1 y T2 cada uno tienen su propio x; C tiene su propio x"
C usa: T1 + T2
"c getXT1, c getXT2 y c getX son independientes"
```

#### Escenario 2: Granting variable access
El cliente accede a variables privadas de los traits bajo nuevos nombres.

```smalltalk
"C accede a x de T1 como xFromT1 y a x de T2 como xFromT2"
T1 @@ {xFromT1 → x} + T2 @@ {xFromT2 → x}
```

#### Escenario 3: Merging de variables
El cliente mapea variables de múltiples traits al mismo nombre, unificándolas en una única variable compartida.

```smalltalk
"x de T1 e y de T2 se fusionan en la variable w"
T1 @@ {w → x} + T2 @@ {w → y}
"c getX, c getY y c getW devuelven el mismo valor"
```

---

### 3.5 Propiedad de aplanamiento preservada

La flattening property se mantiene con stateful traits: se puede aplanar una jerarquía de traits a una clase equivalente. Para preservarla, las variables privadas de los traits se **alpha-renombran** para evitar colisiones de nombres en el scope del cliente (técnicamente, se puede usar name-mangling prefijando el nombre del trait).

---

### 3.6 Impacto limitado de cambios

- Agregar una variable a un trait **no afecta a sus clientes** porque las variables son privadas.
- Eliminar o renombrar una variable solo afecta a clientes directos que explícitamente accedían a esa variable con `@@`.
- Una vez adaptados los clientes directos, **ningún efecto cascada** puede ocurrir en clientes indirectos.
- Esto contrasta fuertemente con los stateless traits, donde un nuevo requerimiento de accessor se propaga transitivamente por toda la jerarquía.

---

### 3.7 Implementación

Se implementaron dos estrategias para manejar la linearización del estado:

#### Estrategia 1: Object state as a dictionary
El estado de un objeto se implementa como una tabla hash donde múltiples claves pueden apuntar al mismo valor (para el merging). Refleja directamente la semántica de los stateful traits, pero tiene overhead de performance.

#### Estrategia 2: Copy-down (adoptada)
Inspirada en Strongtalk. Los métodos que acceden a variables con diferentes offsets son copiados y ajustados. Los métodos que no acceden a variables de instancia ni a `super` se comparten entre trait y sus usuarios.

**Ventajas del copy-down:** no hay overhead en el acceso a variables; el acceso sigue siendo offset-based.

**Benchmarks:** comparando stateless traits, stateful traits y Squeak puro en dos casos de estudio (SyncStream y LinkChecker), **no se observa diferencia de performance** entre las tres implementaciones.

---

### 3.8 Caso de estudio: jerarquía de colecciones refactorizada

En la versión con stateless traits, la clase `Heap` debía definir 3 variables (`array`, `tally`, `sortBlock`) y todos sus accessors, actuando como shell class para satisfacer las necesidades de los traits `TArrayBased` y `TSortBlockBased`.

Con stateful traits, las variables se trasladan a los traits que efectivamente las necesitan:
- `TArrayBased` define `array` y `tally` directamente.
- `TSortBlockBased` define `sortBlock` directamente.
- `Heap` queda vacía de variables y puede incluso absorber los métodos de `THeapImpl`.

**Beneficios obtenidos:**
- Encapsulamiento preservado.
- Menos definiciones de métodos (accessors innecesarios eliminados).
- Menos requerimientos de métodos.
- Eliminación de las shell classes.

---

### 3.9 Comparaciones con otros sistemas

| Sistema | Similitudes | Diferencias clave |
|---------|-------------|-------------------|
| **Eiffel** | Renaming, selección de features en conflicto | Renaming en Eiffel reemplaza el nombre original; aliasing en traits lo mantiene. Merging en Eiffel solo para variables de superclase común. |
| **Jigsaw** | Operadores de composición similares (merge, override, restrict) | Jigsaw es black-box estricto; no permite abrir módulos. Variables no se pueden compartir entre módulos. |
| **Cecil** | Herencia múltiple | Cecil no soporta exclusión ni renaming de features heredados. |
| **Self** | Trait objects como repositorios de comportamiento compartido | Self no tiene operadores específicos de composición; usa objetos padre ordinarios. |
| **Scala** | Traits en Scala | No pueden definir estado en la versión original. |

---

### 3.10 Conclusiones del paper

Los stateful traits representan una **extensión mínima** al modelo de stateless traits que resuelve sus limitaciones prácticas sin sacrificar sus propiedades teóricas. El principio fundamental —el cliente retiene el control de la composición— se preserva. La propiedad de aplanamiento se mantiene mediante alpha-renaming. La implementación copy-down garantiza rendimiento equivalente a herencia simple.

---
---

## Preguntas de análisis comparativo

---

### Pregunta 1: ¿Qué es un Mixin? ¿Qué es un Trait? ¿Cuál es la diferencia?

#### Mixin (Bracha & Cook, 1990)

Un **mixin** es una **subclase abstracta parametrizada por su clase padre**. Es una especificación de subclase que puede aplicarse a diferentes padres para crear una familia de clases modificadas relacionadas. Formalmente, un mixin es un delta independiente: un conjunto de cambios que puede existir sin un padre concreto y ser aplicado a distintos padres en distintos contextos.

```
type GraduateMixin =
  object degree: string
  methods display := displayGraduateMixin
  end;

type Graduate    = Person GraduateMixin;
type GuardDog    = Dog    GraduateMixin;
```

El mixin se **compone linealmente**: la composición respeta un orden, y el mixin que aparece "antes" tiene precedencia sobre el que aparece "después" (o viceversa, según la dirección). La composición de dos mixins produce otro mixin (no una clase).

#### Trait (Scharli et al., 2003)

Un **trait** es una **unidad primitiva de reutilización de comportamiento puro**. Es un conjunto de métodos que provee comportamiento y puede requerir métodos que sirven como parámetros. Los traits **no tienen estado** (en su versión original) y su composición es **simétrica y sin orden**.

```smalltalk
Trait named: #TDrawing uses: {}
  draw
    ↑ self drawOn: World canvas
  bounds
    self requirement
```

El trait no es una clase ni una subclase; es una entidad independiente que sirve exclusivamente como unidad de reuso dentro de una clase.

#### Diferencias fundamentales

| Dimensión | Mixin | Trait |
|-----------|-------|-------|
| **Naturaleza** | Subclase abstracta (delta parametrizado) | Grupo de métodos puro, no ligado a herencia |
| **Composición** | Lineal (ordenada), usa el operador de herencia | Simétrica (no ordenada), operadores propios |
| **Estado** | Puede tener variables de instancia | No (stateless) / Privadas con `@@` (stateful) |
| **Conflictos** | Se resuelven por el orden de composición (implícito) | Deben resolverse **explícitamente** |
| **Glue code** | Distribuido en clases intermedias generadas | Centralizado en la entidad compositora |
| **`super`** | Se resuelve según la cadena lineal de herencia | En la clase que usa el trait (flattening) |
| **Granularidad** | Más gruesa (conjunto de features + posible estado) | Más fina (comportamiento puro) |
| **Lugar en jerarquía** | Tiene un lugar específico en la cadena de herencia | Independiente de la jerarquía |

---

### Pregunta 2: ¿Qué es un conflicto? ¿Cómo se maneja en cada tecnología?

Un **conflicto** ocurre cuando dos o más entidades compuestas proveen una característica (método o variable) con el mismo nombre pero con implementaciones o semánticas distintas.

#### En Mixins

El conflicto se maneja **implícitamente por el orden de composición**. El mixin que aparece primero en la lista (o el que tiene mayor precedencia según la dirección) automáticamente "gana". No es necesario que el programador declare explícitamente la resolución.

```
type ResearchDoctor = PersonMixin GraduateMixin Doctor;
"Si PersonMixin y GraduateMixin ambos definen display,
 la precedencia la determina el orden."
```

**Problema:** si el orden cambia (por refactorización), el comportamiento puede cambiar silenciosamente. Además, puede ser imposible acceder a la implementación "perdedora" del conflicto sin modificar los mixins.

#### En Traits (stateless y stateful)

Un conflicto ocurre **si y solo si** dos traits proveen métodos con el mismo nombre que no se originan del mismo trait. La resolución es **obligatoriamente explícita**: el compositor debe tomar una decisión activa.

Mecanismos disponibles:
- **Sobreescritura directa:** definir el método en la clase compositora.
- **Exclusión:** descartar el método de uno o más traits con el operador `−`.
- **Aliasing:** dar acceso a ambas implementaciones bajo nombres alternativos con `@`, y luego componer explícitamente.

```smalltalk
Object subclass: #Circle
  uses: { TCircle @ {#circleHash -> #hash. #circleEqual: -> #=} .
          TColor  @ {#colorHash  -> #hash. #colorEqual:  -> #=} }
  hash
    ↑ self circleHash bitXor: self colorHash
  = anObject
    ↑ (self circleEqual: anObject) and: [self colorEqual: anObject]
```

**Regla especial (same-operation exception):** si el mismo método (proveniente del mismo trait) se obtiene múltiples veces por diferentes caminos (diamond), **no hay conflicto**.

**Conflicto de variables en stateful traits:** no puede ocurrir por defecto, ya que las variables son privadas al scope del trait. El compositor decide explícitamente si acceder o mergear variables con `@@`.

---

### Pregunta 3: ¿Qué rol tiene la clase en cada tecnología?

#### En Mixin-based inheritance

La clase sigue siendo la unidad central: es el **generador de instancias** y el **punto de composición**. Un mixin, al componerse con una clase, produce otra clase. La clase resultante hereda de la cadena de mixins aplicados. La composición de mixins **es** herencia: produce una nueva clase en la jerarquía.

En este modelo, las clases y los mixins son esencialmente la misma cosa (los mixins son clases abstractas). La diferencia es operacional: los mixins están diseñados para ser aplicados a distintas bases, mientras que una clase ordinaria no usa su parámetro padre.

#### En Traits (stateless)

La clase cumple **dos roles claramente separados:**

1. **Generadora de instancias:** define el estado (variables de instancia) y ocupa su lugar en la jerarquía de herencia simple.
2. **Punto de composición:** ensambla los traits, define el glue code, resuelve conflictos y satisface los requerimientos.

Los traits **no son clases**: no generan instancias, no pertenecen a la jerarquía de herencia. La ecuación es:

```
Clase = Superclase + Estado + Traits + Glue
```

La clase es quien tiene el control total sobre cómo se ensambla su comportamiento. Los traits son simples proveedores de comportamiento reutilizable.

#### En Stateful Traits

El rol de la clase se mantiene igual que en stateless traits, pero con una responsabilidad adicional sobre el estado: la clase puede **decidir la política de visibilidad** de las variables de los traits que usa. Puede mantenerlas privadas (black-box), acceder a ellas bajo nuevos nombres (white-box selectivo), o mergearlas con variables de otros traits.

La clase sigue siendo la única entidad que puede instanciarse. Los traits, aunque ahora tienen estado privado, no son generadores de instancias.

---

### Pregunta 4: ¿Cuándo sugiere cada autor usar mixins o traits? ¿Por qué?

#### Bracha & Cook: cuándo usar Mixins

Los autores sugieren usar mixins cuando:
- Se necesita **aplicar el mismo conjunto de características a múltiples clases base** diferentes (comportamiento cross-cutting).
- Se trabaja con frameworks donde una familia de clases necesita ser enriquecida con comportamiento adicional.
- Se quiere evitar la linearización implícita de CLOS (usar composición explícita).
- El lenguaje base ya soporta herencia simple y se quiere extender el modelo sin modificaciones radicales.

**Por qué:** los mixins hacen explícita la abstracción que CLOS realiza implícitamente. Son la generalización natural del mecanismo subyacente de herencia. Son la herramienta adecuada cuando el comportamiento a reutilizar tiene sentido expresarse como "extensión incremental de un padre".

#### Scharli et al. / Bergel et al.: cuándo usar Traits

Los autores sugieren usar traits cuando:
- Se quiere **factorizar comportamiento reutilizable a nivel intra-clase**, sin impactar la jerarquía de herencia.
- Se necesita **componer múltiples características ortogonales** en una misma clase sin entrar en conflictos implícitos.
- Se trabaja con jerarquías complejas donde el código duplicado o los métodos "mal ubicados" son un problema (como la jerarquía de colecciones de Smalltalk).
- Se necesita **comprensibilidad**: la flattening property permite ver la clase como si los métodos de los traits estuvieran definidos directamente en ella.
- Se quiere que el glue code esté **centralizado** y que los cambios en traits no produzcan efectos cascada en la jerarquía.

**Por qué:** los traits separan la reusabilidad del código del rol de las clases como generadoras de instancias. Esta separación elimina los problemas del diamond problem en estado, la dispersión del glue code y la fragilidad de las jerarquías de mixins. La composición explícita y el flattening garantizan tanto reusabilidad como comprensibilidad.

**Stateful traits** se sugieren cuando:
- Los traits necesitan encapsular su propio estado internamente.
- Se quiere evitar boilerplate de accessors y shell classes.
- Se busca que las evoluciones del estado de un trait no propaguen requerimientos a sus clientes.

---

### Pregunta 5: ¿Qué hay en Ruby y Scala? ¿Qué tienen?

#### Ruby: Modules como Mixins

Ruby no tiene herencia múltiple, pero ofrece **modules** que funcionan como mixins en el sentido de Bracha & Cook.

```ruby
module Greetable
  def greet
    "Hello, I am #{name}"
  end
end

module Farewell
  def bye
    "Goodbye from #{name}"
  end
end

class Person
  include Greetable
  include Farewell

  attr_reader :name
  def initialize(name)
    @name = name
  end
end
```

**Características:**
- Un module es un **namespace + conjunto de métodos** que puede incluirse en clases.
- `include` inserta el módulo en la cadena de herencia mediante **linearización** (similar a CLOS): Ruby usa un algoritmo de resolución llamado MRO (Method Resolution Order) que sigue C3 linearization.
- Los métodos del módulo incluido pueden acceder al estado de la clase que lo incluye via `self`.
- Un módulo **puede tener variables de instancia** si se referencian dentro de sus métodos (el estado existe en la instancia de la clase, no en el módulo mismo).
- **Los conflictos se resuelven por el orden de inclusión:** el último módulo incluido tiene mayor precedencia (overrides anteriores).
- `prepend` inserta el módulo antes de la clase en la cadena (la clase puede llamar a `super` para acceder al método original).
- `extend` agrega los métodos del módulo como métodos de clase (singleton methods).

**En términos de los papers:** Ruby implementa **mixins** (más cercano a Bracha & Cook que a traits). La resolución de conflictos es implícita por linearización, no explícita. No hay flattening property ni operadores de alias/exclusión como en traits. Los módulos pueden tener estado accesible en la instancia.

---

#### Scala: Traits

Scala adopta el término **trait** pero su implementación es más cercana a los mixins con algunas características de traits.

```scala
trait Greetable {
  def name: String  // método requerido (abstracto)
  def greet(): String = s"Hello, I am $name"
}

trait Loggable {
  def log(msg: String): Unit = println(s"[LOG] $msg")
  private var logCount = 0  // ¡puede tener estado!
}

class Person(val name: String) extends Greetable with Loggable {
  def introduce(): Unit = {
    log(greet())
  }
}
```

**Características:**
- Los traits de Scala **pueden tener estado** (variables de instancia y su inicialización).
- Pueden tener métodos concretos y abstractos.
- Se componen con `extends` y `with`, y la linearización sigue el algoritmo C3 (igual que Ruby/Python).
- Los conflictos de métodos con implementación deben resolverse explícitamente en la clase, sobreescribiendo el método. Sin embargo, la composición general sigue siendo lineal.
- `super` en un trait de Scala se resuelve dinámicamente según la linearización (stackable traits pattern), lo que permite cadenas de decoradores.

```scala
// Stackable traits pattern
trait Doubling extends IntQueue {
  abstract override def put(x: Int) = super.put(2 * x)
}
trait Incrementing extends IntQueue {
  abstract override def put(x: Int) = super.put(x + 1)
}
// La composición: extends BasicIntQueue with Incrementing with Doubling
// La llamada put(5) ejecuta: Doubling(put(5)) → Incrementing(put(10)) → BasicIntQueue.put(11)
```

**En términos de los papers:** Scala tiene **traits con estado** (como stateful traits de Bergel et al.), pero la composición sigue siendo **lineal** (como mixins de Bracha & Cook). La linearización de Scala resuelve conflictos implícitamente por orden, salvo cuando dos traits proveen métodos concretos con el mismo nombre, en cuyo caso la clase debe sobreescribir.

---

#### Tabla comparativa final: Ruby vs Scala vs el modelo académico

| Característica | Mixins (Bracha) | Traits (Scharli) | Stateful Traits (Bergel) | Ruby Modules | Scala Traits |
|----------------|-----------------|------------------|--------------------------|--------------|--------------|
| **Composición** | Lineal explícita | Simétrica | Simétrica | Lineal (C3) | Lineal (C3) |
| **Estado en la unidad** | Sí (posible) | No | Sí (privado) | Sí (en instancia) | Sí |
| **Resolución de conflictos** | Implícita (orden) | Explícita obligatoria | Explícita obligatoria | Implícita (orden) | Semi-explícita |
| **Flattening property** | No | Sí | Sí | No | No |
| **Operadores alias/exclusión** | No | Sí | Sí + `@@` | No | No |
| **`super` en trait/mixin** | Resuelto por linearización | Resuelto por clase usante | Resuelto por clase usante | Resuelto por linearización | Resuelto por linearización |
| **Glue code** | Disperso (clases intermedias) | Centralizado en compositor | Centralizado en compositor | En la clase | En la clase |
| **Genera instancias** | No (abstracto) | No | No | No | No |

**Conclusión:** Ruby y Scala implementan conceptos intermedios. Ruby es más fiel al modelo de mixins de Bracha & Cook (linearización para resolver conflictos). Scala es más cercano a stateful traits pero mantiene la linearización como mecanismo de composición. Ninguno implementa la propiedad de aplanamiento ni los operadores de alias y exclusión a nivel de lenguaje que caracterizan a los traits académicos de Scharli et al.
