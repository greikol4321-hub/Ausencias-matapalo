# Implementacion de Boletas (Sin RLS)

Este proyecto ya incluye la vista de boletas en `boletas.html` y el acceso desde `docentes.html`.

Usa este documento cuando quieras dejar el sistema funcionando sin RLS (como esta actualmente el frontend).

## SQL unico para pegar

Ejecuta TODO este bloque en el editor SQL de Supabase:

```sql
begin;

-- 1) Ajustes en usuarios (no toca contrasena)
alter table if exists public.usuarios
  add column if not exists seccion text,
  add column if not exists seccion_guia text;

-- Permitir nuevo rol docente_guia sin romper tu constraint actual
do $$
declare c record;
begin
  for c in
    select conname
    from pg_constraint
    where conrelid = 'public.usuarios'::regclass
      and contype = 'c'
      and pg_get_constraintdef(oid) ilike '%rol%'
  loop
    execute format('alter table public.usuarios drop constraint if exists %I;', c.conname);
  end loop;
end $$;

alter table public.usuarios
  add constraint usuarios_rol_check
  check (rol = any (array['docente','docente_guia','admin']));

-- 2) Tabla estudiantes para autocompletar por cedula/nombre
create table if not exists public.estudiantes (
  id bigint generated always as identity primary key,
  cedula text not null unique,
  nombre_completo text not null,
  seccion text not null,
  correo_encargado text,
  creado_en timestamp without time zone default now()
);

-- 3) Tabla boletas disciplinarias
create table if not exists public.boletas (
  id bigint generated always as identity primary key,
  numero_boleta text not null,
  fecha date not null,
  estudiante_nombre text not null,
  estudiante_cedula text not null,
  seccion text not null,
  docente_nombre text not null,
  docente_usuario text not null,
  correo_encargado text,
  falta_cometida text not null,
  creado_en timestamp without time zone default now(),
  constraint boletas_docente_usuario_fkey foreign key (docente_usuario) references public.usuarios(usuario),
  constraint boletas_estudiante_cedula_fkey foreign key (estudiante_cedula) references public.estudiantes(cedula)
);

create index if not exists idx_estudiantes_nombre on public.estudiantes(nombre_completo);
create index if not exists idx_estudiantes_seccion on public.estudiantes(seccion);
create index if not exists idx_boletas_fecha on public.boletas(fecha desc);
create index if not exists idx_boletas_docente_usuario on public.boletas(docente_usuario);
create index if not exists idx_boletas_seccion on public.boletas(seccion);

-- 4) Dejar TODO sin RLS para que funcione con anon key y frontend actual
do $$
declare t record;
begin
  for t in
    select schemaname, tablename
    from pg_tables
    where schemaname = 'public'
  loop
    execute format('alter table %I.%I disable row level security;', t.schemaname, t.tablename);
  end loop;
end $$;

-- 5) Grants base (por si faltan en tablas nuevas)
grant usage on schema public to anon, authenticated;
grant select, insert, update, delete on all tables in schema public to anon, authenticated;
grant usage, select on all sequences in schema public to anon, authenticated;

alter default privileges in schema public
grant select, insert, update, delete on tables to anon, authenticated;

alter default privileges in schema public
grant usage, select on sequences to anon, authenticated;

commit;
```

## Como marcar un docente guia

```sql
update public.usuarios
set rol = 'docente_guia', seccion_guia = '10-1'
where usuario = 'USUARIO_DEL_DOCENTE';
```

## Cargar estudiantes de ejemplo

```sql
insert into public.estudiantes (cedula, nombre_completo, seccion, correo_encargado)
values
  ('1-1234-5678', 'Estudiante Uno', '10-1', 'encargado1@correo.com'),
  ('2-2345-6789', 'Estudiante Dos', '10-1', 'encargado2@correo.com')
on conflict (cedula) do update
set nombre_completo = excluded.nombre_completo,
    seccion = excluded.seccion,
    correo_encargado = excluded.correo_encargado;
```

## Confirmacion importante

- Este script NO cambia ni encripta la columna `contrasena`.
- Solo crea/ajusta estructura para boletas y docentes guia.
