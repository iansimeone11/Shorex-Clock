create extension if not exists pgcrypto;

create table if not exists public.employees (
  id uuid primary key default gen_random_uuid(),
  name text not null unique,
  pin_hash text not null,
  rate numeric not null default 0,
  active boolean not null default true,
  created_at timestamptz not null default now()
);

create table if not exists public.shifts (
  id uuid primary key default gen_random_uuid(),
  employee_id uuid not null references public.employees(id) on delete cascade,
  location text not null,
  client text not null,
  clock_in timestamptz not null default now(),
  clock_out timestamptz,
  created_at timestamptz not null default now()
);

create table if not exists public.admin_sessions (
  token uuid primary key default gen_random_uuid(),
  expires_at timestamptz not null default now() + interval '12 hours'
);

alter table public.employees enable row level security;
alter table public.shifts enable row level security;
alter table public.admin_sessions enable row level security;

create or replace function public.app_employee_json(p_employee public.employees)
returns jsonb
language sql
stable
as $$
  select jsonb_build_object(
    'id', (p_employee).id,
    'name', (p_employee).name,
    'rate', (p_employee).rate
  );
$$;

create or replace function public.app_shift_json(p_shift public.shifts)
returns jsonb
language sql
stable
as $$
  select jsonb_build_object(
    'id', (p_shift).id,
    'personId', (p_shift).employee_id,
    'location', (p_shift).location,
    'client', (p_shift).client,
    'clockIn', (p_shift).clock_in,
    'clockOut', (p_shift).clock_out
  );
$$;

create or replace function public.app_employee_state(p_employee_id uuid)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  employee_row employees;
begin
  select * into employee_row from employees where id = p_employee_id and active = true;

  if employee_row.id is null then
    return jsonb_build_object('ok', false, 'message', 'Usuario no encontrado.');
  end if;

  return jsonb_build_object(
    'ok', true,
    'people', jsonb_build_array(app_employee_json(employee_row)),
    'shifts', coalesce((
      select jsonb_agg(app_shift_json(s) order by s.clock_in desc)
      from shifts s
      where s.employee_id = p_employee_id
    ), '[]'::jsonb)
  );
end;
$$;

create or replace function public.app_register_employee(p_name text, p_pin text)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  clean_name text := trim(regexp_replace(p_name, '\s+', ' ', 'g'));
  employee_row employees;
begin
  if length(clean_name) < 2 then
    return jsonb_build_object('ok', false, 'message', 'Escribí tu nombre.');
  end if;

  if length(coalesce(p_pin, '')) < 4 then
    return jsonb_build_object('ok', false, 'message', 'El PIN necesita mínimo 4 dígitos.');
  end if;

  insert into employees (name, pin_hash)
  values (clean_name, crypt(p_pin, gen_salt('bf')))
  returning * into employee_row;

  return app_employee_state(employee_row.id);
exception
  when unique_violation then
    return jsonb_build_object('ok', false, 'message', 'Ese usuario ya existe.');
end;
$$;

create or replace function public.app_login_employee(p_name text, p_pin text)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  clean_name text := trim(regexp_replace(p_name, '\s+', ' ', 'g'));
  employee_row employees;
begin
  select * into employee_row
  from employees
  where lower(name) = lower(clean_name)
    and active = true
    and pin_hash = crypt(p_pin, pin_hash);

  if employee_row.id is null then
    return jsonb_build_object('ok', false, 'message', 'Usuario o PIN incorrecto.');
  end if;

  return app_employee_state(employee_row.id);
end;
$$;

create or replace function public.app_clock_in(p_employee_id uuid, p_location text, p_client text)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
begin
  if exists (select 1 from shifts where employee_id = p_employee_id and clock_out is null) then
    return jsonb_build_object('ok', false, 'message', 'Ya tenés un turno abierto.');
  end if;

  insert into shifts (employee_id, location, client)
  values (p_employee_id, p_location, p_client);

  return app_employee_state(p_employee_id);
end;
$$;

create or replace function public.app_clock_out(p_employee_id uuid)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
begin
  update shifts
  set clock_out = now()
  where id = (
    select id from shifts
    where employee_id = p_employee_id and clock_out is null
    order by clock_in desc
    limit 1
  );

  return app_employee_state(p_employee_id);
end;
$$;

create or replace function public.app_admin_login(p_pin text)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
declare
  new_token uuid;
begin
  if p_pin <> coalesce(current_setting('app.admin_pin', true), '1234') then
    return jsonb_build_object('ok', false, 'message', 'PIN de admin incorrecto.');
  end if;

  insert into admin_sessions default values returning token into new_token;
  return jsonb_build_object('ok', true, 'token', new_token);
end;
$$;

create or replace function public.app_admin_state(p_token uuid)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
begin
  if not exists (select 1 from admin_sessions where token = p_token and expires_at > now()) then
    return jsonb_build_object('ok', false, 'message', 'Sesión admin vencida.');
  end if;

  return jsonb_build_object(
    'ok', true,
    'people', coalesce((
      select jsonb_agg(app_employee_json(e) order by e.name)
      from employees e
      where e.active = true
    ), '[]'::jsonb),
    'shifts', coalesce((
      select jsonb_agg(app_shift_json(s) order by s.clock_in desc)
      from shifts s
    ), '[]'::jsonb)
  );
end;
$$;

create or replace function public.app_update_rate(p_token uuid, p_employee_id uuid, p_rate numeric)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
begin
  if not exists (select 1 from admin_sessions where token = p_token and expires_at > now()) then
    return jsonb_build_object('ok', false, 'message', 'Sesión admin vencida.');
  end if;

  update employees set rate = greatest(p_rate, 0) where id = p_employee_id;
  return app_admin_state(p_token);
end;
$$;

create or replace function public.app_clear_shifts(p_token uuid)
returns jsonb
language plpgsql
security definer
set search_path = public
as $$
begin
  if not exists (select 1 from admin_sessions where token = p_token and expires_at > now()) then
    return jsonb_build_object('ok', false, 'message', 'Sesión admin vencida.');
  end if;

  delete from shifts;
  return app_admin_state(p_token);
end;
$$;

grant usage on schema public to anon;
grant execute on all functions in schema public to anon;
