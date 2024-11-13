---
title: "\"Mapeando\" sobre estructuras de datos"
short-title: "Mapeando sobre estructuras"
date: 2024-11-13
---
<!-- LTeX: language=es -->

Hace unos días tuve una conversación con mi equipo sobre el uso de la función `map` en Rust. Como
puede resultar didáctico para aquellos que quieran saber más sobre Programación Funcional he
decidido reproducirlo por aquí.

Ciertamente, la concepción más popular de `map` (y también las otras dos entidades que forman la "triada" de
 funciones de orden superior, `filter` y `reduce` o `fold`) viene de su uso en iteradores como listas,
 *arrays*, etc. Esto no solo ocurre en Rust sino también en otros lenguajes como JavaScript.

Sin embargo, en algunos casos puede ser útil abstraer nuestra concepción de `map` un poco más para
razonar mejor cómo se comporta en otros contextos. Por ejemplo, ¿qué hay de los tipos `Option` y
`Result` en Rust? ¿por qué tienen una implementación de `map`?

Una manera útil de comprender la función `map` que también abarca su disponibilidad como método en
`Option` y `Result` es considerarla como una función implementable para tipos de datos similares a
*contenedores*. De esta forma, `map` opera aplicando la función pasada como parámetro al
*valor del tipo contenido*, sustituyéndolo por el valor de salida de la aplicación,
dejando el *contenedor* intacto.

![Representación de el uso de la función map usando contenedores como analogía](functor-diagram.png)

Así, un valor de tipo `Option<T>` se transforma en un valor de tipo `Option<U>` si llamamos a la
función `map` pasándole una función que implemente `Fn(T) -> U`[^fnonce] que transformará `T` en
`U`, dejando el *envoltorio* `Option<_>` intacto.

`Result` es un poco más interesante. Es un tipo de dato con dos variantes, igual que `Option`, pero
a diferencia de este, `Result` incluye un tipo en cada variante. La implementación de `map` para
`Result` solo actúa en una de sus posibles variantes (`Ok(_)`) ignorando la otra.

> [!tip] ¿Por qué `Ok` y no `Err`?
> Dado que `Result` se usa habitualmente en situaciones en las que propagamos los errores
> en la variante `Err(_)` usando `?`, nos importa la primera ocurrencia de la variante `Err(_)`.
>
> La función `map`, pues, no actúa sobre la variante `Err(_)`[^maperr-maporelse].

Entonces, partiendo de un `Result<T, E>` obtenemos un `Result<U, E>` pasando un
`Fn(T) -> U`[^fnonce]. De nuevo, el valor de tipo `T` se transforma en un valor de tipo `U` pero el
*envoltorio* `Result<_, E>` permanece inalterado.

¡Pero podríamos tener `map`s para muchos otros tipos! ¿Por qué no una tupla con dos (o más)
elementos, como `(T, U)`? ¿Tiene sentido usar `map` aquí? ¿Por qué no un `struct` arbitrario que
hayamos definido?

¿Tendría sentido la existencia de un *trait* llamado `Mappable` o algo parecido?
*¡La respuesta es sí!*

![Mind blown!](./mind_blown.gif)

En lenguajes de programación funcional como Haskell, el comportamiento de `map` está definido en
lo equivalente a un *trait* de Rust (en dicho lenguaje, los *traits* se llaman *typeclasses*).
Este *trait* se llama `Functor`. La documentación sobre `Functor`, aunque algo matemática, parece
estar de acuerdo con nuestro razonamiento anterior:

> Un tipo f es un Functor si proporciona una función fmap que, dados dos tipos arbitrarios a y b
> permite aplicar cualquier función (a -> b) transformando f a en un f b, preservando la estructura
> de f.

In Haskell :haskell-intensifies2:, the behavior of map is indeed defined under a trait (traits are called typeclasses there). This trait is called Functor. The documentation, though a bit "mathy", agrees with our reasoning above:
A type f is a Functor if it provides a function fmap which, given any types a and b lets you apply any function from (a -> b) to turn an f a into an f b, preserving the structure of f.
The function is called fmap instead of map for historical reasons (map was of course initially defined for lists :wink: and then generalized) and ease of use (newcomers to Haskell will begin using map on lists and then abstract out to fmap).
8:35
There's Functor implementations for many types in the ecosystem, such as for:
Lists
Sets
Maps
Maybe (Haskell's Option)
Either (Haskell's Result)
Tuples of various elements
Functions themselves (!!)
... and many others, usually in icreasing level of abstraction xD
(edited)
8:36
The other typical FP functions, filter and fold/reduce, have their own traits: Filterable and Foldable.
8:36
Why is this Functor (or Filterable or Foldable) trait not available in Rust, then? Well, because Rust cannot represent traits for types like this with its current features (a good exercise would be trying it, perhaps via Generic Associated Types).
For being able to do that, Rust would need to support something called higher-kinded types.
8:36
But don't let that get in the way of reasoning about what a map function can represent and what could make a type mappable!
The end xD

[^fnonce]:
    Realmente en Rust es `FnOnce(T) -> U`, pero esto es otra discusión.

[^maperr-maporelse]:
    Aunque también existe `map_err` para actuar sobre la variante `Err(_)` o `map_or_else` para
    actuar sobre las dos variantes.
