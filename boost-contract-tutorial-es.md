# Boost.Contract: Tutorial Completo y Manual de Instrucciones

### Introduccion

Este tutorial es una guia completa sobre programacion por contratos en C++ usando la biblioteca [Boost.Contract](https://www.boost.org/doc/libs/release/libs/contract/doc/html/index.html). Vas a partir sin saber nada de programacion por contratos y vas a terminar manejando todas las funcionalidades que ofrece la biblioteca, incluyendo subcontratacion a traves de jerarquias de clases, manejadores de fallas personalizados, niveles de asercion y configuracion en tiempo de compilacion.

La programacion por contratos es una de las tecnicas mas potentes que existen para escribir software correcto. Cuando llegues al final de este documento, vas a haber escrito contratos para funciones libres, constructores, destructores, funciones miembro publicas, funciones virtuales con subcontratacion completa, y mas. Vas a entender cuando y por que usar cada funcionalidad, y vas a saber como ajustar la verificacion de contratos tanto para builds de desarrollo como de produccion.

El tutorial esta dividido en cuatro partes. La Parte I presenta el concepto de programacion por contratos y te deja listo para usar Boost.Contract. La Parte II cubre las funcionalidades centrales que vas a usar en cada proyecto. La Parte III aborda temas avanzados como herencia, funciones virtuales y semantica de movimiento. La Parte IV trata temas de nivel experto como niveles de asercion, manejadores de fallas, deshabilitacion selectiva y patrones de diseno del mundo real.

## Prerequisitos

Antes de empezar, asegurate de tener lo siguiente:

- Un compilador con soporte C++11. GCC 5 o posterior, Clang 3.4 o posterior, y MSVC 2015 o posterior estan soportados.
- Una instalacion funcional de Boost. Si todavia no tienes Boost instalado, sigue la [Guia de Inicio de Boost](https://www.boost.org/doc/libs/release/more/getting_started/index.html).
- Familiaridad con clases de C++, herencia, funciones virtuales y expresiones lambda. No necesitas ser experto, pero deberias sentirte comodo leyendo codigo que use estas caracteristicas.

---

# Parte I - Fundamentos

## Paso 1 - Entendiendo la Programacion por Contratos

Antes de escribir una sola linea de codigo con Boost.Contract, necesitas entender que es la programacion por contratos y por que existe. Esta seccion te ensena el concepto desde cero.

### La Metafora

Piensa en un contrato legal entre dos partes. Un dueno de casa contrata a un maestro para construir una terraza. El contrato establece que el dueno va a proveer la madera y un sitio despejado (las obligaciones del dueno), y el maestro va a construir una terraza estructuralmente solida para una fecha determinada (la garantia del maestro). Si el dueno no entrega la madera, el maestro no tiene la culpa de no cumplir el plazo. Si el dueno entrega todo lo prometido y la terraza se cae, el maestro tiene la culpa.

La programacion por contratos en software funciona de la misma manera. Cada funcion tiene un *llamador* (el cliente) y un *llamado* (el proveedor). El contrato entre ellos especifica que debe proveer el llamador y que garantiza el llamado a cambio. Cuando ambas partes cumplen sus obligaciones, el software funciona correctamente. Cuando un contrato se viola, la violacion apunta directamente a la parte responsable, haciendo que los bugs sean dramaticamente mas faciles de encontrar.

### Los Tres Pilares

La programacion por contratos se sostiene sobre tres conceptos fundamentales:

**Precondiciones** son condiciones que deben ser verdaderas cuando se llama a una funcion. Son las obligaciones del llamador. Por ejemplo, una funcion de raiz cuadrada podria requerir que su argumento sea no negativo. Si el llamador pasa un numero negativo, el llamador ha violado el contrato, y el llamado no es responsable del resultado.

**Postcondiciones** son condiciones que deben ser verdaderas cuando una funcion retorna (asumiendo que no se lanzo una excepcion). Son las garantias del llamado. Por ejemplo, la funcion de raiz cuadrada garantiza que su resultado, al elevarlo al cuadrado, es igual al argumento original (dentro de la tolerancia de punto flotante). Si esta condicion es falsa despues de que la funcion retorna, el llamado tiene un bug.

**Invariantes de clase** son condiciones que deben cumplirse para cada objeto de una clase siempre que ese objeto sea visible para el codigo cliente. Se verifican despues de que los constructores se completan, antes y despues de cada llamada a funcion miembro publica, y antes de que los destructores se ejecuten. Por ejemplo, una clase `vector` podria mantener el invariante de que su tamano siempre es menor o igual a su capacidad.

### Obligaciones y Beneficios

La relacion entre llamador y llamado se puede resumir asi:

| | Llamador (Cliente) | Llamado (Proveedor) |
|---|---|---|
| **Obligacion** | Satisfacer precondiciones | Satisfacer postcondiciones y mantener invariantes |
| **Beneficio** | Puede confiar en postcondiciones e invariantes | Puede asumir que las precondiciones se cumplen |

Esta tabla captura la esencia del Diseno por Contrato. El llamador se beneficia de las garantias del llamado, y el llamado se beneficia de las obligaciones del llamador. Ninguna de las partes necesita verificar lo que la otra parte es responsable de cumplir, lo que elimina chequeos defensivos redundantes y hace el codigo mas limpio y enfocado.

### Como los Contratos Difieren de la Programacion Defensiva

En la programacion defensiva, cada funcion valida sus entradas sin importar quien es responsable de ellas. Una funcion de raiz cuadrada podria verificar argumentos negativos y retornar un codigo de error o lanzar una excepcion. Este enfoque es seguro, pero tiene costos: los chequeos son redundantes cuando el llamador ya asegura la correctitud, oscurecen la logica central de la funcion, y no queda claro quien es responsable del error.

La programacion por contratos toma una postura diferente. La funcion de raiz cuadrada declara una precondicion de que el argumento debe ser no negativo. Si el llamador viola esta precondicion, el programa tiene un bug, y la violacion del contrato se detecta inmediatamente en lugar de ser enmascarada por un retorno de error elegante. Los contratos tienen que ver con la *correctitud* (encontrar bugs), no con la *robustez* (manejar errores esperados). Ambas tecnicas tienen su lugar, y Boost.Contract no te impide usar excepciones y codigos de error para condiciones de runtime esperadas. Los contratos abordan una preocupacion diferente: asegurar que las suposiciones de las que depende tu codigo sean realmente verdaderas.

### Origenes

La programacion por contratos fue formalizada por Bertrand Meyer en el lenguaje de programacion Eiffel en 1986. Meyer acuno el termino *Diseno por Contrato* y construyo el soporte de contratos directamente en la sintaxis de Eiffel con las palabras clave `require` (precondiciones), `ensure` (postcondiciones) e `invariant` (invariantes de clase). Su libro *Object-Oriented Software Construction* (Prentice Hall, 1997) sigue siendo la referencia definitiva sobre el tema.

Las ideas detras de los contratos son aun mas antiguas, enraizadas en el trabajo de C.A.R. Hoare sobre verificacion de programas de los anos 1960. La logica de Hoare provee la base formal: una *triple de Hoare* `{P} S {Q}` establece que si la precondicion `P` se cumple antes de que se ejecute la sentencia `S`, entonces la postcondicion `Q` se cumple despues. Los contratos son la expresion practica y amigable para el programador de este concepto formal.

En el mundo de C++, los contratos han sido una funcionalidad largamente buscada a nivel de lenguaje. El estandar C++26 introduce aserciones de contrato con los atributos `[[pre:]]`, `[[post:]]` y `[[assert:]]`. Sin embargo, la facilidad del estandar no incluye invariantes de clase, valores antiguos ni subcontratacion. Boost.Contract provee todas estas funcionalidades hoy, en cualquier compilador C++11.

### Referencias

- Bertrand Meyer, *Object-Oriented Software Construction*, 2da edicion, Prentice Hall, 1997
- [Design by Contract - Wikipedia](https://en.wikipedia.org/wiki/Design_by_contract)
- [Building Bug-Free O-O Software - Eiffel.com](https://www.eiffel.com/values/design-by-contract/introduction/)
- [C++26 Contract Assertions - cppreference](https://en.cppreference.com/w/cpp/language/contracts)

Ahora que entiendes que es la programacion por contratos y por que importa, estas listo para empezar a usarla en C++.

## Paso 2 - Empezando con Boost.Contract

En esta seccion vas a configurar Boost.Contract y escribir tu primera funcion con verificacion de contratos.

### Incluyendo la Biblioteca

La forma mas facil de usar Boost.Contract es incluir el header de conveniencia:

```cpp
#include <boost/contract.hpp>
```

Este unico header trae todo lo que necesitas. Si prefieres includes mas granulares, hay headers individuales disponibles bajo `boost/contract/` (como `boost/contract/function.hpp`, `boost/contract/constructor.hpp`, y asi), pero el header de conveniencia es lo recomendado para la mayoria de los usos.

### Linkeo

Boost.Contract es una biblioteca compilada por defecto. Necesitas linkear contra `libboost_contract` (o `boost_contract` dependiendo de tu sistema de build). Hay tres modos de linkeo:

- **Linkeo dinamico** (por defecto): Define `BOOST_CONTRACT_DYN_LINK` cuando compiles tu proyecto. Este es el modo recomendado.
- **Linkeo estatico**: Define `BOOST_CONTRACT_STATIC_LINK` para linkear la biblioteca estaticamente.
- **Solo headers**: Define `BOOST_CONTRACT_HEADER_ONLY` para evitar linkear del todo. Esto es comodo para proyectos chicos pero no se recomienda para codebases mas grandes porque aumenta el tiempo de compilacion y el tamano del binario.

### Tu Primer Contrato

Crea un archivo llamado `first_contract.cpp` y agrega el siguiente codigo:

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

    ++x; // Cuerpo de la funcion.
}

int main() {
    int x = 10;
    inc(x);
    assert(x == 11);
    return 0;
}
```

Esta funcion incrementa un entero. La precondicion establece que `x` no debe estar ya en su valor maximo (de lo contrario el incremento causaria overflow). La postcondicion establece que despues de que la funcion retorna, `x` es uno mas que su valor antiguo. El cuerpo de la funcion es la unica linea `++x;` que aparece despues de la declaracion del contrato.

### Anatomia del Patron

Cada contrato de Boost.Contract sigue el mismo patron estructural. Entenderlo ahora va a hacer que el resto de este tutorial fluya naturalmente.

**`boost::contract::function()`** crea un objeto de especificacion de contrato para una funcion no miembro (o una funcion miembro privada/protegida). Existen otros puntos de entrada para constructores (`boost::contract::constructor`), destructores (`boost::contract::destructor`) y funciones miembro publicas (`boost::contract::public_function`), pero todos siguen la misma API fluida.

**`.precondition([&] { ... })`** recibe una lambda que contiene tus chequeos de precondicion. La lambda se llama al entrar a la funcion.

**`.postcondition([&] { ... })`** recibe una lambda con los chequeos de postcondicion. La lambda se llama cuando la funcion retorna normalmente (no cuando lanza una excepcion).

**`BOOST_CONTRACT_ASSERT(condicion)`** es una macro que verifica una condicion booleana. Si la condicion es falsa, ocurre una violacion de contrato. Por defecto, esto llama a `std::terminate`, pero puedes personalizar este comportamiento (se cubre en una seccion posterior).

**`boost::contract::check c = ...`** es la variable RAII crucial. Debes asignar el resultado de la especificacion de contrato a una variable de tipo `boost::contract::check`, declarada con un tipo explicito (no `auto`), y ubicada inmediatamente antes del cuerpo de la funcion. El constructor del objeto `check` ejecuta las precondiciones, y su destructor ejecuta las postcondiciones. Si olvidas esta variable o la declaras incorrectamente, la biblioteca lo detecta y genera un error en tiempo de ejecucion.

**`boost::contract::old_ptr<T>`** almacena una copia de un valor tomada antes de que se ejecute el cuerpo de la funcion, para que las postcondiciones puedan comparar el estado "despues" con el estado "antes". La macro `BOOST_CONTRACT_OLDOF(expr)` crea la copia.

Con esta base en su lugar, estas listo para explorar cada funcionalidad de contratos en profundidad.

---

# Parte II - Conceptos Centrales

## Paso 3 - Funciones No Miembro

En esta seccion vas a escribir contratos completos para funciones libres (no miembro), usando las cuatro clausulas de contrato: precondiciones, postcondiciones, valores antiguos y garantias de excepcion.

El patron para una funcion no miembro se ve asi:

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

    return result = x++; // Cuerpo de la funcion.
}
```

Esta version de `inc` retorna el valor original mientras incrementa `x`. La postcondicion verifica ambos comportamientos. La garantia de excepcion (`.except()`) establece que si la funcion lanza, `x` queda sin cambios. Para una operacion de incremento trivial, una excepcion es poco probable, pero esta clausula se vuelve esencial para funciones que realizan I/O, asignacion de memoria u otras operaciones que pueden fallar.

### La Cadena de la API Fluida

Los metodos de especificacion de contrato deben llamarse en un orden especifico:

1. `.precondition(...)` - se verifica primero, al entrar a la funcion
2. `.old(...)` - los valores antiguos se copian despues de que pasan las precondiciones
3. `.postcondition(...)` - se verifica cuando la funcion retorna normalmente
4. `.except(...)` - se verifica cuando la funcion lanza una excepcion

Cada clausula es opcional. Puedes especificar cualquier subconjunto en el orden requerido. Si solo necesitas precondiciones y postcondiciones, omite `.old()` y `.except()`. Si solo necesitas postcondiciones, omite el resto. La API fluida impone el orden en tiempo de compilacion: llamar `.precondition()` despues de `.postcondition()` no va a compilar.

Usas el mismo punto de entrada `boost::contract::function()` para funciones no miembro, funciones miembro privadas, funciones miembro protegidas, funciones lambda e incluso cuerpos de bucle. Cualquier codigo que no verifique invariantes de clase usa este punto de entrada.

Con los contratos de funciones no miembro cubiertos, estas listo para ver los valores antiguos mas de cerca.

## Paso 4 - Valores Antiguos

Las postcondiciones frecuentemente necesitan comparar el estado del mundo despues de que una funcion se ejecuta con el estado antes de que se ejecutara. Los valores antiguos hacen esto posible. En esta seccion vas a aprender las diferentes formas de capturar y usar valores antiguos.

### Valores Antiguos Basicos

El patron mas comun declara un `old_ptr` y lo inicializa con `BOOST_CONTRACT_OLDOF`:

```cpp
boost::contract::old_ptr<int> old_x = BOOST_CONTRACT_OLDOF(x);
```

Esto copia el valor actual de `x` antes de que el cuerpo de la funcion se ejecute. En las postcondiciones, dereferencias el puntero con `*old_x` para acceder al valor guardado.

### Copia Diferida de Valores Antiguos

A veces la expresion que quieres guardar como valor antiguo depende de que una precondicion se satisfaga. Por ejemplo, si una precondicion verifica que un indice esta dentro de los limites, y la expresion del valor antiguo accede al elemento en ese indice, no quieres que la copia del valor antiguo se ejecute antes de que la precondicion se verifique.

La clausula `.old()` maneja este caso:

```cpp
char replace(std::string& s, unsigned index, char x) {
    char result;
    boost::contract::old_ptr<char> old_char; // Declarado nulo.
    boost::contract::check c = boost::contract::function()
        .precondition([&] {
            BOOST_CONTRACT_ASSERT(index < s.size());
        })
        .old([&] { // Se ejecuta despues de que pasan las precondiciones.
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

Aqui `old_char` se declara sin inicializacion (empieza como nulo). La lambda `.old()` se ejecuta despues de que la precondicion verifica que `index` es valido, asi que la expresion `s[index]` es segura.

### Tipos No Copiables

En codigo generico, podrias escribir contratos para tipos que no son copiables. La macro `BOOST_CONTRACT_OLDOF` requiere que el tipo sea copiable, e intentar usarla con un tipo no copiable va a producir un error de compilacion. Para esta situacion, Boost.Contract provee `old_ptr_if_copyable<T>`:

```cpp
boost::contract::old_ptr_if_copyable<T> old_val = BOOST_CONTRACT_OLDOF(val);
```

Si `T` es copiable, `old_val` funciona como un `old_ptr` normal. Si `T` no es copiable, `old_val` siempre es nulo, y las postcondiciones que lo dereferencian se saltan silenciosamente. Esto te permite escribir contratos genericos que funcionan con cualquier tipo, proveyendo verificacion mas fuerte cuando el tipo lo soporta y degradandose de forma elegante cuando no.

Ahora que entiendes como capturar el estado pasado, estas listo para aprender sobre invariantes de clase - los contratos que gobiernan objetos durante toda su vida util.

## Paso 5 - Invariantes de Clase

Un invariante de clase es una condicion que cada objeto de una clase debe satisfacer siempre que ese objeto sea visible para el codigo cliente. A diferencia de las precondiciones y postcondiciones, que aplican a llamadas de funcion individuales, los invariantes aplican al objeto como un todo. Son el corazon de la programacion por contratos para clases.

### Cuando se Verifican los Invariantes

Boost.Contract verifica los invariantes de clase en estos puntos:

- **Despues de que un constructor se completa**: el objeto acaba de ser creado y ya debe satisfacer sus invariantes.
- **Antes y despues de cada llamada a funcion miembro publica**: el objeto debe ser consistente cuando la funcion empieza y cuando termina.
- **Antes de que un destructor comience**: el objeto debe estar en un estado valido antes de ser destruido.

Nota que los invariantes *no* se verifican para funciones miembro privadas o protegidas. Estas funciones son detalles de implementacion internos que pueden romper invariantes temporalmente como parte de una operacion mas grande. Solo la interfaz publica, que el codigo cliente puede observar, impone la verificacion de invariantes.

### Escribiendo un Invariante

Defines un invariante agregando una funcion miembro `void invariant() const` a tu clase:

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

La funcion `invariant()` debe ser `const` porque verificar el invariante no deberia modificar el objeto. Puedes poner cualquier cantidad de llamadas a `BOOST_CONTRACT_ASSERT` dentro de ella.

### Invariantes Estaticos

Algunos invariantes aplican a la clase como un todo en lugar de a una instancia particular. Por ejemplo, una clase que rastrea cuantas instancias existen podria aseverar que la cuenta es no negativa. Expresas esto con una funcion miembro estatica:

```cpp
static void static_invariant() {
    BOOST_CONTRACT_ASSERT(instances() >= 0);
}
```

Los invariantes estaticos se verifican para funciones publicas estaticas, y tambien se verifican (junto con los invariantes no estaticos) para constructores, destructores y funciones publicas no estaticas.

### Haciendo los Invariantes Privados

Si quieres mantener tu funcion `invariant()` privada (lo cual es generalmente buena practica, ya que es un detalle de implementacion), necesitas otorgarle acceso a la biblioteca:

```cpp
class my_class {
    friend class boost::contract::access;

    void invariant() const {
        BOOST_CONTRACT_ASSERT(/* ... */);
    }

public:
    // ... interfaz publica
};
```

La declaracion `friend` permite que Boost.Contract llame a la funcion privada `invariant()`. Sin ella, la biblioteca no puede encontrar el invariante y no lo verificara.

Con los invariantes de clase entendidos, estas listo para aprender como los constructores y destructores interactuan con el sistema de contratos.

## Paso 6 - Constructores

Los contratos de constructores son especiales porque el objeto todavia no existe cuando el constructor comienza a ejecutarse. El invariante no se puede verificar al entrar (el objeto aun no esta formado), y las precondiciones no se pueden especificar de la manera usual (no hay puntero `this` para pasar a la API fluida en el punto donde las precondiciones se verificarian).

### El Patron de Contrato de Constructor

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

Como el objeto aun no esta construido cuando las precondiciones necesitan verificarse, las precondiciones de constructor se especifican a traves de la clase base `boost::contract::constructor_precondition<Class>`. Tu clase hereda privadamente de esta base, y le pasas una lambda en la lista de inicializacion de miembros. Esta lambda se ejecuta antes del cuerpo del constructor y antes de cualquier inicializacion de miembros que dependa de que las precondiciones se satisfagan.

### Postcondiciones e Invariantes del Constructor

Dentro del cuerpo del constructor, creas el objeto `boost::contract::check` llamando a `boost::contract::constructor(this)`. Esto configura la verificacion de postcondiciones (se ejecuta cuando el constructor se completa) y la verificacion de invariantes (tambien se ejecuta cuando el constructor se completa, para verificar que el objeto recien construido es valido). No puedes especificar `.precondition()` aqui - usa el mecanismo de la clase base en su lugar.

La interaccion es: las precondiciones se ejecutan primero (via el inicializador de la clase base), luego el cuerpo del constructor se ejecuta, luego las postcondiciones y los invariantes se verifican. Este orden asegura que el objeto este completamente formado antes de que cualquier postcondicion o invariante lo acceda.

Ahora entiendes como nacen los objetos con contratos. La siguiente seccion cubre como se destruyen.

## Paso 7 - Destructores

Los contratos de destructores son mas simples que los contratos de constructores porque el objeto esta completamente formado cuando el destructor comienza. Los invariantes se verifican al entrar (el objeto debe ser valido antes de que comience la destruccion), y las postcondiciones pueden verificar cualquier garantia de limpieza.

### El Patron de Contrato de Destructor

```cpp
virtual ~unique_identifiers() {
    boost::contract::check c = boost::contract::destructor(this);

    // Cuerpo del destructor.
}
```

Los destructores no pueden tener precondiciones. El lenguaje C++ garantiza que los destructores siempre son invocables (no puedes impedir que un objeto sea destruido), asi que no hay precondicion significativa que verificar. La llamada a `boost::contract::destructor(this)` configura la verificacion de invariantes al entrar y la verificacion de postcondiciones al salir.

Si tu destructor no necesita postcondiciones y tu clase no tiene invariantes, puedes omitir el contrato por completo. Pero si tu clase define una funcion `invariant()`, incluir el contrato en el destructor asegura que el invariante se verifique una ultima vez antes de que el objeto desaparezca.

### Seguridad de Excepciones en Destructores

Ten cuidado con los contratos en destructores. Si un invariante o postcondicion falla y el manejador de fallas lanza una excepcion, y el destructor fue llamado durante el desenrollado de pila de otra excepcion, el programa va a llamar a `std::terminate`. Este es el mismo problema que aplica a cualquier destructor que lanza en C++. La seccion sobre manejadores de fallas personalizados mas adelante en este tutorial cubre como manejar esto de forma segura.

Ahora sabes como escribir contratos para cada evento del ciclo de vida de un objeto. La siguiente seccion une todo para funciones miembro publicas ordinarias.

## Paso 8 - Funciones Miembro Publicas

Las funciones miembro publicas son el lugar mas comun donde vas a escribir contratos. Verifican invariantes de clase al entrar y al salir, y soportan precondiciones, postcondiciones, valores antiguos y garantias de excepcion.

### El Patron Basico

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

    // Cuerpo de la funcion.
    return result = std::find(vect_.begin(), vect_.end(), id) != vect_.end();
}
```

La diferencia clave con `boost::contract::function()` es que `boost::contract::public_function(this)` recibe el puntero `this`. Esto le dice a la biblioteca que verifique el invariante de clase antes y despues de que la funcion se ejecute. Para funciones miembro const, solo se verifican los invariantes `const`.

### Funciones de Acceso

Incluso las funciones de acceso simples se benefician de los contratos cuando la clase tiene invariantes:

```cpp
int size() const {
    boost::contract::check c = boost::contract::public_function(this);
    return vect_.size();
}
```

Esta funcion no tiene precondiciones ni postcondiciones propias, pero la llamada a `public_function(this)` asegura que los invariantes se verifiquen. Si tu clase no tiene invariantes y el accessor no tiene pre/postcondiciones, puedes omitir el contrato por completo para mayor eficiencia.

Ya cubriste todos los tipos fundamentales de contratos. En la proxima parte, vas a aprender como los contratos interactuan con herencia, funciones virtuales y otras caracteristicas avanzadas de C++.

---

# Parte III - Temas Avanzados

## Paso 9 - Funciones Virtuales y Subcontratacion

La subcontratacion es una de las funcionalidades mas poderosas de Boost.Contract, y es algo que los contratos de C++26 no proveen. Cuando una clase derivada sobreescribe una funcion virtual, los contratos de la clase base se combinan automaticamente con los contratos de la clase derivada segun reglas precisas enraizadas en el Principio de Sustitucion de Liskov.

### Por Que la Subcontratacion Importa

El Principio de Sustitucion de Liskov establece que si el codigo funciona correctamente con una referencia a la clase base, tambien debe funcionar correctamente con una referencia a la clase derivada. Los contratos imponen este principio: una clase derivada puede aceptar *mas* entradas que la clase base (precondiciones mas debiles), y debe proveer *al menos las mismas* garantias que la clase base (postcondiciones e invariantes mas fuertes). Esto no es meramente una convencion - Boost.Contract lo verifica automaticamente en tiempo de ejecucion.

### Las Reglas de Subcontratacion

- **Las precondiciones se combinan con OR** (debilitadas en clases derivadas). La precondicion de una clase derivada tiene exito si *cualquiera* de la precondicion de la clase base *o* la precondicion propia de la clase derivada se satisface. Esto significa que la clase derivada puede aceptar un rango mas amplio de entradas.
- **Las postcondiciones se combinan con AND** (fortalecidas en clases derivadas). Tanto la postcondicion de la clase base *como* la postcondicion de la clase derivada deben cumplirse. La clase derivada puede hacer promesas adicionales pero no puede romper las promesas de la clase base.
- **Los invariantes se combinan con AND** (fortalecidos en clases derivadas). El invariante de la clase base y el invariante de la clase derivada deben cumplirse ambos.

### Configurando una Funcion Virtual

Para habilitar la subcontratacion, necesitas varias piezas de infraestructura:

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

Hay varios detalles importantes aqui:

**El parametro `virtual_*`.** Cada funcion virtual que participa en la subcontratacion debe tener un parametro extra `boost::contract::virtual_* v = 0` al final. Este parametro es usado internamente por la biblioteca para coordinar la verificacion de contratos entre clases base y derivadas. Los llamadores nunca pasan un valor para el (el valor por defecto es `0`).

**Pasar `v` a `BOOST_CONTRACT_OLDOF`.** Cuando estas dentro de una funcion virtual, las copias de valores antiguos deben recibir el parametro `v`: `BOOST_CONTRACT_OLDOF(v, expr)`. Esto asegura que los valores antiguos se copien en el momento correcto durante la subcontratacion.

**Pasar `v` y `result` a `public_function`.** La llamada `boost::contract::public_function(v, result, this)` pasa el parametro virtual y la variable de resultado. La biblioteca usa estos para coordinar la verificacion de postcondiciones a traves de la jerarquia de clases.

**Postcondicion con parametro `result`.** Cuando una funcion virtual retorna un valor, la lambda de postcondicion toma `result` como parametro en lugar de capturarlo. Esto permite que la biblioteca pase el valor de resultado correcto durante la subcontratacion.

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

Aqui aparecen varios elementos nuevos:

**`BOOST_CONTRACT_BASE_TYPES(BASES)`** declara un typedef `base_types` que le dice a la biblioteca sobre la jerarquia de clases. El patron `#define BASES` / `#undef BASES` es una convencion que mantiene la lista de clases base en un solo lugar.

**`boost::contract::public_function<override_push_back>(...)`** usa el trait de tipo de sobreescritura generado por `BOOST_CONTRACT_OVERRIDE(push_back)`. Esto le dice a la biblioteca que funcion esta siendo sobreescrita para que pueda buscar los contratos de la clase base.

**La funcion de sobreescritura recibe argumentos adicionales**: el parametro `v`, el `result`, un puntero a la funcion miembro (`&identifiers::push_back`), el puntero `this`, y todos los argumentos de la funcion (`id`).

**`BOOST_CONTRACT_OVERRIDE(push_back)`** debe aparecer en el cuerpo de la clase (tipicamente despues de la definicion de la funcion). Genera un tipo `override_push_back` que la biblioteca usa para la subcontratacion. Si tienes multiples sobreescrituras, puedes usar `BOOST_CONTRACT_OVERRIDES(func1, func2, func3)` para declararlas todas de una vez.

### Funciones Virtuales Puras

Las funciones virtuales puras tambien pueden tener contratos. Escribes el contrato en la implementacion por defecto fuera de linea:

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

El cuerpo de la implementacion por defecto de una funcion virtual pura nunca es ejecutado realmente por la biblioteca. Solo sus contratos se usan para la subcontratacion. El `assert(false)` es una red de seguridad en caso de que el cuerpo sea llamado accidentalmente.

Ahora que entiendes la subcontratacion, la siguiente seccion aborda la mecanica especifica de los valores de retorno en funciones virtuales.

## Paso 10 - Valores de Retorno en Funciones Virtuales

Cuando una funcion virtual retorna un valor, la postcondicion necesita acceso a ese valor de retorno. Boost.Contract maneja esto a traves de un patron especifico que difiere ligeramente dependiendo de si el tipo de retorno es un valor o una referencia.

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

La lambda de postcondicion toma `result` como *parametro* en lugar de capturarlo. Esto es esencial para la subcontratacion: la biblioteca pasa el valor de retorno real a traves de este parametro cuando verifica las postcondiciones de la clase base.

### Tipos de Retorno por Referencia

Cuando el tipo de retorno es una referencia, no puedes usar una variable local simple porque las referencias deben inicializarse en la declaracion. En su lugar, usa `boost::optional`:

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

El `boost::optional` envuelve la referencia, permitiendo que empiece sin inicializar. La lambda de postcondicion recibe `boost::optional<T const&> const&` y la dereferencia con `*result` para acceder al valor de retorno real.

Ahora entiendes toda la maquinaria para contratos de funciones virtuales. La siguiente seccion cubre las funciones publicas estaticas.

## Paso 11 - Funciones Publicas Estaticas

Las funciones publicas estaticas no operan sobre un objeto especifico, asi que no pueden verificar invariantes no estaticos. Sin embargo, pueden y verifican invariantes estaticos.

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

La diferencia clave es la llamada `boost::contract::public_function<make>()` con un parametro de template explicito y sin puntero `this`. Esto le dice a la biblioteca que verifique el invariante estatico de la clase `make`. Los invariantes no estaticos no se verifican porque no hay objeto sobre el cual verificarlos.

Con las funciones estaticas cubiertas, estas listo para manejar funciones sobrecargadas.

## Paso 12 - Funciones Sobrecargadas

Cuando una clase tiene multiples sobrecargas de la misma funcion virtual, la macro `BOOST_CONTRACT_OVERRIDES` necesita invocarse solo una vez para todas las sobrecargas de ese nombre. El desafio es resolver el puntero de funcion ambiguo cuando lo pasas a `public_function`. Manejas esto con `static_cast`:

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
        // Cuerpo de la funcion.
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
        // Cuerpo de la funcion.
    }
};
```

El `static_cast` le dice al compilador exactamente que sobrecarga quieres decir. Cada sobrecarga tiene su propia especificacion de contrato con sus propias precondiciones y postcondiciones, pero todas comparten el unico tipo `override_put` generado por `BOOST_CONTRACT_OVERRIDES(put)`.

El mismo patron aplica a sobrecargas const/no-const, sobrecargas con diferentes aridades y sobrecargas con parametros por defecto. En cada caso, `static_cast` resuelve la ambiguedad.

Ahora estas equipado para manejar cualquier firma de funcion. La siguiente seccion muestra que los contratos no se limitan a funciones con nombre.

## Paso 13 - Contratos para Lambdas, Bucles y Bloques de Codigo

Uno de los aspectos elegantes de Boost.Contract es que `boost::contract::function()` no esta restringido a funciones con nombre. Puedes usarlo en cualquier lugar donde quieras especificar contratos para un bloque de codigo.

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

Esta lambda acumula un total. La precondicion protege contra overflow de enteros, y la postcondicion verifica que cada suma es correcta. El contrato se verifica en cada invocacion de la lambda.

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

Este patron convierte cada iteracion del bucle en una operacion verificada por contrato. La precondicion actua como un guardia del bucle, y la postcondicion verifica el efecto del cuerpo del bucle en cada iteracion. Esto es lo mas cercano que puedes llegar a invariantes de bucle formales en C++ sin soporte a nivel de lenguaje.

### Verificaciones de Implementacion

Para verificaciones inline simples que no necesitan la ceremonia completa de precondicion/postcondicion, puedes asignar una lambda sin argumentos directamente a una variable `check`:

```cpp
boost::contract::check c = [] {
    BOOST_CONTRACT_ASSERT(gcd(12, 28) == 4);
    BOOST_CONTRACT_ASSERT(gcd(4, 14) == 2);
};
```

La lambda se ejecuta inmediatamente cuando `c` se construye. Esto es util para auto-tests, verificaciones de sanidad, o verificar el estado del programa en un punto particular de tu codigo.

Ahora vas a aprender como los contratos interactuan con una de las caracteristicas mas importantes del C++ moderno: la semantica de movimiento.

## Paso 14 - Semantica de Movimiento y Contratos

Los constructores de movimiento y los operadores de asignacion por movimiento presentan un desafio unico para los contratos. Despues de un movimiento, el objeto fuente esta en un estado valido pero no especificado - todavia debe ser destruible, pero sus invariantes pueden estar debilitados. Boost.Contract maneja esto de forma elegante.

### El Patron

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
        // (ej., los necesarios para destruccion segura) van aqui.
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

El invariante esta condicionado en `!moved()`, de modo que la verificacion completa del invariante solo aplica a objetos vivos (no movidos). La precondicion del constructor de movimiento asegura que no muevas desde un objeto ya movido. Las postcondiciones verifican que la transferencia fue completa: el destino esta vivo y la fuente esta marcada como movida.

Nota que la asignacion por movimiento no requiere `!moved()` como precondicion en `this`. Un objeto movido puede ser el destino de una nueva asignacion (ya sea por copia o por movimiento), lo que lo restaura a un estado valido. Solo la *fuente* no debe estar ya movida.

---

# Parte IV - Temas de Experto

## Paso 15 - Niveles de Asercion

No todas las aserciones son igualmente costosas de verificar. Boost.Contract provee tres niveles de asercion que te permiten controlar el equilibrio costo-beneficio:

### Aserciones por Defecto

```cpp
BOOST_CONTRACT_ASSERT(x > 0);
```

Las aserciones por defecto siempre se verifican cuando los contratos estan habilitados. Usalas para chequeos que son baratos en relacion al costo propio de la funcion.

### Aserciones de Auditoria

```cpp
BOOST_CONTRACT_ASSERT_AUDIT(std::is_sorted(first, last));
```

Las aserciones de auditoria solo se verifican cuando la macro `BOOST_CONTRACT_AUDITS` esta definida. Usalas para chequeos costosos que quieres durante pruebas exhaustivas pero no en builds de debug normales. El chequeo `std::is_sorted` en un rango, por ejemplo, es O(n) y podria ser demasiado costoso para una funcion que de otro modo es O(log n).

Tambien puedes proteger copias costosas de valores antiguos con la misma macro:

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

Las aserciones de axioma *nunca* se verifican en tiempo de ejecucion. Sirven como documentacion ejecutable para propiedades que son verdaderas pero no pueden ser verificadas eficientemente (o no pueden ser verificadas del todo en C++). Por ejemplo, "este rango de iteradores es valido" es una propiedad significativa, pero no hay una forma general de verificarla. Las aserciones de axioma declaran la verdad intencionada para beneficio de lectores humanos y herramientas de analisis estatico.

Los tres niveles te dan un enfoque graduado: siempre activo para chequeos baratos, opt-in para chequeos costosos, y solo documentacion para propiedades no verificables. Una estrategia recomendada es habilitar los tres niveles en tu suite de pruebas de CI (`BOOST_CONTRACT_AUDITS` definido), usar solo los por defecto en builds de debug regulares, y deshabilitar todos los contratos en builds de release (ver la seccion sobre deshabilitar contratos).

## Paso 16 - Manejadores de Fallas Personalizados

Por defecto, una violacion de contrato llama a `std::terminate`. Este es un valor por defecto seguro - si un contrato se viola, el programa tiene un bug, y continuar podria causar mas dano. Sin embargo, podrias querer un comportamiento diferente: registrar la violacion, lanzar una excepcion para dejar que el programa se recupere, o personalizar el comportamiento de forma diferente para distintos contextos.

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
                throw; // Re-lanzar la excepcion assertion_failure.
            }
        }
    ))));

    boost::contract::set_except_failure(
        [] (boost::contract::from) {
            // Ya hay una excepcion activa, asi que no lanzar otra.
            std::clog << "falla de garantia de excepcion ignorada"
                      << std::endl;
        }
    );

    boost::contract::set_check_failure(
        [] {
            throw; // Re-lanzar para verificaciones de implementacion.
        }
    );
}
```

### El Enum `from`

El parametro `boost::contract::from` te dice en que contexto ocurrio la falla:

- `boost::contract::from_constructor` - un contrato de constructor fallo
- `boost::contract::from_destructor` - un contrato de destructor fallo
- `boost::contract::from_function` - cualquier otro contrato de funcion fallo

Esto es esencial para la seguridad de destructores. Si un contrato falla dentro de un destructor y el manejador de fallas lanza, y el destructor fue llamado durante el desenrollado de pila, el programa va a llamar a `std::terminate`. El patron de arriba maneja esto registrando en lugar de lanzar cuando `where == from_destructor`.

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

Cuando el manejador de fallas usa `throw;` (re-lanzar), propaga cualquier excepcion que la lambda del contrato haya lanzado originalmente - ya sea `boost::contract::assertion_failure` de `BOOST_CONTRACT_ASSERT` o tu propio tipo de excepcion personalizado.

## Paso 17 - Implementacion de Cuerpo Separado

Para proyectos mas grandes, podrias querer separar las declaraciones de contratos (que van en headers) de las implementaciones del cuerpo de las funciones (que van en archivos fuente). Esto mejora el tiempo de compilacion porque los cambios al cuerpo de la funcion no requieren recompilar archivos que solo dependen de la especificacion del contrato.

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

Los contratos son completamente visibles en el header (que sirve como la especificacion de la funcion), mientras que los detalles de implementacion estan ocultos en el archivo fuente. Cambiar el cuerpo no invalida el header, asi que las unidades de traduccion dependientes no necesitan recompilacion.

## Paso 18 - Deshabilitando Contratos Selectivamente

En builds de produccion, podrias querer deshabilitar parte o toda la verificacion de contratos por rendimiento. Boost.Contract provee un conjunto completo de macros para este proposito.

### Deshabilitar por Tipo de Contrato

Define estas macros antes de incluir cualquier header de Boost.Contract:

| Macro | Efecto |
|---|---|
| `BOOST_CONTRACT_NO_PRECONDITIONS` | Deshabilita todos los chequeos de precondicion |
| `BOOST_CONTRACT_NO_POSTCONDITIONS` | Deshabilita todos los chequeos de postcondicion |
| `BOOST_CONTRACT_NO_INVARIANTS` | Deshabilita todos los chequeos de invariante |
| `BOOST_CONTRACT_NO_EXCEPTS` | Deshabilita todos los chequeos de garantia de excepcion |
| `BOOST_CONTRACT_NO_OLDS` | Deshabilita todas las copias de valores antiguos |
| `BOOST_CONTRACT_NO_ALL` | Deshabilita todo |

### Deshabilitar por Tipo de Operacion

| Macro | Efecto |
|---|---|
| `BOOST_CONTRACT_NO_CONSTRUCTORS` | Deshabilita contratos para constructores |
| `BOOST_CONTRACT_NO_DESTRUCTORS` | Deshabilita contratos para destructores |
| `BOOST_CONTRACT_NO_PUBLIC_FUNCTIONS` | Deshabilita contratos para funciones miembro publicas |
| `BOOST_CONTRACT_NO_FUNCTIONS` | Deshabilita contratos para funciones no miembro/privadas/protegidas |

### Compilacion Condicional para Cero Overhead

Cuando los contratos se deshabilitan via estas macros, la biblioteca elimina el codigo de verificacion de contratos. Sin embargo, las *declaraciones* de contrato (lambdas, punteros old, etc.) pueden tener todavia un pequeno overhead. Para overhead verdaderamente cero, puedes usar guardias `#ifdef`:

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

Este enfoque es verboso, pero garantiza que ningun codigo relacionado con contratos se compile cuando los contratos estan deshabilitados. La estrategia recomendada para la mayoria de los proyectos es:

- **Builds de debug y pruebas**: Habilitar todos los contratos (sin macros de deshabilitacion).
- **Builds de CI**: Habilitar todos los contratos mas `BOOST_CONTRACT_AUDITS` para verificacion exhaustiva.
- **Builds de release**: Definir `BOOST_CONTRACT_NO_ALL` para cero overhead, o selectivamente mantener las precondiciones habilitadas como red de seguridad.

## Paso 19 - Referencia de Configuracion

Boost.Contract ofrece varias macros de configuracion mas alla de las macros de deshabilitacion cubiertas anteriormente.

### `BOOST_CONTRACT_MAX_ARGS`

Por defecto: 10. El numero maximo de argumentos que `public_function` puede aceptar para funciones de sobreescritura. Si tus funciones de sobreescritura tienen mas de 10 parametros (lo cual es raro), aumenta este valor. Valores mas altos aumentan el tiempo de compilacion.

### Personalizacion de Nombres

| Macro | Por Defecto | Proposito |
|---|---|---|
| `BOOST_CONTRACT_BASES_TYPEDEF` | `base_types` | Nombre del typedef de tipos base |
| `BOOST_CONTRACT_INVARIANT_FUNC` | `invariant` | Nombre de la funcion de invariante |
| `BOOST_CONTRACT_STATIC_INVARIANT_FUNC` | `static_invariant` | Nombre de la funcion de invariante estatico |

Estas macros te permiten evitar colisiones de nombres si tu codigo ya usa `invariant` o `base_types` para otros propositos.

### `BOOST_CONTRACT_PERMISSIVE`

Cuando esta definida, la biblioteca es mas permisiva con ciertos errores de uso. Por ejemplo, no va a generar un error en tiempo de ejecucion si la variable `check` se declara con el tipo incorrecto. Esto es util durante la migracion pero no deberia usarse en produccion.

### `BOOST_CONTRACT_ON_MISSING_CHECK_DECL`

Controla que pasa cuando el resultado de una especificacion de contrato no se asigna a una variable `boost::contract::check`. Por defecto, esto es un error en tiempo de ejecucion. Definir esta macro te permite cambiar el comportamiento (ej., a una advertencia o ignorar silenciosamente).

### `BOOST_CONTRACT_DISABLE_THREADS`

Boost.Contract usa un mutex global para prevenir la verificacion recursiva de contratos (que de otro modo podria causar bucles infinitos cuando un invariante llama a una funcion publica que verifica el invariante de nuevo). Si tu programa es single-threaded, definir `BOOST_CONTRACT_DISABLE_THREADS` elimina el overhead del mutex.

## Paso 20 - Patrones del Mundo Real

Esta seccion presenta ejemplos completos y realistas tomados de la suite de ejemplos de Boost.Contract.

### El Stack (Clasico de Meyer)

Esta es una traduccion directa del iconico ejemplo de stack de Bertrand Meyer de *Object-Oriented Software Construction*. Demuestra como una estructura de datos completa se especifica con contratos:

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

Estudia este ejemplo con cuidado. Cada funcion publica tiene precondiciones/postcondiciones explicitas o como minimo verifica el invariante de clase. El invariante captura las propiedades fundamentales de un stack: la cuenta esta acotada, y el vacio es consistente con la cuenta. Las funciones `put` y `remove` tienen precondiciones que previenen el overflow y el underflow, y sus postcondiciones verifican los cambios de estado esperados usando valores antiguos.

### Cuando Usar Contratos vs. Excepciones vs. Aserciones

Los contratos, las excepciones y las aserciones sirven propositos diferentes y no son intercambiables:

**Contratos** expresan requisitos de correctitud. Un contrato violado significa que hay un bug en el programa. Los contratos se verifican durante el desarrollo y las pruebas y pueden deshabilitarse en produccion. Responden a la pregunta: "Es correcto este codigo?"

**Excepciones** manejan fallas de runtime esperadas. Quedarse sin memoria, no poder abrir un archivo, o recibir entrada de red malformada no son bugs - son condiciones anticipadas que el programa debe manejar de forma elegante. Las excepciones nunca deberian deshabilitarse. Responden a la pregunta: "Puede esta operacion tener exito ahora mismo?"

**Aserciones** (`assert()`) son una forma liviana de contrato para chequeos de implementacion interna. Son utiles pero carecen de la estructura y expresividad de Boost.Contract (sin postcondiciones, sin valores antiguos, sin invariantes, sin subcontratacion).

Una buena regla general: si la condicion que se verifica es responsabilidad del llamador, es una precondicion. Si es la garantia del llamado, es una postcondicion. Si es una propiedad del objeto, es un invariante. Si no es ninguna de estas y es solo un chequeo de sanidad sobre estado interno, `assert()` o `BOOST_CONTRACT_CHECK` es apropiado. Si es una condicion de runtime esperada que podria ocurrir legitimamente, usa una excepcion o codigo de error.

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

Este es un hito significativo para C++, pero la facilidad del lenguaje y Boost.Contract abordan ambitos diferentes:

| Funcionalidad | Boost.Contract | Contratos C++26 |
|---|---|---|
| Precondiciones | Si | Si |
| Postcondiciones | Si (con valores antiguos) | Si (limitado) |
| Invariantes de clase | Si (verificacion automatica) | No |
| Valores antiguos | Si (`old_ptr<T>`) | No |
| Subcontratacion | Si (OR/AND automatico) | No |
| Garantias de excepcion | Si (`.except()`) | No |
| Niveles de asercion | Si (default/audit/axiom) | Si (default/audit) |
| Funciona en C++11 | Si | No (solo C++26) |

Boost.Contract provee un modelo de programacion por contratos sustancialmente mas rico. Los invariantes de clase, los valores antiguos y la subcontratacion son funcionalidades centrales de la metodologia de Diseno por Contrato que C++26 no aborda.

Si estas empezando un proyecto nuevo en un compilador C++26, podrias usar `[[pre:]]` y `[[post:]]` para contratos basicos de funcion y Boost.Contract para invariantes de clase, valores antiguos y subcontratacion. Los dos enfoques pueden coexistir: contratos a nivel de lenguaje en la declaracion de la funcion y Boost.Contract dentro del cuerpo de la funcion para las funcionalidades que el lenguaje no provee.

Si estas trabajando con C++11 a C++23, Boost.Contract es tu unica opcion para soporte de contratos a nivel de biblioteca, y sigue siendo una solucion potente y completa.

## Conclusion

Ya cubriste todo el espectro de Boost.Contract, desde los conceptos fundacionales de la programacion por contratos hasta la mecanica de nivel experto de niveles de asercion, manejadores de fallas y configuracion en tiempo de compilacion.

Aqui tienes un resumen de referencia rapida de los patrones que aprendiste:

| Contexto | Punto de Entrada |
|---|---|
| Funcion no miembro / privada / protegida | `boost::contract::function()` |
| Constructor | `boost::contract::constructor(this)` |
| Precondicion de constructor | `boost::contract::constructor_precondition<Class>` (clase base) |
| Destructor | `boost::contract::destructor(this)` |
| Funcion miembro publica (no virtual) | `boost::contract::public_function(this)` |
| Funcion miembro publica (virtual, base) | `boost::contract::public_function(v, result, this)` |
| Funcion miembro publica (sobreescritura) | `boost::contract::public_function<Override>(v, result, &C::f, this, args...)` |
| Funcion publica estatica | `boost::contract::public_function<Class>()` |

Y la cadena de la API fluida:

```
.precondition([&] { ... })
.old([&] { ... })
.postcondition([&] { ... })  // o  .postcondition([&] (auto const& result) { ... })
.except([&] { ... })
```

Para seguir leyendo:

- [Documentacion Oficial de Boost.Contract](https://www.boost.org/doc/libs/release/libs/contract/doc/html/index.html)
- [Referencia de API de Boost.Contract](https://www.boost.org/doc/libs/release/libs/contract/doc/html/reference.html)
- [Codigo Fuente y Ejemplos en GitHub](https://github.com/boostorg/contract)
- [Lista de Correo de Boost (tag: contract)](https://www.boost.org/community/groups.html#main)

La programacion por contratos atrapa bugs en su origen, documenta las suposiciones de tu codigo de forma verificable por maquina, y hace tu software mas confiable. Ahora que tienes las herramientas, ponlas a trabajar.
