# Shorex Clock

App simple de clock in / clock out preparada para Vercel + Supabase.

## Modo local

Abrir `index.html` en el navegador. Si no hay Supabase configurado, la app usa almacenamiento local.

## Modo nube con Supabase

1. Crear un proyecto en Supabase.
2. En Supabase SQL Editor, ejecutar `supabase/schema.sql`.
3. En Vercel, configurar variables de entorno:
   - `SUPABASE_URL`
   - `SUPABASE_ANON_KEY`
4. Subir este proyecto a GitHub y conectarlo con Vercel.

La app lee esas variables desde `/api/config` cuando corre en Vercel.

## PIN admin

Por defecto es `1234`. Para producción conviene cambiarlo en Supabase con:

```sql
alter database postgres set app.admin_pin = 'TU_PIN_SEGURO';
```

Luego reiniciar/conectar nuevamente el proyecto Supabase si hiciera falta.
