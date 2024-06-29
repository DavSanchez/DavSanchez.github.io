---
title: "Entornos de desarrollo reproducibles con Nix"
date: 2024-06-26
---
<!-- LTeX: language=es -->

Tras la introducción a Nix que vimos en el artículo anterior, continuamos con uno de los casos de uso más potentes de esta herramienta.

> [!warning]
> De forma similar al [artículo anterior](./2024-06-16-conociendo-nix.md), a partir de aquí asumiremos que has instalado Nix y activado dos configuraciones opcionales que facilitan mucho la experiencia de usuario con Nix. Si usaste el instalador oficial, puedes activarlas individualmente para cada comando del terminal usando `nix --extra-experimental-features "nix-command flakes"`, o activarlas globalmente añadiendo lo siguiente a `~/.config/nix/nix.conf` o `/etc/nix/nix.conf`:
>
> > ```conf
> > experimental-features = nix-command flakes
> > ```
>
> (Por supuesto, estas configuraciones pueden ser descritas declarativamente con Nix, pero aún no hemos llegado a eso).

Imagina que tienes un proyecto de software en el que estás trabajando, quizá en las etapas iniciales. No tienes interés en empaquetar el producto en Nixpkgs ni nada parecido, porque quizá aún no tienes nada que lanzar aún, pero sí que te gustaría:

- Asegurarte de que tienes todas las herramientas que tienes para desarrollar este proyecto, como por ejemplo:
  - La versión *nightly* de Rust con varios objetivos de compilación diferentes al por defecto, como WASM u otros.
  - Una versión de Go específica, diferente a una que puedas tener instalada globalmente vía otras herramientas como Homebrew.
  - La CLI de AWS/Ansible/Terraform.
  - Alguna utilidad de Python que normalmente instalarías con `pip`, como `jmespath`.
  - Variables de entorno específicas fijadas a un valor particular (`AWS_PROFILE` o alguna configuración de git).
- Asegurarte de que tus *pipelines* de CI/CD tienen estas mismas herramientas, en las mismas versiones, para que tu equipo pueda ejecutar localmente las mismas acciones que la *pipeline* CI/CD y obtener el mismo resultado.
- Forzar un estilo particular de mensajes en los *commits*, o ejecutar las mismas herramientas (formateadores, *linters* o una herramienta propia) con la misma configuración cada vez que un miembro del equipo crea una revisión o *commit*.
- Prevenir que las herramientas que has instalado para tu proyecto entren en conflicto con otros proyectos de tu equipo, sin por ello tener que utilizar contenedores:
  - El Proyecto A usa algúna funcionalidad solo disponible en Go 1.20 y posterior.
  - El Proyecto B no está listo para actualizar más allá de Go 1.19 por ciertos problemas con CGO y `libresolv`.
- Hacer que cada nueva incorporación al equipo esté listo para usar todas estas herramientas y convenciones fácilmente.

¿Cómo podrías lograr todo esto sin recurrir a contenedores, Docker y demás?

Averigüémoslo.

Recuerda que una de las posibles [*salidas*](./2024-06-16-conociendo-nix.md) que pueden definirse en un *Nix flake* son **entornos de desarrollo**. Esto significa que, usando Nix, puedes ir al repositorio de tu proyecto, "convertirlo" en un *flake* (tan solo añadiendo un fichero `flake.nix` en su raíz), y con solo algunas instrucciones en el lenguaje Nix puedes configurar todo lo que mencionamos en la lista. Añade el fichero a tu control de versiones y todos los miembros de tu equipo, las *pipelines* CI/CD, todo el mundo podrá comenzar a usarlo.

No más "pues en mi máquina funciona" o instalar la misma herramienta con `apt-get` en Ubuntu, `Dockerfile` con `FROM ubuntu:latest` en macOS, una GitHub Action en tu CI/CD y golpearte la cabeza contra la pared cuando alguno de los tres entornos falla.

## Nuestro primer *flake*

> [!tip]
> Esta seguramente sea tu primera exposición al lenguaje de programación Nix. No entraremos en demasiado detalle por ahora, confiando en que no será difícil intuir lo que ocurre en los comienzos, pero si necesitas recursos para profundizar, echa un vistazo a [esta web interactiva](https://zaynetro.com/explainix) o al [tutorial oficial](https://nix.dev/tutorials/nix-language.html). ¡O no dudes en preguntarme!

### Estructura básica

Un fichero `flake.nix` contiene lo que en lenguaje Nix se denomina un **conjunto de atributos** (*attribute set* o *attrset* para abreviar), algo muy similar a un objeto JSON:

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs = {}; # Omitimos los contenidos de momento

  outputs = {}; # Esto que lees es un comentario
}
```

Recordemos que Nix hace que nuestras *builds* sean declarativas y reproducibles. Cualquier salida que definamos vendrá determinada por las entradas que definamos, y por nada más. Veamos, entonces, cómo se definen estos *inputs* (entradas) y *outputs* (salidas).

#### Entradas

Para declarar paquetes como dependencias, estos paquetes tienen que venir de alguna parte. ¿De dónde vienen, pues? ¿Quién los define?

La respuesta la tenemos en el artículo anterior, en el que mencioné que Nixpkgs es un repositorio de más de 100000 paquetes instalables. Declaremos entonces Nixpkgs, que también es un *flake*, como entrada:

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs = {
    # Puedes darle un nombre cualquiera a tu input, lo que determina qué es es la URL.
    nixpkgs = {
      # Consulta el artículo anterior para recordar qué significaba esta referencia.
      url = "github:NixOS/nixpkgs/nixos-23.11";
    };
  };

  outputs = {};
}
```

Esta sintaxis es útil si queremos personalizar cada entrada, pero no cubriremos esto en este artículo. Si solo queremos añadir entradas de una forma básica, solo necesitando su URL, podemos hacer esto en su lugar:

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs.nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
  inputs.git-hooks.url = "github:cachix/git-hooks.nix";

  outputs = {};
}
```

O agrupar las entradas de esta otra forma:

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
    git-hooks.url = "github:cachix/git-hooks.nix";
  };

  outputs = {};
}
```

#### Salidas

Aquí es donde se pone interesante. Las salidas definen lo que ofrece nuestro *flake*, y deberían utilizar de alguna forma las entradas que hemos definido anteriormente. Como ya hemos comentado, Nix recibe mucha influencia de la programación funcional, por tanto, **la salida está definida como una función que recibe las entradas como parámetro, y devuelve un nuevo conjunto de atributos**. Las funciones en Nix se definen con la forma `myFunction = arg1: arg2: ... argN: returned_value` (sí, esto es [currying](https://en.wikipedia.org/wiki/Currying)). Por tanto, para nuestra salida:

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
    git-hooks.url = "github:cachix/git-hooks.nix";
  };

  outputs = {
    nixpkgs,
    git-hooks
  }: {}; # Esto último es el conjunto de atributos que devuelve la función. Vacío de momento.
}
```

Supón que hemos definido muchas entradas, pero solo un pequeño subconjunto de ellas es importante y queremos que exista una referencia fácil a ellas en todo momento, mientras que las demás solo se referenciarán ocasionalmente. Podemos des-estructurar el conjunto de atributos de la siguiente forma (que reconocerán los programadores de Haskell o Rust):

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
    git-hooks.url = "github:cachix/git-hooks.nix";
  };

  outputs = inputs @ { # Vinculamos todas las entradas a "inputs".
    nixpkgs,           # Queremos referencia fácil a nixpkgs
    ...                # Omitimos lo demás
  }: {
                       # Ahora podemos usar las referencias aquí.
                       # Ya sea refiriéndose a "nixpkgs" directamente o,
                       # si queremos acceder a git-hooks, usando
                       # "inputs.git-hooks".
  };
}
```

##### Añadiendo dependencias

Ya tenemos la forma básica y queremos tener algo disponible rápidamente. Definamos nuestro entorno de desarrollo, una *shell* personalizada con el atributo [**devShell**](https://nixos.org/manual/nixpkgs/stable/#sec-pkgs-mkShell):

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
    git-hooks.url = "github:cachix/git-hooks.nix";
  };

  outputs = {
    nixpkgs,
    ...
  }: let
    macM1Packages = nixpkgs.legacyPackages.aarch64-darwin;
    linuxAMD64Packages = nixpkgs.legacyPackages.x86_64-linux;
  in {
    devShells.aarch64-darwin.default = macM1Packages.mkShell {
      packages = [
        # Al momento de escribir esto, la versión de
        # Go_1_19 en nixpkgs/nixos-23.11 es 1.19.13
        macM1Packages.go_1_19
      ];
    };

    devShells.x86_64-linux.default = linuxAMD64Packages.mkShell {
      # Nada por ahora
    };
  };
}
```

###### ¿Es obligatorio definir *devShells* y usar referencias a los conjuntos de paquetes para cada sistema por separado?

**La respuesta rápida es *¡No!***.

La respuesta larga es que Nix sirve para construir paquetes, una herramienta o sistema de *builds*, y que por tanto es necesario poder ser consciente del sistema o *target* para el que estamos construyendo un paquete, *shell* o salida en general, pues en general un software no puede ser compilado sin más para todos los posibles sistemas que existen.

Dicho esto, para el caso que nos ocupa en este tutorial, combinaciones de sistema operativo y arquitectura bastante extendidas y de propósito general, sí que podemos asumir que cierto software pueda compilar para ellas sin demasiado problema. Otros desarrolladores han hecho el trabajo duro por nosotros y ofrecen algunas funciones y sistemas para minimizar la repetición, a cambio de un poquito más de código Nix. Una de estas utilidades es [`flake-utils`](https://github.com/numtide/flake-utils). Para utilizarla imagino que ya intuirás lo que necesitamos: Definirla como entrada.

```nix
{
  description = "Nix flake para mi proyecto en Go";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-23.11";
    git-hooks.url = "github:cachix/git-hooks.nix";
    flake-utils.url = "github:numtide/flake-utils"; # Nuevo input!
  };

  outputs = {
    nixpkgs,
    flake-utils,
    ...
  }: flake-utils.lib.eachDefaultSystem (system: let
    pkgs = nixpkgs.legacyPackages.${system};
  in {
    devShells.default = pkgs.mkShell {
      packages = with pkgs; [
        go_1_19
      ];
    };
  });
}
```


