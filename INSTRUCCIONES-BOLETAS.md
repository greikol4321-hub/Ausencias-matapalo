# Implementacion sistema de boletas

Este proyecto ya incluye la vista de boletas en boletas.html y el acceso desde docentes.html.

## 1) Estructura recomendada en Supabase

Ejecuta este SQL en el editor SQL de Supabase para tener tablas y campos base:

```sql
-- Asegura columnas para rol docente_guia y seccion
alter table if exists usuarios
add column if not exists seccion_guia text,
add column if not exists seccion text;

-- Tabla de estudiantes para autocompletado
create table if not exists estudiantes (
  id bigint generated always as identity primary key,
  cedula text unique not null,
  nombre_completo text not null,
  seccion text not null,
  correo_encargado text,
  created_at timestamptz default now()
);

-- Tabla de boletas disciplinarias
create table if not exists boletas (
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
  created_at timestamptz default now()
);

create index if not exists idx_estudiantes_cedula on estudiantes (cedula);
create index if not exists idx_estudiantes_nombre on estudiantes (nombre_completo);
create index if not exists idx_estudiantes_seccion on estudiantes (seccion);
create index if not exists idx_boletas_docente on boletas (docente_usuario);
create index if not exists idx_boletas_seccion on boletas (seccion);
create index if not exists idx_boletas_fecha on boletas (fecha desc);
```

## 2) Como marcar un usuario como docente_guia

```sql
update usuarios
set rol = 'docente_guia', seccion_guia = '10-1'
where usuario = 'USUARIO_DEL_DOCENTE';
```

## 3) Flujo implementado

- Rol docente: puede crear boletas y ver sus boletas recientes.
- Rol docente_guia: puede crear boletas y ver boletas de su seccion guia.
- Autocompletado: al escribir cedula o nombre, consulta estudiantes y autocompleta nombre, cedula, seccion y correo de encargado.

## 4) Recomendaciones de seguridad (siguiente paso)

- Activar RLS en boletas y estudiantes.
- Crear policies para que docente solo vea/cree sus boletas y docente_guia vea su seccion.
- Mover credenciales de Supabase fuera del frontend y usar autenticacion real de Supabase Auth.
