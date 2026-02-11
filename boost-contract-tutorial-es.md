# Boost.Contract: Tutorial Completo y Manual de Instrucciones

### Introducción

Este tutorial es una guía completa sobre programación por contratos en C++ usando la biblioteca [Boost.Contract](https://www.boost.org/doc/libs/release/libs/contract/doc/html/index.html). Vas a partir sin saber nada de programación por contratos y vas a terminar manejando todas las funcionalidades que ofrece la biblioteca, incluyendo subcontratación a través de jerarquías de clases, manejadores de fallas personalizados, niveles de aserción y configuración en tiempo de compilación.

La programación por contratos es una de las técnicas más potentes que existen para escribir software correcto. Cuando llegues al final de este documento, vas a haber escrito contratos para funciones libres, constructores, destructores, funciones miembro públicas, funciones virtuales con subcontratación completa, y más. Vas a entender cuándo y por qué usar cada funcionalidad, y vas a saber cómo ajustar la verificación de contratos tanto para builds de desarrollo como de producción.

El tutorial está dividido en cuatro partes. La Parte I presenta el concepto de programación por contratos y te deja listo para usar Boost.Contract. La Parte II cubre las funcionalidades centrales que vas a usar en cada proyecto. La Parte III aborda temas avanzados como herencia, funciones virtuales y semántica de movimiento. La Parte IV trata temas de nivel experto como niveles de aserción, manejadores de fallas, deshabilitación selectiva y patrones de diseño del mundo real.

## Prerequisitos

Antes de empezar, asegúrate de tener lo siguiente:

- Un compilador con soporte C++11. GCC 5 o posterior, Clang 3.4 o posterior, y MSVC 2015 o posterior están soportados.
- Una instalación funcional de Boost. Si todavía no tienes Boost instalado, sigue la [Guía de Inicio de Boost](https://www.boost.org/doc/libs/release/more/getting_started/index.html).
- Familiaridad con clases de C++, herencia, funciones virtuales y expresiones lambda. No necesitas ser experto, pero deberías sentirte cómodo leyendo código que use estas características.

---

# Parte I - Fundamentos

## Paso 1 - Entendiendo la Programación por Contratos

Antes de escribir una sola línea de código con Boost.Contract, necesitas entender qué es la programación por contratos y por qué existe. Esta sección te enseña el concepto desde cero.

### La Metáfora

Piensa en un contrato legal entre dos partes. Un dueño de casa contrata a un maestro para construir una terraza. El contrato establece que el dueño va a proveer la madera y un sitio despejado (las obligaciones del dueño), y el maestro va a construir una terraza estructuralmente sólida para una fecha determinada (la garantía del maestro). Si el dueño no entrega la madera, el maestro no tiene la culpa de no cumplir el plazo. Si el dueño entrega todo lo prometido y la terraza se cae, el maestro tiene la culpa.

La programación por contratos en software funciona de la misma manera. Cada función tiene un *llamador* (el cliente) y un *llamado* (el proveedor). El contrato entre ellos especifica qué debe proveer el llamador y qué garantiza el llamado a cambio. Cuando ambas partes cumplen sus obligaciones, el software funciona correctamente. Cuando un contrato se viola, la violación apunta directamente a la parte responsable, haciendo que los bugs sean dramáticamente más fáciles de encontrar.

### Los Tres Pilares

La programación por contratos se sostiene sobre tres conceptos fundamentales:

**Precondiciones** son condiciones que deben ser verdaderas cuando se llama a una función. Son las obligaciones del llamador. Por ejemplo, una función de raíz cuadrada podría requerir que su argumento sea no negativo. Si el llamador pasa un número negativo, el llamador ha violado el contrato, y el llamado no es responsable del resultado.

**Postcondiciones** son condiciones que deben ser verdaderas cuando una función retorna (asumiendo que no se lanzó una excepción). Son las garantías del llamado. Por ejemplo, la función de raíz cuadrada garantiza que su resultado, al elevarlo al cuadrado, es igual al argumento original (dentro de la tolerancia de punto flotante). Si esta condición es falsa después de que la función retorna, el llamado tiene un bug.

**Invariantes de clase** son condiciones que deben cumplirse para cada objeto de una clase siempre que ese objeto sea visible para el código cliente. Se verifican después de que los constructores se completan, antes y después de cada llamada a función miembro pública, y antes de que los destructores se ejecuten. Por ejemplo, una clase `vector` podría mantener el invariante de que su tamaño siempre es menor o igual a su capacidad.

### Obligaciones y Beneficios

La relación entre llamador y llamado se puede resumir así:

| | Llamador (Cliente) | Llamado (Proveedor) |
|---|---|---|
| **Obligación** | Satisfacer precondiciones | Satisfacer postcondiciones y mantener invariantes |
| **Beneficio** | Puede confiar en postcondiciones e invariantes | Puede asumir que las precondiciones se cumplen |

Esta tabla captura la esencia del Diseño por Contrato. El llamador se beneficia de las garantías del llamado, y el llamado se beneficia de las obligaciones del llamador. Ninguna de las partes necesita verificar lo que la otra parte es responsable de cumplir, lo que elimina chequeos defensivos redundantes y hace el código más limpio y enfocado.

### Cómo los Contratos Difieren de la Programación Defensiva

En la programación defensiva, cada función valida sus entradas sin importar quién es responsable de ellas. Una función de raíz cuadrada podría verificar argumentos negativos y retornar un código de error o lanzar una excepción. Este enfoque es seguro, pero tiene costos: los chequeos son redundantes cuando el llamador ya asegura la correctitud, oscurecen la lógica central de la función, y no queda claro quién es responsable del error.

La programación por contratos toma una postura diferente. La función de raíz cuadrada declara una precondición de que el argumento debe ser no negativo. Si el llamador viola esta precondición, el programa tiene un bug, y la violación del contrato se detecta inmediatamente en lugar de ser enmascarada por un retorno de error elegante. Los contratos tienen que ver con la *correctitud* (encontrar bugs), no con la *robustez* (manejar errores esperados). Ambas técnicas tienen su lugar, y Boost.Contract no te impide usar excepciones y códigos de error para condiciones de runtime esperadas. Los contratos abordan una preocupación diferente: asegurar que las suposiciones de las que depende tu código sean realmente verdaderas.

### Orígenes

La programación por contratos fue formalizada por Bertrand Meyer en el lenguaje de programación Eiffel en 1986. Meyer acuñó el término *Diseño por Contrato* y construyó el soporte de contratos directamente en la sintaxis de Eiffel con las palabras clave `require` (precondiciones), `ensure` (postcondiciones) e `invariant` (invariantes de clase). Su libro *Object-Oriented Software Construction* (Prentice Hall, 1997) sigue siendo la referencia definitiva sobre el tema.

Las ideas detrás de los contratos son aún más antiguas, enraizadas en el trabajo de C.A.R. Hoare sobre verificación de programas de los años 1960. La lógica de Hoare provee la base formal: una *triple de Hoare* `{P} S {Q}` establece que si la precondición `P` se cumple antes de que se ejecute la sentencia `S`, entonces la postcondición `Q` se cumple después. Los contratos son la expresión práctica y amigable para el programador de este concepto formal.

En el mundo de C++, los contratos han sido una funcionalidad largamente buscada a nivel de lenguaje. El estándar C++26 introduce aserciones de contrato con los atributos `[[pre:]]`, `[[post:]]` y `[[assert:]]`. Sin embargo, la facilidad del estándar no incluye invariantes de clase, valores antiguos ni subcontratación. Boost.Contract provee todas estas funcionalidades hoy, en cualquier compilador C++11.

### Referencias

- Bertrand Meyer, *Object-Oriented Software Construction*, 2da edición, Prentice Hall, 1997
- [Design by Contract - Wikipedia](https://en.wikipedia.org/wiki/Design_by_contract)
- [Building Bug-Free O-O Software - Eiffel.com](https://www.eiffel.com/values/design-by-contract/introduction/)
- [C++26 Contract Assertions - cppreference](https://en.cppreference.com/w/cpp/language/contracts)

Ahora que entiendes qué es la programación por contratos y por qué importa, estás listo para empezar a usarla en C++.

## Paso 2 - Empezando con Boost.Contract

En esta sección vas a configurar Boost.Contract y escribir tu primera función con verificación de contratos.

### Incluyendo la Biblioteca

La forma más fácil de usar Boost.Contract es incluir el header de conveniencia:

```cpp
#include <boost/contract.hpp>
```

Este único header trae todo lo que necesitas. Si prefieres includes más granulares, hay headers individuales disponibles bajo `boost/contract/` (como `boost/contract/function.hpp`, `boost/contract/constructor.hpp`, y así), pero el header de conveniencia es lo recomendado para la mayoría de los usos.

### Linkeo

Boost.Contract es una biblioteca compilada por defecto. Necesitas linkear contra `libboost_contract` (o `boost_contract` dependiendo de tu sistema de build). Hay tres modos de linkeo:

- **Linkeo dinámico** (por defecto): Define `BOOST_CONTRACT_DYN_LINK` cuando compiles tu proyecto. Este es el modo recomendado.
- **Linkeo estático**: Define `BOOST_CONTRACT_STATIC_LINK` para linkear la biblioteca estáticamente.
- **Solo headers**: Define `BOOST_CONTRACT_HEADER_ONLY` para evitar linkear del todo. Esto es cómodo para proyectos chicos pero no se recomienda para codebases más grandes porque aumenta el tiempo de compilación y el tamaño del binario.

### Tu Primer Contrato

Crea un archivo llamado `first_contract.cpp` y agrega el siguiente código:

```cpp
#include <boost/contract.hpp>
#include <limits>
#include <cassert>

void inc(int& x) {
    boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(x < std::numeric_limits<int>::max());
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(x == *old_x + 1);
        })
    ;

    ++x; // Cuerpo de la función.
}

int main() {
    int x = 10;
    inc(x);
    assert(x == 11);
    return 0;
}
```

Esta función incrementa un entero. La precondición establece que `x` no debe estar ya en su valor máximo (de lo contrario el incremento causaría overflow). La postcondición establece que después de que la función retorna, `x` es uno más que su valor antiguo. El cuerpo de la función es la única línea `++x;` que aparece después de la declaración del contrato.

### Anatomía del Patrón

Cada contrato de Boost.Contract sigue el mismo patrón estructural. Entenderlo ahora va a hacer que el resto de este tutorial fluya naturalmente.

**`boost::contract::function()`** crea un objeto de especificación de contrato para una función no miembro (o una función miembro privada/protegida). Existen otros puntos de entrada para constructores (`boost::contract::constructor`), destructores (`boost::contract::destructor`) y funciones miembro públicas (`boost::contract::public_function`), pero todos siguen la misma API fluida.

**`.precondition([&] { ... })`** recibe una lambda que contiene tus chequeos de precondición. La lambda se llama al entrar a la función.

**`.postcondition([&] { ... })`** recibe una lambda con los chequeos de postcondición. La lambda se llama cuando la función retorna normalmente (no cuando lanza una excepción).

**`BOOST_CONTRACT_ASSERT(condición)`** es una macro que verifica una condición booleana. Si la condición es falsa, ocurre una violación de contrato. Por defecto, esto llama a `std::terminate`, pero puedes personalizar este comportamiento (se cubre en una sección posterior).

**`boost::contract::check c = ...`** es la variable RAII crucial. Debes asignar el resultado de la especificación de contrato a una variable de tipo `boost::contract::check`, declarada con un tipo explícito (no `auto`), y ubicada inmediatamente antes del cuerpo de la función. El constructor del objeto `check` ejecuta las precondiciones, y su destructor ejecuta las postcondiciones. Si olvidas esta variable o la declaras incorrectamente, la biblioteca lo detecta y genera un error en tiempo de ejecución.

**`boost::contract::old_ptr<T>`** almacena una copia de un valor tomada antes de que se ejecute el cuerpo de la función, para que las postcondiciones puedan comparar el estado "después" con el estado "antes". La macro `BOOST_CONTRACT_OLDOF(expr)` crea la copia.

Con esta base en su lugar, estás listo para explorar cada funcionalidad de contratos en profundidad.

---

# Parte II - Conceptos Centrales

## Paso 3 - Funciones No Miembro

En esta sección vas a escribir contratos completos para funciones libres (no miembro), usando las cuatro cláusulas de contrato: precondiciones, postcondiciones, valores antiguos y garantías de excepción.

El patrón para una función no miembro se ve así:

```cpp
#include <boost/contract.hpp>
#include <limits>

int inc(int& x) {
    int result;
    boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(x < std::numeric_limits<int>::max());
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(x == *old_x + 1);
            BOOST_CONTRACT_ASSERT(result == *old_x);
        })
        .except([&] {
            BOOST_CONTRACT_ASSERT(x == *old_x);
        })
    ;

    return result = x++; // Cuerpo de la función.
}
```

Esta versión de `inc` retorna el valor original mientras incrementa `x`. La postcondición verifica ambos comportamientos. La garantía de excepción (`.except()`) establece que si la función lanza, `x` queda sin cambios. Para una operación de incremento trivial, una excepción es poco probable, pero esta cláusula se vuelve esencial para funciones que realizan I/O, asignación de memoria u otras operaciones que pueden fallar.

### La Cadena de la API Fluida

Los métodos de especificación de contrato deben llamarse en un orden específico:

1. `.precondition(...)` - se verifica primero, al entrar a la función
2. `.old(...)` - los valores antiguos se copian después de que pasan las precondiciones
3. `.postcondition(...)` - se verifica cuando la función retorna normalmente
4. `.except(...)` - se verifica cuando la función lanza una excepción

Cada cláusula es opcional. Puedes especificar cualquier subconjunto en el orden requerido. Si solo necesitas precondiciones y postcondiciones, omite `.old()` y `.except()`. Si solo necesitas postcondiciones, omite el resto. La API fluida impone el orden en tiempo de compilación: llamar `.precondition()` después de `.postcondition()` no va a compilar.

Usas el mismo punto de entrada `boost::contract::function()` para funciones no miembro, funciones miembro privadas, funciones miembro protegidas, funciones lambda e incluso cuerpos de bucle. Cualquier código que no verifique invariantes de clase usa este punto de entrada.

Con los contratos de funciones no miembro cubiertos, estás listo para ver los valores antiguos más de cerca.

## Paso 4 - Valores Antiguos

Las postcondiciones frecuentemente necesitan comparar el estado del mundo después de que una función se ejecuta con el estado antes de que se ejecutara. Los valores antiguos hacen esto posible. En esta sección vas a aprender las diferentes formas de capturar y usar valores antiguos.

### Valores Antiguos Básicos

El patrón más común declara un `old_ptr` y lo inicializa con `BOOST_CONTRACT_OLDOF`:

```cpp
boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
```

Esto copia el valor actual de `x` antes de que el cuerpo de la función se ejecute. En las postcondiciones, dereferencias el puntero con `*old_x` para acceder al valor guardado.

### Copia Diferida de Valores Antiguos

A veces la expresión que quieres guardar como valor antiguo depende de que una precondición se satisfaga. Por ejemplo, si una precondición verifica que un índice está dentro de los límites, y la expresión del valor antiguo accede al elemento en ese índice, no quieres que la copia del valor antiguo se ejecute antes de que la precondición se verifique.

La cláusula `.old()` maneja este caso:

```cpp
char replace(std::string& s, unsigned index, char x) {
    char result;
    boost::contract::old_ptr<char> old_char; // Declarado nulo.
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(index < s.size());
        })
        .old([&] { // Se ejecuta después de que pasan las precondiciones.
            old_char = BOOST_CONTRACT_OLDOF(s[index]);
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(s[index] == x);
            BOOST_CONTRACT_ASSERT(result == *old_char);
        })
    ;

    result = s[index];
    s[index] = x;
    return result;
}
```

Aquí `old_char` se declara sin inicialización (empieza como nulo). La lambda `.old()` se ejecuta después de que la precondición verifica que `index` es válido, así que la expresión `s[index]` es segura.

### Tipos No Copiables

En código genérico, podrías escribir contratos para tipos que no son copiables. La macro `BOOST_CONTRACT_OLDOF` requiere que el tipo sea copiable, e intentar usarla con un tipo no copiable va a producir un error de compilación. Para esta situación, Boost.Contract provee `old_ptr_if_copyable<T>`:

```cpp
boost::contract::old_ptr_if_copyable<T> old_val = BOOST_CONTRACT_OLDOF(val);
```

Si `T` es copiable, `old_val` funciona como un `old_ptr` normal. Si `T` no es copiable, `old_val` siempre es nulo, y las postcondiciones que lo dereferencian se saltan silenciosamente. Esto te permite escribir contratos genéricos que funcionan con cualquier tipo, proveyendo verificación más fuerte cuando el tipo lo soporta y degradándose de forma elegante cuando no.

Ahora que entiendes cómo capturar el estado pasado, estás listo para aprender sobre invariantes de clase - los contratos que gobiernan objetos durante toda su vida útil.

## Paso 5 - Invariantes de Clase

Un invariante de clase es una condición que cada objeto de una clase debe satisfacer siempre que ese objeto sea visible para el código cliente. A diferencia de las precondiciones y postcondiciones, que aplican a llamadas de función individuales, los invariantes aplican al objeto como un todo. Son el corazón de la programación por contratos para clases.

### Cuándo se Verifican los Invariantes

Boost.Contract verifica los invariantes de clase en estos puntos:

- **Después de que un constructor se completa**: el objeto acaba de ser creado y ya debe satisfacer sus invariantes.
- **Antes y después de cada llamada a función miembro pública**: el objeto debe ser consistente cuando la función empieza y cuando termina.
- **Antes de que un destructor comience**: el objeto debe estar en un estado válido antes de ser destruido.

Nota que los invariantes *no* se verifican para funciones miembro privadas o protegidas. Estas funciones son detalles de implementación internos que pueden romper invariantes temporalmente como parte de una operación más grande. Solo la interfaz pública, que el código cliente puede observar, impone la verificación de invariantes.

### Escribiendo un Invariante

Defines un invariante agregando una función miembro `void invariant() const` a tu clase:

```cpp
class unique_identifiers :
    private boost::contract::constructor_precondition<unique_identifiers>
{
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() >= 0);
    }

    // ... resto de la clase
};
```

La función `invariant()` debe ser `const` porque verificar el invariante no debería modificar el objeto. Puedes poner cualquier cantidad de llamadas a `BOOST_CONTRACT_ASSERT` dentro de ella.

### Invariantes Estáticos

Algunos invariantes aplican a la clase como un todo en lugar de a una instancia particular. Por ejemplo, una clase que rastrea cuántas instancias existen podría aseverar que la cuenta es no negativa. Expresas esto con una función miembro estática:

```cpp
static void static_invariant() {
    BOOST_CONTRACT_ASSERT(instances() >= 0);
}
```

Los invariantes estáticos se verifican para funciones públicas estáticas, y también se verifican (junto con los invariantes no estáticos) para constructores, destructores y funciones públicas no estáticas.

### Haciendo los Invariantes Privados

Si quieres mantener tu función `invariant()` privada (lo cual es generalmente buena práctica, ya que es un detalle de implementación), necesitas otorgarle acceso a la biblioteca:

```cpp
class my_class {
    friend class boost::contract::access;

    void invariant() const {
        BOOST_CONTRACT_ASSERT(/* ... */);
    }

public:
    // ... interfaz pública
};
```

La declaración `friend` permite que Boost.Contract llame a la función privada `invariant()`. Sin ella, la biblioteca no puede encontrar el invariante y no lo verificará.

Con los invariantes de clase entendidos, estás listo para aprender cómo los constructores y destructores interactúan con el sistema de contratos.

## Paso 6 - Constructores

Los contratos de constructores son especiales porque el objeto todavía no existe cuando el constructor comienza a ejecutarse. El invariante no se puede verificar al entrar (el objeto aún no está formado), y las precondiciones no se pueden especificar de la manera usual (no hay puntero `this` para pasar a la API fluida en el punto donde las precondiciones se verificarían).

### El Patrón de Contrato de Constructor

```cpp
class unique_identifiers :
    private boost::contract::constructor_precondition<unique_identifiers>
{
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() >= 0);
    }

    unique_identifiers(int from, int to) :
        boost::contract::constructor_precondition<unique_identifiers>([&] {
            BOOST_CONTRACT_ASSERT(from >= 0);
            BOOST_CONTRACT_ASSERT(to >= from);
        })
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(size() == (to - from + 1));
            })
        ;

        // Cuerpo del constructor.
        for (int id = from; id <= to; ++id) vect_.push_back(id);
    }

private:
    std::vector<int> vect_;
};
```

### Precondiciones de Constructor

Como el objeto aún no está construido cuando las precondiciones necesitan verificarse, las precondiciones de constructor se especifican a través de la clase base `boost::contract::constructor_precondition<Class>`. Tu clase hereda privadamente de esta base, y le pasas una lambda en la lista de inicialización de miembros. Esta lambda se ejecuta antes del cuerpo del constructor y antes de cualquier inicialización de miembros que dependa de que las precondiciones se satisfagan.

### Postcondiciones e Invariantes del Constructor

Dentro del cuerpo del constructor, creas el objeto `boost::contract::check` llamando a `boost::contract::constructor(this)`. Esto configura la verificación de postcondiciones (se ejecuta cuando el constructor se completa) y la verificación de invariantes (también se ejecuta cuando el constructor se completa, para verificar que el objeto recién construido es válido). No puedes especificar `.precondition()` aquí - usa el mecanismo de la clase base en su lugar.

La interacción es: las precondiciones se ejecutan primero (vía el inicializador de la clase base), luego el cuerpo del constructor se ejecuta, luego las postcondiciones y los invariantes se verifican. Este orden asegura que el objeto esté completamente formado antes de que cualquier postcondición o invariante lo acceda.

Ahora entiendes cómo nacen los objetos con contratos. La siguiente sección cubre cómo se destruyen.

## Paso 7 - Destructores

Los contratos de destructores son más simples que los contratos de constructores porque el objeto está completamente formado cuando el destructor comienza. Los invariantes se verifican al entrar (el objeto debe ser válido antes de que comience la destrucción), y las postcondiciones pueden verificar cualquier garantía de limpieza.

### El Patrón de Contrato de Destructor

```cpp
virtual ~unique_identifiers() {
    boost::contract::check c = boost::contract::destructor(this);

    // Cuerpo del destructor.
}
```

Los destructores no pueden tener precondiciones. El lenguaje C++ garantiza que los destructores siempre son invocables (no puedes impedir que un objeto sea destruido), así que no hay precondición significativa que verificar. La llamada a `boost::contract::destructor(this)` configura la verificación de invariantes al entrar y la verificación de postcondiciones al salir.

Si tu destructor no necesita postcondiciones y tu clase no tiene invariantes, puedes omitir el contrato por completo. Pero si tu clase define una función `invariant()`, incluir el contrato en el destructor asegura que el invariante se verifique una última vez antes de que el objeto desaparezca.

### Seguridad de Excepciones en Destructores

Ten cuidado con los contratos en destructores. Si un invariante o postcondición falla y el manejador de fallas lanza una excepción, y el destructor fue llamado durante el desenrollado de pila de otra excepción, el programa va a llamar a `std::terminate`. Este es el mismo problema que aplica a cualquier destructor que lanza en C++. La sección sobre manejadores de fallas personalizados más adelante en este tutorial cubre cómo manejar esto de forma segura.

Ahora sabes cómo escribir contratos para cada evento del ciclo de vida de un objeto. La siguiente sección une todo para funciones miembro públicas ordinarias.

## Paso 8 - Funciones Miembro Públicas

Las funciones miembro públicas son el lugar más común donde vas a escribir contratos. Verifican invariantes de clase al entrar y al salir, y soportan precondiciones, postcondiciones, valores antiguos y garantías de excepción.

### El Patrón Básico

```cpp
bool find(int id) const {
    bool result;
    boost::contract::check c = boost::contract::public_function(this)
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(id >= 0);
        })
        .postcondition([&] {
            if (size() == 0) BOOST_CONTRACT_ASSERT(!result);
        })
    ;

    // Cuerpo de la función.
    return result = std::find(vect_.begin(), vect_.end(), id) != vect_.end();
}
```

La diferencia clave con `boost::contract::function()` es que `boost::contract::public_function(this)` recibe el puntero `this`. Esto le dice a la biblioteca que verifique el invariante de clase antes y después de que la función se ejecute. Para funciones miembro const, solo se verifican los invariantes `const`.

### Funciones de Acceso

Incluso las funciones de acceso simples se benefician de los contratos cuando la clase tiene invariantes:

```cpp
int size() const {
    boost::contract::check c = boost::contract::public_function(this);
    return vect_.size();
}
```

Esta función no tiene precondiciones ni postcondiciones propias, pero la llamada a `public_function(this)` asegura que los invariantes se verifiquen. Si tu clase no tiene invariantes y el accessor no tiene pre/postcondiciones, puedes omitir el contrato por completo para mayor eficiencia.

Ya cubriste todos los tipos fundamentales de contratos. En la próxima parte, vas a aprender cómo los contratos interactúan con herencia, funciones virtuales y otras características avanzadas de C++.

---

# Parte III - Temas Avanzados

## Paso 9 - Funciones Virtuales y Subcontratación

La subcontratación es una de las funcionalidades más poderosas de Boost.Contract, y es algo que los contratos de C++26 no proveen. Cuando una clase derivada sobreescribe una función virtual, los contratos de la clase base se combinan automáticamente con los contratos de la clase derivada según reglas precisas enraizadas en el Principio de Sustitución de Liskov.

### Por Qué la Subcontratación Importa

El Principio de Sustitución de Liskov establece que si el código funciona correctamente con una referencia a la clase base, también debe funcionar correctamente con una referencia a la clase derivada. Los contratos imponen este principio: una clase derivada puede aceptar *más* entradas que la clase base (precondiciones más débiles), y debe proveer *al menos las mismas* garantías que la clase base (postcondiciones e invariantes más fuertes). Esto no es meramente una convención - Boost.Contract lo verifica automáticamente en tiempo de ejecución.

### Las Reglas de Subcontratación

- **Las precondiciones se combinan con OR** (debilitadas en clases derivadas). La precondición de una clase derivada tiene éxito si *cualquiera* de la precondición de la clase base *o* la precondición propia de la clase derivada se satisface. Esto significa que la clase derivada puede aceptar un rango más amplio de entradas.
- **Las postcondiciones se combinan con AND** (fortalecidas en clases derivadas). Tanto la postcondición de la clase base *como* la postcondición de la clase derivada deben cumplirse. La clase derivada puede hacer promesas adicionales pero no puede romper las promesas de la clase base.
- **Los invariantes se combinan con AND** (fortalecidos en clases derivadas). El invariante de la clase base y el invariante de la clase derivada deben cumplirse ambos.

### Configurando una Función Virtual

Para habilitar la subcontratación, necesitas varias piezas de infraestructura:

```cpp
class unique_identifiers {
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() >= 0);
    }

    virtual int push_back(int id, boost::contract::virtual_* v = 0) {
        int result;
        boost::contract::old_ptr<bool> old_find =
                BOOST_CONTRACT_OLDOF(v, find(id));
        boost::contract::old_ptr<int> old_size =
                BOOST_CONTRACT_OLDOF(v, size());
        boost::contract::check c = boost::contract::public_function(
                v, result, this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(id >= 0);
                BOOST_CONTRACT_ASSERT(!find(id));
            })
            .postcondition([&] (int const result) {
                if (!*old_find) {
                    BOOST_CONTRACT_ASSERT(find(id));
                    BOOST_CONTRACT_ASSERT(size() == *old_size + 1);
                }
                BOOST_CONTRACT_ASSERT(result == id);
            })
        ;

        vect_.push_back(id);
        return result = id;
    }

private:
    std::vector<int> vect_;
};
```

Hay varios detalles importantes aquí:

**El parámetro `virtual_*`.** Cada función virtual que participa en la subcontratación debe tener un parámetro extra `boost::contract::virtual_* v = 0` al final. Este parámetro es usado internamente por la biblioteca para coordinar la verificación de contratos entre clases base y derivadas. Los llamadores nunca pasan un valor para él (el valor por defecto es `0`).

**Pasar `v` a `BOOST_CONTRACT_OLDOF`.** Cuando estás dentro de una función virtual, las copias de valores antiguos deben recibir el parámetro `v`: `BOOST_CONTRACT_OLDOF(v, expr)`. Esto asegura que los valores antiguos se copien en el momento correcto durante la subcontratación.

**Pasar `v` y `result` a `public_function`.** La llamada `boost::contract::public_function(v, result, this)` pasa el parámetro virtual y la variable de resultado. La biblioteca usa estos para coordinar la verificación de postcondiciones a través de la jerarquía de clases.

**Postcondición con parámetro `result`.** Cuando una función virtual retorna un valor, la lambda de postcondición toma `result` como parámetro en lugar de capturarlo. Esto permite que la biblioteca pase el valor de resultado correcto durante la subcontratación.

### Escribiendo la Sobreescritura

```cpp
class identifiers
    #define BASES public unique_identifiers
    : BASES
{
public:
    typedef BOOST_CONTRACT_BASE_TYPES(BASES) base_types;
    #undef BASES

    void invariant() const {
        BOOST_CONTRACT_ASSERT(empty() == (size() == 0));
    }

    int push_back(int id, boost::contract::virtual_* v = 0) /* override */ {
        int result;
        boost::contract::old_ptr<bool> old_find =
                BOOST_CONTRACT_OLDOF(v, find(id));
        boost::contract::old_ptr<int> old_size =
                BOOST_CONTRACT_OLDOF(v, size());
        boost::contract::check c = boost::contract::public_function<
            override_push_back
        >(v, result, &identifiers::push_back, this, id)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(id >= 0);
                BOOST_CONTRACT_ASSERT(find(id)); // Puede insertar duplicados.
            })
            .postcondition([&] (int const result) {
                if (*old_find) BOOST_CONTRACT_ASSERT(size() == *old_size);
            })
        ;

        if (!find(id)) unique_identifiers::push_back(id);
        return result = id;
    }
    BOOST_CONTRACT_OVERRIDE(push_back)

    // ... resto de la clase derivada
};
```

Aquí aparecen varios elementos nuevos:

**`BOOST_CONTRACT_BASE_TYPES(BASES)`** declara un typedef `base_types` que le dice a la biblioteca sobre la jerarquía de clases. El patrón `#define BASES` / `#undef BASES` es una convención que mantiene la lista de clases base en un solo lugar.

**`boost::contract::public_function<override_push_back>(...)`** usa el trait de tipo de sobreescritura generado por `BOOST_CONTRACT_OVERRIDE(push_back)`. Esto le dice a la biblioteca qué función está siendo sobreescrita para que pueda buscar los contratos de la clase base.

**La función de sobreescritura recibe argumentos adicionales**: el parámetro `v`, el `result`, un puntero a la función miembro (`&identifiers::push_back`), el puntero `this`, y todos los argumentos de la función (`id`).

**`BOOST_CONTRACT_OVERRIDE(push_back)`** debe aparecer en el cuerpo de la clase (típicamente después de la definición de la función). Genera un tipo `override_push_back` que la biblioteca usa para la subcontratación. Si tienes múltiples sobreescrituras, puedes usar `BOOST_CONTRACT_OVERRIDES(func1, func2, func3)` para declararlas todas de una vez.

### Funciones Virtuales Puras

Las funciones virtuales puras también pueden tener contratos. Escribes el contrato en la implementación por defecto fuera de línea:

```cpp
template<typename Iterator>
class range {
public:
    virtual Iterator begin(boost::contract::virtual_* v = 0) = 0;
    virtual Iterator end() = 0;
    virtual bool empty() const = 0;
};

template<typename Iterator>
Iterator range<Iterator>::begin(boost::contract::virtual_* v) {
    Iterator result;
    boost::contract::check c = boost::contract::public_function(v, result, this)
        .postcondition([&] (Iterator const& result) {
            if (empty()) BOOST_CONTRACT_ASSERT(result == end());
        })
    ;
    assert(false); // Virtual pura - el cuerpo nunca se ejecuta.
    return result;
}
```

El cuerpo de la implementación por defecto de una función virtual pura nunca es ejecutado realmente por la biblioteca. Solo sus contratos se usan para la subcontratación. El `assert(false)` es una red de seguridad en caso de que el cuerpo sea llamado accidentalmente.

Ahora que entiendes la subcontratación, la siguiente sección aborda la mecánica específica de los valores de retorno en funciones virtuales.

## Paso 10 - Valores de Retorno en Funciones Virtuales

Cuando una función virtual retorna un valor, la postcondición necesita acceso a ese valor de retorno. Boost.Contract maneja esto a través de un patrón específico que difiere ligeramente dependiendo de si el tipo de retorno es un valor o una referencia.

### Tipos de Retorno por Valor

Para funciones que retornan por valor, declaras una variable de resultado local y la pasas a `public_function`:

```cpp
virtual int get(boost::contract::virtual_* v = 0) const {
    int result;
    boost::contract::check c = boost::contract::public_function(
            v, result, this)
        .postcondition([&] (int const result) {
            BOOST_CONTRACT_ASSERT(result <= 0);
        })
    ;
    return result = n_;
}
```

La lambda de postcondición toma `result` como *parámetro* en lugar de capturarlo. Esto es esencial para la subcontratación: la biblioteca pasa el valor de retorno real a través de este parámetro cuando verifica las postcondiciones de la clase base.

### Tipos de Retorno por Referencia

Cuando el tipo de retorno es una referencia, no puedes usar una variable local simple porque las referencias deben inicializarse en la declaración. En su lugar, usa `boost::optional`:

```cpp
virtual T& at(unsigned index, boost::contract::virtual_* v = 0) {
    boost::optional<T&> result;
    boost::contract::check c = boost::contract::public_function(v, result, this)
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(index < size());
        })
        .postcondition([&] (boost::optional<T const&> const& result) {
            BOOST_CONTRACT_ASSERT(*result == operator[](index));
        })
    ;
    return *(result = vect_[index]);
}
```

El `boost::optional` envuelve la referencia, permitiendo que empiece sin inicializar. La lambda de postcondición recibe `boost::optional<T const&> const&` y la dereferencia con `*result` para acceder al valor de retorno real.

Ahora entiendes toda la maquinaria para contratos de funciones virtuales. La siguiente sección cubre las funciones públicas estáticas.

## Paso 11 - Funciones Públicas Estáticas

Las funciones públicas estáticas no operan sobre un objeto específico, así que no pueden verificar invariantes no estáticos. Sin embargo, pueden y verifican invariantes estáticos.

```cpp
template<class C>
class make {
public:
    static void static_invariant() {
        BOOST_CONTRACT_ASSERT(instances() >= 0);
    }

    static int instances() {
        boost::contract::check c = boost::contract::public_function<make>();
        return instances_;
    }

    make() : object() {
        boost::contract::old_ptr<int> old_instances =
                BOOST_CONTRACT_OLDOF(instances());
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(instances() == *old_instances + 1);
            })
        ;
        ++instances_;
    }

    ~make() {
        boost::contract::old_ptr<int> old_instances =
                BOOST_CONTRACT_OLDOF(instances());
        boost::contract::check c = boost::contract::destructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(instances() == *old_instances - 1);
            })
        ;
        --instances_;
    }

    C object;

private:
    static int instances_;
};

template<class C>
int make<C>::instances_ = 0;
```

La diferencia clave es la llamada `boost::contract::public_function<make>()` con un parámetro de template explícito y sin puntero `this`. Esto le dice a la biblioteca que verifique el invariante estático de la clase `make`. Los invariantes no estáticos no se verifican porque no hay objeto sobre el cual verificarlos.

Con las funciones estáticas cubiertas, estás listo para manejar funciones sobrecargadas.

## Paso 12 - Funciones Sobrecargadas

Cuando una clase tiene múltiples sobrecargas de la misma función virtual, la macro `BOOST_CONTRACT_OVERRIDES` necesita invocarse solo una vez para todas las sobrecargas de ese nombre. El desafío es resolver el puntero de función ambiguo cuando lo pasas a `public_function`. Manejas esto con `static_cast`:

```cpp
class string_lines
    #define BASES public lines
    : BASES
{
public:
    typedef BOOST_CONTRACT_BASE_TYPES(BASES) base_types;
    #undef BASES

    BOOST_CONTRACT_OVERRIDES(put)

    void put(std::string const& x,
            boost::contract::virtual_* v = 0) /* override */ {
        boost::contract::check c = boost::contract::public_function<
                override_put>(
            v,
            static_cast<void (string_lines::*)(std::string const&,
                    boost::contract::virtual_*)>(&string_lines::put),
            this, x
        )
            .postcondition([&] { /* ... */ })
        ;
        // Cuerpo de la función.
    }

    void put(char x, boost::contract::virtual_* v = 0) /* override */ {
        boost::contract::check c = boost::contract::public_function<
                override_put>(
            v,
            static_cast<void (string_lines::*)(char,
                    boost::contract::virtual_*)>(&string_lines::put),
            this, x
        )
            .postcondition([&] { /* ... */ })
        ;
        // Cuerpo de la función.
    }
};
```

El `static_cast` le dice al compilador exactamente qué sobrecarga quieres decir. Cada sobrecarga tiene su propia especificación de contrato con sus propias precondiciones y postcondiciones, pero todas comparten el único tipo `override_put` generado por `BOOST_CONTRACT_OVERRIDES(put)`.

El mismo patrón aplica a sobrecargas const/no-const, sobrecargas con diferentes aridades y sobrecargas con parámetros por defecto. En cada caso, `static_cast` resuelve la ambigüedad.

Ahora estás equipado para manejar cualquier firma de función. La siguiente sección muestra que los contratos no se limitan a funciones con nombre.

## Paso 13 - Contratos para Lambdas, Bucles y Bloques de Código

Uno de los aspectos elegantes de Boost.Contract es que `boost::contract::function()` no está restringido a funciones con nombre. Puedes usarlo en cualquier lugar donde quieras especificar contratos para un bloque de código.

### Funciones Lambda

```cpp
int total = 0;
std::for_each(v.cbegin(), v.cend(),
    [&total] (int const x) {
        boost::contract::old_ptr<int> old_total =
                BOOST_CONTRACT_OLDOF(total);
        boost::contract::check c = boost::contract::function()
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(
                        total < std::numeric_limits<int>::max() - x);
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(total == *old_total + x);
            })
        ;

        total += x;
    }
);
```

Esta lambda acumula un total. La precondición protege contra overflow de enteros, y la postcondición verifica que cada suma es correcta. El contrato se verifica en cada invocación de la lambda.

### Cuerpos de Bucle

```cpp
int total = 0;
for (std::vector<int>::const_iterator i = v.begin(); i != v.end(); ++i) {
    boost::contract::old_ptr<int> old_total = BOOST_CONTRACT_OLDOF(total);
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(
                    total < std::numeric_limits<int>::max() - *i);
        })
        .postcondition([&] {
            BOOST_CONTRACT_ASSERT(total == *old_total + *i);
        })
    ;

    total += *i;
}
```

Este patrón convierte cada iteración del bucle en una operación verificada por contrato. La precondición actúa como un guardia del bucle, y la postcondición verifica el efecto del cuerpo del bucle en cada iteración. Esto es lo más cercano que puedes llegar a invariantes de bucle formales en C++ sin soporte a nivel de lenguaje.

### Verificaciones de Implementación

Para verificaciones inline simples que no necesitan la ceremonia completa de precondición/postcondición, puedes asignar una lambda sin argumentos directamente a una variable `check`:

```cpp
boost::contract::check c = [] {
    BOOST_CONTRACT_ASSERT(gcd(12, 28) == 4);
    BOOST_CONTRACT_ASSERT(gcd(4, 14) == 2);
};
```

La lambda se ejecuta inmediatamente cuando `c` se construye. Esto es útil para auto-tests, verificaciones de sanidad, o verificar el estado del programa en un punto particular de tu código.

Ahora vas a aprender cómo los contratos interactúan con una de las características más importantes del C++ moderno: la semántica de movimiento.

## Paso 14 - Semántica de Movimiento y Contratos

Los constructores de movimiento y los operadores de asignación por movimiento presentan un desafío único para los contratos. Después de un movimiento, el objeto fuente está en un estado válido pero no especificado - todavía debe ser destruible, pero sus invariantes pueden estar debilitados. Boost.Contract maneja esto de forma elegante.

### El Patrón

La idea clave es usar un flag `moved()` para distinguir entre objetos normales y objetos movidos:

```cpp
class circular_buffer :
        private boost::contract::constructor_precondition<circular_buffer> {
public:
    void invariant() const {
        if (!moved()) {
            BOOST_CONTRACT_ASSERT(index() < size());
        }
        // Invariantes que deben cumplirse incluso para objetos movidos
        // (ej., los necesarios para destrucción segura) van aquí.
    }

    circular_buffer(circular_buffer&& other) :
        boost::contract::constructor_precondition<circular_buffer>([&] {
            BOOST_CONTRACT_ASSERT(!other.moved());
        })
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!moved());
                BOOST_CONTRACT_ASSERT(other.moved());
            })
        ;
        move(std::forward<circular_buffer>(other));
    }

    circular_buffer& operator=(circular_buffer&& other) {
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!other.moved());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!moved());
                BOOST_CONTRACT_ASSERT(other.moved());
            })
        ;
        return move(std::forward<circular_buffer>(other));
    }

    // Los objetos movidos siempre pueden ser destruidos.
    ~circular_buffer() {
        boost::contract::check c = boost::contract::destructor(this);
    }

    bool moved() const {
        boost::contract::check c = boost::contract::public_function(this);
        return moved_;
    }

private:
    bool moved_;
    // ... otros miembros
};
```

El invariante está condicionado en `!moved()`, de modo que la verificación completa del invariante solo aplica a objetos vivos (no movidos). La precondición del constructor de movimiento asegura que no muevas desde un objeto ya movido. Las postcondiciones verifican que la transferencia fue completa: el destino está vivo y la fuente está marcada como movida.

Nota que la asignación por movimiento no requiere `!moved()` como precondición en `this`. Un objeto movido puede ser el destino de una nueva asignación (ya sea por copia o por movimiento), lo que lo restaura a un estado válido. Solo la *fuente* no debe estar ya movida.

---

# Parte IV - Temas de Experto

## Paso 15 - Niveles de Aserción

No todas las aserciones son igualmente costosas de verificar. Boost.Contract provee tres niveles de aserción que te permiten controlar el equilibrio costo-beneficio:

### Aserciones por Defecto

```cpp
BOOST_CONTRACT_ASSERT(x > 0);
```

Las aserciones por defecto siempre se verifican cuando los contratos están habilitados. Úsalas para chequeos que son baratos en relación al costo propio de la función.

### Aserciones de Auditoría

```cpp
BOOST_CONTRACT_ASSERT_AUDIT(std::is_sorted(first, last));
```

Las aserciones de auditoría solo se verifican cuando la macro `BOOST_CONTRACT_AUDITS` está definida. Úsalas para chequeos costosos que quieres durante pruebas exhaustivas pero no en builds de debug normales. El chequeo `std::is_sorted` en un rango, por ejemplo, es O(n) y podría ser demasiado costoso para una función que de otro modo es O(log n).

También puedes proteger copias costosas de valores antiguos con la misma macro:

```cpp
boost::contract::old_ptr<vector> old_me, old_other;
#ifdef BOOST_CONTRACT_AUDITS
    old_me = BOOST_CONTRACT_OLDOF(*this);
    old_other = BOOST_CONTRACT_OLDOF(other);
#endif
```

### Aserciones de Axioma

```cpp
BOOST_CONTRACT_ASSERT_AXIOM(!valid(begin(), end()));
```

Las aserciones de axioma *nunca* se verifican en tiempo de ejecución. Sirven como documentación ejecutable para propiedades que son verdaderas pero no pueden ser verificadas eficientemente (o no pueden ser verificadas del todo en C++). Por ejemplo, "este rango de iteradores es válido" es una propiedad significativa, pero no hay una forma general de verificarla. Las aserciones de axioma declaran la verdad intencionada para beneficio de lectores humanos y herramientas de análisis estático.

Los tres niveles te dan un enfoque graduado: siempre activo para chequeos baratos, opt-in para chequeos costosos, y solo documentación para propiedades no verificables. Una estrategia recomendada es habilitar los tres niveles en tu suite de pruebas de CI (`BOOST_CONTRACT_AUDITS` definido), usar solo los por defecto en builds de debug regulares, y deshabilitar todos los contratos en builds de release (ver la sección sobre deshabilitar contratos).

## Paso 16 - Manejadores de Fallas Personalizados

Por defecto, una violación de contrato llama a `std::terminate`. Este es un valor por defecto seguro - si un contrato se viola, el programa tiene un bug, y continuar podría causar más daño. Sin embargo, podrías querer un comportamiento diferente: registrar la violación, lanzar una excepción para dejar que el programa se recupere, o personalizar el comportamiento de forma diferente para distintos contextos.

### Configurando Manejadores

Boost.Contract provee funciones setter para cada tipo de falla de contrato:

```cpp
int main() {
    boost::contract::set_precondition_failure(
    boost::contract::set_postcondition_failure(
    boost::contract::set_invariant_failure(
    boost::contract::set_old_failure(
        [] (boost::contract::from where) {
            if (where == boost::contract::from_destructor) {
                // Nunca lanzar desde destructores.
                std::clog << "falla de contrato de destructor ignorada"
                          << std::endl;
            } else {
                throw; // Re-lanzar la excepción assertion_failure.
            }
        }
    ))));

    boost::contract::set_except_failure(
        [] (boost::contract::from) {
            // Ya hay una excepción activa, así que no lanzar otra.
            std::clog << "falla de garantía de excepción ignorada"
                      << std::endl;
        }
    );

    boost::contract::set_check_failure(
        [] {
            throw; // Re-lanzar para verificaciones de implementación.
        }
    );
}
```

### El Enum `from`

El parámetro `boost::contract::from` te dice en qué contexto ocurrió la falla:

- `boost::contract::from_constructor` - un contrato de constructor falló
- `boost::contract::from_destructor` - un contrato de destructor falló
- `boost::contract::from_function` - cualquier otro contrato de función falló

Esto es esencial para la seguridad de destructores. Si un contrato falla dentro de un destructor y el manejador de fallas lanza, y el destructor fue llamado durante el desenrollado de pila, el programa va a llamar a `std::terminate`. El patrón de arriba maneja esto registrando en lugar de lanzar cuando `where == from_destructor`.

### Lanzando Excepciones Personalizadas

Dentro de tus lambdas de contrato, puedes lanzar tus propias excepciones en lugar de usar `BOOST_CONTRACT_ASSERT`:

```cpp
explicit cstring(char const* chars) :
    boost::contract::constructor_precondition<cstring>([&] {
        BOOST_CONTRACT_ASSERT(chars); // Lanza assertion_failure.
        if (std::strlen(chars) > MaxSize) throw too_large_error(); // Personalizada.
    })
{
    // ...
}
```

Cuando el manejador de fallas usa `throw;` (re-lanzar), propaga cualquier excepción que la lambda del contrato haya lanzado originalmente - ya sea `boost::contract::assertion_failure` de `BOOST_CONTRACT_ASSERT` o tu propio tipo de excepción personalizado.

## Paso 17 - Implementación de Cuerpo Separado

Para proyectos más grandes, podrías querer separar las declaraciones de contratos (que van en headers) de las implementaciones del cuerpo de las funciones (que van en archivos fuente). Esto mejora el tiempo de compilación porque los cambios al cuerpo de la función no requieren recompilar archivos que solo dependen de la especificación del contrato.

### Archivo Header

```cpp
// separate_body.hpp
class iarray :
        private boost::contract::constructor_precondition<iarray> {
public:
    void invariant() const {
        BOOST_CONTRACT_ASSERT(size() <= capacity());
    }

    explicit iarray(unsigned max, unsigned count = 0) :
        boost::contract::constructor_precondition<iarray>([&] {
            BOOST_CONTRACT_ASSERT(count <= max);
        }),
        values_(new int[max]),
        capacity_(max)
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(capacity() == max);
                BOOST_CONTRACT_ASSERT(size() == count);
            })
        ;
        constructor_body(max, count); // Delegar al archivo fuente.
    }

    virtual void push_back(int value, boost::contract::virtual_* v = 0) {
        boost::contract::old_ptr<unsigned> old_size =
                BOOST_CONTRACT_OLDOF(v, size());
        boost::contract::check c = boost::contract::public_function(v, this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(size() < capacity());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(size() == *old_size + 1);
            })
        ;
        push_back_body(value); // Delegar al archivo fuente.
    }

private:
    void constructor_body(unsigned max, unsigned count);
    void push_back_body(int value);
    // ...
};
```

### Archivo Fuente

```cpp
// separate_body.cpp
#include "separate_body.hpp"

void iarray::constructor_body(unsigned max, unsigned count) {
    for (unsigned i = 0; i < count; ++i) values_[i] = int();
    size_ = count;
}

void iarray::push_back_body(int value) {
    values_[size_++] = value;
}
```

Los contratos son completamente visibles en el header (que sirve como la especificación de la función), mientras que los detalles de implementación están ocultos en el archivo fuente. Cambiar el cuerpo no invalida el header, así que las unidades de traducción dependientes no necesitan recompilación.

## Paso 18 - Deshabilitando Contratos Selectivamente

En builds de producción, podrías querer deshabilitar parte o toda la verificación de contratos por rendimiento. Boost.Contract provee un conjunto completo de macros para este propósito.

### Deshabilitar por Tipo de Contrato

Define estas macros antes de incluir cualquier header de Boost.Contract:

| Macro | Efecto |
|---|---|
| `BOOST_CONTRACT_NO_PRECONDITIONS` | Deshabilita todos los chequeos de precondición |
| `BOOST_CONTRACT_NO_POSTCONDITIONS` | Deshabilita todos los chequeos de postcondición |
| `BOOST_CONTRACT_NO_INVARIANTS` | Deshabilita todos los chequeos de invariante |
| `BOOST_CONTRACT_NO_EXCEPTS` | Deshabilita todos los chequeos de garantía de excepción |
| `BOOST_CONTRACT_NO_OLDS` | Deshabilita todas las copias de valores antiguos |
| `BOOST_CONTRACT_NO_ALL` | Deshabilita todo |

### Deshabilitar por Tipo de Operación

| Macro | Efecto |
|---|---|
| `BOOST_CONTRACT_NO_CONSTRUCTORS` | Deshabilita contratos para constructores |
| `BOOST_CONTRACT_NO_DESTRUCTORS` | Deshabilita contratos para destructores |
| `BOOST_CONTRACT_NO_PUBLIC_FUNCTIONS` | Deshabilita contratos para funciones miembro públicas |
| `BOOST_CONTRACT_NO_FUNCTIONS` | Deshabilita contratos para funciones no miembro/privadas/protegidas |

### Compilación Condicional para Cero Overhead

Cuando los contratos se deshabilitan vía estas macros, la biblioteca elimina el código de verificación de contratos. Sin embargo, las *declaraciones* de contrato (lambdas, punteros old, etc.) pueden tener todavía un pequeño overhead. Para overhead verdaderamente cero, puedes usar guardias `#ifdef`:

```cpp
#include <boost/contract/core/config.hpp>
#ifndef BOOST_CONTRACT_NO_ALL
    #include <boost/contract.hpp>
#endif

int inc(int& x) {
    int result;
    #ifndef BOOST_CONTRACT_NO_OLDS
        boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
    #endif
    #ifndef BOOST_CONTRACT_NO_FUNCTIONS
        boost::contract::check c = boost::contract::function()
            #ifndef BOOST_CONTRACT_NO_PRECONDITIONS
                .precondition([&] {
                    BOOST_CONTRACT_ASSERT(
                            x < std::numeric_limits<int>::max());
                })
            #endif
            #ifndef BOOST_CONTRACT_NO_POSTCONDITIONS
                .postcondition([&] {
                    BOOST_CONTRACT_ASSERT(x == *old_x + 1);
                    BOOST_CONTRACT_ASSERT(result == *old_x);
                })
            #endif
        ;
    #endif

    return result = x++;
}
```

Este enfoque es verboso, pero garantiza que ningún código relacionado con contratos se compile cuando los contratos están deshabilitados. La estrategia recomendada para la mayoría de los proyectos es:

- **Builds de debug y pruebas**: Habilitar todos los contratos (sin macros de deshabilitación).
- **Builds de CI**: Habilitar todos los contratos más `BOOST_CONTRACT_AUDITS` para verificación exhaustiva.
- **Builds de release**: Definir `BOOST_CONTRACT_NO_ALL` para cero overhead, o selectivamente mantener las precondiciones habilitadas como red de seguridad.

## Paso 19 - Referencia de Configuración

Boost.Contract ofrece varias macros de configuración más allá de las macros de deshabilitación cubiertas anteriormente.

### `BOOST_CONTRACT_MAX_ARGS`

Por defecto: 10. El número máximo de argumentos que `public_function` puede aceptar para funciones de sobreescritura. Si tus funciones de sobreescritura tienen más de 10 parámetros (lo cual es raro), aumenta este valor. Valores más altos aumentan el tiempo de compilación.

### Personalización de Nombres

| Macro | Por Defecto | Propósito |
|---|---|---|
| `BOOST_CONTRACT_BASES_TYPEDEF` | `base_types` | Nombre del typedef de tipos base |
| `BOOST_CONTRACT_INVARIANT_FUNC` | `invariant` | Nombre de la función de invariante |
| `BOOST_CONTRACT_STATIC_INVARIANT_FUNC` | `static_invariant` | Nombre de la función de invariante estático |

Estas macros te permiten evitar colisiones de nombres si tu código ya usa `invariant` o `base_types` para otros propósitos.

### `BOOST_CONTRACT_PERMISSIVE`

Cuando está definida, la biblioteca es más permisiva con ciertos errores de uso. Por ejemplo, no va a generar un error en tiempo de ejecución si la variable `check` se declara con el tipo incorrecto. Esto es útil durante la migración pero no debería usarse en producción.

### `BOOST_CONTRACT_ON_MISSING_CHECK_DECL`

Controla qué pasa cuando el resultado de una especificación de contrato no se asigna a una variable `boost::contract::check`. Por defecto, esto es un error en tiempo de ejecución. Definir esta macro te permite cambiar el comportamiento (ej., a una advertencia o ignorar silenciosamente).

### `BOOST_CONTRACT_DISABLE_THREADS`

Boost.Contract usa un mutex global para prevenir la verificación recursiva de contratos (que de otro modo podría causar bucles infinitos cuando un invariante llama a una función pública que verifica el invariante de nuevo). Si tu programa es single-threaded, definir `BOOST_CONTRACT_DISABLE_THREADS` elimina el overhead del mutex.

## Paso 20 - Patrones del Mundo Real

Esta sección presenta ejemplos completos y realistas tomados de la suite de ejemplos de Boost.Contract.

### El Stack (Clásico de Meyer)

Esta es una traducción directa del icónico ejemplo de stack de Bertrand Meyer de *Object-Oriented Software Construction*. Demuestra cómo una estructura de datos completa se especifica con contratos:

```cpp
#include <boost/contract.hpp>

template<typename T>
class stack
    #define BASES private boost::contract::constructor_precondition<stack<T>>
    : BASES
{
    friend boost::contract::access;
    typedef BOOST_CONTRACT_BASE_TYPES(BASES) base_types;
    #undef BASES

    void invariant() const {
        BOOST_CONTRACT_ASSERT(count() >= 0);
        BOOST_CONTRACT_ASSERT(count() <= capacity());
        BOOST_CONTRACT_ASSERT(empty() == (count() == 0));
    }

public:
    explicit stack(int n) :
        boost::contract::constructor_precondition<stack>([&] {
            BOOST_CONTRACT_ASSERT(n >= 0);
        })
    {
        boost::contract::check c = boost::contract::constructor(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(capacity() == n);
            })
        ;
        capacity_ = n;
        count_ = 0;
        array_ = new T[n];
    }

    virtual ~stack() {
        boost::contract::check c = boost::contract::destructor(this);
        delete[] array_;
    }

    int capacity() const {
        boost::contract::check c = boost::contract::public_function(this);
        return capacity_;
    }

    int count() const {
        boost::contract::check c = boost::contract::public_function(this);
        return count_;
    }

    T const& item() const {
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!empty());
            })
        ;
        return array_[count_ - 1];
    }

    bool empty() const {
        bool result;
        boost::contract::check c = boost::contract::public_function(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(result == (count() == 0));
            })
        ;
        return result = (count_ == 0);
    }

    bool full() const {
        bool result;
        boost::contract::check c = boost::contract::public_function(this)
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(result == (count() == capacity()));
            })
        ;
        return result = (count_ == capacity_);
    }

    void put(T const& x) {
        boost::contract::old_ptr<int> old_count =
                BOOST_CONTRACT_OLDOF(count());
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!full());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!empty());
                BOOST_CONTRACT_ASSERT(item() == x);
                BOOST_CONTRACT_ASSERT(count() == *old_count + 1);
            })
        ;
        array_[count_++] = x;
    }

    void remove() {
        boost::contract::old_ptr<int> old_count =
                BOOST_CONTRACT_OLDOF(count());
        boost::contract::check c = boost::contract::public_function(this)
            .precondition([&] {
                BOOST_CONTRACT_ASSERT(!empty());
            })
            .postcondition([&] {
                BOOST_CONTRACT_ASSERT(!full());
                BOOST_CONTRACT_ASSERT(count() == *old_count - 1);
            })
        ;
        --count_;
    }

private:
    int capacity_;
    int count_;
    T* array_;
};
```

Estudia este ejemplo con cuidado. Cada función pública tiene precondiciones/postcondiciones explícitas o como mínimo verifica el invariante de clase. El invariante captura las propiedades fundamentales de un stack: la cuenta está acotada, y el vacío es consistente con la cuenta. Las funciones `put` y `remove` tienen precondiciones que previenen el overflow y el underflow, y sus postcondiciones verifican los cambios de estado esperados usando valores antiguos.

### Cuándo Usar Contratos vs. Excepciones vs. Aserciones

Los contratos, las excepciones y las aserciones sirven propósitos diferentes y no son intercambiables:

**Contratos** expresan requisitos de correctitud. Un contrato violado significa que hay un bug en el programa. Los contratos se verifican durante el desarrollo y las pruebas y pueden deshabilitarse en producción. Responden a la pregunta: "¿Es correcto este código?"

**Excepciones** manejan fallas de runtime esperadas. Quedarse sin memoria, no poder abrir un archivo, o recibir entrada de red malformada no son bugs - son condiciones anticipadas que el programa debe manejar de forma elegante. Las excepciones nunca deberían deshabilitarse. Responden a la pregunta: "¿Puede esta operación tener éxito ahora mismo?"

**Aserciones** (`assert()`) son una forma liviana de contrato para chequeos de implementación interna. Son útiles pero carecen de la estructura y expresividad de Boost.Contract (sin postcondiciones, sin valores antiguos, sin invariantes, sin subcontratación).

Una buena regla general: si la condición que se verifica es responsabilidad del llamador, es una precondición. Si es la garantía del llamado, es una postcondición. Si es una propiedad del objeto, es un invariante. Si no es ninguna de estas y es solo un chequeo de sanidad sobre estado interno, `assert()` o `BOOST_CONTRACT_CHECK` es apropiado. Si es una condición de runtime esperada que podría ocurrir legítimamente, usa una excepción o código de error.

## Paso 21 - Boost.Contract y los Contratos de C++26

C++26 introduce soporte de contratos a nivel de lenguaje con tres nuevos atributos:

```cpp
int sqrt(int x)
    [[pre: x >= 0]]
    [[post r: r >= 0]]
{
    // ...
}
```

Este es un hito significativo para C++, pero la facilidad del lenguaje y Boost.Contract abordan ámbitos diferentes:

| Funcionalidad | Boost.Contract | Contratos C++26 |
|---|---|---|
| Precondiciones | Sí | Sí |
| Postcondiciones | Sí (con valores antiguos) | Sí (limitado) |
| Invariantes de clase | Sí (verificación automática) | No |
| Valores antiguos | Sí (`old_ptr<T>`) | No |
| Subcontratación | Sí (OR/AND automático) | No |
| Garantías de excepción | Sí (`.except()`) | No |
| Niveles de aserción | Sí (default/audit/axiom) | Sí (default/audit) |
| Funciona en C++11 | Sí | No (solo C++26) |

Boost.Contract provee un modelo de programación por contratos sustancialmente más rico. Los invariantes de clase, los valores antiguos y la subcontratación son funcionalidades centrales de la metodología de Diseño por Contrato que C++26 no aborda.

Si estás empezando un proyecto nuevo en un compilador C++26, podrías usar `[[pre:]]` y `[[post:]]` para contratos básicos de función y Boost.Contract para invariantes de clase, valores antiguos y subcontratación. Los dos enfoques pueden coexistir: contratos a nivel de lenguaje en la declaración de la función y Boost.Contract dentro del cuerpo de la función para las funcionalidades que el lenguaje no provee.

Si estás trabajando con C++11 a C++23, Boost.Contract es tu única opción para soporte de contratos a nivel de biblioteca, y sigue siendo una solución potente y completa.

## Conclusión

Ya cubriste todo el espectro de Boost.Contract, desde los conceptos fundacionales de la programación por contratos hasta la mecánica de nivel experto de niveles de aserción, manejadores de fallas y configuración en tiempo de compilación.

Aquí tienes un resumen de referencia rápida de los patrones que aprendiste:

| Contexto | Punto de Entrada |
|---|---|
| Función no miembro / privada / protegida | `boost::contract::function()` |
| Constructor | `boost::contract::constructor(this)` |
| Precondición de constructor | `boost::contract::constructor_precondition<Class>` (clase base) |
| Destructor | `boost::contract::destructor(this)` |
| Función miembro pública (no virtual) | `boost::contract::public_function(this)` |
| Función miembro pública (virtual, base) | `boost::contract::public_function(v, result, this)` |
| Función miembro pública (sobreescritura) | `boost::contract::public_function<Override>(v, result, &C::f, this, args...)` |
| Función pública estática | `boost::contract::public_function<Class>()` |

Y la cadena de la API fluida:

```
.precondition([&] { ... })
.old([&] { ... })
.postcondition([&] { ... })  // o  .postcondition([&] (auto const& result) { ... })
.except([&] { ... })
```

Para seguir leyendo:

- [Documentación Oficial de Boost.Contract](https://www.boost.org/doc/libs/release/libs/contract/doc/html/index.html)
- [Referencia de API de Boost.Contract](https://www.boost.org/doc/libs/release/libs/contract/doc/html/reference.html)
- [Código Fuente y Ejemplos en GitHub](https://github.com/boostorg/contract)
- [Lista de Correo de Boost (tag: contract)](https://www.boost.org/community/groups.html#main)

La programación por contratos atrapa bugs en su origen, documenta las suposiciones de tu código de forma verificable por máquina, y hace tu software más confiable. Ahora que tienes las herramientas, ponlas a trabajar.
