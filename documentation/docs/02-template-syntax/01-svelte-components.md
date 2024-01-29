---
title: Componentes de Svelte
---

Los componentes son los bloques de construcción de las aplicaciones Svelte. Se escriben como archivos `.svelte`, utilizando un superconjunto de HTML.

Las tres secciones — script, estilos y marcado — son opcionales.

```svelte
<script>
	// La lógica va aquí
</script>

<!-- El marcado (cero o más items) va aquí -->

<style>
	/* Los estilos van aquí. */
</style>
```

## &lt;script&gt;

Un bloque `<script>` lleva el código JavaScript que se ejecuta cuando se crea una instancia del componente. Las variables declaradas (o importadas) en la parte superior son "visibles" desde el marcado del componente. Existen cuatro reglas adicionales:

### 1. `export` crea un componente con props

Svelte usa la palabra clave `export` para marcar una declaración de variable como _propiedad_ o _prop_, lo que significa que se convierte en accesible para los consumidores del componente (véase la sección sobre [atributos y props](/docs/basic-markup#attributes-and-props) para más información).

```svelte
<script>
	export let foo;

	// Los valores pasados como props
	// están disponibles inmediatamente
	console.log({ foo });
</script>
```

Se puede especificar un valor inicial por defecto para una prop. Se utilizará si el consumidor del componente no especifica el prop en el componente (o si su valor inicial es `undefined`) al instanciar el componente. Tenga en cuenta que si los valores de las props se actualizan posteriormente, entonces cualquier prop cuyo valor no se especifique se establecerá a `undefined` (en lugar de su valor inicial).

En modo de desarrollo, se mostrará una advertencia si no se proporciona un valor inicial por defecto y si el consumidor no especifica un valor (vea las [opciones de compilación](/docs/svelte-compiler#compile)). Para silenciar esta advertencia, asegúrese de que se especifica un valor inicial por defecto, incluso si es `undefined`.

```svelte
<script>
	export let bar = 'valor inicial por defecto, opcional';
	export let baz = undefined;
</script>
```

Si exporta una `const`, `class` o `function`, es de sólo lectura desde fuera del componente. Sin embargo, las funciones son valores prop válidos, como se muestra a continuación.

```svelte
<!--- file: App.svelte --->
<script>
	// estos son de sólo lectura
	export const thisIs = 'readonly';

	/** @param {string} name */
	export function greet(name) {
		alert(`Hola, ${name}!`);
	}

	// esto es una prop
	export let format = (n) => n.toFixed(2);
</script>
```

Se puede acceder a las props de sólo lectura como propiedades del elemento, que estén viculadas al componente mediante [la sintaxis `bind:this`](/docs/component-directives#bind-this).

Puede utilizar palabras reservadas como nombres de props.

```svelte
<!--- file: App.svelte --->
<script>
	/** @type {string} */
	let className;

	// crea una propiedad `class`, aunque
	// sea una palabra reservada
	export { className as class };
</script>
```

### 2. Las asignaciones son 'reactivas'

Para cambiar el estado de un componente y volver a renderizarlo, basta con asignarlo a una variable declarada localmente.

Las expresiones de actualización (`count += 1`) y las asignaciones de propiedades (`obj.x = y`) tienen el mismo efecto.

```svelte
<script>
	let count = 0;

	function handleClick() {
		// La llamada a esta función provocará una
		// actualización si el marcado hace referencia a `count`
		count = count + 1;
	}
</script>
```

Dado que la reactividad de Svelte se basa en asignaciones, el uso de métodos de array como `.push()` o `.splice()` no activará automáticamente las actualizaciones. Se requiere de una asignación posterior para activar la actualización. Esto y más detalles se pueden encontrar en el [tutorial](https://learn.svelte.dev/tutorial/updating-arrays-and-objects).

```svelte
<script>
	let arr = [0, 1];

	function handleClick() {
		// esta llamada al método no provoca una actualización
		arr.push(2);
		// esta asignación provocará una actualización
		// si el marcado hace referencia a `arr`
		arr = arr;
	}
</script>
```

Los bloques `<script>` de Svelte se ejecutan sólo cuando se crea el componente, por lo que las asignaciones dentro de un bloque `<script>` no se vuelven a ejecutar automáticamente cuando se actualiza una prop. Si desea realizar un seguimiento de los cambios en un componente, consulte el siguiente ejemplo de la siguiente sección.

```svelte
<script>
	export let person;
	// esto sólo establecerá `name` en la creación del componente
	// no se actualizará cuando pase a `person`
	let { name } = person;
</script>
```

### 3. `$:` marks a statement as reactive

Any top-level statement (i.e. not inside a block or a function) can be made reactive by prefixing it with the `$:` [JS label syntax](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label). Reactive statements run after other script code and before the component markup is rendered, whenever the values that they depend on have changed.

```svelte
<script>
	export let title;
	export let person;

	// this will update `document.title` whenever
	// the `title` prop changes
	$: document.title = title;

	$: {
		console.log(`multiple statements can be combined`);
		console.log(`the current title is ${title}`);
	}

	// this will update `name` when 'person' changes
	$: ({ name } = person);

	// don't do this. it will run before the previous line
	let name2 = name;
</script>
```

Only values which directly appear within the `$:` block will become dependencies of the reactive statement. For example, in the code below `total` will only update when `x` changes, but not `y`.

```svelte
<!--- file: App.svelte --->
<script>
	let x = 0;
	let y = 0;

	/** @param {number} value */
	function yPlusAValue(value) {
		return value + y;
	}

	$: total = yPlusAValue(x);
</script>

Total: {total}
<button on:click={() => x++}> Increment X </button>

<button on:click={() => y++}> Increment Y </button>
```

It is important to note that the reactive blocks are ordered via simple static analysis at compile time, and all the compiler looks at are the variables that are assigned to and used within the block itself, not in any functions called by them. This means that `yDependent` will not be updated when `x` is updated in the following example:

```svelte
<script>
	let x = 0;
	let y = 0;

	/** @param {number} value */
	function setY(value) {
		y = value;
	}

	$: yDependent = y;
	$: setY(x);
</script>
```

Moving the line `$: yDependent = y` below `$: setY(x)` will cause `yDependent` to be updated when `x` is updated.

If a statement consists entirely of an assignment to an undeclared variable, Svelte will inject a `let` declaration on your behalf.

```svelte
<!--- file: App.svelte --->
<script>
	/** @type {number} */
	export let num;

	// we don't need to declare `squared` and `cubed`
	// — Svelte does it for us
	$: squared = num * num;
	$: cubed = squared * num;
</script>
```

### 4. Prefix stores with `$` to access their values

A _store_ is an object that allows reactive access to a value via a simple _store contract_. The [`svelte/store` module](/docs/svelte-store) contains minimal store implementations which fulfil this contract.

Any time you have a reference to a store, you can access its value inside a component by prefixing it with the `$` character. This causes Svelte to declare the prefixed variable, subscribe to the store at component initialization and unsubscribe when appropriate.

Assignments to `$`-prefixed variables require that the variable be a writable store, and will result in a call to the store's `.set` method.

Note that the store must be declared at the top level of the component — not inside an `if` block or a function, for example.

Local variables (that do not represent store values) must _not_ have a `$` prefix.

```svelte
<script>
	import { writable } from 'svelte/store';

	const count = writable(0);
	console.log($count); // logs 0

	count.set(1);
	console.log($count); // logs 1

	$count = 2;
	console.log($count); // logs 2
</script>
```

#### Store contract

```ts
// @noErrors
store = { subscribe: (subscription: (value: any) => void) => (() => void), set?: (value: any) => void }
```

You can create your own stores without relying on [`svelte/store`](/docs/svelte-store), by implementing the _store contract_:

1. A store must contain a `.subscribe` method, which must accept as its argument a subscription function. This subscription function must be immediately and synchronously called with the store's current value upon calling `.subscribe`. All of a store's active subscription functions must later be synchronously called whenever the store's value changes.
2. The `.subscribe` method must return an unsubscribe function. Calling an unsubscribe function must stop its subscription, and its corresponding subscription function must not be called again by the store.
3. A store may _optionally_ contain a `.set` method, which must accept as its argument a new value for the store, and which synchronously calls all of the store's active subscription functions. Such a store is called a _writable store_.

For interoperability with RxJS Observables, the `.subscribe` method is also allowed to return an object with an `.unsubscribe` method, rather than return the unsubscription function directly. Note however that unless `.subscribe` synchronously calls the subscription (which is not required by the Observable spec), Svelte will see the value of the store as `undefined` until it does.

## &lt;script context="module"&gt;

A `<script>` tag with a `context="module"` attribute runs once when the module first evaluates, rather than for each component instance. Values declared in this block are accessible from a regular `<script>` (and the component markup) but not vice versa.

You can `export` bindings from this block, and they will become exports of the compiled module.

You cannot `export default`, since the default export is the component itself.

> Variables defined in `module` scripts are not reactive — reassigning them will not trigger a rerender even though the variable itself will update. For values shared between multiple components, consider using a [store](/docs/svelte-store).

```svelte
<script context="module">
	let totalComponents = 0;

	// the export keyword allows this function to imported with e.g.
	// `import Example, { alertTotal } from './Example.svelte'`
	export function alertTotal() {
		alert(totalComponents);
	}
</script>

<script>
	totalComponents += 1;
	console.log(`total number of times this component has been created: ${totalComponents}`);
</script>
```

## &lt;style&gt;

CSS inside a `<style>` block will be scoped to that component.

This works by adding a class to affected elements, which is based on a hash of the component styles (e.g. `svelte-123xyz`).

```svelte
<style>
	p {
		/* this will only affect <p> elements in this component */
		color: burlywood;
	}
</style>
```

To apply styles to a selector globally, use the `:global(...)` modifier.

```svelte
<style>
	:global(body) {
		/* this will apply to <body> */
		margin: 0;
	}

	div :global(strong) {
		/* this will apply to all <strong> elements, in any
			 component, that are inside <div> elements belonging
			 to this component */
		color: goldenrod;
	}

	p:global(.red) {
		/* this will apply to all <p> elements belonging to this
			 component with a class of red, even if class="red" does
			 not initially appear in the markup, and is instead
			 added at runtime. This is useful when the class
			 of the element is dynamically applied, for instance
			 when updating the element's classList property directly. */
	}
</style>
```

If you want to make @keyframes that are accessible globally, you need to prepend your keyframe names with `-global-`.

The `-global-` part will be removed when compiled, and the keyframe then be referenced using just `my-animation-name` elsewhere in your code.

```svelte
<style>
	@keyframes -global-my-animation-name {
		/* code goes here */
	}
</style>
```

There should only be 1 top-level `<style>` tag per component.

However, it is possible to have `<style>` tag nested inside other elements or logic blocks.

In that case, the `<style>` tag will be inserted as-is into the DOM, no scoping or processing will be done on the `<style>` tag.

```svelte
<div>
	<style>
		/* this style tag will be inserted as-is */
		div {
			/* this will apply to all `<div>` elements in the DOM */
			color: red;
		}
	</style>
</div>
```
