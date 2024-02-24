---
title: Etiquetas especiales
---

## {@html ...}

```svelte
<!--- copy: false --->
{@html expression}
```

En una expresión de texto, caracteres como `<` y `>` se escapan; sin embargo, con expresiones HTML, no lo son.

La expresión debe ser HTML válida por sí sola — `{@html "<div>"}contenido{@html "</div>"}` no funcionará, porque `</div>` no es HTML válido. Tampoco compilará código Svelte.

> Svelte no sanitiza las expresiones antes de inyectar HTML. Si los datos provienen de una fuente no confiable, debes sanitizarlos, o estarás exponiendo a tus usuarios a una vulnerabilidad XSS.

```svelte
<div class="blog-post">
	<h1>{post.title}</h1>
	{@html post.content}
</div>
```

## {@debug ...}

```svelte
<!--- copy: false --->
{@debug}
```

```svelte
<!--- copy: false --->
{@debug var1, var2, ..., varN}
```

La etiqueta `{@debug ...}` ofrece una alternativa a `console.log(...)`. Registra los valores de variables específicas cada vez que cambian y pausa la ejecución del código si tienes las herramientas de desarrollo abiertas.

```svelte
<script>
	let user = {
		firstname: 'Ada',
		lastname: 'Lovelace'
	};
</script>

{@debug user}

<h1>Hola {user.firstname}!</h1>
```

`{@debug ...}` acepta una lista de nombres de variables separados por comas (no expresiones arbitrarias).

```svelte
<!-- Compila -->
{@debug user}
{@debug user1, user2, user3}

<!-- NO compila -->
{@debug user.firstname}
{@debug myArray[0]}
{@debug !isReady}
{@debug typeof user === 'object'}
```

La etiqueta `{@debug}` sin argumentos insertará una declaración `debugger` que se activa cuando _any_ estado cambia, en lugar de las variables especificadas.

## {@const ...}

```svelte
<!--- copy: false --->
{@const assignment}
```

La etiqueta `{@const ...}` define una constante local.

```svelte
<script>
	export let boxes;
</script>

{#each boxes as box}
	{@const area = box.width * box.height}
	{box.width} * {box.height} = {area}
{/each}
```

`{@const}` solo está permitido como hijo directo de `{#if}`, `{:else if}`, `{:else}`, `{#each}`, `{:then}`, `{:catch}`, `<Component />` o `<svelte:fragment />`.
