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

## Soporte para varias secciones por docente

Si un docente imparte varias secciones, ejecuta este bloque adicional.

```sql
begin;

-- Tabla para secciones impartidas por cada docente
create table if not exists public.docente_secciones (
  id bigint generated always as identity primary key,
  usuario text not null references public.usuarios(usuario) on delete cascade,
  seccion text not null,
  es_guia boolean not null default false,
  creado_en timestamp without time zone default now(),
  constraint docente_secciones_unq unique (usuario, seccion)
);

-- Tabla opcional para multiples secciones guia
create table if not exists public.docente_guia_secciones (
  id bigint generated always as identity primary key,
  usuario text not null references public.usuarios(usuario) on delete cascade,
  seccion text not null,
  creado_en timestamp without time zone default now(),
  constraint docente_guia_secciones_unq unique (usuario, seccion)
);

create index if not exists idx_docente_secciones_usuario on public.docente_secciones(usuario);
create index if not exists idx_docente_secciones_seccion on public.docente_secciones(seccion);
create index if not exists idx_docente_guia_secciones_usuario on public.docente_guia_secciones(usuario);
create index if not exists idx_docente_guia_secciones_seccion on public.docente_guia_secciones(seccion);

-- Migracion inicial desde columnas viejas (seccion / seccion_guia)
insert into public.docente_secciones (usuario, seccion, es_guia)
select u.usuario, u.seccion, false
from public.usuarios u
where u.seccion is not null and trim(u.seccion) <> ''
on conflict (usuario, seccion) do nothing;

insert into public.docente_secciones (usuario, seccion, es_guia)
select u.usuario, u.seccion_guia, true
from public.usuarios u
where u.seccion_guia is not null and trim(u.seccion_guia) <> ''
on conflict (usuario, seccion)
do update set es_guia = true;

insert into public.docente_guia_secciones (usuario, seccion)
select u.usuario, u.seccion_guia
from public.usuarios u
where u.seccion_guia is not null and trim(u.seccion_guia) <> ''
on conflict (usuario, seccion) do nothing;

-- Permisos para frontend sin RLS
grant select, insert, update, delete on public.docente_secciones to anon, authenticated;
grant select, insert, update, delete on public.docente_guia_secciones to anon, authenticated;
grant usage, select on all sequences in schema public to anon, authenticated;

commit;
```

### Ejemplos de asignacion de varias secciones

```sql
insert into public.docente_secciones (usuario, seccion, es_guia)
values
  ('docente1', '10-1', false),
  ('docente1', '10-2', false),
  ('guia101', '10-1', true),
  ('guia101', '11-2', true)
on conflict (usuario, seccion)
do update set es_guia = excluded.es_guia;

insert into public.docente_guia_secciones (usuario, seccion)
values
  ('guia101', '10-1'),
  ('guia101', '11-2')
on conflict (usuario, seccion) do nothing;
```
