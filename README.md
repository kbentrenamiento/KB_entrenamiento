# KB Entrenamiento — Guía de Setup

## Paso 1: Crear cuenta en Supabase (gratis)

1. Andá a [https://supabase.com](https://supabase.com) y hacé clic en **"Start your project"**
2. Registrate con tu email de Google o creá una cuenta nueva (gratis, sin tarjeta)
3. Hacé clic en **"New project"**
4. Completá:
   - **Organization:** tu nombre o el del gimnasio
   - **Project name:** `kb-entrenamiento`
   - **Database password:** elegí una contraseña segura (guardala)
   - **Region:** `South America (São Paulo)` (el más cercano)
5. Esperá 2 minutos mientras se crea el proyecto

---

## Paso 2: Copiar tus credenciales

1. En el panel de Supabase, andá a **Settings → API**
2. Copiá:
   - **Project URL** (algo como `https://abcxyz.supabase.co`)
   - **anon public** key (una clave larga)
3. Abrí el archivo `js/config.js` y reemplazá:

```js
const SUPABASE_URL = 'TU_SUPABASE_URL_AQUI';   // ← pegá la Project URL
const SUPABASE_KEY = 'TU_SUPABASE_ANON_KEY_AQUI'; // ← pegá la anon key
```

---

## Paso 3: Crear las tablas en Supabase

1. En Supabase, andá a **SQL Editor** (ícono de código en la barra izquierda)
2. Hacé clic en **"New query"**
3. Pegá y ejecutá el siguiente SQL:

```sql
-- Tabla de perfiles de usuarios
create table profiles (
  id uuid references auth.users on delete cascade primary key,
  full_name text,
  dni text,
  phone text,
  birth_date date,
  address text,
  emergency_contact text,
  emergency_phone text,
  is_admin boolean default false,
  membership_start date,
  membership_end date,
  apto_fisico_date date,
  apto_fisico_url text,
  payment_status text default 'pending',
  created_at timestamptz default now()
);

-- Habilitar Row Level Security
alter table profiles enable row level security;

-- Políticas de acceso
create policy "Users can view own profile" on profiles for select using (auth.uid() = id);
create policy "Users can update own profile" on profiles for update using (auth.uid() = id);
create policy "Admins can view all" on profiles for select using (
  exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);
create policy "Admins can update all" on profiles for update using (
  exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);
create policy "Allow insert on signup" on profiles for insert with check (auth.uid() = id);

-- Tipos de clase
create table class_types (
  id uuid default gen_random_uuid() primary key,
  name text not null,
  description text,
  color text,
  created_at timestamptz default now()
);
alter table class_types enable row level security;
create policy "Anyone can view class types" on class_types for select using (true);
create policy "Admins can manage class types" on class_types for all using (
  exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);

-- Insertar tipos de clase iniciales
insert into class_types (name) values
  ('Funcional'), ('Yoga / Pilates'), ('Running'),
  ('Funcional Kids'), ('Zumba'), ('Tercera Edad');

-- Clases (horarios semanales)
create table classes (
  id uuid default gen_random_uuid() primary key,
  class_type_id uuid references class_types on delete cascade,
  instructor text,
  day_of_week int not null check (day_of_week between 0 and 6),
  start_time time not null,
  end_time time not null,
  capacity int not null default 15,
  is_active boolean default true,
  created_at timestamptz default now()
);
alter table classes enable row level security;
create policy "Anyone can view active classes" on classes for select using (is_active = true);
create policy "Admins can manage classes" on classes for all using (
  exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);

-- Reservas
create table reservations (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references auth.users on delete cascade,
  class_id uuid references classes on delete cascade,
  reservation_date date not null,
  status text default 'confirmed',
  created_at timestamptz default now(),
  unique(user_id, class_id, reservation_date)
);
alter table reservations enable row level security;
create policy "Users can view own reservations" on reservations for select using (auth.uid() = user_id);
create policy "Users can insert own reservations" on reservations for insert with check (auth.uid() = user_id);
create policy "Users can update own reservations" on reservations for update using (auth.uid() = user_id);
create policy "Admins can view all reservations" on reservations for select using (
  exists (select 1 from profiles where id = auth.uid() and is_admin = true)
);
```

4. Hacé clic en **"Run"** (o Ctrl+Enter)

---

## Paso 3b: Cargar el horario de clases de Funcional

En una **nueva query** del SQL Editor, pegá y ejecutá este SQL para pre-cargar todas las clases de Funcional según el horario:

```sql
-- Cargar horario completo de Funcional (Kb.entrenamiento)
-- day_of_week: 0=Lunes, 1=Martes, 2=Miércoles, 3=Jueves, 4=Viernes, 5=Sábado
-- Duración estimada de 1 hora por clase (podés editar los horarios desde el panel admin)

insert into classes (class_type_id, instructor, day_of_week, start_time, end_time, capacity, is_active)
select ct.id, null, d.dow, d.start_t::time, d.end_t::time, 15, true
from class_types ct,
(values
  -- LUNES
  (0, '07:00', '08:00'), (0, '08:00', '09:00'), (0, '09:00', '10:00'),
  (0, '13:30', '14:30'), (0, '18:00', '19:00'), (0, '19:00', '20:00'),
  -- MARTES
  (1, '08:00', '09:00'), (1, '09:00', '10:00'),
  (1, '18:00', '19:00'), (1, '19:00', '20:00'),
  -- MIÉRCOLES
  (2, '07:00', '08:00'), (2, '08:00', '09:00'), (2, '09:00', '10:00'),
  (2, '13:30', '14:30'), (2, '18:00', '19:00'), (2, '19:00', '20:00'),
  -- JUEVES
  (3, '08:00', '09:00'), (3, '09:00', '10:00'),
  (3, '18:00', '19:00'), (3, '19:00', '20:00'),
  -- VIERNES
  (4, '07:00', '08:00'), (4, '08:00', '09:00'), (4, '09:00', '10:00'),
  (4, '13:30', '14:30'), (4, '18:00', '19:00'), (4, '19:00', '20:00'),
  -- SÁBADO
  (5, '08:00', '09:00'), (5, '09:00', '10:00')
) as d(dow, start_t, end_t)
where ct.name = 'Funcional';
```

> Esto carga las 28 clases semanales de Funcional que aparecen en el horario. Podés cambiar el cupo (capacity), horarios o instructor desde el panel admin en `admin.html` → Clases.

---

## Paso 4: Crear el storage para aptos físicos

1. En Supabase, andá a **Storage** (ícono de archivos)
2. Hacé clic en **"New bucket"**
3. Nombre: `aptos`
4. Marcá **Public bucket** (para que las alumnas puedan ver sus archivos)
5. Hacé clic en **"Save"**

---

## Paso 5: Crear tu usuario admin

1. Andá a **Authentication → Users**
2. Hacé clic en **"Add user"** → **"Create new user"**
3. Completá tu email y contraseña
4. Copiá el **User UID** (lo vas a necesitar en el siguiente paso)
5. Andá a **SQL Editor** y ejecut