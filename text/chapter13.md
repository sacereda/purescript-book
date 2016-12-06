# Verificación generativa (generative testing)

## Objetivos del capítulo

En este capítulo veremos una aplicación particularmente elegante de las clases de tipos al problema de probar código. En lugar de probar nuestro código diciendo al compilador _cómo_ probar, simplemente decimos _qué_ propiedades debe tener nuestro código. Los casos de prueba se pueden generar de manera aleatoria a partir de esta especificación usando clases de tipos para ocultar el código repetitivo que genera los datos aleatorios. Esto se llama _verificación generativa_ (generative testing) o _verificación basada en propiedades_ (property-based testing), una técnica popularizada en Haskell por la biblioteca [QuickCheck](http://www.haskell.org/haskellwiki/Introduction_to_QuickCheck1).

El paquete `purescript-quickcheck` es un conversión de la biblioteca QuickCheck de Haskell a PureScript, que mayormente preserva los tipos y sintaxis de la biblioteca original. Veremos cómo usar `purescript-quickcheck` para verificar una biblioteca simple, usando Pulp para integrar nuestro conjunto de pruebas en nuestro proceso de desarrollo.

## Preparación del proyecto

El proyecto de este capítulo añade `purescript-quickcheck` como dependencia Bower.

En un proyecto Pulp, el código fuente de las pruebas debe ponerse en el directorio `test`, y el módulo principal del conjunto de pruebas debe llamarse `Test.Main`. El conjunto de pruebas se puede ejecutar usando el comando `pulp test`.

## Escribiendo propiedades

El módulo `Merge` implementa una función simple `merge` que usaremos para demostrar las capacidades de la biblioteca `purescript-quickcheck`.

```haskell
merge :: Array Int -> Array Int -> Array Int
```

`merge` toma dos arrays de enteros ordenados y mezcla sus elementos de manera que el resultado también está ordenado. Por ejemplo:

```text
> import Merge
> merge [1, 3, 5] [2, 4, 6]

[1, 2, 3, 4, 5, 6]
```

En un conjunto de pruebas típico, podemos probar `merge` generando unos pocos casos de prueba pequeños como este a mano, asegurando que los resultados deben ser iguales a los valores apropiados. Sin embargo, todo lo que necesitamos saber sobre la función `merge` se puede resumir en dos propiedades:

- (Orden) Si `xs` e `ys` están ordenados, entonces `merge xs ys` está también ordenado.
- (Subarray) `xs` e `ys` son ambos subarrays de `merge xs ys`, y sus elementos aparecen en el mismo orden.

`purescript-quickcheck` nos permite verificar estas propiedades directamente generando casos de prueba aleatorios. Podemos simplemente dar las propiedades que queremos que tenga nuestro código como funciones:

```haskell
main = do
  quickCheck \xs ys ->
    isSorted $ merge (sort xs) (sort ys)
  quickCheck \xs ys ->
    xs `isSubarrayOf` merge xs ys
```

Aquí, `isSorted` e `isSubarrayOf` se implementan como funciones auxiliares con los siguientes tipos:

```haskell
isSorted :: forall a. Ord a => Array a -> Boolean
isSubarrayOf :: forall a. Eq a => Array a -> Array a -> Boolean
```

Cuando ejecutamos este código, `purescript-quickcheck` intentará falsear las propiedades que afirmamos, generando entradas aleatorias de `xs` e `ys`, y pasándolas a nuestras funciones. Si nuestra función devuelve `false` para cualquier entrada, la propiedad es incorrecta y la biblioteca lanzará un error. Afortunadamente, la biblioteca es incapaz de falsear nuestras propiedades tras generar 100 casos de prueba aleatorios:

```text
$ pulp test

* Build successful. Running tests...

100/100 test(s) passed.
100/100 test(s) passed.

* Tests OK.
```

Si introducimos un fallo deliberadamente en la función `merge` (por ejemplo, cambiando la comprobación menor-que por mayor-que), se lanza una excepción en tiempo de ejecución tras el primer caso de prueba fallido:

```text
Error: Test 1 failed:
Test returned false
```

Como vemos, este mensaje de error no es de mucha ayuda, pero puede mejorarse con un poco de trabajo.

## Mejorando los mensajes de error

Para proporcionar mensajes de error junto a nuestros casos de prueba fallidos, `purescript-quickcheck` suministra el operador `<?>`. Simplemente separa la definición de la propiedad del mensaje de error usando `<?>` como sigue:

```haskell
quickCheck \xs ys ->
  let
    result = merge (sort xs) (sort ys)
  in
    xs `isSubarrayOf` result <?> show xs <> " not a subarray of " <> show result
```

Esta vez, si modificamos el código para introducir un fallo, vemos nuestro mensaje mejorado tras el primer caso de prueba fallido:

```text
Error: Test 6 failed:
[79168] not a subarray of [-752832,686016]
```

Fíjate en cómo las entradas `xs` e `ys` fueron generadas como arrays de enteros seleccionados al azar.

X> ## Ejercicios
X>
X> 1. (Fácil) Escribe una propiedad que asegura que mezclar un array con el array vacío no modifica el array original.
X> 1. (Fácil) Añade un mensaje de error apropiado a la propiedad restante de `merge`.

## Verificando código polimórfico

El módulo `Merge` define una generalización de la función `merge` llamada `mergePoly`, que trabaja no sólo con arrays de números, sino con cualquier array que pertenezca a la clase de tipos `Ord`:

```haskell
mergePoly :: forall a. Ord a => Array a -> Array a -> Array a
```

Si modificamos nuestras pruebas originales para usar `mergePoly` en lugar de `merge`, vemos el siguiente mensaje de error:

```text
No type class instance was found for

  Test.QuickCheck.Arbitrary.Arbitrary t0

The instance head contains unknown type variables.
Consider adding a type annotation.
```

Este mensaje de error indica que el compilador no pude generar casos de prueba aleatorios, porque no sabía qué tipo de elementos queríamos que tuviesen nuestros arrays. En estos casos, podemos usar una función auxiliar para forzar al compilador a inferir un tipo particular. Por ejemplo, si definimos una función `ints` como sinónimo de la función identidad:

```haskell
ints :: Array Int -> Array Int
ints = id
```

podemos modificar nuestras pruebas de manera que el compilador infiera el tipo `Array Int` para nuestros dos argumentos de tipo array:

```haskell
quickCheck \xs ys ->
  isSorted $ ints $ mergePoly (sort xs) (sort ys)
quickCheck \xs ys ->
  ints xs `isSubarrayOf` mergePoly xs ys
```

Aquí, `xs` e `ys` tienen ambos el tipo `Array Int`, ya que hemos usado la función `ints` para desambiguar los tipos desconocidos.

X> ## Ejercicios
X>
X> 1. (Fácil) Escribe una función `bools` que fuerza que los tipos de `xs` e `ys` sean `Array Boolean`, y añade propiedades adicionales que comprueban `mergePoly` con ese tipo.
X> 1. (Medio) Elige una función pura de las bibliotecas estándar (por ejemplo, del paquete `purescript-arrays`) y escribe una propiedad QuickCheck para ella, incluyendo un mensaje de error apropiado. Tu propiedad debe usar una función auxiliar para fijar cualquier tipo polimórfico a `Int` o `Boolean`.

## Generando datos arbitrarios

Ahora veremos cómo la biblioteca `purescript-quickcheck` es capaz de generar casos de prueba para nuestras propiedades de manera aleatoria.

Los tipos cuyos valores pueden ser generados aleatoriamente están capturados por la clase de tipos `Arbitrary`:

```haskell
class Arbitrary t where
  arbitrary :: Gen t
```

El constructor de tipo `Gen` representa los efectos secundarios de _generación de datos aleatorios determinista_. Usa un generador de números pseudo-aleatorios para generar argumentos de función deterministas a partir de un valor semilla. El módulo `Test.QuickCheck.Gen` define varios combinadores útiles para construir generadores.

`Gen` es también una mónada y un funtor aplicativo, de manera que tenemos la habitual colección de combinadores a nuestra disposición para crear nuevas instancias de la clase de tipos `Arbitrary`.

Por ejemplo, podemos usar la instancia `Arbitrary` para el tipo `Int` proporcionada en la biblioteca `purescript-quickcheck` para crear una distribución en los 256 valores posibles de un byte, usando la instancia `Functor` de `Gen` para mapear una función de enteros a bytes sobre valores enteros arbitrarios:

```haskell
newtype Byte = Byte Int

instance arbitraryByte :: Arbitrary Byte where
  arbitrary = map intToByte arbitrary
    where
    intToByte n | n >= 0 = Byte (n `mod` 256)
                | otherwise = intToByte (-n)
```

Aquí definimos un tipo `Byte` de valores enteros entre 0 y 255. La instancia `Arbitrary` usa la función `map` para elevar la función `intToByte` sobre la acción `arbitrary`. El tipo de la acción `arbitrary` interna se infiere como `Gen Int`.

Podemos también usar esta idea para mejorar nuestra prueba de orden de `merge`:

```haskell
quickCheck \xs ys ->
  isSorted $ numbers $ mergePoly (sort xs) (sort ys)
```

En esta prueba, hemos generado los arrays arbitrarios `xs` e `ys`, pero hemos tenido que ordenarlos porque `merge` espera entradas ordenadas. Por otra parte, podríamos crear un newtype representando arrays ordenados y escribir una instancia `Arbitrary` que genera datos ordenados:

```haskell
newtype Sorted a = Sorted (Array a)

sorted :: forall a. Sorted a -> Array a
sorted (Sorted xs) = xs

instance arbSorted :: (Arbitrary a, Ord a) => Arbitrary (Sorted a) where
  arbitrary = map (Sorted <<< sort) arbitrary
```

Con este constructor de tipo, podemos modificar nuestras pruebas como sigue:

```haskell
quickCheck \xs ys ->
  isSorted $ ints $ mergePoly (sorted xs) (sorted ys)
```

Esto parece un cambio pequeño, pero los tipos de `xs` e `ys` han cambiado a `Sorted Int` en lugar de simplemente `Array Int`. Esto comunica nuestra _intención_ de manera más clara; la función `mergePoly` toma entradas ordenadas. Idealmente, el tipo de la función `mergePoly` se actualizaría para usar el constructor de tipo `Sorted`.

Como ejemplo más interesante, el módulo `Tree` define un tipo de árboles binarios ordenados con valores en las ramas:

```haskell
data Tree a
  = Leaf
  | Branch (Tree a) a (Tree a)
```

El módulo `Tree` define la siguiente API:

```haskell
insert    :: forall a. Ord a => a -> Tree a -> Tree a
member    :: forall a. Ord a => a -> Tree a -> Boolean
fromArray :: forall a. Ord a => Array a -> Tree a
toArray   :: forall a. Tree a -> Array a
```

La función `insert` se usa para insertar un nuevo elemento en un árbol ordenado, y la función `member` se puede usar para preguntar al árbol por un valor particular. Por ejemplo:

```text
> import Tree

> member 2 $ insert 1 $ insert 2 Leaf
true

> member 1 Leaf
false
```

Las funciones `toArray` y `fromArray` se pueden usar para convertir árboles ordenados en arrays y viceversa. Podemos usar `fromArray` para escribir una instancia `Arbitrary` para árboles:

```haskell
instance arbTree :: (Arbitrary a, Ord a) => Arbitrary (Tree a) where
  arbitrary = map fromArray arbitrary
```

Podemos ahora usar `Tree a` como el tipo de un argumento a nuestras propiedades de prueba, siempre que haya una instancia de `Arbitrary` disponible para el tipo `a`. Por ejemplo, podemos verificar que `member` siempre devuelve `true` para un cierto valor tras insertarlo:

```haskell
quickCheck \t a ->
  member a $ insert a $ treeOfInt t
```

Aquí, el argumento `t` es un árbol generado aleatoriamente de tipo `Tree Int`, donde el tipo del argumento está desambiguado por la función identidad `treeOfInt`.

X> ## Ejercicios
X>
X> 1. (Medio) Crea un newtype para `String` con una instancia asociada de `Arbitrary` que genera colecciones de carateres aleatorios en el rango `a-z`. _Pista_: usa las funciones `elements` y `arrayOf` del módulo `Test.QuickCheck.Gen`.
X> 1. (Difícil) Escribe una propiedad que asegura que un valor insertado en un árbol es todavía miembro de ese árbol tras un cierto número arbitrario de inserciones.

## Probando funciones de orden mayor

El módulo `Merge` define otra generalización de la función `merge`; la función `mergeWith` toma como argumento adicional una función que se usa para determinar el orden en que los elementos deben mezclarse. Esto es, `mergeWith` es una función de orden mayor.

Por ejemplo, podemos pasar la función `length` como primer argumento para mezclar dos arrays que están ordenados por la longitud de sus elementos. El resultado debe estar también ordenado:

```haskell
> import Data.String

> mergeWith length
    ["", "ab", "abcd"]
    ["x", "xyz"]

["","x","ab","xyz","abcd"]
```

¿Cómo podemos verificar dicha función? Idealmente, querríamos generar valores para los tres argumentos, incluyendo el primer argumento que es una función.

Hay una segunda clase de tipos que nos permite crear funciones generadas aleatoriamente. Se llama `Coarbitrary` y está definida como sigue:

```haskell
class Coarbitrary t where
  coarbitrary :: forall r. t -> Gen r -> Gen r
```

La función `coarbitrary` toma como argumento una función de tipo `t`, y un generador aleatorio para un resultado de función de tipo `r`, y usa la función argumento para _perturbar_ el generador aleatorio. Esto es, para obtener el resultado aplica la función proporcionada para modificar la salida del generador aleatorio.

Además, hay una instancia de clase de tipos que nos da funciones `Arbitrary` si el dominio de la función es `Coarbitrary` y el codominio de la función es `Arbitrary`:

```haskell
instance arbFunction :: (Coarbitrary a, Arbitrary b) => Arbitrary (a -> b)
```

En la práctica esto significa que podemos escribir propiedades que toman funciones como argumentos. En el caso de la función `mergeWith` podemos generar el primer argumento aleatoriamente, modificando nuestras pruebas para que tengan en cuenta el nuevo argumento.

En el caso de la propiedad de orden, no podemos garantizar que el resultado estará ordenado (no tenemos necesariamente una instancia de `Ord`) pero podemos esperar que el resultado esté ordenado con respecto a la función `f` que pasamos como argumento. Además, necesitamos que los arrays de entrada estén ordenados con respecto a `f`, de manera que usamos la función `sortBy` para ordenar `xs` e `ys` basándonos en la comparación tras aplicar la función `f`:

```haskell
quickCheck \xs ys f ->
  isSorted $
    map f $
      mergeWith (intToBool f)
                (sortBy (compare `on` f) xs)
                (sortBy (compare `on` f) ys)
```

Aquí usamos una función `intToBool` para desambiguar el tipo de la función `f`:

```haskell
intToBool :: (Int -> Boolean) -> Int -> Boolean
intToBool = id
```

En el caso de la propiedad de subarray, simplemente tenemos que cambiar el nombre de la función a `mergeWith`; seguimos esperando que nuestros arrays de entrada sean subarrays del resultado:

```haskell
quickCheck \xs ys f ->
  xs `isSubarrayOf` mergeWith (numberToBool f) xs ys
```

Además de ser `Arbitrary`, las funciones son también `Coarbitrary`:

```haskell
instance coarbFunction :: (Arbitrary a, Coarbitrary b) => Coarbitrary (a -> b)
```

Esto significa que no estamos limitados únicamente a valores y funciones; podemos generar aleatoriamente _funciones de orden mayor_, o funciones cuyos argumentos son funciones de orden mayor, etc.

## Escribiendo instancias de Coarbitrary

Al igual que podemos escribir instancias de `Arbitrary` para nuestros tipos de datos usando las instancias `Monad` y `Applicative` de `Gen`, podemos escribir nuestras propias instancias de `Coarbitrary` también. Esto nos permite usar nuestros propios tipos de datos como dominio de funciones generadas aleatoriamente.

Escribamos una instancia de `Coarbitrary` para nuestro tipo `Tree`. Necesitaremos una instancia de `Coarbitrary` para el tipo de los elementos almacenados en las ramas:

```haskell
instance coarbTree :: Coarbitrary a => Coarbitrary (Tree a) where
```

Tenemos que escribir una función que perturba un generador aleatorio dado un valor de tipo `Tree a`. Si el valor de entrada es una `Leaf`, simplemente devolvemos el generador sin cambios:

```haskell
  coarbitrary Leaf = id
```

Si el árbol es una `Branch`, perturbaremos el generador usando el subárbol izquierdo, el valor y el subárbol derecho, usando composición de funciones para crear nuestra función perturbadora:

```haskell
  coarbitrary (Branch l a r) =
    coarbitrary l <<<
    coarbitrary a <<<
    coarbitrary r
```

Ahora somos libres de escribir propiedades cuyos argumentos incluyan funciones que toman árboles como argumentos. Por ejemplo, el módulo `Tree` define una función `anywhere` que comprueba si un predicado se mantiene en cualquier subárbol de su argumento:

```haskell
anywhere :: forall a. (Tree a -> Boolean) -> Tree a -> Boolean
```

Ahora somos capaces de generar la función predicado aleatoriamente. Por ejemplo, esperamos que la función `anywhere` _respete la disyunción_:

```haskell
quickCheck \f g t ->
  anywhere (\s -> f s || g s) t ==
    anywhere f (treeOfInt t) || anywhere g t
```

Aquí, la función `treeOfInt` se usa para fijar el tipo de los valores contenidos en el árbol al tipo `Int`:

```haskell
treeOfInt :: Tree Int -> Tree Int
treeOfInt = id
```

## Verificando sin efectos secundarios

Para propósitos de verificación, normalmente incluimos llamadas a la función `quickCheck` en la acción `main` de nuestro conjunto de pruebas. Sin embargo, hay una variante de la función `quickCheck` llamada `quickCheckPure` que no usa efectos secundarios. En su lugar, es una función pura que toma una semilla aleatoria como entrada y devuelve un array de resultados de la prueba.

Podemos probar `quickCheckPure` usando PSCi. Aquí probamos que la operación `merge` es asociativa:

```text
> import Prelude
> import Merge
> import Test.QuickCheck
> import Test.QuickCheck.LCG (mkSeed)

> :paste
… quickCheckPure (mkSeed 12345) 10 \xs ys zs ->
…   ((xs `merge` ys) `merge` zs) ==
…     (xs `merge` (ys `merge` zs))
… ^D

Success : Success : ...
```

`quickCheckPure` toma tres argumentos: la semilla del generador aleatorio, el número de casos de prueba a generar, y la propiedad a verificar. Si todas las pruebas pasan, debes ver un array de constructores de dato `Success` impresos en la consola.

`quickCheckPure` puede ser útil en otras situaciones, como generar datos de entrada aleatorios para pruebas de rendimiento, o para generar datos de formulario de ejemplo para aplicaciones web.

X> ## Ejercicios
X>
X> 1. (Fácil) Escribe instancias de `Coarbitrary` para los constructores de tipo `Byte` y `Sorted`.
X> 1. (Medio) Escribe una propiedad (de orden mayor) que asegura la asociatividad de la función `mergeWith f` para cualquier función `f`. Prueba tu propiedad en PSCi usando `quickCheckPure`.
X> 1. (Medio) Escribe instancias `Arbitrary` y `Coarbitrary` para el siguiente tipo de datos:
X>
X>     ```haskell
X>     data OneTwoThree a = One a | Two a a | Three a a a
X>     ```
X>
X>     _Pista_: Usa la función `oneOf` definida en `Test.QuickCheck.Gen` para definir tu instancia `Arbitrary`.
X> 1. (Medio) Usa la función `all` para simplificar el resultado de la función `quickCheckPure`. Tu función debe devolver `true` si todas las pruebas pasan y `false` en caso contrario. Intenta usar el monoide `First` definido en `purescript-monoids` con la función `foldMap` para preservar el primer error en caso de fallo.

## Conclusión

En este capítulo hemos conocido el paquete `purescript-quickcheck`, que se puede usar para escribir pruebas de manera declarativa usando el paradigma de _verificación generativa_. En particular:

- Vimos cómo automatizar las pruebas QuickCheck usando `pulp test`.
- Vimos cómo escribir propiedades como funciones, y cómo usar el operador `<?>` para mejorar los mensajes de error.
- Vimos cómo las clases de tipos `Arbitrary` y `Coarbitrary` permiten la generación de código de prueba repetitivo, y cómo nos permiten probar propiedades de orden mayor.
- Vimos cómo implementar instancias `Arbitrary` y `Coarbitrary` a medida para nuestros propios tipos de datos.
