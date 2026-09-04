# Consentimiento: el `consent default` va antes del `config` y aceptar activa GA4

Corrige dos fallos que dejaban Google Analytics fuera del control del banner de
cookies. Ninguno es de SEO: los dos son de RGPD.

## 1. Orden de Consent Mode

El bloque de GA4 se insertó al principio del `<head>`, así que `gtag('config', …)`
se ejecutaba **antes** del `gtag('consent','default', …)` que vive al final del body:

```
línea 10   gtag('config', 'G-G688LPED1M');                            ← primero
línea 345  gtag('consent','default',{analytics_storage:'denied', …})  ← tarde
```

Consent Mode exige el orden contrario. Tal como estaba, la primera medición salía
con almacenamiento permitido y el `denied` llegaba después: se escribían cookies de
analítica antes de que el visitante tocara el banner, que es exactamente lo que el
banner existe para impedir.

El default pasa al bloque del `<head>`, delante del `config`, y se borra el
duplicado del final.

## 2. `ckOn()` estaba vacía

```js
function ckOn(){
}
```

Es la función que se llama al pulsar **Aceptar** y también al volver a entrar con
la cookie ya puesta. No hacía nada, así que aceptar las cookies no activaba la
analítica. Venía de antes de este trabajo — hasta ahora daba igual porque no había
GA instalado. Ahora emite:

```js
gtag('consent','update',{'analytics_storage':'granted'})
```

y `ckNo()` emite el `denied` correspondiente, para que rechazar después de haber
aceptado también surta efecto.

## 3. Las 18 páginas legales

`legal.html`, `privacidad.html` y `cookies.html` de los seis idiomas llevan GA4
pero no el banner, así que no tenían forma de conceder el consentimiento. Leen la
misma cookie `amarra_consent` y aplican el `granted` si el visitante ya aceptó en
otra página. Sin cookie se quedan en `denied`.

## Validación

Comprobado sobre los 24.840 archivos con GA4:

- un único `consent default` por página, siempre por delante del `config`;
- `ckOn` y `ckNo` rellenadas en las 24.822 con banner;
- lectura de la cookie en las 18 legales;
- segunda pasada del script: 0 archivos modificados (idempotencia);
- el diff no toca ninguna línea que no sea del consentimiento: 0 `canonical`,
  0 `hreflang`, 0 `<title>`, `vercel.json` sin tocar.

**Archivos modificados: 24.840**

## Cómo comprobarlo tras el deploy

En una ventana nueva, con la consola abierta: antes de tocar el banner,
`dataLayer` debe mostrar el `consent default` con `analytics_storage: 'denied'`
por delante del `config`, y no debe existir la cookie `_ga`. Al pulsar Aceptar
aparece el `consent update` con `granted` y entonces sí se crea `_ga`.
