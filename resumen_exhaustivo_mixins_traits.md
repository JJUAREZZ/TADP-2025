# Resumen Exhaustivo: Mixins, Traits y Stateful Traits

> **Para quién es este documento:** programadores con conocimiento de POO (clases, herencia, métodos, variables de instancia) que quieren entender en profundidad los modelos académicos de composición de comportamiento.

---

## PAPER 1: Mixin-based Inheritance
**Autores:** Gilad Bracha y William Cook
**Publicado en:** Proceedings of OOPSLA/ECOOP 1990

---

### ¿De qué trata este paper?

Bracha y Cook parten de una pregunta aparentemente simple: ¿por qué Smalltalk, Beta y CLOS tienen mecanismos de herencia tan distintos entre sí? ¿Son realmente diferentes o hay algo común debajo?

La respuesta que encuentran es sorprendente: **los tres lenguajes usan el mismo mecanismo subyacente, solo que en direcciones o formas diferentes**. A partir de esta observación, proponen generalizar ese mecanismo en algo llamado **mixin-based inheritance**, que unifica los tres enfoques y los supera.

---

### Parte 1: El mecanismo subyacente de la herencia

Para entender qué tienen en común Smalltalk y Beta, los autores formalizan cómo funciona la herencia en términos matemáticos simples.

#### El concepto de "delta"

Cuando creamos una subclase, en realidad estamos especificando un **conjunto de cambios** (un "delta") respecto a la clase padre. Por ejemplo:

```
class Person:
    name = ""
    def display():
        print(name)

class Graduate extends Person:
    degree = ""
    def display():        # ← Este método es el "delta": el cambio
        super.display()
        print(degree)
```

El delta de `Graduate` es simplemente: "reemplazar `display` por una versión que llama a la original y luego muestra el grado".

Los autores formalizan esto así:

```
C = Δ(P) ⊕ P
```

Donde:
- `P` es la clase padre (con todos sus métodos)
- `Δ` es el delta (los cambios que hace la subclase)
- `⊕` es una operación de "combinación" que toma los campos de dos conjuntos, dando preferencia al izquierdo cuando hay duplicados
- `Δ(P)` significa: evaluar el delta teniendo acceso a los métodos de P (esto es lo que permite que `super` funcione)

El resultado `C` es la nueva clase: tiene todos los métodos de P, pero donde hay un método redefinido en el delta, gana el del delta.

---

### Parte 2: Herencia en Smalltalk — "la subclase manda"

En Smalltalk, la subclase tiene el control total. Puede **reemplazar completamente** cualquier método del padre. La subclase llama al padre cuando quiere, usando `super`.

```
┌──────────────────────────────────────────────────────┐
│  SMALLTALK: La subclase tiene precedencia            │
│                                                      │
│  ┌──────────────┐                                    │
│  │    Person    │  ← define display() original       │
│  └──────┬───────┘                                    │
│         │ hereda                                     │
│  ┌──────▼───────┐                                    │
│  │   Graduate   │  ← REEMPLAZA display()             │
│  │              │    puede llamar a super si quiere  │
│  └──────────────┘                                    │
│                                                      │
│  Dirección del control: subclase → superclase        │
│  La subclase decide cuándo (si es que) llama a super │
└──────────────────────────────────────────────────────┘
```

**Ejemplo concreto:**
```
Person.display()   →  imprime "A. Smith"
Graduate.display() →  llama super.display(), luego imprime "Ph.D."
                   →  resultado: "A. Smith Ph.D."
```

**La limitación clave:** el delta (`Graduate`) **no puede existir de forma independiente**. Está siempre pegado a un padre concreto (`Person`). Si queremos aplicar la misma lógica de "mostrar el grado" a otras clases (digamos, `Doctor`), tenemos que **copiar el código**:

```
class GraduateDoctor extends Doctor:
    degree = ""
    def display():
        super.display()   # ← mismo código que Graduate, copiado
        print(degree)
```

Esto es exactamente lo que los mixins vienen a resolver.

---

### Parte 3: Herencia en Beta — "el padre manda"

Beta invierte completamente el control: es el **padre quien decide** cómo se ejecuta el comportamiento. El padre usa un comando llamado `inner` que dice "aquí ejecutá lo que define el hijo".

```
┌──────────────────────────────────────────────────────┐
│  BETA: El padre (prefix) tiene precedencia           │
│                                                      │
│  ┌──────────────┐                                    │
│  │    Person    │  ← define display() con inner      │
│  │  (prefix)    │    EL PADRE controla la ejecución  │
│  └──────┬───────┘                                    │
│         │ prefixes                                   │
│  ┌──────▼───────┐                                    │
│  │   Graduate   │  ← EXTIENDE display()              │
│  │  (suffix)    │    NO puede ignorar al padre       │
│  └──────────────┘                                    │
│                                                      │
│  Dirección del control: superclase → subclase        │
│  El padre decide cuándo (si es que) llama a inner    │
└──────────────────────────────────────────────────────┘
```

**Ejemplo concreto:**
```
Person.display():
    print(name)
    inner()         # ← aquí le cede el control al hijo

Graduate.display() (el "inner behavior"):
    print(degree)
    inner()         # ← cede al nieto (si lo hay)

# Para una instancia de Graduate:
# resultado: "A. Smith Ph.D."
```

La diferencia fundamental: en Beta, el hijo **no puede saltarse** la lógica del padre. El padre siempre se ejecuta primero y decide en qué momento (o si) llama al hijo. Esto da **seguridad**: un framework puede garantizar que cierta lógica del padre siempre se ejecuta.

**La limitación de Beta:** para aplicar el mismo "suffix" (Graduate) a una clase diferente (Doctor), hay que **copiar la clase base**. Si en Smalltalk copiabas el delta, en Beta copiás la base.

---

### Parte 4: La simetría entre Smalltalk y Beta

Esta es una de las observaciones más elegantes del paper: Smalltalk y Beta son **el mismo mecanismo, pero invertido**.

```
┌─────────────────────────────────────────────────────────────┐
│  COMPARACIÓN: Smalltalk vs Beta                             │
│                                                             │
│  SMALLTALK          │  BETA                                 │
│  ─────────────────  │  ─────────────────                    │
│  Subclase/Superclase│  Superpatrón/Subpatrón                │
│  super              │  inner                                │
│  La subclase llama  │  El padre llama al hijo               │
│  al padre           │                                       │
│                     │                                       │
│  SMALLTALK:         │  BETA:                                │
│  Child ──super──►  Parent  │  Prefix ──inner──► Suffix      │
│  (el nuevo llama    │  (el original llama                   │
│   al original)      │   al nuevo)                           │
│                     │                                       │
│  Son inversamente   equivalentes:                           │
│  C_smalltalk = Δ ⊲ P  ≡  C_beta = P ⊲ Δ                   │
└─────────────────────────────────────────────────────────────┘
```

En términos del operador `⊲`: Smalltalk es `Δ ⊲ P` (el nuevo tiene precedencia), Beta es `P ⊲ Δ` (el original tiene precedencia).

---

### Parte 5: Herencia múltiple en CLOS y los Mixins

CLOS (Common Lisp Object System) permite que una clase herede de **múltiples padres**. Esto crea un problema inmediato: si dos padres definen el mismo método, ¿cuál se usa?

#### El problema del diamante

```
┌─────────────────────────────────────────────────────┐
│  EL PROBLEMA DEL DIAMANTE                           │
│                                                     │
│         ┌──────────┐                               │
│         │  Person  │  ← define display()           │
│         └─────┬────┘                               │
│        ┌──────┴───────┐                            │
│  ┌─────▼────┐   ┌─────▼────┐                       │
│  │  Doctor  │   │ Graduate │  ← ambos heredan      │
│  │          │   │          │    de Person y         │
│  └─────┬────┘   └─────┬────┘    redefinen display() │
│        └──────┬────────┘                            │
│         ┌─────▼──────────┐                          │
│         │ ResearchDoctor │  ← ¿qué display() usa?  │
│         └────────────────┘                          │
│                                                     │
│  Si se ejecuta Person.display() dos veces,          │
│  el nombre aparece duplicado en la salida.          │
└─────────────────────────────────────────────────────┘
```

CLOS resuelve esto con **linearización**: convierte el grafo de herencia en una lista lineal donde cada ancestro aparece exactamente una vez. Para `ResearchDoctor`, la lista podría ser:

```
ResearchDoctor → Doctor → Graduate → Person
```

Ahora cada clase en la cadena llama a la siguiente con `call-next-method` (equivalente a `super` en Smalltalk). El resultado es que `Person.display()` se ejecuta exactamente una vez.

#### Los Mixins en CLOS

En CLOS existe una técnica llamada **mixin programming**: se definen clases abstractas pequeñas diseñadas para "mezclarse" con otras clases.

```lisp
;; Un mixin: clase abstracta que añade comportamiento de "grado"
(defclass Graduate-mixin () (degree))
(defmethod display ((self Graduate-mixin))
  (call-next-method)           ; ← llama al siguiente en la cadena
  (display (slot-value self 'degree)))

;; Se aplica a Person:
(defclass Graduate (Graduate-mixin Person) ())
;; Linearización: Graduate → Graduate-mixin → Person

;; Se aplica a Dog:
(defclass GuardDog (Graduate-mixin Dog) ())
;; Linearización: GuardDog → Graduate-mixin → Dog
```

El mixin `Graduate-mixin` hace `call-next-method` sin saber a quién va a llamar. La linearización se encarga de "inyectar" la clase correcta (`Person` o `Dog`) entre el mixin y la raíz.

**El problema:** esto es implícito y mágico. El programador no controla explícitamente cómo se forma la cadena. Cambiar la jerarquía puede cambiar silenciosamente el comportamiento.

---

### Parte 6: La propuesta — Mixins como construcción explícita

Bracha y Cook proponen hacer el mixin un **ciudadano de primera clase** del lenguaje, no solo una convención de programación.

La idea central: un mixin es una función que toma una clase padre y devuelve una nueva clase.

```
┌──────────────────────────────────────────────────────────┐
│  MIXIN COMO FUNCIÓN                                      │
│                                                          │
│  GraduateMixin es una función:                          │
│                                                          │
│  GraduateMixin(padre) = {                               │
│      degree: string,                                    │
│      display: llama a padre.display() + muestra degree  │
│  } combinado con padre                                  │
│                                                          │
│  Aplicaciones:                                          │
│  Graduate    = GraduateMixin(Person)                    │
│  GuardDog    = GraduateMixin(Dog)                       │
│  GuardDoctor = GraduateMixin(Doctor)                    │
│                                                          │
│  El mixin se reutiliza sin copiar código.               │
└──────────────────────────────────────────────────────────┘
```

El operador de composición de mixins `?` combina dos mixins:

```
M₁ ? M₂ = fun(i) M₁(M₂(i) ⊕ i) ⊕ M₂(i)
```

Lo importante de esta fórmula:
- El resultado es **otro mixin** (no una clase aún), que sigue esperando un padre.
- `M₁` ve a `M₂` como su "super", y `M₂` ve al parámetro `i` como su "super".
- El operador es **asociativo**: `(M₁ ? M₂) ? M₃ = M₁ ? (M₂ ? M₃)`.
- En caso de conflicto, **el primero en la lista tiene precedencia** (igual que en Smalltalk).

#### Ejemplo en Modula-3 extendido

```
type GraduateMixin =
  object degree: string
  methods display := displayGraduateMixin
  super display() := No_Op   (* declara qué necesita del padre *)
  end;

type PersonMixin =
  object name: string
  methods display := displayPersonMixin
  super display() := No_Op
  end;

(* Composición explícita: Graduate = Person + GraduateMixin *)
type Graduate = PersonMixin GraduateMixin;

(* El mismo mixin aplicado a otro padre *)
type GuardDog = DogMixin GraduateMixin;
```

La gran diferencia con CLOS: el programador **declara explícitamente** el orden de composición. No hay linearización automática ni sorpresas.

#### Ventajas de la composición explícita

```
┌──────────────────────────────────────────────────────────────────┐
│  IMPLÍCITO (CLOS)          vs   EXPLÍCITO (Bracha & Cook)        │
│                                                                  │
│  El compilador/runtime          El programador decide            │
│  decide el orden                el orden                         │
│                                                                  │
│  Cambiar la jerarquía           Cambiar la jerarquía             │
│  puede cambiar silenciosamente  requiere cambiar                 │
│  el comportamiento              explícitamente las composiciones │
│                                                                  │
│  Difícil de razonar:            Fácil de razonar:                │
│  hay que conocer el             la cadena está escrita           │
│  algoritmo de linearización     en el código                     │
└──────────────────────────────────────────────────────────────────┘
```

---

### Parte 7: Sistema de tipos para mixins

Una contribución importante del paper es proponer reglas de tipado para mixins en un lenguaje con tipos estáticos (Modula-3).

La regla central: si `T₁ = T₂ T₃` (T₁ se compone de T₂ y T₃), entonces:
- `T₁` es subtipo de `T₂`
- `T₁` es subtipo de `T₃`

Esto tiene sentido: si un tipo es la combinación de dos cosas, puede usarse donde se espera cualquiera de las dos.

Además, el operador `?` es asociativo, lo que se refleja en los tipos:
```
Si PGMixin = PersonMixin GraduateMixin
Entonces ResearchDoctor = PGMixin Doctor
Por lo tanto ResearchDoctor <: PGMixin
```

Esto permite verificar estáticamente que una composición de mixins es correcta, lo que era novedoso para la época.

---

### Conclusión del Paper 1

El paper de Bracha y Cook es seminal porque:
1. **Unifica** tres mecanismos de herencia aparentemente distintos bajo un modelo común.
2. **Formaliza** el concepto de mixin como entidad independiente y reutilizable.
3. **Elimina la linearización implícita** reemplazándola por composición explícita.
4. **Extiende** Modula-3 para demostrar que mixins son viables en lenguajes fuertemente tipados.

La limitación principal que quedan sin resolver (y que el Paper 2 ataca directamente) es que la composición de mixins sigue siendo **lineal y ordenada**: cuando dos mixins tienen métodos con el mismo nombre, el orden de composición decide cuál gana, sin que el programador pueda elegir fácilmente combinar partes de ambos.

---
---

## PAPER 2: Traits: Composable Units of Behaviour
**Autores:** Nathanael Scharli, Stéphane Ducasse, Oscar Nierstrasz, Andrew P. Black
**Publicado en:** ECOOP 2003, LNCS 2743, pp. 248–274

---

### ¿De qué trata este paper?

Los autores identifican que todos los mecanismos de herencia existentes (simple, múltiple y por mixins) tienen problemas conceptuales y prácticos que limitan la reutilización de código. Proponen un nuevo modelo llamado **traits** que resuelve estos problemas siendo:
- Más fino en granularidad que una clase o un mixin.
- Completamente independiente de la jerarquía de herencia.
- Con composición simétrica (sin orden) y resolución explícita de conflictos.
- Sin estado propio (solo comportamiento).

La implementación de referencia es en **Squeak** (un dialecto de Smalltalk-80).

---

### Parte 1: Los problemas que motivan los traits

#### Problema 1: El conflicto entre "clase como generadora de instancias" y "clase como unidad de reuso"

Una clase tiene dos responsabilidades que a menudo compiten:

```
┌────────────────────────────────────────────────────────────┐
│  LAS DOS CARAS DE UNA CLASE                                │
│                                                            │
│  Rol 1: Generadora de instancias                          │
│  ─────────────────────────────                            │
│  • Debe ser COMPLETA (todos los métodos implementados)    │
│  • Tiene un lugar FIJO en la jerarquía                    │
│  • Es el "plano" del objeto                               │
│                                                            │
│  Rol 2: Unidad de reutilización                           │
│  ───────────────────────────────                          │
│  • Debe ser PEQUEÑA (fácil de entender y combinar)        │
│  • Debe ser FLEXIBLE (aplicable en distintos contextos)   │
│                                                            │
│  TENSIÓN: una clase grande y completa es mala para        │
│  reutilizar. Una clase pequeña no puede instanciarse      │
│  directamente sin código adicional.                       │
└────────────────────────────────────────────────────────────┘
```

Los traits resuelven esto siendo **solo** unidades de reuso: nunca se instancian.

#### Problema 2: Herencia simple fuerza duplicación de código

Con herencia simple, si dos ramas independientes de la jerarquía necesitan el mismo comportamiento, no hay forma de compartirlo sin copiarlo o subirlo a un ancestro común (lo que contamina la jerarquía).

```
┌─────────────────────────────────────────────────────────┐
│  DUPLICACIÓN CON HERENCIA SIMPLE                        │
│                                                         │
│         Animal                                          │
│        /      \                                         │
│    Mammal    Bird                                       │
│    /    \      \                                        │
│  Dog    Cat   Penguin                                   │
│                                                         │
│  Problema: Dog, Cat y Penguin necesitan el método       │
│  "canSwim()" pero están en ramas distintas.             │
│                                                         │
│  Opción A: Subir canSwim() a Animal ← contamina        │
│            (no todos los animales nadan)               │
│  Opción B: Copiar canSwim() en cada clase ← duplicar   │
│  Opción C: Crear SwimmingAnimal ← fuerza la jerarquía  │
└─────────────────────────────────────────────────────────┘
```

#### Problema 3: El problema del "wrapper genérico" con mixins y herencia múltiple

Este es un problema específico que los autores ilustran con el ejemplo de la sincronización. Supongamos que tenemos:

```
class A:
    def read():  ...    # acceso no sincronizado
    def write(): ...

class SyncA extends A:
    def read():
        acquireLock()
        value = super.read()   # llama a A.read()
        releaseLock()
        return value
    def write(): ... (similar)
```

Hasta aquí todo bien. Pero ¿qué pasa si también tenemos `class B` con `read()` y `write()` y queremos `SyncB`?

```
┌─────────────────────────────────────────────────────────────────┐
│  EL PROBLEMA DEL WRAPPER GENÉRICO                               │
│                                                                 │
│  INTENTO 1: Factorizar la sincronización en una superclase      │
│                                                                 │
│       ┌──────────────────┐                                      │
│       │  SyncReadWrite   │  ← tiene syncRead, syncWrite         │
│       │  (con super.read)│    pero super.read ¿es A o B?        │
│       └──────────────────┘                                      │
│          ↗          ↖                                           │
│  ┌───────┴───┐  ┌────┴──────┐                                  │
│  │   SyncA   │  │   SyncB   │                                   │
│  │ extends A │  │ extends B │                                   │
│  └───────────┘  └───────────┘                                   │
│                                                                 │
│  PROBLEMA: los super-sends son ESTÁTICOS.                       │
│  super.read() en SyncReadWrite no puede referirse               │
│  a A.read() en un caso y a B.read() en otro.                    │
│                                                                 │
│  SOLUCIÓN PARCIAL (figura 1c del paper): usar self-sends        │
│  abstractos en lugar de super, pero esto obliga a duplicar      │
│  4 métodos en SyncA y SyncB de todas formas.                    │
└─────────────────────────────────────────────────────────────────┘
```

Con traits, este problema desaparece porque `super` en un trait se resuelve en el contexto de la clase que usa el trait (ver sección sobre la flattening property).

#### Problema 4: Los tres problemas de los mixins

Los autores identifican tres problemas específicos de la composición de mixins que no se mencionan explícitamente en la literatura previa:

**4a. Ordenamiento total obligatorio**

```
┌───────────────────────────────────────────────────────────┐
│  PROBLEMA: ORDENAMIENTO TOTAL EN MIXINS                   │
│                                                           │
│  Queremos una clase con Color y Border.                   │
│  Ambos definen asString().                                │
│                                                           │
│  Opción 1: Rectangle + MColor + MBorder                  │
│  → MBorder.asString() prevalece sobre MColor.asString()  │
│                                                           │
│  Opción 2: Rectangle + MBorder + MColor                  │
│  → MColor.asString() prevalece sobre MBorder.asString()  │
│                                                           │
│  Pero ¿qué pasa si queremos COMBINAR ambos asString()?   │
│  No hay un "orden total" que lo exprese.                  │
│  El compositor no tiene control sobre esto.              │
└───────────────────────────────────────────────────────────┘
```

**4b. Dispersión del glue code**

Con mixins, el código que "conecta" los comportamientos queda repartido en clases intermediarias que se generan automáticamente al aplicar cada mixin. El compositor final (la clase que usa los mixins) **no tiene control** sobre cómo se ensamblan.

```
┌──────────────────────────────────────────────────────────────┐
│  DISPERSIÓN DEL GLUE CODE                                    │
│                                                              │
│  Rectangle                                                   │
│      ↓ aplica MColor                                         │
│  Rectangle + MColor  ← clase intermediaria generada         │
│      ↓ aplica MBorder                                        │
│  Rectangle + MColor + MBorder  ← clase intermediaria        │
│      ↓                                                       │
│  MyRectangle                                                 │
│                                                              │
│  En "Rectangle + MColor + MBorder":                         │
│  → puedo acceder al comportamiento de MBorder               │
│  → puedo acceder al comportamiento MEZCLADO de MColor+Rect  │
│  → NO puedo acceder al asString ORIGINAL de MColor solo     │
│    ni al de Rectangle solo                                  │
│                                                              │
│  Si quiero cambiar cómo se combinan los asString(),         │
│  tengo que MODIFICAR los mixins involucrados.               │
└──────────────────────────────────────────────────────────────┘
```

**4c. Jerarquías frágiles**

Si el mixin `MBorder` no tenía `asString()` originalmente, y `MyRectangle` usaba el de `MColor`, todo estaba bien. Pero si alguien **agrega** `asString()` a `MBorder` en el futuro, silenciosamente pasa a tener precedencia sobre el de `MColor`, y el comportamiento de `MyRectangle` cambia sin ninguna advertencia.

---

### Parte 2: La definición formal de un Trait

Un **trait** tiene exactamente dos cosas:

1. **Métodos provistos (provided methods):** los métodos que el trait implementa y ofrece a quien lo use.
2. **Métodos requeridos (required methods):** los métodos que el trait necesita que existan para que su comportamiento funcione, pero que no implementa él mismo.

```
┌──────────────────────────────────────────────────────────────┐
│  ANATOMÍA DE UN TRAIT                                        │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │                    TDrawing                          │    │
│  │                                                      │    │
│  │  Provistos        │  Requeridos                     │    │
│  │  ─────────────    │  ──────────────                 │    │
│  │  draw()           │  bounds()   ← necesita saber    │    │
│  │  refresh()        │              dónde dibujar      │    │
│  │  refreshOn(c)     │  drawOn(c)  ← necesita saber    │    │
│  │                   │              cómo dibujar        │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  Los required methods son los "parámetros" del trait.       │
│  El trait es como una función incompleta que depende de     │
│  que alguien externo provea esas partes.                    │
└──────────────────────────────────────────────────────────────┘
```

**Lo que un trait NO tiene:**
- Variables de instancia
- Posición en la jerarquía de herencia
- Lógica de inicialización

**Código del trait TDrawing en Smalltalk:**
```smalltalk
Trait named: #TDrawing uses: {}

"Métodos provistos:"
draw
    ↑ self drawOn: World canvas    "llama a drawOn:, que es requerido"

refresh
    ↑ self refreshOn: World canvas

refreshOn: aCanvas
    aCanvas form
        deferUpdatesIn: self bounds   "bounds es requerido"
        while: [self drawOn: aCanvas]

"Los métodos requeridos se declaran así (cuerpo = error):"
bounds
    self requirement    "← si nadie lo provee, esto falla"

drawOn: aCanvas
    self requirement
```

---

### Parte 3: La ecuación fundamental de los Traits

La propuesta central del paper se resume en:

```
Clase = Superclase + Estado + Traits + Glue
```

Desglosando cada componente:

```
┌──────────────────────────────────────────────────────────────────┐
│  LOS CUATRO COMPONENTES DE UNA CLASE CON TRAITS                  │
│                                                                  │
│  SUPERCLASE                                                      │
│  → La clase de la que se hereda (herencia simple, como siempre) │
│  → Provee el "esqueleto" de la jerarquía                        │
│                                                                  │
│  ESTADO                                                          │
│  → Las variables de instancia de la clase                       │
│  → Solo la clase define estado; los traits NO                   │
│                                                                  │
│  TRAITS                                                          │
│  → Los comportamientos reutilizables que se "enchufan"          │
│  → Pueden ser varios a la vez                                   │
│  → No tienen orden entre sí                                     │
│                                                                  │
│  GLUE (código de pegamento)                                      │
│  → Los métodos que conectan los traits entre sí                 │
│  → Los accessors que satisfacen los required methods            │
│  → La resolución de conflictos                                  │
│  → Siempre está EN la clase, nunca repartido afuera            │
└──────────────────────────────────────────────────────────────────┘
```

**Ejemplo completo — la clase Circle:**

```
┌──────────────────────────────────────────────────────────────────┐
│  Circle = Object + {center, radius} + {TCircle, TDrawing} + Glue │
│                                                                  │
│            ┌─────────────────────────────────────────┐          │
│            │                 Circle                  │          │
│            │                                         │          │
│  ESTADO ── │  center  (variable de instancia)        │          │
│            │  radius  (variable de instancia)        │          │
│            │                                         │          │
│  GLUE ──── │  initialize()    "center=0@0; radius=50"│          │
│            │  center()        "↑ center"             │          │
│            │  center: aPoint  "center := aPoint"     │          │
│            │  radius()        "↑ radius"             │          │
│            │  radius: n       "radius := n"          │          │
│            │  drawOn: aCanvas "dibuja el círculo"     │          │
│            │                                         │          │
│            │       ┌─────────────────┐               │          │
│  TRAIT ──  │       │    TCircle      │               │          │
│            │       │                 │               │          │
│            │       │ area()          │ center ◄──────┼─ glue    │
│            │       │ bounds()        │ center: ◄─────┼─ glue    │
│            │       │ circumference() │ radius ◄──────┼─ glue    │
│            │       │ scaleBy:()      │ radius: ◄─────┼─ glue    │
│            │       └─────────────────┘               │          │
│            │       ┌─────────────────┐               │          │
│  TRAIT ──  │       │    TDrawing     │               │          │
│            │       │                 │               │          │
│            │       │ draw()          │ bounds ◄──────┼─ TCircle │
│            │       │ refresh()       │ drawOn: ◄─────┼─ glue    │
│            │       │ refreshOn:()    │               │          │
│            │       └─────────────────┘               │          │
│            └─────────────────────────────────────────┘          │
│                                                                  │
│  Nota: el required "bounds" de TDrawing lo satisface TCircle,   │
│  no la clase. Los traits pueden satisfacerse entre sí.          │
└──────────────────────────────────────────────────────────────────┘
```

```smalltalk
Object subclass: #Circle
    instanceVariableNames: 'center radius'
    uses: { TCircle . TDrawing }

initialize
    center := 0@0.
    radius := 50

center         center: aPoint
    ↑ center       center := aPoint

radius         radius: aNumber
    ↑ radius       radius := aNumber

drawOn: aCanvas
    aCanvas fillOval: self bounds color: Color black
```

---

### Parte 4: La Propiedad de Aplanamiento (Flattening Property)

Esta propiedad es la más importante del modelo de traits. Dice:

> **"La semántica de una clase definida usando traits es exactamente la misma que la de una clase construida directamente a partir de todos los métodos no sobreescritos de los traits."**

En términos simples: usar traits es **semánticamente equivalente** a copiar y pegar los métodos de los traits directamente en la clase.

```
┌──────────────────────────────────────────────────────────────────┐
│  LA PROPIEDAD DE APLANAMIENTO                                    │
│                                                                  │
│  CON TRAITS:                   EQUIVALENTE A:                   │
│  ──────────────────────        ──────────────────────────────   │
│  class Circle                  class Circle                     │
│    uses: TCircle, TDrawing       initialize() ...               │
│    initialize() ...              center() ...                   │
│    center() ...                  radius() ...                   │
│    radius() ...                  drawOn() ...                   │
│    drawOn() ...                  area()        ← de TCircle     │
│                                  bounds()      ← de TCircle     │
│                                  draw()        ← de TDrawing    │
│                                  refresh()     ← de TDrawing    │
│                                  refreshOn()   ← de TDrawing    │
│                                                                  │
│  Son IDÉNTICOS en comportamiento.                               │
│                                                                  │
│  Consecuencia importante: super en un trait simplemente         │
│  busca el método en la SUPERCLASE de la clase que usa el        │
│  trait. No hay "super trait" ni nada especial.                  │
└──────────────────────────────────────────────────────────────────┘
```

Esta propiedad tiene consecuencias prácticas enormes:
- Se puede **ver** una clase como si no tuviera traits (vista aplanada).
- Se puede **entender** una clase sin necesidad de rastrear toda la jerarquía de traits.
- Los traits no introducen ningún mecanismo de lookup especial en runtime.
- El `super` en un método de un trait simplemente referencia la superclase de la clase que lo usa.

---

### Parte 5: Los Operadores de Composición de Traits

Los traits tienen cuatro operadores para combinar comportamiento:

#### Operador 1: Suma (+)
Combina dos traits en uno. El resultado contiene todos los métodos de ambos. Si hay un nombre en conflicto, ese nombre queda marcado como "conflicto" (no se elige ninguno automáticamente).

```
TCircle + TDrawing = trait con todos los métodos de ambos
                    (si no hay conflictos)

TColor + TCircle  = trait con todos los métodos, PERO
                    hash() y =() están MARCADOS como conflicto
                    porque ambos los definen diferente
```

#### Operador 2: Exclusión (-)
Elimina métodos específicos de un trait para evitar conflictos, dejando esos métodos como "requeridos".

```smalltalk
"Eliminar = y hash de TColor para que no conflictúen con TCircle"
TColor - {#=. #hash}

"Ahora TColor ya no provee = ni hash, pero tampoco los requiere;
 simplemente no los tiene."
```

#### Operador 3: Alias (@)
Agrega un nombre alternativo para un método. El método original sigue existiendo con su nombre original, y ahora también existe con el nuevo nombre. Ambos apuntan al mismo código.

```
┌────────────────────────────────────────────────────────────┐
│  ALIAS vs RENAMING                                         │
│                                                            │
│  ALIAS (traits):                                           │
│  TCircle @ {circleHash → hash}                            │
│  Resultado: TCircle ahora tiene TANTO hash() COMO         │
│             circleHash() (apuntan al mismo código)        │
│                                                            │
│  RENAMING (Eiffel):                                        │
│  rename hash as circleHash                                │
│  Resultado: circleHash() existe, hash() YA NO existe      │
│             Todas las referencias a hash en métodos        │
│             internos deben actualizarse                    │
│                                                            │
│  Por qué alias y no renaming: el renaming viola el        │
│  flattening property porque obliga a modificar el         │
│  cuerpo de los métodos que referencian el nombre viejo.   │
└────────────────────────────────────────────────────────────┘
```

#### Operador 4: Sobreescritura (Override)
Los métodos definidos directamente en la clase (o en el trait compuesto) tienen precedencia sobre los métodos que vienen de los traits usados.

---

### Parte 6: Traits compuestos

Un trait puede componerse de otros traits, igual que una clase. Esto permite construir una jerarquía de reusabilidad:

```
┌──────────────────────────────────────────────────────────────────┐
│  TRAITS COMPUESTOS — Ejemplo con TCircle                         │
│                                                                  │
│  TEquality                                                       │
│  ─────────                                                       │
│  Provee: ~=()                                                    │
│  Requiere: =(), hash()                                           │
│                                                                  │
│         ↓ usado por                                              │
│                                                                  │
│  TMagnitude                         TGeometry                   │
│  ───────────                        ──────────                   │
│  Usa: TEquality                     Provee: area(), bounds(),   │
│  Provee: <=(), >=(), max:(),                 diameter(),        │
│          between:and:()                      scaleBy:()         │
│  Requiere: <(), =(), hash()         Requiere: center, radius    │
│  (propagados desde TEquality)                                   │
│                                                                  │
│         ↓ ambos usados por                                       │
│                                                                  │
│  TCircle                                                         │
│  ───────                                                         │
│  Usa: TMagnitude + TGeometry                                     │
│  Define: =(), hash(), <()           ← satisface requeridos de   │
│          (comparación geométrica)      TMagnitude/TEquality      │
│  Propaga: center, radius, center:, radius: ← de TGeometry      │
│                                                                  │
│  El resultado: TCircle tiene TODOS los métodos de todos los     │
│  subtraits más los suyos propios. La propiedad de aplanamiento  │
│  aplica en todos los niveles.                                   │
└──────────────────────────────────────────────────────────────────┘
```

```smalltalk
Trait named: #TCircle uses: { TMagnitude . TGeometry }

= other
    ↑ self radius = other radius and: [self center = other center]

hash
    ↑ self radius hash bitXor: self center hash

< other
    ↑ self radius < other radius
```

---

### Parte 7: Resolución de conflictos — ejemplos detallados

Un conflicto ocurre cuando dos traits proveen métodos con el mismo nombre **con implementaciones diferentes**. La regla es simple: debe resolverse de forma **explícita y obligatoria**.

#### Ejemplo: Circle con Color

```
┌──────────────────────────────────────────────────────────────────┐
│  CONFLICTO: hash() y =() en TColor y TCircle                    │
│                                                                  │
│  TColor:                        TCircle:                        │
│  ────────────────────────────   ────────────────────────────    │
│  hash()  → self rgb hash        hash()  → self radius hash      │
│  =()     → self rgb = other.rgb =()     → geometric comparison  │
│                                                                  │
│  Ambos vienen de TEquality pero con implementaciones distintas. │
│                                                                  │
│  Al componer TColor + TCircle:                                  │
│  → hash y = quedan marcados como CONFLICTO                      │
│  → ~= NO es conflicto (ambas implementaciones vienen del        │
│     mismo trait: TEquality, aunque lleguen por caminos          │
│     distintos — "same-operation exception")                     │
└──────────────────────────────────────────────────────────────────┘
```

**Resolución usando aliases:**
```smalltalk
Object subclass: #Circle
    instanceVariableNames: 'center radius rgb'
    uses: {
        TCircle @ {#circleHash -> #hash. #circleEqual: -> #=} .
        TDrawing .
        TColor  @ {#colorHash  -> #hash. #colorEqual:  -> #=}
    }

"Ahora resolvemos el conflicto en la clase:"
hash
    "Un hash que combina geometría y color"
    ↑ self circleHash bitXor: self colorHash

= anObject
    "Igualdad: misma geometría Y mismo color"
    ↑ (self circleEqual: anObject) and: [self colorEqual: anObject]
```

**Resolución usando exclusión (si el color no importa para la igualdad):**
```smalltalk
Object subclass: #Circle
    instanceVariableNames: 'center radius rgb'
    uses: {
        TCircle .
        TDrawing .
        TColor - {#=. #hash}   "eliminamos = y hash de TColor"
    }
"Ahora solo TCircle define = y hash, no hay conflicto."
```

---

### Parte 8: La regla "same-operation exception" y el Diamond en Traits

¿Qué pasa cuando el mismo método llega por dos caminos pero proviene del mismo trait original?

```
┌──────────────────────────────────────────────────────────────┐
│  DIAMOND EN TRAITS: NO hay conflicto si es el mismo código   │
│                                                              │
│          TEquality                                           │
│          ─────────                                           │
│          define ~=()                                         │
│          ↗           ↖                                       │
│  TMagnitude         TColor                                   │
│  ────────────       ───────                                  │
│  usa TEquality      usa TEquality                            │
│  "hereda" ~=()      "hereda" ~=()                            │
│          ↘           ↙                                       │
│          Circle                                              │
│          ──────                                              │
│          ¿conflicto con ~=() ?                               │
│                                                              │
│  NO, porque las dos copias de ~=() son EL MISMO código      │
│  del MISMO trait (TEquality). No hay ambigüedad semántica.   │
│  Circle simplemente tiene ~=() una vez.                      │
└──────────────────────────────────────────────────────────────┘
```

Esta regla es **semánticamente correcta** porque los traits no tienen estado. Si dos caminos llegan al mismo método, ejecutarlo dos veces o una vez es equivalente (no hay estado que se duplique).

---

### Parte 9: Ventajas de los Traits sobre los Mixins

Volviendo al ejemplo del wrapper de sincronización (figura 1c del paper), con traits la solución (b) del paper funciona perfectamente:

```
┌──────────────────────────────────────────────────────────────────┐
│  SOLUCIÓN CON TRAITS (imposible con mixins)                      │
│                                                                  │
│  TSyncReadWrite (TRAIT):                                         │
│  ───────────────────────                                         │
│  read()   → acquireLock(); value = super.read(); releaseLock()  │
│  write()  → acquireLock(); value = super.write(); releaseLock() │
│  Requiere: read (sin sync), write (sin sync)                    │
│                                                                  │
│  SyncA extends A uses TSyncReadWrite:                           │
│  → super.read() en el trait se resuelve como A.read()          │
│     (por flattening: super busca en la superclase de SyncA)    │
│                                                                  │
│  SyncB extends B uses TSyncReadWrite:                           │
│  → super.read() en el trait se resuelve como B.read()          │
│     (por flattening: super busca en la superclase de SyncB)    │
│                                                                  │
│  El mismo trait, sin modificaciones, funciona para ambos.       │
│  No hay duplicación. El glue code está en SyncA y SyncB,       │
│  que son triviales (solo declaran qué trait usar).             │
└──────────────────────────────────────────────────────────────────┘
```

---

### Parte 10: Implementación en Squeak

La implementación de traits en Squeak es elegante y sin cambios en la VM:

**Cómo se implementan:**
1. Cada clase tiene una variable adicional con su **composition clause** (lista de traits, exclusiones y aliases).
2. Al compilar una clase, el **diccionario de métodos** se extiende con todos los métodos de los traits no sobreescritos.
3. Los alias agregan una segunda entrada en el diccionario que apunta al mismo bytecode.
4. Los métodos de traits que usan `super` deben copiarse (no pueden compartirse) porque tienen una referencia estática a la superclase.
5. Los métodos sin `super` pueden **compartirse** entre el trait y todas las clases que lo usan.

**Performance:**
- Sin penalización en tiempo de ejecución.
- Sin duplicación de bytecode (salvo para métodos con `super`).
- Equivalente a haber escrito todos los métodos directamente en la clase.

**Herramientas (browser extendido):**
- Vista **aplanada**: la clase se muestra como si no usara traits.
- Vista de **composición**: se ven los traits, sus métodos, los conflictos y el glue.
- Generación automática de accessors.
- Lista de "cosas pendientes" cuando un cambio genera un nuevo conflicto.

---

### Parte 11: Caso de estudio — Jerarquía de Colecciones de Smalltalk-80

La prueba de fuego fue refactorizar la jerarquía de colecciones de Squeak, que había sido construida durante más de 20 años.

**El problema original:** las colecciones tienen muchas combinaciones de propiedades (ordenada/desordenada, extensible/inmutable, con clave/sin clave, etc.). La herencia simple no puede expresar todas las combinaciones sin duplicar código o ubicar métodos en lugares incorrectos.

```
┌──────────────────────────────────────────────────────────────────┐
│  PROBLEMA EN LA JERARQUÍA ORIGINAL                               │
│                                                                  │
│  Ejemplo: isEmpty() se implementa en Collection (base),         │
│  aunque hay subclases donde simplemente no tiene sentido        │
│  o tiene una implementación diferente. Se sube demasiado        │
│  alto en la jerarquía para compartir código, y luego se         │
│  sobreescribe o deshabilita en las subclases que no lo          │
│  necesitan. Esto viola el Principio de Sustitución.             │
└──────────────────────────────────────────────────────────────────┘
```

**La solución con traits:**
- Se crearon traits para cada propiedad: "emptiness" (requiere `size()`, provee `isEmpty()`, `notEmpty()`, `ifEmpty:`), "enumeration" (requiere `do:`, provee `collect:`, `select:`, `detect:`), etc.
- Se usó herencia para **ordenar** los traits más específicos sobre los más generales.

**Resultados:**
- 48 traits, 567 métodos.
- **10% menos métodos** que la implementación original.
- **12% menos código** en total.
- 9% de métodos que antes estaban "mal ubicados" (demasiado alto en la jerarquía) fueron eliminados.
- Algunas clases usan hasta 22 traits, pero gracias al flattening se pueden entender como si fueran clases simples.

---

### Conclusión del Paper 2

Los traits son una evolución natural de la herencia simple que:
1. **Separa** la reutilización de comportamiento del rol de generador de instancias de las clases.
2. **Elimina** los problemas de los mixins (orden, glue disperso, fragilidad) haciendo la composición simétrica y la resolución de conflictos explícita.
3. **Preserva** la semántica de la herencia simple gracias al flattening property.
4. **Mejora** la comprensibilidad porque la clase siempre puede verse de forma aplanada.
5. **Escala** a jerarquías complejas como lo demuestra el caso de las colecciones.

---
---

## PAPER 3: Stateful Traits
**Autores:** Alexandre Bergel, Stéphane Ducasse, Oscar Nierstrasz, Roel Wuyts
**Publicado en:** 14th International Smalltalk Conference (ISC 2006), LNCS vol. 4406

---

### ¿De qué trata este paper?

Los autores son, en parte, los mismos que definieron los traits en el Paper 2. En este paper reconocen que los traits sin estado tienen limitaciones prácticas importantes, y proponen una extensión: los **stateful traits**, que permiten a los traits tener variables de instancia **privadas**, manteniendo todos los principios del modelo original.

La pregunta central es: ¿cómo agregar estado a los traits sin repetir los problemas que los traits vinieron a resolver?

---

### Parte 1: El problema con "state = required accessors"

Recordemos: en el Paper 2, los traits no pueden tener estado. Entonces, si un trait necesita usar una variable (por ejemplo, un lock para sincronización), lo que hace es **requerir los accessors** para esa variable:

```
┌──────────────────────────────────────────────────────────────────┐
│  ESTADO VÍA REQUIRED ACCESSORS (stateless traits)                │
│                                                                  │
│  TSyncReadWrite:                                                 │
│  ───────────────                                                 │
│  Provee: syncRead(), syncWrite(), hash()                        │
│  Requiere: read(), write(), lock(), lock:()  ← accessors!       │
│                                                                  │
│  La clase que usa TSyncReadWrite DEBE proveer:                  │
│  • una variable de instancia "lock"                             │
│  • el método lock  → ↑ lock                                     │
│  • el método lock: → lock := aLock                              │
│  • la inicialización: lock := Lock new                          │
│                                                                  │
│  Esto debe repetirse en CADA clase que use TSyncReadWrite:      │
│                                                                  │
│       SyncFile      SyncStream    SyncSocket                    │
│       ─────────     ───────────   ───────────                   │
│       lock          lock          lock        ← DUPLICADO       │
│       lock()        lock()        lock()      ← DUPLICADO       │
│       lock:()       lock:()       lock:()     ← DUPLICADO       │
│       initialize    initialize    initialize  ← DUPLICADO       │
└──────────────────────────────────────────────────────────────────┘
```

Esto viola el principio más básico de los traits: evitar la duplicación de código.

---

### Parte 2: Los cuatro problemas de los stateless traits

#### Problema 1: Reusabilidad limitada — la interfaz queda "contaminada"

La interfaz requerida de un trait debería mostrar solo los métodos que son conceptualmente responsabilidad del cliente. Los accessors de estado interno no lo son: son un detalle de implementación que se filtra hacia afuera.

Imagina documentación de un trait que dice:
```
TSyncReadWrite requiere:
  • read()          ← esto SÍ es conceptualmente importante
  • write()         ← esto SÍ es conceptualmente importante
  • lock()          ← esto es ruido de implementación
  • lock:()         ← esto es ruido de implementación
```

El programador que quiere usar `TSyncReadWrite` necesita saber que "read y write son los hooks importantes", pero esa información se pierde entre el ruido de los accessors.

#### Problema 2: Boilerplate glue code — las "shell classes"

Una shell class es una clase que no tiene comportamiento propio: solo existe para declarar variables y sus accessors para satisfacer los required methods de los traits que usa.

```
┌──────────────────────────────────────────────────────────────────┐
│  SHELL CLASS — solo existe como "pegamento"                      │
│                                                                  │
│  class SyncFile extends File                                     │
│    uses: TSyncReadWrite                                          │
│                                                                  │
│    lock           ← esta clase existe ÚNICAMENTE para esto:     │
│    lock → ↑ lock  ← declarar la variable y el accessor          │
│    lock: l → lock := l                                          │
│    initialize → super initialize. lock := Lock new             │
│                                                                  │
│  Todo el "comportamiento" real está en TSyncReadWrite.          │
│  Esta clase no aporta nada conceptual. Es puro boilerplate.     │
│                                                                  │
│  En la jerarquía de Smalltalk refactorizada con stateless       │
│  traits: el 24% de las clases (7 de 29) eran shell classes.    │
└──────────────────────────────────────────────────────────────────┘
```

#### Problema 3: Propagación transitiva de cambios

Si el trait `TSyncReadWrite` evoluciona y necesita una nueva variable (por ejemplo, `waitingCount` para contar hilos esperando), debe agregar dos nuevos required methods: `waitingCount()` y `waitingCount:()`.

Estos required methods se propagan a **todos** los clientes directos e indirectos del trait. Si `TSyncReadWrite` era usado por `THighPrioritySync` que era usado por otras tres clases, ahora todas esas clases reciben un required method nuevo que deben satisfacer, aunque su interfaz pública no haya cambiado en absoluto.

```
TSyncReadWrite (agrega waitingCount)
       ↓ propaga required: waitingCount, waitingCount:
THighPrioritySync
       ↓ propaga los mismos required
 SyncFile   SyncStream   SyncSocket
 ↑ todos deben agregar la variable y los accessors
```

Esto es exactamente el tipo de fragilidad que los traits vinieron a resolver.

#### Problema 4: Violación de encapsulamiento

En Smalltalk, todos los métodos son públicos. Si un trait requiere `lock()` y `lock:()`, la clase cliente debe proveerlos como métodos públicos, aunque lógicamente `lock` sea un detalle de implementación interno que nadie debería tocar desde afuera.

---

### Parte 3: La solución — Stateful Traits

La solución que proponen los autores es elegante: permitir que los traits tengan variables de instancia, pero hacerlas **privadas al scope del trait** por defecto.

**Los tres principios del modelo:**

```
┌──────────────────────────────────────────────────────────────────┐
│  LOS TRES PRINCIPIOS DE LOS STATEFUL TRAITS                      │
│                                                                  │
│  1. PRIVACIDAD POR DEFAULT                                       │
│     Las variables de un trait son invisibles para el exterior.   │
│     Dos traits pueden tener una variable "x" sin conflicto;     │
│     son variables completamente distintas.                       │
│                                                                  │
│  2. ACCESO EXPLÍCITO CON @@                                      │
│     El cliente (la clase o trait compuesto) puede ELEGIR        │
│     acceder a una variable del trait bajo un nuevo nombre.      │
│     Esto le da un nombre local dentro del scope del cliente.    │
│                                                                  │
│  3. MERGING EXPLÍCITO CON @@                                     │
│     El cliente puede mapear variables de DISTINTOS traits       │
│     al mismo nombre, unificándolas en una sola variable         │
│     compartida por todos esos traits.                           │
└──────────────────────────────────────────────────────────────────┘
```

---

### Parte 4: El operador `@@` en detalle

La única adición sintáctica al modelo es el operador `@@` (variable access operator).

#### Sintaxis básica:
```
TraitExpresion @@ { nombreExterno → nombreInterno }
```

Donde `nombreInterno` es el nombre de la variable dentro del trait, y `nombreExterno` es el nombre bajo el cual será visible en el scope del cliente.

---

#### Escenario A: Variables completamente privadas (sin `@@`)

Cuando se componen dos traits y **no se usa** `@@`, cada trait conserva sus variables privadas. No pueden conflictuar porque no son visibles entre sí.

```
┌──────────────────────────────────────────────────────────────────┐
│  ESCENARIO A — Variables privadas sin acceso externo             │
│                                                                  │
│  T1 tiene: x (privada)         T2 tiene: x (privada)           │
│  T1 provee: getXT1(), setXT1:  T2 provee: getXT2(), setXT2:    │
│                                                                  │
│  C tiene: x (propia, privada)                                    │
│  C provee: getX(), setX:                                        │
│  C usa: T1 + T2                                                  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐        │
│  │                      C                              │        │
│  │      x  ← propia variable de C                      │        │
│  │   getX() → ↑ x                                      │        │
│  │   setX: → x := v                                    │        │
│  │                                                      │        │
│  │   ┌─────────────────┐    ┌─────────────────┐        │        │
│  │   │       T1        │    │       T2        │        │        │
│  │   │  x (privada)    │    │  x (privada)    │        │        │
│  │   │  getXT1()       │    │  getXT2()       │        │        │
│  │   │  setXT1:()      │    │  setXT2:()      │        │        │
│  │   └─────────────────┘    └─────────────────┘        │        │
│  └──────────────────────────────────────────────────────┘        │
│                                                                  │
│  c setXT1: 1. c setXT2: 2. c setX: 3.                          │
│  c getXT1 = 1  (la x de T1)                                     │
│  c getXT2 = 2  (la x de T2, ¡distinta!)                        │
│  c getX   = 3  (la x de C, también distinta)                   │
└──────────────────────────────────────────────────────────────────┘
```

Esto es **black-box reuse**: cada trait es una caja negra con su propio estado interno. Ningún trait sabe nada del estado de los demás.

---

#### Escenario B: Granting variable access (dar acceso a una variable privada)

El cliente puede exponer una variable privada de un trait bajo un nuevo nombre, que luego puede usar en sus propios métodos.

```
┌──────────────────────────────────────────────────────────────────┐
│  ESCENARIO B — Granting access                                   │
│                                                                  │
│  T1 tiene: x (privada)         T2 tiene: x (privada)           │
│  T1 provee: getXT1(), setXT1:  T2 provee: getXT2(), setXT2:    │
│                                                                  │
│  C usa: T1 @@ {xFromT1 → x}                                     │
│       + T2 @@ {xFromT2 → x}                                     │
│                                                                  │
│  C ahora puede ver:                                              │
│  • xFromT1 (que es la x de T1)                                  │
│  • xFromT2 (que es la x de T2)                                  │
│  • Puede escribir métodos que usen directamente estas variables: │
│                                                                  │
│  C provee: sum() → ↑ xFromT1 + xFromT2                         │
│                                                                  │
│  c setXT1: 1. c setXT2: 2.                                       │
│  c sum = 3   ← C accede directamente a las variables privadas   │
│               de sus traits para componer nuevo comportamiento  │
└──────────────────────────────────────────────────────────────────┘
```

Importante: los métodos **dentro del trait** siguen usando el nombre interno (`x`). Solo el cliente usa el nombre externo (`xFromT1`). Esto es análogo a cómo funciona el alias de métodos `@`.

---

#### Escenario C: Merging de variables

El cliente puede decidir que variables de distintos traits "son la misma", mapeándolas al mismo nombre. Esto crea una única variable compartida.

```
┌──────────────────────────────────────────────────────────────────┐
│  ESCENARIO C — Merging de variables                              │
│                                                                  │
│  T1 tiene: x (privada)         T2 tiene: y (privada)           │
│  T1 provee: getX(), setX:      T2 provee: getY(), setY:        │
│                                                                  │
│  C usa: T1 @@ {w → x}                                           │
│       + T2 @@ {w → y}                                           │
│                                                                  │
│  Ambas variables apuntan al mismo "slot" en la instancia:       │
│                                                                  │
│  ┌──────────────────────────────────────────────────────┐        │
│  │                       C                             │        │
│  │                   getW(), setW:                     │        │
│  │                        │                            │        │
│  │          ┌─────────────┴──────────────┐             │        │
│  │   ┌──────▼────────┐          ┌────────▼──────┐      │        │
│  │   │      T1       │          │      T2       │      │        │
│  │   │ x ────────────┼──────────┼──── y         │      │        │
│  │   │ (ambas son    │          │  la misma     │      │        │
│  │   │  "w" en C)    │          │  variable w)  │      │        │
│  │   └───────────────┘          └───────────────┘      │        │
│  └──────────────────────────────────────────────────────┘        │
│                                                                  │
│  c setW: 3.                                                      │
│  c getX = 3  ← T1 accede a su x, que es la misma que w         │
│  c getY = 3  ← T2 accede a su y, que es la misma que w         │
│  c getW = 3  ← C accede a w directamente                       │
└──────────────────────────────────────────────────────────────────┘
```

¿Cuándo es útil el merging? Cuando dos traits conceptualmente trabajan sobre el **mismo dato**. Por ejemplo, si `TSerializer` y `TCompressor` ambos necesitan saber el "tamaño máximo del buffer", tiene sentido que compartan esa variable en lugar de mantener dos copias.

---

### Parte 5: Ejemplo completo — SyncStream con Stateful Traits

Veamos cómo queda el ejemplo del wrapper de sincronización que antes requería código duplicado en cada clase cliente:

**Antes (stateless traits):**
```
TSyncReadWrite no puede tener lock.
→ Cada clase cliente debe definir lock, lock:() y initialize().
→ Código duplicado en SyncFile, SyncStream, SyncSocket.
```

**Después (stateful traits):**
```smalltalk
"El trait ahora tiene su propio lock:"
Trait named: #TSyncReadWrite
    uses: {}
    instVarNames: 'lock'   "← variable privada del trait"

syncRead
    lock acquire.
    value := self read.    "read es still required"
    lock release.
    ↑ value

syncWrite
    lock acquire.
    value := self write.
    lock release.
    ↑ value

initialize
    super initialize.
    lock := Lock new       "el trait inicializa su propio estado"

hash
    ^ ...   "implementación propia"
```

```smalltalk
"La clase que lo usa:"
Object subclass: #SyncStream
    uses: TSyncReadWrite @ {#hashFromSync → #hash}
                         @@ {syncLock → lock}    "opcional: si quiero acceder al lock"
         + TStream       @ {#hashFromStream → #hash}
    instVarNames: ''   "¡ya no necesita declarar lock!"

isBusy
    ↑ syncLock isAcquired   "puede acceder al lock via el alias @@"

hash
    ↑ self hashFromSync bitAnd: self hashFromStream
```

```
┌──────────────────────────────────────────────────────────────────┐
│  COMPARACIÓN ANTES Y DESPUÉS                                     │
│                                                                  │
│  ANTES (stateless):                                              │
│                                                                  │
│       TSyncReadWrite                                             │
│       ─────────────────────────────────────────                 │
│       syncRead, syncWrite, hash                                  │
│       REQUIERE: read, write, lock, lock: ← ruido!               │
│                                                                  │
│       SyncFile   SyncStream   SyncSocket                        │
│       ─────────  ───────────  ─────────                         │
│       lock       lock         lock       ← duplicado            │
│       lock()     lock()       lock()     ← duplicado            │
│       lock:()    lock:()      lock:()    ← duplicado            │
│       initialize initialize   initialize ← duplicado            │
│                                                                  │
│  DESPUÉS (stateful):                                             │
│                                                                  │
│       TSyncReadWrite                                             │
│       ─────────────────────────────────────────                 │
│       VARIABLE: lock (privada)                                   │
│       syncRead, syncWrite, hash, initialize                     │
│       REQUIERE: read, write ← solo los verdaderamente importantes│
│                                                                  │
│       SyncFile   SyncStream   SyncSocket                        │
│       ─────────  ───────────  ─────────                         │
│       (vacías, solo componen traits)                            │
│                                                                  │
│  Las shell classes desaparecen.                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

### Parte 6: La Flattening Property con Stateful Traits

¿Se puede seguir "aplanando" una jerarquía de traits con estado? Sí, con un paso adicional: **alpha-renaming**.

Cuando se aplana un trait con variables privadas, esas variables se **renombran** para garantizar que no colisionen con las variables de otros traits o de la clase cliente.

```
┌──────────────────────────────────────────────────────────────────┐
│  FLATTENING CON VARIABLES PRIVADAS                               │
│                                                                  │
│  VERSIÓN CON TRAITS:                                             │
│  class SyncStream uses: TSyncReadWrite + TStream                 │
│  TSyncReadWrite tiene: lock (privada)                            │
│                                                                  │
│  VERSIÓN APLANADA EQUIVALENTE:                                   │
│  class SyncStream                                                │
│    TSyncReadWrite_lock    ← renombrada para evitar colisiones   │
│    syncRead() { TSyncReadWrite_lock.acquire(); ... }            │
│    syncWrite() { ... }                                          │
│    initialize() { TSyncReadWrite_lock = Lock.new }              │
│    read() { ... }   ← de TStream                                │
│    write() { ... }  ← de TStream                                │
│    hash() { ... }                                               │
│                                                                  │
│  La clase aplanada es SEMÁNTICAMENTE IDÉNTICA a la              │
│  versión con traits.                                             │
└──────────────────────────────────────────────────────────────────┘
```

Esto preserva una propiedad clave: los traits siguen siendo puramente una herramienta de **estructuración** del código, no un mecanismo semántico nuevo. Siempre se puede razonar sobre la clase "como si no tuviera traits".

---

### Parte 7: Impacto limitado de los cambios

Una de las victorias más importantes de los stateful traits es que restauran la resistencia al cambio:

```
┌──────────────────────────────────────────────────────────────────┐
│  IMPACTO DE AGREGAR UNA VARIABLE AL TRAIT                        │
│                                                                  │
│  STATELESS TRAITS:                                               │
│  Si TSyncReadWrite agrega "waitingCount":                        │
│  → Debe agregar required: waitingCount, waitingCount:           │
│  → Todos los traits que usen TSyncReadWrite heredan estos req.  │
│  → Todas las clases al final de la cadena deben agregar         │
│    la variable y los accessors                                  │
│  → Efecto cascada por toda la jerarquía                         │
│                                                                  │
│  STATEFUL TRAITS:                                                │
│  Si TSyncReadWrite agrega "waitingCount" como variable privada: │
│  → Cero impacto en clientes. La variable es privada al trait.   │
│  → Ningún cliente sabe que la variable existe.                  │
│  → Solo se impacta si un cliente usa @@ para acceder a ella.   │
│                                                                  │
│  Si un cliente decide exponer la variable con @@:               │
│  → Solo ese cliente directo necesita adaptarse.                 │
│  → Sus clientes no se ven afectados.                            │
└──────────────────────────────────────────────────────────────────┘
```

---

### Parte 8: Implementación y Performance

Agregar estado a los traits introduce un problema técnico fundamental: la **linearización del estado**.

En lenguajes con herencia simple, el estado de un objeto es un array lineal de slots. Cada variable tiene un índice fijo. Esto es eficiente porque acceder a `self.x` se compila como "leer el slot en el índice N del objeto".

Con traits que pueden usarse en múltiples contextos, el índice de una variable puede ser diferente según quién use el trait:

```
┌──────────────────────────────────────────────────────────────────┐
│  EL PROBLEMA DE LINEARIZACIÓN DEL ESTADO                         │
│                                                                  │
│  T1 tiene: x (índice 0), y (índice 1), z (índice 2)             │
│  T2 tiene: v (índice 0), x (índice 1)                           │
│                                                                  │
│  En T3 = T1:                 En T4 = T1 + T2:                   │
│  slot 0: T1.x                slot 0: T1.x                       │
│  slot 1: T1.y                slot 1: T1.y                       │
│  slot 2: T1.z                slot 2: T1.z                       │
│                               slot 3: T2.v                      │
│                               slot 4: T2.x                      │
│                                                                  │
│  Si getV() en T2 dice "↑ slot[0]", funciona en T2 solo,        │
│  pero en T4 el slot 0 es T1.x, ¡no T2.v!                       │
│  Los métodos de T2 no pueden compartirse directamente.          │
└──────────────────────────────────────────────────────────────────┘
```

Los autores evaluaron dos estrategias:

**Estrategia 1: Object state as a dictionary (hash table)**
El estado del objeto se implementa como una tabla hash donde el nombre de la variable es la clave. Múltiples claves pueden apuntar al mismo valor (para el merging).

- Ventaja: refleja directamente la semántica.
- Desventaja: overhead de performance (acceso por hash en lugar de por índice).

**Estrategia 2: Copy-down (adoptada en la implementación)**
Inspirada en Strongtalk (un Smalltalk de alto rendimiento). Los métodos que acceden a variables de instancia se **copian** para cada usuario del trait, con los índices de variable ajustados según el layout del objeto que los usa.

- Ventaja: mismo rendimiento que herencia simple.
- Desventaja: duplicación de bytecode para los métodos con acceso a variables (no para los demás).

**Benchmarks realizados:**

| Caso de estudio | Sin traits | Stateless traits | Stateful traits |
|-----------------|------------|------------------|-----------------|
| SyncStream (I/O intensivo) | 13912 ms | 13913 ms | 13912 ms |
| LinkChecker (parsing + visitas) | 2564 ms | 2563 ms | 2564 ms |

**Conclusión:** sin diferencia de performance medible entre las tres implementaciones.

---

### Parte 9: Caso de estudio — Refactorización con Stateful Traits

El ejemplo más ilustrativo es la clase `Heap` en la jerarquía de colecciones de Smalltalk:

**Con stateless traits:**
```
┌──────────────────────────────────────────────────────────────────┐
│  HEAP CON STATELESS TRAITS — Shell class obligatoria            │
│                                                                  │
│  Heap (shell class)                                              │
│  ───────────────────────────────────────────                    │
│  Variables: array, tally, sortBlock  ← declaradas aquí solo    │
│  Métodos: array/array:, tally/tally:, sortBlock/sortBlock:      │
│           (todos son accessors de boilerplate)                   │
│                                                                  │
│  THeapImpl requiere todos esos accessors porque está           │
│  compuesto de TArrayBased (necesita array, tally) y            │
│  TSortBlockBased (necesita sortBlock).                          │
│                                                                  │
│  La clase Heap no tiene comportamiento propio:                  │
│  solo existe para satisfacer los required methods.             │
└──────────────────────────────────────────────────────────────────┘
```

**Con stateful traits:**
```
┌──────────────────────────────────────────────────────────────────┐
│  HEAP CON STATEFUL TRAITS — Las variables van al trait           │
│                                                                  │
│  TArrayBased                 TSortBlockBased                    │
│  ─────────────────────       ──────────────────────             │
│  VARIABLE: array (privada)   VARIABLE: sortBlock (privada)     │
│  VARIABLE: tally (privada)                                      │
│  Provee: size(), capacity()  Provee: sortBlock:()               │
│  (usa sus propias variables) (usa su propia variable)           │
│  NO requiere accessors       NO requiere accessors              │
│                                                                  │
│  Heap                                                            │
│  ─────────────────────────────────────────────────────────      │
│  (sin variables declaradas, sin accessors boilerplate)          │
│  Provee: add:(), copy(), grow(), removeAt:()  ← comportamiento  │
│  real, no boilerplate                                            │
└──────────────────────────────────────────────────────────────────┘
```

**Beneficios concretos del refactoring:**
- Encapsulamiento preservado: TArrayBased no expone `array` ni `tally` al exterior.
- Menos métodos en total (desaparecen los accessors boilerplate).
- Menos required methods (las dependencias de estado son internas al trait).
- La clase `Heap` deja de ser una shell class y puede tener comportamiento propio.

---

### Parte 10: Comparaciones con sistemas relacionados

#### Eiffel

Eiffel tiene herencia múltiple con un sistema sofisticado de control sobre las features heredadas (renaming, selección). Las diferencias clave con stateful traits son:

| Aspecto | Eiffel | Stateful Traits |
|---------|--------|-----------------|
| Renaming | Renaming reemplaza el nombre original. Todos los usos del nombre viejo en otros métodos se actualizan automáticamente. | Aliasing (`@`) mantiene el nombre original y agrega uno nuevo. |
| Merging de variables | Solo se pueden mergear variables que provienen de un ancestro común. | Se pueden mergear variables de cualquier trait, sin importar su origen. |
| Modelo base | Herencia múltiple con linearización | Composición simétrica sin linearización |

#### Jigsaw (Bracha, 1992)

Jigsaw es un framework teórico de modularidad que tiene operadores similares a los de los traits (merge, override, rename, restrict). Diferencias:

- Jigsaw es **black-box estricto**: un módulo no puede "abrirse" para acceder a su estado interno.
- Jigsaw tiene **renaming** (no aliasing): cambia el nombre original.
- En Jigsaw, las variables **no se pueden compartir entre módulos**.
- Los stateful traits permiten white-box controlado (con `@@`) manteniendo al cliente en control.

#### Scala Traits

- Scala tiene traits con estado (variables de instancia).
- La composición es **lineal** (C3), no simétrica.
- Conflictos de métodos concretos deben resolverse en la clase, pero la linearización sigue vigente.
- No tiene los operadores `@` (alias) y `-` (exclusión) del modelo académico.

---

### Conclusión del Paper 3

Los stateful traits son una extensión **mínima y coherente** del modelo de stateless traits. Agregan exactamente lo necesario para eliminar las limitaciones prácticas (boilerplate, shell classes, propagación de cambios), sin sacrificar ninguna de las propiedades teóricas que hacen valiosos a los traits:

- La **flattening property** se preserva via alpha-renaming.
- El **cliente retiene el control** de la composición.
- Los **conflictos de variables** no pueden ocurrir por accidente (privacidad por defecto).
- El **impacto de cambios** queda confinado a los clientes directos.
- El **rendimiento** es idéntico al de herencia simple.

La única nueva responsabilidad que adquiere el programador es decidir, al componer un trait, si desea mantener sus variables privadas (black-box) o hacerlas accesibles (white-box selectivo) bajo nombres controlados.

---

## Glosario de términos clave

| Término | Definición |
|---------|-----------|
| **Delta** | Conjunto de cambios que una subclase aplica a su padre. |
| **Linearización** | Proceso de convertir un grafo de herencia en una lista lineal donde cada ancestro aparece exactamente una vez. |
| **Mixin** | Subclase abstracta parametrizada por su padre; puede aplicarse a distintas bases. |
| **Trait** | Grupo de métodos puros (sin estado) que sirve como unidad de reutilización de comportamiento. |
| **Required method** | Método que un trait necesita que exista pero no implementa; es un "parámetro" del trait. |
| **Provided method** | Método que un trait implementa y ofrece a quien lo use. |
| **Glue code** | Código de pegamento que conecta traits entre sí, satisface required methods y resuelve conflictos. |
| **Shell class** | Clase vacía que solo existe para proveer boilerplate (variables y accessors) a los traits que usa. |
| **Flattening property** | Propiedad que garantiza que una clase con traits es semánticamente idéntica a una clase donde los métodos de los traits estuvieran escritos directamente. |
| **Same-operation exception** | Regla que dice que obtener el mismo método por múltiples caminos no constituye un conflicto, si proviene del mismo trait. |
| **Alpha-renaming** | Técnica de renombrar variables para evitar colisiones de nombres al aplanar una jerarquía de traits con estado. |
| **Copy-down** | Técnica de implementación que duplica los métodos de un trait para cada usuario, ajustando los offsets de variables. |
| **`@` (alias)** | Operador de traits que agrega un nombre alternativo a un método sin eliminar el original. |
| **`-` (exclusión)** | Operador de traits que elimina un método específico de un trait para evitar conflictos. |
| **`@@` (variable access)** | Operador de stateful traits que permite al cliente acceder a una variable privada de un trait bajo un nuevo nombre. |
