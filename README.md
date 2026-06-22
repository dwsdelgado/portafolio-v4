# darwindelgado.com — Sitio personal (Hugo)

Sitio hecho con [Hugo](https://gohugo.io) y el tema [Archie](https://github.com/athul/archie).

## Requisitos

Hugo instalado (`hugo version` para confirmar que está disponible).

## Levantar el sitio en local (mientras editas/escribes)

El `baseURL` en `hugo.toml` está puesto a tu dominio real (`https://darwindelgado.com/`). Para que el preview local cargue bien los CSS/JS, hay que decirle al servidor que use `localhost` en vez del dominio real:

```bash
hugo server --disableFastRender --baseURL http://localhost:1313/
```

Abre [http://localhost:1313/](http://localhost:1313/) en el navegador. Cada vez que guardas un archivo dentro de `content/`, el sitio se recarga solo — no hace falta reiniciar el comando.

Para detenerlo: `Ctrl+C` en la terminal donde está corriendo.

## Crear un post nuevo

1. Crea una carpeta dentro de `content/blog/AAAA/MM/tu-slug/` (el `MM` es solo organización tuya en el disco, no afecta la URL final).
2. Dentro, un archivo llamado exactamente `index.md`.
3. Usa `TEMPLATE-POST.md` (en la raíz del proyecto) como punto de partida: copia su contenido y reemplázalo con el tuyo.
4. Frontmatter mínimo:

   ```yaml
   ---
   date: '2026-06-20T09:00:00-05:00'
   tags: ['tag1', 'tag2']
   title: 'Título del post'
   ---
   ```

   - `date` controla el orden de los posts, el RSS, y el año que aparece en la URL.
   - Si quieres una URL distinta al nombre de la carpeta, agrega `slug: 'mi-url-personalizada'`.

5. Para controlar qué se muestra en el listado del home antes del botón "Read more", agrega la línea `<!--more-->` en el punto exacto donde quieres que corte el resumen. Si no la pones, Hugo corta automáticamente a las ~70 palabras, sin importar dónde caiga.
6. Las imágenes del post van en la misma carpeta que su `index.md`, referenciadas así:

   ```
   {{< figure src="./nombre-imagen.jpg" alt="Descripción de la imagen" width="700" height="auto" class="insert-image" >}}
   ```

## Cuando el post ya está listo

Revísalo en `http://localhost:1313/` con el servidor corriendo: abre la home (para ver el preview/"Read more") y el post completo, confirma que las imágenes cargan y los links funcionan.

## Build de producción (antes de publicar)

A diferencia del modo desarrollo, el build final **sí** debe usar el dominio real — que ya es el `baseURL` configurado en `hugo.toml` — así que **no** se pasa `--baseURL` aquí:

```bash
hugo --gc --minify
```

Esto genera la carpeta `public/` con el sitio compilado para producción, usando `https://darwindelgado.com/` en todos los links y assets. `public/` está en `.gitignore`, así que nunca se sube al repo — solo se sube el código fuente (`content/`, `layouts/`, `hugo.toml`, `assets/`, etc.). Tu hosting (o pipeline de despliegue) es quien corre este build y sirve lo que queda en `public/`.

## Subir los cambios al repo

```bash
git add .
git commit -m "mensaje describiendo el cambio"
git push
```

(Esto asume que esta carpeta ya está conectada a un repositorio remoto vía `git remote add origin <url>`.)

## Resumen rápido

| Qué quieres hacer | Comando |
|---|---|
| Ver el sitio mientras editas/escribes | `hugo server --disableFastRender --baseURL http://localhost:1313/` |
| Generar el sitio final para publicar | `hugo --gc --minify` |
| Subir cambios al repo | `git add . && git commit -m "..." && git push` |
