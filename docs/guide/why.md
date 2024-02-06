# ¿Por qué Vite?

## Problemas

Antes de los que módulos ES estuvieran disponibles para navegadores web, los desarrolladores no tenían un mecanismo nativo para crear código Javascript en forma modular. Este es el por qué estamos familizariados con el concepto de "empaquetado": el uso de herramientas que analizan, procesan y concatenan nuestros módulos en archivos que puedan ser ejecutados en el navegador.

Con el tiempo hemos visto herramientas como [webpack](https://webpack.js.org/), [Rollup](https://rollupjs.org) y [Parcel](https://parceljs.org/), las cuales han mejorado considerablemente la experiencia de desarrollo de los desarrolladores frontend.

Sin embargo, a medida que empezamos a crear aplicaciones cada vez más ambiciosas, la cantidad de código JavaScript con la que estamos trabajando tambien irá incrementandose dramáticamente. No es raro que los proyectos a gran escala contengan miles de módulos. En este punto empezamos a encontrarnos con un cuello de botella en el rendimiento de las herramientas basadas en JavaScript: a menudo puede llevar a una espera excesivamente larga (¡a veces hasta de minutos!) poner en marcha un servidor de desarrollo, e incluso con Hot Module Replacement (HMR), las ediciones de archivos pueden tardar un par de segundos en reflejarse en el navegador. El ciclo lento de retroalimentación puede afectar en gran medida la productividad y felicidad de los desarrolladores.

Vite viene a corregir estos problemas aprovechando los nuevos avances en el ecosistema: la disponibilidad de módulos ES nativos en el navegador, y el surgimiento de herramientas JavaScript escritas en lenguajes de compilación a nativo.

### Inicio lento del servidor

Cuando se inicia desde cero el servidor de desarrollo, la configuración de compilación basada en empaquetadores tiene que analizar y compilar de forma estricta toda la aplicación antes de que pueda ser servida.

Vite mejora el tiempo de inicio del servidor de desarrollo dividiendo primero los módulos de una aplicación en dos categorías: **dependencias** y **código fuente**.

- Las **dependencias** son en su mayoría código JavaScript plano que no cambia con frecuencia durante el desarrollo. Algunas dependencias grandes (por ejemplo, librerías de componentes con cientos de módulos) también son bastante complejas de procesar. Las dependencias también pueden estar disponibles en varios formatos de módulos (por ejemplo, ESM o CommonJS).

  Vite [preempaqueta dependencias](./dep-pre-bundling) usando [esbuild](https://esbuild.github.io/). esbuild está escrito en Go y preempaqueta dependencias de 10 a 100 veces más rápido que los empaquetadores basados en JavaScript.

- **El código fuente** a menudo contiene código JavaScript no plano que necesita transformación (por ejemplo, JSX, CSS o componentes Vue/Svelte) y que se editará con mucha frecuencia. Además, no es necesario cargar todo el código fuente al mismo tiempo (por ejemplo, con división de código basado en rutas).

Vite sirve código fuente sobre [ESM nativo](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules). Básicamente, esto permitirá que el navegador se haga cargo de parte del trabajo de un empaquetador: Vite solo necesita transformar y servir código fuente a petición, según como lo solicite el navegador. El código detrás de las importaciones dinámicas condicionales sólo es procesado si realmente es usado en la pantalla actual.

<script setup>
import bundlerSvg from '../images/bundler.svg?raw'
import esmSvg from '../images/esm.svg?raw'
</script>
<svg-image :svg="bundlerSvg" />
<svg-image :svg="esmSvg" />

### Actualizaciones lentas

Cuando se edita un archivo con una configuración de compilación basada en empaquetadores, es ineficiente recompilar todo el paquete por una razón obvia: la velocidad de actualización se degradará linealmente con el tamaño de la aplicación.

En algunos empaquetadores, el servidor de desarrollo ejecuta el empaquetado en memoria, por lo que solo necesita invalidar parte de su gráfico de módulo cuando cambia un archivo, pero aún así necesita recompilar todo el paquete y recargar la página web. Recompilar el paquete puede ser costoso y recargar la página arruina el estado actual de la aplicación. Esta es la razón por la que algunos empaquetadores tienen soporte para Hot Module Replacement (HMR): permite que un módulo se "reemplace en caliente" a sí mismo sin afectar el resto de la página. Esto mejora enormemente la experiencia de desarrollo; sin embargo, en la práctica hemos descubierto que incluso con la actualización vía HMR la velocidad se deteriora significativamente a medida que crece el tamaño de la aplicación.

En Vite, el HMR es realizado sobre ESM nativo. Cuando se edita un archivo, Vite solo necesita invalidar precisamente la relación entre el módulo editado y su límite HMR más cercano (la mayoría de las veces solo el módulo en sí), lo que hace que las actualizaciones vía HMR sean consistentemente rápidas, independientemente del tamaño de tu aplicación.

Vite también aprovecha los encabezados HTTP para acelerar los refrescos completos de páginas (nuevamente, permitiendo que el navegador haga más trabajo por nosotros): las solicitudes de módulos de código fuente se hacen de forma condicional vía `304 Not Modified`, y las solicitudes de módulos de dependencia son almacenanados fuertemente en caché vía `Cache-Control: max-age=31536000,immutable` para que no lleguen al servidor nuevamente una vez almacenados en caché.

Una vez experimentes lo rápido que es Vite, dudamos mucho que estés dispuesto a trabajar nuevamente con el desarrollo en empaquetados.

## ¿Por qué empaquetar en producción?

Aunque el ESM nativo ahora es ampliamente compatible, distribuir ESM desempaquetado en producción sigue siendo ineficiente (incluso con HTTP/2) debido a las rondas de peticiones adicionales causadas ​​por importaciones anidadas. Para obtener el rendimiento óptimo de carga en producción, aún sigue siendo mejor empaquetar tu código con tree-shaking, lazy-loading y división de fragmentos comunes (para un mejor almacenamiento en caché).

Garantizar un resultado óptimo y una coherencia de comportamiento entre el servidor de desarrollo y la compilación en producción no es fácil. Esta es la razón por la que Vite tiene disponible un [comando de compilación](./build) preconfigurado que incorpora muchas [optimizaciones de rendimiento](./features#optimizaciones-de-compilacion) listas para usar.

## ¿Por qué no empaquetar con esbuild?

La API de complementos actual de Vite no es compatible con el uso de `esbuild` como paquete. A pesar de que `esbuild` es más rápido, la adopción por parte de Vite de la infraestructura y API de complementos flexible de Rollup contribuyó en gran medida a su éxito en el ecosistema. Por el momento, creemos que Rollup ofrece una mejor compensación entre rendimiento y flexibilidad.

Rollup también ha estado trabajando en mejoras de rendimiento, [cambiando su analizador a SWC en la versión 4](https://github.com/rollup/rollup/pull/5073). Además, hay un esfuerzo en curso para construir una versión en Rust de Rollup llamada Rolldown. Una vez que Rolldown esté listo, podría reemplazar tanto a Rollup como a esbuild en Vite, mejorando significativamente el rendimiento de la compilación y eliminando las inconsistencias entre el desarrollo y la compilación. Puedes ver [la presentación principal de Evan You en ViteConf 2023 para obtener más detalles](https://youtu.be/hrdwQHoAp0M).

## ¿En qué se diferencia Vite de X?

Puedes consultar la sección [Comparaciones](./comparisons) para obtener más detalles sobre cómo Vite difiere de otras herramientas similares.
