# SmartStand — MVP Backlog (10 тижнів)

B2B SaaS-платформа для ресторанів і кафе: NFC/QR-меню зі столу, цифрові сервісні кнопки, базове замовлення без онлайн-оплати, real-time дашборд офіціанта, фізичні IoT-кнопки, multi-tenancy з піддоменами та custom-доменами. Розробка: один full-stack розробник з AI-асистуванням (Claude Code).

---

## 0. Загальна архітектура

### 0.1 Стек і ключові пакети

- **Next.js 15+** (App Router, TypeScript, Server Components + Server Actions) — фронтенд і BFF.
- **Tailwind CSS + shadcn/ui** — UI-шар.
- **Supabase** — PostgreSQL, Auth (email/password для персоналу), Realtime (`supabase.channel()` + `postgres_changes`); service role key лише для довірених серверних операцій.
- **Vercel** — хостинг, wildcard-піддомени (`*.platform.com`), Vercel Domains API для custom-доменів.
- **Cloudflare R2** — зображення меню та логотипи (S3-compatible API, presigned PUT, публічне читання).
- Пакети: `@supabase/supabase-js`, `@supabase/ssr`, `zod`, `react-hook-form`, `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`, `qrcode`, `lucide-react`, `sonner`; dev: `supabase` CLI (версіоновані міграції), `tsx` (скрипти).

### 0.2 Структура проєкту

```
smartstand/
├── next.config.ts                  # images.remotePatterns (R2), security headers
├── .env.example                    # повний перелік env-змінних
├── supabase/
│   └── migrations/
│       ├── 0001_core_tenancy.sql   # ресторани, домени, брендинг, персонал, RLS-хелпери
│       ├── 0002_operational.sql    # столи, меню, замовлення, запити, hardware-кнопки
│       └── 0003_close_table.sql    # RPC close_table
├── scripts/
│   ├── seed.ts                     # демо-дані двох ресторанів
│   └── security-check.ts           # автоматична перевірка RLS/ізоляції (фаза 4)
└── src/
    ├── middleware.ts               # tenant resolution + auth session refresh
    ├── app/
    │   ├── (platform)/
    │   │   ├── login/page.tsx
    │   │   └── dashboard/
    │   │       ├── layout.tsx          # auth guard + StaffContext
    │   │       ├── page.tsx            # активні столи
    │   │       ├── requests/page.tsx   # сервісні запити
    │   │       ├── orders/page.tsx     # замовлення
    │   │       └── (admin)/            # layout: requireAdmin
    │   │           ├── menu/page.tsx
    │   │           ├── tables/page.tsx
    │   │           ├── hardware/page.tsx
    │   │           └── settings/page.tsx  # брендинг, персонал, домен
    │   ├── r/[slug]/
    │   │   ├── layout.tsx              # брендинг → CSS variables
    │   │   ├── page.tsx                # публічне меню
    │   │   └── table/[tableNumber]/route.ts  # валідація токена → cookie → redirect
    │   ├── api/
    │   │   ├── service-requests/route.ts
    │   │   ├── orders/route.ts
    │   │   ├── hardware-call/route.ts
    │   │   └── uploads/presign/route.ts
    │   ├── domain-not-found/page.tsx
    │   ├── layout.tsx
    │   └── globals.css
    ├── components/
    │   ├── ui/                     # shadcn
    │   ├── guest/                  # menu, cart, service-buttons
    │   └── dashboard/              # tables-board, requests, orders, realtime-provider
    └── lib/
        ├── supabase/
        │   ├── client.ts           # browser client (anon)
        │   ├── server.ts           # server client (cookies, сесія користувача)
        │   ├── admin.ts            # service role, import 'server-only'
        │   └── middleware.ts       # updateSession helper
        ├── tenant.ts               # parseHost, getGuestBasePath
        ├── table-session.ts        # HMAC-підписана table-сесія (cookie)
        ├── branding.ts             # hexToHsl, contrastForeground
        ├── r2.ts                   # S3Client + presign
        ├── vercel.ts               # Domains API (фаза 4)
        ├── format.ts               # formatPrice (UAH)
        └── database.types.ts       # згенеровані типи Supabase
```

### 0.3 Модель даних (огляд)

| Таблиця | Призначення | Ключові зв'язки |
|---|---|---|
| `restaurants` | Tenant-корінь: назва, унікальний `slug`, `is_active` | — |
| `restaurant_domains` | Custom-домени і мапінг domain → ресторан для middleware | → restaurants |
| `restaurant_branding` | `primary_color`, `logo_url`, `font_family` (1:1) | → restaurants |
| `staff_users` | Персонал: `id` = `auth.users.id`, роль `admin`/`waiter`, `email`, `is_active` | → auth.users, restaurants |
| `tables` | Столи: `table_number`, `status`, **`access_token`** (токен доступу столу — колонка, не окрема таблиця: один активний токен на стіл, ротація перегенерацією) | → restaurants |
| `menu_categories` | Категорії меню, `sort_order`, `is_active` | → restaurants |
| `menu_items` | Страви: ціна, фото (R2), `is_available`, `sort_order` | → restaurants, menu_categories |
| `service_requests` | Виклики: `call_waiter`/`request_bill`, `digital`/`hardware`, `open`/`handled` | → restaurants, tables, staff_users |
| `orders` | Замовлення: `status`, `payment_status`, коментар гостя | → restaurants, tables, staff_users |
| `order_items` | Позиції зі **snapshot** назви та ціни (редаговані офіціантом) | → orders, menu_items, restaurants |
| `hardware_buttons` | Фізичні кнопки: `button_id`, `secret`, прив'язка до столу і типу запиту | → restaurants, tables |

Enum-типи: `staff_role` (admin, waiter), `table_status` (free, occupied), `service_request_type` (call_waiter, request_bill), `service_request_source` (digital, hardware), `service_request_status` (open, handled), `order_status` (new, in_progress, completed, cancelled), `payment_status` (unpaid, cash, terminal, paid_manually).

### 0.4 Ключові архітектурні рішення (безпека та масштабування)

1. **Tenant resolution у Middleware.** Три класи хостів: root-домен (dashboard + шляхи `/r/[slug]/...`), піддомен `{slug}.platform.com` і custom-домен (lookup у `restaurant_domains` з in-memory кешем TTL 60s). Піддомени/custom-домени rewrite'яться на `/r/[slug]/...`; контекст доступний маршрутам через сегмент `[slug]` та request headers `x-restaurant-slug` / `x-tenant-host`. Додавання нового ресторану = лише дані, нуль змін коду.
2. **Модель безпеки гостя.** Гість анонімний. QR/NFC-токен (`tables.access_token`, 48 hex) валідується server-side і обмінюється на **HMAC-підписану httpOnly cookie** `ss_table` (TTL 4 год). Усі guest-write API (`/api/service-requests`, `/api/orders`) читають контекст ресторану/столу ВИКЛЮЧНО з cookie й пишуть через service role. Жодних anon INSERT-політик у БД.
3. **RLS-модель.** `anon` читає лише публічні дані (активні ресторани, брендинг, домени, доступне меню). Персонал — через security definer хелпери `is_staff_of()` / `is_admin_of()` (без RLS-рекурсії). Таблиця `tables` НЕ доступна anon (містить токени). Матриця доступів визначена в задачах 1.2 і 1.3.
4. **Realtime.** Один канал на ресторан: `supabase.channel('restaurant:{id}')` + `postgres_changes` з фільтром `restaurant_id=eq.{id}` по таблицях `service_requests`, `orders`, `order_items`, `tables`. RLS гарантує, що клієнт отримує події лише свого ресторану.
5. **Ціноутворення.** Сервер ніколи не довіряє цінам з клієнта: `order_items` зберігає snapshot назви/ціни з БД на момент замовлення; офіціант редагує snapshot вручну. Видалення страви з меню не ламає історію (FK `on delete set null`).
6. **Денормалізація `restaurant_id`** у кожній tenant-таблиці (включно з `order_items`) + композитні індекси `(restaurant_id, status, created_at)` — прості RLS-політики, дешеві realtime-фільтри, готовність до шардування в майбутньому.
7. **Stateless-додаток.** Уся довгоживуча взаємодія — у Supabase (дані, сесії, realtime); Vercel-функції безстанові й масштабуються горизонтально; медіа — на R2 (CDN). Версіоновані SQL-міграції через Supabase CLI + згенеровані TypeScript-типи.
8. **Антидубль на рівні БД.** Частковий унікальний індекс `(table_id, type) WHERE status='open'` у `service_requests` — жодних дублів викликів від спаму цифрової чи фізичної кнопки, незалежно від шляху створення.

### 0.5 Порядок виконання і залежності

Задачі виконуються послідовно за нумерацією. Критичний шлях: `1.1 → 1.2 → 1.3 → 1.4 → 1.5 → 1.6 → 1.7 → 2.1 → 2.2 → 2.3 → 2.4 → 2.5 → 3.1 → 3.2 → 3.3 → 3.4 → 3.5 → 3.6 → 3.7 → 3.8 → 3.9 → 4.1 → 4.2 → 4.3 → 4.4 → 4.5`.

Допустимі відхилення: 3.7–3.9 незалежні одна від одної (після 3.1); 4.1–4.2 незалежні від 4.3–4.4; задача 4.5 — завжди остання (реліз-gate).

---

## Фаза 1 — Foundation, DB Schema & Multi-Tenancy Architecture (тижні 1–2)

### Задача 1.1

- Title: Ініціалізація Next.js 15 проєкту, shadcn/ui та Supabase-клієнтів

- Goal / Description:
  Як розробник, я хочу мати повністю налаштований каркас проєкту (Next.js 15 + TypeScript + Tailwind + shadcn/ui + Supabase-клієнти + env-конфігурація), щоб усі наступні задачі виконувались на готовій основі без повторного налаштування інфраструктури.

- Technical Implementation Details for AI:
  - `pnpm create next-app@latest` у корені репозиторію: TypeScript, App Router, Tailwind, каталог `src/`, alias `@/*`, ESLint.
  - `npx shadcn@latest init`, потім add: `button card badge dialog sheet input label select separator switch table tabs textarea dropdown-menu form sonner skeleton alert radio-group`.
  - Залежності: `@supabase/supabase-js @supabase/ssr zod react-hook-form @hookform/resolvers lucide-react`; dev: `tsx`, Supabase CLI (`supabase init` → каталог `supabase/`, `supabase link` до hosted-проєкту).
  - Supabase-клієнти в `src/lib/supabase/`:
    - `client.ts` — `createBrowserClient(NEXT_PUBLIC_SUPABASE_URL, NEXT_PUBLIC_SUPABASE_ANON_KEY)` для Client Components;
    - `server.ts` — `createServerClient` з `await cookies()` (Next 15) для RSC / Server Actions / Route Handlers;
    - `admin.ts` — `createClient(url, SUPABASE_SERVICE_ROLE_KEY, { auth: { autoRefreshToken: false, persistSession: false } })` + перший рядок `import 'server-only'`;
    - `middleware.ts` — заготовка `updateSession(request)` за офіційним патерном `@supabase/ssr`.
  - `.env.example` одразу з повним переліком: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`, `NEXT_PUBLIC_ROOT_DOMAIN` (dev: `localhost:3000`), `TABLE_SESSION_SECRET`, `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_BUCKET`, `R2_PUBLIC_BASE_URL`, `VERCEL_TOKEN`, `VERCEL_PROJECT_ID`, `VERCEL_TEAM_ID`.
  - `next.config.ts`: `images.remotePatterns` для хоста з `R2_PUBLIC_BASE_URL`; глобальні security headers (`X-Content-Type-Options: nosniff`, `Referrer-Policy: strict-origin-when-cross-origin`, `X-Frame-Options: DENY`).
  - Створити каркас каталогів зі структури 0.2 (порожні `components/guest`, `components/dashboard`, `lib`, `scripts`).
  - `src/lib/format.ts`: `formatPrice(value: number | string)` через `Intl.NumberFormat('uk-UA', { style: 'currency', currency: 'UAH' })` — валюта в MVP фіксована.

- Acceptance Criteria:
  - `pnpm dev` запускається, головна сторінка рендериться без помилок і warning'ів у консолі.
  - shadcn-компоненти встановлені та рендеряться (тестова кнопка на головній).
  - Усі чотири Supabase-клієнти компілюються; імпорт `admin.ts` у Client Component валить build (`server-only` спрацьовує).
  - `.env.example` містить повний перелік змінних; `.env.local` у `.gitignore`.
  - Supabase CLI під'єднаний до проєкту (`supabase migration list` працює).

### Задача 1.2

- Title: Міграція БД №1 — tenant-ядро (restaurants, domains, branding, staff) + RLS-фундамент

- Goal / Description:
  Як система, я хочу мати таблиці ресторанів, доменів, брендингу та персоналу з ролями і суворою tenant-ізоляцією на рівні Row Level Security, щоб кожен наступний шар (auth, middleware, дашборд) будувався на безпечному фундаменті.

- Technical Implementation Details for AI:
  - Файл `supabase/migrations/0001_core_tenancy.sql`:

  ```sql
  create extension if not exists pgcrypto;

  create type staff_role as enum ('admin', 'waiter');

  create table restaurants (
    id         uuid primary key default gen_random_uuid(),
    name       text not null,
    slug       text not null unique check (slug ~ '^[a-z0-9]([a-z0-9-]*[a-z0-9])?$'),
    is_active  boolean not null default true,
    created_at timestamptz not null default now()
  );

  create table restaurant_domains (
    id            uuid primary key default gen_random_uuid(),
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    domain        text not null unique check (domain = lower(domain)),
    is_primary    boolean not null default false,
    verified_at   timestamptz,
    created_at    timestamptz not null default now()
  );
  create index idx_restaurant_domains_restaurant on restaurant_domains(restaurant_id);

  create table restaurant_branding (
    restaurant_id uuid primary key references restaurants(id) on delete cascade,
    primary_color text not null default '#1e293b' check (primary_color ~ '^#[0-9a-fA-F]{6}$'),
    logo_url      text,
    font_family   text,
    updated_at    timestamptz not null default now()
  );

  create table staff_users (
    id            uuid primary key references auth.users(id) on delete cascade,
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    role          staff_role not null default 'waiter',
    email         text not null default '',
    full_name     text not null default '',
    is_active     boolean not null default true,
    created_at    timestamptz not null default now()
  );
  create index idx_staff_users_restaurant on staff_users(restaurant_id);

  -- Хелпери security definer: обходять RLS всередині себе → без рекурсії політик
  create or replace function public.is_staff_of(rid uuid) returns boolean
  language sql stable security definer set search_path = public as $$
    select exists (
      select 1 from staff_users su
      where su.id = auth.uid() and su.restaurant_id = rid and su.is_active
    );
  $$;

  create or replace function public.is_admin_of(rid uuid) returns boolean
  language sql stable security definer set search_path = public as $$
    select exists (
      select 1 from staff_users su
      where su.id = auth.uid() and su.restaurant_id = rid
        and su.is_active and su.role = 'admin'
    );
  $$;

  alter table restaurants enable row level security;
  alter table restaurant_domains enable row level security;
  alter table restaurant_branding enable row level security;
  alter table staff_users enable row level security;

  create policy "public_read_active_restaurants" on restaurants
    for select using (is_active);
  create policy "admin_update_own_restaurant" on restaurants
    for update using (is_admin_of(id)) with check (is_admin_of(id));

  -- Публічний select доменів потрібен middleware для резолву custom-доменів
  create policy "public_read_domains" on restaurant_domains
    for select using (true);

  create policy "public_read_branding" on restaurant_branding
    for select using (true);
  create policy "admin_insert_branding" on restaurant_branding
    for insert with check (is_admin_of(restaurant_id));
  create policy "admin_update_branding" on restaurant_branding
    for update using (is_admin_of(restaurant_id)) with check (is_admin_of(restaurant_id));

  create policy "staff_read_own_team" on staff_users
    for select using (is_staff_of(restaurant_id));
  create policy "admin_update_own_team" on staff_users
    for update using (is_admin_of(restaurant_id)) with check (is_admin_of(restaurant_id));
  ```

  - Матриця доступів (операції без політики доступні лише `service_role`, який обходить RLS):
    - `restaurants`: SELECT — усі (лише `is_active`); UPDATE — admin свого ресторану; INSERT/DELETE — service_role (онбординг нового ресторану виконує оператор платформи).
    - `restaurant_domains`: SELECT — усі; всі записи — service_role (через API-route з перевіркою адміна, задача 4.4).
    - `restaurant_branding`: SELECT — усі; INSERT/UPDATE — admin; DELETE — service_role.
    - `staff_users`: SELECT — персонал свого ресторану; UPDATE — admin свого ресторану; INSERT/DELETE — service_role (flow `auth.admin.createUser`, задача 3.9).
  - Застосувати: `supabase db push`. Згенерувати типи: `supabase gen types typescript --linked > src/lib/database.types.ts`; типізувати всі клієнти з 1.1 (`createClient<Database>`).

- Acceptance Criteria:
  - Міграція застосовується без помилок; повторний `db push` нічого не змінює.
  - Анонімний клієнт бачить активні ресторани/брендинг/домени і НЕ бачить `staff_users` (0 рядків).
  - INSERT у `restaurants` під anon/authenticated → помилка RLS; під service_role — успіх.
  - Виклик `is_staff_of()` всередині політики `staff_users` не спричиняє рекурсії (select власного рядка працює).
  - Типи згенеровано в `database.types.ts` і підключено до Supabase-клієнтів.

### Задача 1.3

- Title: Міграція БД №2 — операційні таблиці (столи, меню, замовлення, запити, hardware) + Realtime

- Goal / Description:
  Як система, я хочу повну операційну модель даних із RLS, індексами, антидубль-обмеженнями та Realtime-публікацією, щоб гостьові та офіціантські фічі читали й писали дані безпечно і швидко.

- Technical Implementation Details for AI:
  - Файл `supabase/migrations/0002_operational.sql`:

  ```sql
  create type table_status as enum ('free', 'occupied');
  create type service_request_type as enum ('call_waiter', 'request_bill');
  create type service_request_source as enum ('digital', 'hardware');
  create type service_request_status as enum ('open', 'handled');
  create type order_status as enum ('new', 'in_progress', 'completed', 'cancelled');
  create type payment_status as enum ('unpaid', 'cash', 'terminal', 'paid_manually');

  create table tables (
    id            uuid primary key default gen_random_uuid(),
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    table_number  int not null check (table_number > 0),
    label         text,
    access_token  text not null unique default encode(gen_random_bytes(24), 'hex'),
    status        table_status not null default 'free',
    is_active     boolean not null default true,
    created_at    timestamptz not null default now(),
    unique (restaurant_id, table_number)
  );
  create index idx_tables_restaurant on tables(restaurant_id);

  create table menu_categories (
    id            uuid primary key default gen_random_uuid(),
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    name          text not null,
    sort_order    int not null default 0,
    is_active     boolean not null default true,
    created_at    timestamptz not null default now()
  );
  create index idx_menu_categories_rest_sort on menu_categories(restaurant_id, sort_order);

  create table menu_items (
    id            uuid primary key default gen_random_uuid(),
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    category_id   uuid not null references menu_categories(id) on delete cascade,
    name          text not null,
    description   text,
    price         numeric(10,2) not null check (price >= 0),
    image_url     text,
    is_available  boolean not null default true,
    sort_order    int not null default 0,
    created_at    timestamptz not null default now()
  );
  create index idx_menu_items_category_sort on menu_items(category_id, sort_order);
  create index idx_menu_items_restaurant on menu_items(restaurant_id);

  create table service_requests (
    id            uuid primary key default gen_random_uuid(),
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    table_id      uuid not null references tables(id) on delete cascade,
    type          service_request_type not null,
    source        service_request_source not null default 'digital',
    status        service_request_status not null default 'open',
    created_at    timestamptz not null default now(),
    handled_at    timestamptz,
    handled_by    uuid references staff_users(id) on delete set null
  );
  create index idx_service_requests_rest_status
    on service_requests(restaurant_id, status, created_at desc);
  -- Антидубль: лише один відкритий запит кожного типу на стіл
  create unique index uniq_open_service_request
    on service_requests(table_id, type) where status = 'open';

  create table orders (
    id             uuid primary key default gen_random_uuid(),
    restaurant_id  uuid not null references restaurants(id) on delete cascade,
    table_id       uuid not null references tables(id) on delete cascade,
    status         order_status not null default 'new',
    payment_status payment_status not null default 'unpaid',
    guest_comment  text,
    created_at     timestamptz not null default now(),
    closed_at      timestamptz,
    closed_by      uuid references staff_users(id) on delete set null
  );
  create index idx_orders_rest_status on orders(restaurant_id, status, created_at desc);
  create index idx_orders_table on orders(table_id, status);

  create table order_items (
    id            uuid primary key default gen_random_uuid(),
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    order_id      uuid not null references orders(id) on delete cascade,
    menu_item_id  uuid references menu_items(id) on delete set null,
    item_name     text not null,                -- snapshot назви на момент замовлення
    unit_price    numeric(10,2) not null check (unit_price >= 0),  -- snapshot, редагує офіціант
    quantity      int not null default 1 check (quantity between 1 and 50),
    created_at    timestamptz not null default now()
  );
  create index idx_order_items_order on order_items(order_id);

  create table hardware_buttons (
    id            uuid primary key default gen_random_uuid(),
    restaurant_id uuid not null references restaurants(id) on delete cascade,
    table_id      uuid not null references tables(id) on delete cascade,
    button_id     text not null unique,
    secret        text not null default encode(gen_random_bytes(24), 'hex'),
    request_type  service_request_type not null default 'call_waiter',
    is_active     boolean not null default true,
    last_seen_at  timestamptz,
    created_at    timestamptz not null default now()
  );

  alter table tables enable row level security;
  alter table menu_categories enable row level security;
  alter table menu_items enable row level security;
  alter table service_requests enable row level security;
  alter table orders enable row level security;
  alter table order_items enable row level security;
  alter table hardware_buttons enable row level security;

  -- tables: НЕ публічна (містить access_token). Гість ніколи не читає її напряму.
  create policy "staff_select_tables" on tables
    for select using (is_staff_of(restaurant_id));
  create policy "staff_update_tables" on tables
    for update using (is_staff_of(restaurant_id)) with check (is_staff_of(restaurant_id));
  create policy "admin_insert_tables" on tables
    for insert with check (is_admin_of(restaurant_id));
  create policy "admin_delete_tables" on tables
    for delete using (is_admin_of(restaurant_id));

  -- Меню: гість (anon) бачить лише активне/доступне, персонал — усе своє
  create policy "anon_read_active_categories" on menu_categories
    for select to anon using (is_active);
  create policy "staff_read_categories" on menu_categories
    for select to authenticated using (is_staff_of(restaurant_id));
  create policy "admin_write_categories" on menu_categories
    for all to authenticated using (is_admin_of(restaurant_id)) with check (is_admin_of(restaurant_id));

  create policy "anon_read_available_items" on menu_items
    for select to anon using (is_available);
  create policy "staff_read_items" on menu_items
    for select to authenticated using (is_staff_of(restaurant_id));
  create policy "admin_write_items" on menu_items
    for all to authenticated using (is_admin_of(restaurant_id)) with check (is_admin_of(restaurant_id));

  -- service_requests / orders / order_items: INSERT лише через server API (service_role)
  create policy "staff_select_requests" on service_requests
    for select using (is_staff_of(restaurant_id));
  create policy "staff_update_requests" on service_requests
    for update using (is_staff_of(restaurant_id)) with check (is_staff_of(restaurant_id));

  create policy "staff_select_orders" on orders
    for select using (is_staff_of(restaurant_id));
  create policy "staff_update_orders" on orders
    for update using (is_staff_of(restaurant_id)) with check (is_staff_of(restaurant_id));

  create policy "staff_select_order_items" on order_items
    for select using (is_staff_of(restaurant_id));
  create policy "staff_update_order_items" on order_items
    for update using (is_staff_of(restaurant_id)) with check (is_staff_of(restaurant_id));
  create policy "staff_delete_order_items" on order_items
    for delete using (is_staff_of(restaurant_id));

  create policy "staff_select_buttons" on hardware_buttons
    for select using (is_staff_of(restaurant_id));
  create policy "admin_write_buttons" on hardware_buttons
    for all using (is_admin_of(restaurant_id)) with check (is_admin_of(restaurant_id));

  -- Realtime для дашборда офіціанта
  alter publication supabase_realtime
    add table service_requests, orders, order_items, tables;
  ```

  - Матриця доступів: гостьові INSERT (запити, замовлення) — ВИКЛЮЧНО service_role через API-routes фази 2; `admin_write_buttons` покриває insert/update/delete для адміна; `hardware_buttons.secret` недоступний anon (таблиця без anon-політик).
  - Перегенерувати типи `database.types.ts`.

- Acceptance Criteria:
  - Міграція застосовується; `select * from pg_publication_tables where pubname='supabase_realtime'` містить 4 таблиці.
  - Другий INSERT відкритого запиту того самого типу на той самий стіл падає з `23505` (unique violation).
  - Anon-клієнт: `select` з `tables`, `service_requests`, `orders`, `hardware_buttons` → 0 рядків; `select` з `menu_items` повертає лише `is_available = true`.
  - Authenticated без рядка в `staff_users` не бачить жодних операційних даних.
  - Типи перегенеровано; enum-значення доступні в TypeScript.

### Задача 1.4

- Title: Seed-скрипт демо-даних (два ресторани для перевірки tenant-ізоляції)

- Goal / Description:
  Як розробник, я хочу відтворюване наповнення БД демо-даними двох ресторанів одним скриптом, щоб розробляти й тестувати multi-tenancy, RLS та всі наступні фічі на реалістичних даних без ручних вставок.

- Technical Implementation Details for AI:
  - `scripts/seed.ts` (запуск `pnpm seed` → `tsx --env-file=.env.local scripts/seed.ts`), використовує admin-клієнт (service role).
  - Створює ДВА ресторани (другий обов'язковий — на ньому перевіряється ізоляція у всіх фазах і в задачі 4.5):
    - `demo-bistro` («Demo Bistro», primary_color `#c2410c`) і `sushi-master` («Sushi Master», primary_color `#0f766e`, інший `font_family`).
  - Для кожного ресторану:
    - рядок `restaurant_branding` (лого поки `null` — з'явиться після 3.7/3.9);
    - staff через `supabase.auth.admin.createUser({ email, password, email_confirm: true })` + INSERT у `staff_users` (з колонкою `email`): `admin@demo-bistro.test` / `waiter@demo-bistro.test` (аналогічно для другого), пароль з env або константа для dev;
    - столи: 8 для першого, 4 для другого (`access_token` генерується DB default);
    - меню: 3–4 категорії, 12+ страв з цінами та описами, 2 страви з `is_available = false`, 1 неактивна категорія;
    - 1 запис `hardware_buttons` для demo-bistro (для розробки фази 4).
  - Ідемпотентність: upsert за `slug` / `email` / `(restaurant_id, table_number)`; повторний запуск не створює дублікатів (для auth-користувачів — пошук через `listUsers` за email перед створенням).
  - У кінці друкує в консоль: креденшали персоналу, URL меню обох ресторанів, повні table-URL з токенами (`/r/{slug}/table/{n}?token={token}`).

- Acceptance Criteria:
  - `pnpm seed` виконується без помилок; повторний запуск не дублює дані.
  - Логін seed-користувачами через Supabase Auth працює (перевіряється в 1.5).
  - У БД дві повністю розділені tenant-множини даних.
  - Консоль друкує готові до використання посилання і креденшали.

### Задача 1.5

- Title: Auth персоналу — логін, сесії @supabase/ssr, захист dashboard-маршрутів і ролі

- Goal / Description:
  Як співробітник ресторану, я хочу входити за email і паролем та мати стабільну захищену сесію, щоб лише персонал мого ресторану бачив службові розділи, а адмін-розділи були доступні лише ролі admin.

- Technical Implementation Details for AI:
  - У Supabase Auth вимкнути публічну реєстрацію (Sign up disabled / email signups off) — користувачів створює лише `auth.admin.createUser` (seed і задача 3.9). Увімкнути leaked password protection.
  - `/login` (`app/(platform)/login/page.tsx`): Client Component, shadcn Form + zod (`email`, `password`), `supabase.auth.signInWithPassword`, помилки українською («Невірний email або пароль»), редірект на `/dashboard` (або `?next=`).
  - `src/lib/supabase/middleware.ts` — повний `updateSession`: `createServerClient` на request/response cookies, `await supabase.auth.getUser()` (саме `getUser`, не `getSession` — довіряємо лише перевіреному токену). Підключити у `src/middleware.ts`: без user запит на `/dashboard/**` → redirect `/login?next={path}`; з user запит на `/login` → redirect `/dashboard`.
  - `app/(platform)/dashboard/layout.tsx` (RSC): `getUser()` → запит `staff_users` (server-клієнт, RLS поверне власний рядок) разом з `restaurants(name, slug)`; якщо рядка немає або `is_active = false` → `signOut()` + redirect `/login`. Передає у клієнтський `<DashboardProvider>` контекст: `{ staff: { id, role, fullName }, restaurant: { id, slug, name } }`.
  - `src/lib/auth.ts`: `getStaffContext()` з React `cache()` — перевикористання в Server Actions; `requireAdmin()` — кидає помилку/redirect, якщо роль не admin.
  - Route group `app/(platform)/dashboard/(admin)/layout.tsx`: server-side перевірка `role === 'admin'`, інакше сторінка 403 («Недостатньо прав»).
  - Header dashboard: dropdown користувача (ім'я, роль, «Вийти» → `signOut` + redirect `/login`).

- Acceptance Criteria:
  - Логін seed-користувачами працює; невірний пароль дає зрозумілу помилку.
  - Неавтентифікований запит будь-якого `/dashboard/**` → redirect на `/login`, після логіну — повернення на цільову сторінку.
  - Сесія автоматично оновлюється (повернення на дашборд через > 1 год без повторного логіну).
  - Auth-користувач без рядка `staff_users` (або `is_active=false`) не потрапляє в dashboard.
  - Waiter отримує 403 на прямий URL admin-розділу; публічна реєстрація в Supabase вимкнена.

### Задача 1.6

- Title: Middleware — tenant/domain resolution і rewrite у контекст ресторану

- Goal / Description:
  Як система, я хочу визначати ресторан за піддоменом або custom-доменом у Next.js Middleware і робити rewrite на `/r/[slug]/...`, щоб один деплой обслуговував усі tenant-домени, а контекст ресторану був доступний маршрутам.

- Technical Implementation Details for AI:
  - `src/middleware.ts`, matcher: `['/((?!api|_next/static|_next/image|favicon.ico|sounds|.*\\..*).*)']` — `/api/**` і статика не переписуються (guest-фронтенд на tenant-домені викликає `/api/...` same-origin без rewrite).
  - `src/lib/tenant.ts`: `parseHost(host, rootDomain)` → `{ kind: 'root' | 'subdomain' | 'custom', slug?: string }`: відкинути порт; `www`/`app` → root; dev-root — `localhost`.
  - Гілки:
    - `root` → виконати лише auth-гілку з 1.5 (dashboard/login), `/r/**` працює як є (шляховий доступ — канонічний формат QR-посилань).
    - `subdomain` → `slug` = перший лейбл; rewrite: `/` → `/r/{slug}`, `/table/{n}` → `/r/{slug}/table/{n}` (query зберігається), інші шляхи → `/r/{slug}{path}`; запити `/dashboard`|`/login` на tenant-домені → redirect `https://{ROOT_DOMAIN}{path}`.
    - `custom` → lookup: `fetch('{SUPABASE_URL}/rest/v1/restaurant_domains?domain=eq.{host}&select=restaurants!inner(slug,is_active)', { headers: { apikey: ANON_KEY } })` (edge-сумісний REST, RLS дозволяє anon-select) → module-level `Map<string, { slug: string | null, expiresAt: number }>` з TTL 60 сек (негативний результат теж кешується); знайдено → rewrite як subdomain; ні → rewrite на `/domain-not-found` (проста статична сторінка).
  - При будь-якому tenant-rewrite прокидати request headers: `x-restaurant-slug: {slug}` та `x-tenant-host: 1` (через `NextResponse.rewrite(url, { request: { headers } })`) — RSC читають через `headers()`.
  - `src/lib/tenant.ts`: `getGuestBasePath(slug)` (server helper): якщо `headers().get('x-tenant-host')` → `''` (гість на tenant-домені), інакше `/r/{slug}` — використовується всіма guest-редіректами (задача 2.2).
  - Dev-перевірка піддоменів: `demo-bistro.localhost:3000` (браузери резолвлять `*.localhost` у 127.0.0.1); custom-домен у dev — `curl -H "Host: mydomain.test" http://localhost:3000/`.

- Acceptance Criteria:
  - `localhost:3000/r/demo-bistro` і `demo-bistro.localhost:3000` віддають ту саму сторінку (заглушку меню до 2.1).
  - Невідомий піддомен/домен → сторінка `domain-not-found` (404-семантика).
  - Custom-домен, доданий у `restaurant_domains` seed-ом, резолвиться (перевірка через `curl -H "Host: ..."`); повторний запит протягом 60 с не б'є в Supabase (кеш, перевірити логом).
  - `/dashboard` на tenant-домені редіректить на root-домен; статика і `/api/**` не проходять tenant-логіку.
  - `headers().get('x-restaurant-slug')` доступний у Server Components після rewrite.

### Задача 1.7

- Title: Динамічний white-label брендинг через CSS variables і shadcn-токени

- Goal / Description:
  Як гість, я хочу бачити меню у фірмових кольорах, з логотипом і шрифтом закладу, щоб платформа виглядала як власний сайт ресторану, а не як сторонній сервіс.

- Technical Implementation Details for AI:
  - `app/r/[slug]/layout.tsx` (Server Component): `const { slug } = await params`; helper `getRestaurantBySlug(slug)` з React `cache()` — один запит server-клієнтом (anon): `restaurants` (за `slug`, тільки `is_active`, інакше `notFound()`) + `restaurant_branding`. Без додаткового кешування в MVP (pk-lookup дешевий); точка майбутньої оптимізації — `unstable_cache` + `revalidateTag`.
  - `src/lib/branding.ts`: `hexToHsl(hex)` → рядок `"H S% L%"` (формат shadcn-токенів); `contrastForeground(hex)` → за YIQ-luminance повертає світлий або темний foreground.
  - Інжект у layout: обгортка guest-дерева `<div className="guest-theme">` + інлайн `<style>`:
    `.guest-theme { --primary: {hsl}; --primary-foreground: {contrast}; --ring: {hsl}; }` — shadcn-компоненти (Button, Badge) всередині guest-дерева автоматично підхоплюють колір; дашборд (поза `/r/**`) залишається зі стандартною темою.
  - Шрифт: allowlist-мапа `{ inter, roboto, open-sans, montserrat, lora }` → `<link rel="stylesheet">` на Google Fonts лише для значення з allowlist + `style={{ fontFamily }}` на обгортці; `font_family = null` → системний стек.
  - Guest-header у layout: `logo_url` через `next/image` (fallback — назва ресторану текстом), назва.
  - Сторінка-заглушка `app/r/[slug]/page.tsx` («Меню незабаром») — повноцінне меню в 2.1.

- Acceptance Criteria:
  - Два seed-ресторани відкриті в сусідніх вкладках показують різні кольори/шрифти одночасно — стилі не «протікають» між tenant'ами.
  - Зміна `primary_color` у БД видна на сторінці після reload; невалідний `slug` → 404.
  - Кнопки primary мають автоматично контрастний текст і для світлого, і для темного фірмового кольору.
  - `font_family` поза allowlist ігнорується (жодних довільних URL у `<link>`).
  - Дашборд не змінює вигляд.

---

## Фаза 2 — Guest Experience (тижні 3–4)

### Задача 2.1

- Title: Публічна сторінка меню `/r/[slug]` (SSR, mobile-first)

- Goal / Description:
  Як гість, я хочу миттєво бачити актуальне меню закладу з категоріями, фото та цінами на своєму телефоні, щоб обрати страви без встановлення застосунків і без логіну.

- Technical Implementation Details for AI:
  - `app/r/[slug]/page.tsx` — Server Component: один запит server-клієнтом (anon): `menu_categories` з вкладеними `menu_items` (`select('*, menu_items(*)')`), `order('sort_order')` для обох рівнів; RLS anon-політики самі відфільтрують неактивні категорії та недоступні страви. `restaurant` — з `getRestaurantBySlug` (кешовано в межах запиту, 1.7).
  - Читання table-сесії: `cookies()` → `verifyTableSession` (з'явиться у 2.2; у цій задачі — заглушка, що повертає `null`), у клієнтські компоненти передається `tableSession: { tableId, tableNumber } | null`. Без сесії сторінка read-only: зони кошика і сервісних кнопок не рендеряться.
  - UI (mobile-first, ширина від 360px):
    - sticky header з layout (лого + назва) + горизонтальна навігація категорій (chips зі scroll-to-anchor, `scroll-margin-top` на секціях);
    - секції категорій; `MenuItemCard`: фото (`next/image` з R2-хоста, `sizes`, lazy; без фото — placeholder-блок), назва, опис (`line-clamp-2`), ціна через `formatPrice`;
    - категорії без жодної доступної страви не рендеряться; ресторан без меню → порожній стан («Меню наповнюється»).
  - `?e=invalid_table` у searchParams (ставить 2.2) → shadcn `Alert` угорі: «Посилання столу недійсне. Зверніться до персоналу.»
  - Продуктивність: один DB roundtrip на меню; зображення lazy; сторінка динамічна (читає cookies), додаткових fetch на клієнті немає.

- Acceptance Criteria:
  - Меню обох seed-ресторанів рендериться з фото, описами й цінами у порядку `sort_order`.
  - Страви з `is_available = false` та неактивні категорії відсутні у HTML-відповіді (не просто приховані CSS).
  - Навігація категорій прокручує до відповідної секції; верстка коректна на 360px.
  - Без table-сесії немає елементів кошика/сервісних кнопок.
  - LCP на демо-даних у throttled mobile-профілі < 2.5s (перевірка Lighthouse).

### Задача 2.2

- Title: Доступ до столу за токеном `/r/[slug]/table/[n]?token=` і підписана table-сесія

- Goal / Description:
  Як гість, я хочу відсканувати QR/NFC-мітку на столі й одразу отримати меню, прив'язане до мого столу, щоб мої виклики та замовлення приходили офіціанту з правильним номером столу. Як система, я хочу криптографічно валідувати посилання і сесію, щоб ніхто не міг створювати запити від імені чужого столу.

- Technical Implementation Details for AI:
  - `src/lib/table-session.ts` (server-only):
    - `createTableSession({ restaurantId, tableId, tableNumber })` → payload `base64url(JSON { rid, tid, tn, exp: now + 4h })` + `'.'` + `base64url(HMAC-SHA256(payload, TABLE_SESSION_SECRET))`;
    - `verifyTableSession(cookieValue, expectedRestaurantId)` → перевірка підпису через `crypto.timingSafeEqual`, перевірка `exp`, перевірка `rid === expectedRestaurantId`; будь-яка невідповідність → `null`;
    - константа `TABLE_SESSION_COOKIE = 'ss_table'`.
  - Route Handler `app/r/[slug]/table/[tableNumber]/route.ts` (GET):
    - `const { slug, tableNumber } = await params`; `token` з `request.nextUrl.searchParams`; zod: `tableNumber` — ціле > 0, `token` — hex довжиною 48;
    - admin-клієнт (service role): `tables` join `restaurants` за `slug` + `table_number`, обидва `is_active = true`;
    - порівняння токена: `crypto.timingSafeEqual` (після перевірки довжини);
    - успіх → `Set-Cookie ss_table` (`httpOnly`, `sameSite: 'lax'`, `secure` у production, `path: '/'`, `maxAge: 4h`) + redirect 303 на `getGuestBasePath(slug) || '/'` (на tenant-домені — `/`, на root-домені — `/r/{slug}`);
    - невдача (немає столу / токен не збігся / стіл або ресторан неактивні) → redirect на `{base}?e=invalid_table` БЕЗ cookie.
  - Замінити заглушку в 2.1: сторінка меню читає cookie і викликає `verifyTableSession(value, restaurant.id)`; при валідній сесії показує badge «Стіл {tn}» у header і вмикає гостьові дії (2.3, 2.4).
  - Токен ніколи не потрапляє в client-side JS/localStorage — лише httpOnly cookie; наявність токена в URL QR-коду прийнятна (він фізично надрукований на столі).

- Acceptance Criteria:
  - Перехід за коректним посиланням (із виводу seed) ставить cookie і відкриває меню з badge стола — і на `/r/{slug}/table/{n}`, і на tenant-домені `/table/{n}`.
  - Невірний/чужий токен → банер «Посилання недійсне», cookie не встановлюється.
  - Cookie з підробленим підписом або простроченим `exp` трактується як відсутність сесії (read-only меню, guest-API повертають 401).
  - Table-сесія ресторану A не активує дії на сторінці ресторану B (`rid` mismatch → null).
  - Деактивований стіл (`is_active = false`) відхиляється.

### Задача 2.3

- Title: Локальний кошик гостя (context + localStorage + Sheet UI)

- Goal / Description:
  Як гість, я хочу додавати страви в кошик, змінювати кількість і бачити загальну суму, щоб сформувати замовлення перед відправкою офіціанту.

- Technical Implementation Details for AI:
  - `components/guest/cart-context.tsx` (Client): `CartProvider` монтується з guest-layout лише за наявності валідної table-сесії; `useReducer` з діями `add | remove | setQty | clear`; елемент: `{ menuItemId, name, price, quantity }` (`price` — лише для відображення, сервер перерахує у 2.5); максимум 20 шт. на позицію.
  - Персистентність: localStorage-ключ `ss_cart_{restaurantId}_{tableId}`; hydration-safe: initial state порожній, читання з localStorage у `useEffect`, запис — при кожній зміні. Кошики різних столів/ресторанів не перетинаються.
  - UI:
    - `AddToCartButton` у `MenuItemCard`: «Додати» → stepper `−/+` коли позиція вже в кошику;
    - плаваючий `CartBar` знизу екрана (кількість позицій + сума + «Переглянути замовлення»), видимий лише при непорожньому кошику;
    - `CartSheet` (shadcn Sheet bottom): список позицій (назва, ціна, stepper, видалити), `Textarea` «Коментар до замовлення» (max 500), загальна сума, кнопка «Надіслати замовлення» (submit підключить 2.5; тут — prop `onSubmit`).
  - Всі суми — через `formatPrice`; без table-сесії компоненти кошика не рендеряться взагалі (узгоджено з 2.1).

- Acceptance Criteria:
  - Додавання/зміна кількості/видалення миттєво оновлюють CartBar і суму.
  - Кошик переживає reload сторінки (localStorage) і не змішується між двома столами/ресторанами у тому самому браузері.
  - Ліміт 20 шт. на позицію дотримується; порожній кошик ховає CartBar.
  - Жодних hydration-помилок у консолі.

### Задача 2.4

- Title: Сервісні кнопки «Покликати офіціанта» / «Попросити рахунок» + `POST /api/service-requests`

- Goal / Description:
  Як гість, я хочу одним дотиком покликати офіціанта або попросити рахунок, щоб отримати сервіс без очікування і без пошуку персоналу очима.

- Technical Implementation Details for AI:
  - UI `components/guest/service-buttons.tsx` (Client): два великі помітні button (primary-стиль — підхоплюють брендинг) у sticky-зоні під header меню: «Покликати офіціанта» (іконка Bell) і «Попросити рахунок» (іконка Receipt). Стани: idle → loading → success (sonner toast «Офіціанта сповіщено» / «Запит на рахунок передано») → cooldown 30 с (disabled + таймер на кнопці). Рендеряться лише при валідній table-сесії.
  - API `app/api/service-requests/route.ts` (POST):
    - zod body: `{ type: 'call_waiter' | 'request_bill' }`;
    - контекст: cookie `ss_table` → `verifyTableSession` → 401 якщо відсутня/невалідна; `restaurant_id`/`table_id` беруться ВИКЛЮЧНО з cookie — жодних id з body;
    - admin-клієнт: INSERT `{ restaurant_id, table_id, type, source: 'digital' }`; при помилці `23505` (частковий унікальний індекс з 1.3 — запит уже відкритий) → відповідь 200 `{ status: 'already_open' }`; при успіху → `UPDATE tables SET status='occupied' WHERE id=... AND status='free'` → 201 `{ status: 'created' }`;
    - інші методи → 405; невалідне body → 400.
  - UI-обробка `already_open`: toast «Запит уже передано — офіціант скоро підійде».

- Acceptance Criteria:
  - Клік створює рядок `service_requests` з правильними `restaurant_id`/`table_id`/`type`/`source='digital'`.
  - Повторний клік до обробки запиту не створює дубля (гілка 23505) і показує коректне повідомлення.
  - Стіл переходить у `occupied` після першого запиту.
  - Прямий POST без cookie (або з підробленою) → 401; підміна id у body неможлива (body id відсутні в контракті).
  - Cooldown блокує спам-кліки на клієнті; кнопки не рендеряться без table-сесії.

### Задача 2.5

- Title: Відправка замовлення + `POST /api/orders` зі server-side snapshot цін

- Goal / Description:
  Як гість, я хочу надіслати сформоване замовлення з кошика, щоб офіціант миттєво його побачив. Як система, я хочу розраховувати назви й ціни виключно на сервері з БД, щоб гість не міг маніпулювати вартістю замовлення.

- Technical Implementation Details for AI:
  - `app/api/orders/route.ts` (POST):
    - zod body: `{ items: [{ menuItemId: uuid, quantity: int 1..20 }] (1..50 елементів), comment?: string ≤ 500 }`; дублікати `menuItemId` зливаються з сумуванням quantity;
    - контекст лише з cookie: `verifyTableSession` → 401;
    - admin-клієнт: `SELECT id, name, price FROM menu_items WHERE id IN (...) AND restaurant_id = {rid} AND is_available` — якщо хоч один id не знайдено → 400 `{ error: 'items_unavailable', missing: [...] }` (захист від чужих/прихованих страв);
    - INSERT `orders { restaurant_id, table_id, status: 'new', payment_status: 'unpaid', guest_comment }` → bulk INSERT `order_items { restaurant_id, order_id, menu_item_id, item_name: name, unit_price: price, quantity }` (значення ТІЛЬКИ з БД, ціни з клієнта ігноруються); при збої вставки позицій — компенсаційний `DELETE order` (щоб не було порожніх замовлень);
    - `UPDATE tables SET status='occupied'`; відповідь 201 `{ orderId, total }` (total рахує сервер).
  - UI: `CartSheet.onSubmit` → fetch POST → loading; успіх → екран/стан підтвердження (іконка, «Замовлення передано», список позицій + сума) → `clear` кошика (включно з localStorage); помилка `items_unavailable` → toast «Деякі страви вже недоступні», видалення недоступних позицій з кошика + `router.refresh()` для оновлення меню.

- Acceptance Criteria:
  - Замовлення створює `orders` + `order_items` зі snapshot назв/цін саме з БД: підміна ціни в localStorage/devtools не впливає на збережені значення.
  - `total` у відповіді = `sum(unit_price × quantity)` вставлених рядків.
  - Недоступна/чужа страва в кошику → 400 без часткових вставок (компенсація відпрацьовує, порожніх orders немає).
  - Кошик очищується після успіху; `guest_comment` зберігається.
  - POST без валідної table-сесії → 401.

---

## Фаза 3 — Waiter Dashboard & Real-Time Alert System (тижні 5–7)

### Задача 3.1

- Title: Каркас дашборда — layout, навігація, контекст персоналу

- Goal / Description:
  Як співробітник ресторану, я хочу єдиний захищений робочий простір зі зручною навігацією по розділах, щоб швидко перемикатися між столами, запитами, замовленнями та налаштуваннями зі свого планшета або телефона.

- Technical Implementation Details for AI:
  - Розширити `app/(platform)/dashboard/layout.tsx` (guard уже з 1.5) повноцінним UI: sidebar на desktop / bottom tabs на mobile (основний пристрій офіціанта — планшет/телефон).
  - Пункти навігації: «Столи» (`/dashboard`), «Запити» (`/dashboard/requests`), «Замовлення» (`/dashboard/orders`); admin-група (рендер лише для `role='admin'`): «Меню» (`/dashboard/menu`), «Столи (конфігурація)» (`/dashboard/tables`), «Кнопки» (`/dashboard/hardware`), «Налаштування» (`/dashboard/settings`).
  - Header: назва ресторану, місце під sound-toggle та індикатор з'єднання (реалізує 3.2), user dropdown з 1.5.
  - `DashboardProvider` (client context) отримує з layout server-props: `{ staff, restaurant }` — використовуватиметься realtime-шаром і сторінками.
  - Badge-лічильники в навігації біля «Запити» і «Замовлення»: initial значення — server-side `count` (`service_requests` зі `status='open'`, `orders` зі `status='new'`); live-оновлення підключить 3.2.
  - Створити сторінки-заглушки всіх розділів з порожніми станами, щоб навігація була повністю робочою одразу.

- Acceptance Criteria:
  - Навігація працює на mobile (bottom tabs) і desktop (sidebar); активний розділ підсвічений.
  - Waiter не бачить admin-пунктів і отримує 403 на їх прямі URL (route group з 1.5).
  - Header показує назву ресторану залогіненого користувача (tenant-контекст правильний).
  - Лічильники показують реальні counts з БД при завантаженні.

### Задача 3.2

- Title: Realtime-шар (supabase.channel) і система звукових/візуальних сповіщень

- Goal / Description:
  Як офіціант, я хочу миттєво чути й бачити нові виклики та замовлення без оновлення сторінки, щоб реагувати за лічені секунди з будь-якого розділу дашборда.

- Technical Implementation Details for AI:
  - `components/dashboard/realtime-provider.tsx` (Client, монтується в dashboard layout всередині `DashboardProvider`):
    - один канал: `supabase.channel('restaurant:' + restaurant.id)` з чотирма bindings `.on('postgres_changes', { event: '*', schema: 'public', table, filter: 'restaurant_id=eq.' + restaurant.id }, handler)` для `service_requests`, `orders`, `order_items`, `tables`;
    - browser-клієнт авторизований JWT персоналу → RLS гарантує лише події свого ресторану (фільтр — оптимізація, RLS — захист);
    - внутрішній event-bus: `useRealtimeEvents(table, handler)` — сторінки реєструють обробники (Set у контексті), provider розсилає payload'и; окрема подія `'resync'` для повного refetch.
  - Глобальні алерти (незалежно від активного розділу): на INSERT `service_requests` → sonner toast «Виклик офіціанта — стіл N» / «Просять рахунок — стіл N» з кнопкою «Переглянути» (навігація на `/dashboard/requests`) + звук `public/sounds/new-request.mp3`; на INSERT `orders` → toast «Нове замовлення — стіл N» + `new-order.mp3`.
  - `AudioManager`: `Audio`-об'єкти з preload; браузери блокують autoplay до першого жесту → `SoundToggle` у header: стан у localStorage (`ss_sound`), при вмиканні — unlock (тихий програш); вимкнено → лише візуальні алерти.
  - Live-лічильники: provider тримає counts (initial із server-props 3.1, дельти з подій INSERT/UPDATE) і віддає їх nav-багжам.
  - Стійкість: `.subscribe(status => ...)`: на `'CHANNEL_ERROR' | 'TIMED_OUT'` — індикатор з'єднання червоний; на відновлення `'SUBSCRIBED'` після обриву → розіслати `'resync'` (сторінки перезавантажують дані). Індикатор з'єднання (зелена/сіра/червона крапка) в header.

- Acceptance Criteria:
  - Створення сервісного запиту гостем (другий браузер/інкогніто) → toast + звук на дашборді < 2 с; те саме для замовлення.
  - Waiter ресторану A не отримує подій ресторану B (перевірити двома залогіненими вкладками).
  - Звук вмикається лише після взаємодії з toggle; стан переживає reload; при вимкненому звуці алерти лише візуальні.
  - Відключення мережі на ~10 с → індикатор червоніє; після відновлення дані ресинхронізуються без reload сторінки.
  - Nav-лічильники оновлюються live при створенні/обробці запитів і замовлень.

### Задача 3.3

- Title: Екран «Активні столи» (головна дашборда)

- Goal / Description:
  Як офіціант, я хочу бачити всі столи з їхнім станом, відкритими викликами й замовленнями на одному екрані, щоб одним поглядом розуміти ситуацію в залі й нічого не пропустити.

- Technical Implementation Details for AI:
  - `/dashboard/page.tsx` (RSC): initial fetch server-клієнтом (RLS): активні `tables` (order by `table_number`) + відкриті `service_requests` + `orders` зі статусами `new`/`in_progress` разом з `order_items` (для сум) → передати в `TablesBoard` (Client).
  - `TableCard`: номер + label; колір за `status` (`free` — нейтральний, `occupied` — акцент); badges: «Виклик» (pulse-анімація + час найстарішого відкритого виклику), «Рахунок», «Замовлень: N»; сума відкритих замовлень столу (`formatPrice`). Відносний час («5 хв тому») оновлюється інтервалом 30 с.
  - Клік по картці → `TableDetailSheet`: відкриті запити столу з кнопкою «Обробити» (та сама мутація, що у 3.4), список замовлень столу (статус, сума; деталі — компонент 3.5), кнопка «Закрити стіл» (підключить 3.6, поки disabled з tooltip).
  - Realtime: `useRealtimeEvents` для `service_requests` / `orders` / `order_items` / `tables` → інкрементальне оновлення локальної `Map` по `table_id`; подія `'resync'` → повний refetch клієнтським supabase-клієнтом (RLS) через функцію `loadBoard()`.
  - Grid-сортування стабільне за `table_number` (без перестрибувань); привертання уваги — лише підсвіткою і badges.

- Acceptance Criteria:
  - Запит/замовлення гостя миттєво (< 2 с) підсвічує відповідний стіл без reload.
  - `free`/`occupied` і суми відкритих замовлень відображаються коректно.
  - Деталі столу відкриваються з актуальними даними; «Обробити» працює звідси (після 3.4).
  - Таймери «N хв тому» оновлюються; порядок карток стабільний.
  - Дані іншого ресторану не з'являються (RLS + фільтр каналу).

### Задача 3.4

- Title: Панель сервісних запитів і позначення обробленими

- Goal / Description:
  Як офіціант, я хочу бачити чергу відкритих викликів (FIFO) і позначати їх обробленими одним дотиком, щоб жоден гість не залишився без уваги, а команда бачила, хто вже відреагував.

- Technical Implementation Details for AI:
  - `/dashboard/requests/page.tsx` (RSC initial): відкриті запити (join `tables(table_number, label)`, найстаріші зверху) + оброблені за сьогодні (останні 20, join `staff_users(full_name)` по `handled_by`) → `RequestsList` (Client) з Tabs «Відкриті (N)» / «Оброблені сьогодні».
  - Item: тип українською («Виклик офіціанта» / «Просять рахунок») + іконка, «Стіл {n}», badge джерела (`digital` → «QR», `hardware` → «Кнопка»), відносний час + абсолютний у tooltip, кнопка «Обробити».
  - Мутація «Обробити»: клієнтський supabase (RLS `staff_update_requests`): `update service_requests set status='handled', handled_at=now-ISO, handled_by=staff.id where id=...` з optimistic UI (миттєвий перенос у «Оброблені», rollback при помилці + toast).
  - Realtime: INSERT → prepend у «Відкриті» з коротким highlight; UPDATE (колега обробив) → прибрати з «Відкритих» у всіх; `'resync'` → refetch обох списків.
  - Той самий мутаційний хук `useHandleRequest()` експортується для `TableDetailSheet` (3.3).

- Acceptance Criteria:
  - Обробка запиту прибирає його з «Відкритих» у всіх підключених офіціантів < 2 с.
  - `handled_at`/`handled_by` заповнюються; вкладка «Оброблені» показує, хто обробив.
  - Optimistic UI відкочується при збої мережі з повідомленням про помилку.
  - Порядок відкритих — FIFO (найстаріший зверху).
  - Badge джерела коректний для обох значень `source` (hardware перевіряється SQL-вставкою до фази 4).

### Задача 3.5

- Title: Управління замовленнями — статуси та ручне редагування цін позицій

- Goal / Description:
  Як офіціант, я хочу переглядати замовлення з позиціями, змінювати їх статус і вручну коригувати ціни/кількість/склад, щоб рахунок відповідав реальності (акції, заміни, домовленості з гостем).

- Technical Implementation Details for AI:
  - `/dashboard/orders/page.tsx`: Tabs «Нові (N)» / «В роботі» / «Завершені сьогодні»; RSC initial fetch: `orders` + `order_items` + `tables(table_number)` свого ресторану → `OrdersList` (Client).
  - `OrderCard`: «Стіл {n}», час створення, кількість позицій, сума (`sum(unit_price × quantity)` рахується з items на клієнті), поточний статус; кнопка «Прийняти» на `new` (→ `in_progress`); Select статусу (`new → in_progress → completed / cancelled`) — клієнтський supabase update (RLS), optimistic.
  - `OrderDetailSheet` (перевикористовується у 3.3): `guest_comment`, список позицій: `item_name` (snapshot), stepper `quantity` (1..50), інлайн-редагування `unit_price` (numeric input ≥ 0, крок 0.01), «Прибрати позицію» (delete з confirm-dialog); мутації — клієнтський supabase `update`/`delete` на `order_items` (політики `staff_update_order_items` / `staff_delete_order_items`); сума перераховується миттєво.
  - Валідація: zod на клієнті + DB check-constraints (`unit_price >= 0`, `quantity 1..50`) як друга лінія.
  - Realtime: INSERT `orders` → prepend у «Нові» (глобальний toast уже робить 3.2); UPDATE `orders`/`order_items` → синхронізація списку і відкритого Sheet (редагування колегою видно live); `'resync'` → refetch.

- Acceptance Criteria:
  - Нове замовлення гостя з'являється у «Нових» < 2 с без reload.
  - Зміна `unit_price`/`quantity` зберігається в БД; сума перераховується у списку, деталях і на картці столу (3.3).
  - Зміна статусу синхронізується між двома відкритими дашбордами; «Завершені сьогодні» показує завершені/скасовані.
  - Видалення позиції вимагає підтвердження і працює; замовлення без позицій показує 0 суму (не ламається).
  - Спроба редагувати order_item чужого ресторану неможлива (RLS; фінальна перевірка — скрипт 4.5).

### Задача 3.6

- Title: Закриття столу з фіксацією способу оплати (RPC `close_table`)

- Goal / Description:
  Як офіціант, я хочу закрити стіл із зазначенням способу розрахунку (готівка / термінал / оплачено вручну), щоб атомарно завершити візит: закрити всі замовлення столу, погасити запити і звільнити стіл.

- Technical Implementation Details for AI:
  - Міграція `supabase/migrations/0003_close_table.sql`:

  ```sql
  create or replace function public.close_table(p_table_id uuid, p_payment payment_status)
  returns json language plpgsql security definer set search_path = public as $$
  declare
    v_rid uuid;
    v_orders int;
    v_requests int;
  begin
    select restaurant_id into v_rid from tables where id = p_table_id;
    if v_rid is null or not is_staff_of(v_rid) then
      raise exception 'forbidden';
    end if;
    if p_payment not in ('cash', 'terminal', 'paid_manually') then
      raise exception 'invalid_payment_status';
    end if;

    update orders
      set status = 'completed', payment_status = p_payment,
          closed_at = now(), closed_by = auth.uid()
      where table_id = p_table_id and status in ('new', 'in_progress');
    get diagnostics v_orders = row_count;

    update service_requests
      set status = 'handled', handled_at = now(), handled_by = auth.uid()
      where table_id = p_table_id and status = 'open';
    get diagnostics v_requests = row_count;

    update tables set status = 'free' where id = p_table_id;

    return json_build_object('orders_closed', v_orders, 'requests_closed', v_requests);
  end $$;

  revoke execute on function public.close_table from anon, public;
  grant execute on function public.close_table to authenticated;
  ```

  - UI `CloseTableDialog` (виклик з `TableDetailSheet` 3.3): підсумок відкритих замовлень столу і загальна сума; `RadioGroup` способу оплати: «Готівка» (`cash`) / «Термінал» (`terminal`) / «Оплачено вручну» (`paid_manually`) — обов'язковий вибір, якщо є відкриті замовлення; якщо відкритих замовлень немає (лише виклики/occupied) — вибір прихований, передається `paid_manually` як технічне значення (замовлень для маркування все одно 0).
  - Виклик: `supabase.rpc('close_table', { p_table_id, p_payment })`; успіх → toast «Стіл N закрито (замовлень: X)»; помилка → toast з описом.
  - Realtime-оновлення (стіл → free, замовлення → completed, запити → handled) розлітаються підписникам автоматично через 3.2 — окремих refetch не потрібно.

- Acceptance Criteria:
  - Закриття встановлює `payment_status` УСІМ відкритим замовленням столу і лише їм (раніше завершені не змінюються).
  - Стіл стає `free`, відкриті запити — `handled`; операція атомарна (функція = одна транзакція).
  - `close_table` з `table_id` чужого ресторану → помилка `forbidden` (перевірити під waiter'ом B).
  - `p_payment='unpaid'` відхиляється; anon не може викликати функцію (revoke).
  - Обидва відкриті дашборди бачать звільнення столу < 2 с; повторне закриття вільного столу недоступне в UI.

### Задача 3.7

- Title: Управління меню (CRUD категорій і страв) + завантаження фото в Cloudflare R2

- Goal / Description:
  Як адміністратор закладу, я хочу самостійно керувати категоріями і стравами з фотографіями, щоб меню завжди було актуальним без участі розробника платформи.

- Technical Implementation Details for AI:
  - `src/lib/r2.ts` (`import 'server-only'`): `S3Client({ region: 'auto', endpoint: 'https://{R2_ACCOUNT_ID}.r2.cloudflarestorage.com', credentials })`; `createPresignedUploadUrl(key, contentType)` через `getSignedUrl(PutObjectCommand, { expiresIn: 300 })`; публічне читання — через `R2_PUBLIC_BASE_URL` (r2.dev або кастомний домен bucket'а).
  - `app/api/uploads/presign/route.ts` (POST): auth: server-клієнт `getUser` + `requireAdmin`; zod: `{ scope: 'menu-item' | 'logo', contentType in (image/jpeg|png|webp), size <= 5MB }`; key: `restaurants/{restaurantId}/{scope}/{crypto.randomUUID()}.{ext}` — **tenant-префікс обов'язковий** (`restaurantId` зі staff-контексту, не з body); відповідь `{ uploadUrl, publicUrl }`. Клієнт: `fetch(uploadUrl, { method: 'PUT', body: file })` → зберігає `publicUrl`.
  - R2 CORS (dev): AllowedOrigins `http://localhost:3000`, AllowedMethods `PUT`, AllowedHeaders `content-type` (prod-конфіг — задача 4.3); зафіксувати JSON конфігурації в коментарі `lib/r2.ts`.
  - `/dashboard/(admin)/menu/page.tsx`: master-detail: список категорій (name, sort_order input, `is_active` switch, delete з confirm «Разом з категорією буде видалено її страви») → страви вибраної категорії таблицею (мініатюра фото, name, price, `is_available` switch, sort_order, edit, delete з confirm).
  - Форми (Dialog + react-hook-form + zod): `CategoryForm { name 1..80, sort_order }`; `ItemForm { name 1..120, description ≤ 500, price ≥ 0 (decimal input, зберігати як рядок numeric), category select, image (presign-flow з preview і заміною), is_available, sort_order }`.
  - Мутації — Server Actions (`actions.ts` поряд зі сторінкою): `requireAdmin` + zod + server-клієнт користувача (RLS admin-політики = друга лінія захисту) → `revalidatePath('/dashboard/menu')`; guest-меню без кешу — зміни видно одразу після reload.
  - Видалення страви — hard delete: історія замовлень не ламається (`order_items.menu_item_id → set null`, snapshot лишається).

- Acceptance Criteria:
  - Адмін створює/редагує/ховає/видаляє категорії та страви; зміни одразу видно в guest-меню (reload).
  - Фото вантажиться напряму в R2 через presigned URL і рендериться в дашборді та guest-меню через `next/image`.
  - `/api/uploads/presign` без сесії → 401, під waiter → 403; ключ завжди з префіксом ресторану викликача.
  - Waiter не має доступу до розділу (403) і не може мутувати меню напряму (RLS).
  - Перемикання `is_available` прибирає страву з guest-меню; видалення страви не змінює позиції існуючих замовлень.

### Задача 3.8

- Title: Управління столами — створення, QR-коди і ротація токенів

- Goal / Description:
  Як адміністратор, я хочу створювати столи й отримувати для кожного QR-код і посилання з токеном, щоб роздрукувати QR-ки чи записати NFC-мітки та фізично розгорнути систему в залі.

- Technical Implementation Details for AI:
  - `/dashboard/(admin)/tables/page.tsx`: таблиця столів (номер, label, статус, `is_active` switch, дата створення) + дії.
  - Створення (Dialog + Server Action): `{ table_number: int > 0, label?: string ≤ 60 }`; `requireAdmin`; помилка `23505` (unique `(restaurant_id, table_number)`) → «Стіл з таким номером уже існує». Редагування label; деактивація (`is_active=false` → route handler 2.2 відхиляє доступ).
  - QR-Dialog («QR і посилання» на кожному столі): формує URL `http(s)://{NEXT_PUBLIC_ROOT_DOMAIN}/r/{slug}/table/{table_number}?token={access_token}` (канонічний формат з вимог; працює завжди, незалежно від custom-доменів); рендер QR клієнтською бібліотекою `qrcode` (`toDataURL`, 512px, margin 2); кнопки «Копіювати URL» (clipboard API) і «Завантажити PNG» (`<a download="table-{n}.png">`).
  - Ротація: кнопка «Перегенерувати токен» з confirm («Старі QR/NFC цього столу перестануть працювати») → Server Action: `crypto.randomBytes(24).toString('hex')` → update `tables.access_token`.
  - Примітка безпеки: `access_token` читається через RLS всім staff (політика `staff_select_tables`), але UI-розділ доступний лише admin (route group + `requireAdmin` в actions) — прийнятний компроміс MVP, зафіксувати коментарем.

- Acceptance Criteria:
  - Створений стіл одразу доступний за своїм QR-URL (e2e: сканування телефоном → меню з badge стола).
  - Дублікат номера дає зрозумілу помилку без 500.
  - PNG QR-коду сканується стандартною камерою телефона.
  - Після ротації старе посилання показує «недійсне», нове працює.
  - Деактивований стіл відхиляє доступ за токеном; waiter не має доступу до розділу.

### Задача 3.9

- Title: Налаштування ресторану — брендинг і персонал

- Goal / Description:
  Як адміністратор, я хочу самостійно налаштовувати фірмовий стиль (колір, логотип, шрифт) і керувати обліковими записами персоналу, щоб заклад був повністю самодостатнім без звернень до оператора платформи.

- Technical Implementation Details for AI:
  - `/dashboard/(admin)/settings/page.tsx` — Tabs «Брендинг» / «Персонал» (вкладку «Домен» додасть 4.4).
  - Брендинг:
    - форма: `primary_color` (`<input type="color">` + синхронне hex-поле, zod `^#[0-9a-fA-F]{6}$`), логотип (presign-flow з 3.7, `scope: 'logo'`, preview, кнопка «Прибрати лого» → `logo_url = null`), `font_family` — Select з allowlist 1.7 (+ «Системний»);
    - LivePreview-картка поруч (кнопка + badge, локально застосований колір через style-var);
    - Server Action: `requireAdmin` → upsert `restaurant_branding` (RLS admin-політики); guest-меню без кешу — зміни видно з наступного запиту.
  - Персонал:
    - таблиця `staff_users` свого ресторану (RLS): `full_name`, `email`, роль, `is_active`, дата;
    - «Додати співробітника» (Dialog): `{ full_name, email, password (кнопка «Згенерувати», показ один раз), role: waiter | admin }`; Server Action: `requireAdmin` → admin-клієнт `auth.admin.createUser({ email, password, email_confirm: true })` → INSERT `staff_users { id, restaurant_id: з контексту адміна, role, full_name, email }`; при збої INSERT — компенсаційний `auth.admin.deleteUser` (без «підвислих» auth-користувачів); дублікат email → зрозуміла помилка;
    - деактивація: switch `is_active` (Server Action; заборонено деактивувати себе) — `is_staff_of`/`is_admin_of` перевіряють `is_active`, тож доступ зникає з наступного запиту, а dashboard layout (1.5) викидає користувача;
    - зміна ролі: Select у рядку (заборонено знімати роль admin із самого себе).

- Acceptance Criteria:
  - Зміна кольору/лого/шрифту відображається на guest-меню; невалідний hex відхиляється формою і БД-constraint'ом.
  - Створений waiter логіниться і бачить лише операційні розділи.
  - Деактивований співробітник втрачає доступ (найпізніше — при наступній навігації по dashboard).
  - Адмін не може деактивувати себе або зняти з себе роль admin.
  - Помилка створення (дублікат email, збій insert) не залишає auth-користувача без рядка `staff_users`.

---

## Фаза 4 — Physical Hardware Integration & Multi-Domain Deployment (тижні 8–10)

### Задача 4.1

- Title: Endpoint `POST /api/hardware-call` з webhook-валідацією

- Goal / Description:
  Як система, я хочу приймати HTTP POST від фізичних Wi-Fi/IoT-кнопок, автентифікувати кожну кнопку секретом і створювати такий самий сервісний запит, як від цифрової кнопки, щоб виклик офіціанта працював навіть без смартфона гостя.

- Technical Implementation Details for AI:
  - `app/api/hardware-call/route.ts` (POST, `export const runtime = 'nodejs'`):
    - контракт: JSON `{ button_id: string }`; секрет — header `Authorization: Bearer {secret}` АБО поле `secret` у body (fallback для прошивок без кастомних headers); zod-валідація; ліміт body ~1KB;
    - admin-клієнт: `SELECT hardware_buttons + tables(is_active) + restaurants(is_active) WHERE button_id = ...`;
    - не знайдено АБО секрет не збігся (`crypto.timingSafeEqual` після перевірки довжини) → **однакова** відповідь 401 `{ error: 'unauthorized' }` (без enumeration, без деталей);
    - кнопка/стіл/ресторан неактивні → 403 `{ error: 'inactive' }`;
    - успіх: INSERT `service_requests { restaurant_id, table_id, type: request_type кнопки, source: 'hardware' }` (усе мапиться З РЯДКА КНОПКИ, нічого з body); `23505` → 200 `{ status: 'already_open' }`; інакше 201 `{ status: 'created' }`; в обох випадках `UPDATE hardware_buttons SET last_seen_at = now()` і `tables → occupied`;
    - інші методи → 405; ціль p95 відповіді < 500 мс (2–3 індексовані запити), без redirect'ів (прошивки їх не следують).
  - Дашборд отримує подію автоматично через realtime 3.2; badge «Кнопка» вже підтриманий у 3.4.
  - Додати в код задокументовані curl-приклади: валідний виклик, невірний секрет, повторне натискання.

- Acceptance Criteria:
  - curl з коректними `button_id` + secret створює `service_requests` правильного типу для правильного столу; алерт на дашборді < 2 с.
  - Невірний секрет і невідомий `button_id` дають ідентичну відповідь 401.
  - Неактивна кнопка/стіл/ресторан → 403; повторне натискання при відкритому запиті → 200 `already_open` без дубля в БД.
  - `last_seen_at` оновлюється при кожному автентифікованому виклику (включно з `already_open`).
  - Секрет через body-поле працює так само, як через header.

### Задача 4.2

- Title: UI конфігурації фізичних кнопок (реєстрація і мапінг на столи)

- Goal / Description:
  Як адміністратор, я хочу реєструвати фізичні кнопки, прив'язувати їх до столів і типу запиту та бачити час останнього сигналу, щоб розгортати й діагностувати обладнання в залі без розробника.

- Technical Implementation Details for AI:
  - `/dashboard/(admin)/hardware/page.tsx`: таблиця кнопок: `button_id`, стіл, тип запиту, `is_active` switch, `last_seen_at` («ніколи» / відносний час; зелений індикатор, якщо < 24 год), дії; кнопка «Оновити» (`router.refresh()` — realtime для цієї таблиці не потрібен).
  - «Зареєструвати кнопку» (Dialog + Server Action `requireAdmin`): `{ button_id: zod 3..64 [a-zA-Z0-9_-] (MAC/серійник пристрою), table: select активних столів, request_type: radio call_waiter | request_bill }`; `secret` генерує DB default; `23505` → «Такий button_id вже зареєстровано».
  - «Конфігурація пристрою» (Dialog на кожному рядку): endpoint URL `https://{ROOT_DOMAIN}/api/hardware-call`, `button_id`, secret (з кнопкою copy), готовий curl-приклад і JSON-payload для прошивки.
  - «Перегенерувати секрет» (Server Action з confirm) → новий `crypto.randomBytes(24).toString('hex')`; старий секрет миттєво невалідний.
  - Редагування прив'язки (стіл, тип, `is_active`) — Server Actions (RLS `admin_write_buttons` — друга лінія); видалення кнопки з confirm.

- Acceptance Criteria:
  - Зареєстрована кнопка одразу приймається endpoint'ом (e2e: curl зі скопійованої конфігурації).
  - Зміна прив'язки стола/типу змінює адресацію наступних викликів.
  - Після перегенерації секрета старий → 401, новий працює.
  - Деактивація → 403 на виклики; `last_seen_at` відображається після refresh.
  - Waiter не має доступу до розділу (403).

### Задача 4.3

- Title: Production-деплой на Vercel — wildcard-піддомени, env, R2 CORS

- Goal / Description:
  Як система, я хочу працювати на продакшн-домені з wildcard-піддоменами для tenant'ів і коректно налаштованими Supabase/R2, щоб кожен ресторан автоматично отримував `{slug}.platform.com` без ручних дій у Vercel.

- Technical Implementation Details for AI:
  - Vercel: проєкт з git-інтеграцією; всі env-змінні з `.env.example` з prod-значеннями; `NEXT_PUBLIC_ROOT_DOMAIN` = продакшн-домен (без протоколу); `TABLE_SESSION_SECRET` — 32+ випадкових байтів (`openssl rand -hex 32`).
  - Supabase production: окремий проєкт; `supabase link` + `supabase db push` (усі міграції); `pnpm seed` як операторська дія; Auth: Site URL = `https://{ROOT_DOMAIN}`, signups вимкнені (як у 1.5).
  - Домени у Vercel: apex `platform.com`, `www.platform.com`, wildcard `*.platform.com`; wildcard вимагає делегування DNS домену на Vercel nameservers (`ns1.vercel-dns.com`, `ns2.vercel-dns.com`) — виконати і зафіксувати кроки в README-блоці репозиторію; TLS-сертифікати (включно з wildcard) Vercel видає автоматично.
  - R2 production: bucket + публічне читання (custom domain типу `images.platform.com` або `r2.dev`) → `R2_PUBLIC_BASE_URL`; CORS: `AllowedOrigins: ["https://{ROOT_DOMAIN}"]` (аплоади йдуть лише з дашборда на root-домені), `AllowedMethods: ["PUT"]`, `AllowedHeaders: ["content-type"]`; `next.config.ts` `images.remotePatterns` — prod-хост R2.
  - Прод-верифікація (чекліст у задачі): логін персоналу; guest-flow на `https://{slug}.platform.com/table/{n}?token=...` і на `https://{ROOT_DOMAIN}/r/{slug}/...`; realtime на два пристрої; `hardware-call` через curl на prod; QR-URL з 3.8 генеруються з prod-домену; cookie з атрибутами `Secure; HttpOnly; SameSite=Lax`; security headers присутні у відповідях.

- Acceptance Criteria:
  - `https://{ROOT_DOMAIN}/r/demo-bistro` і `https://demo-bistro.{ROOT_DOMAIN}` віддають брендоване меню.
  - Повний цикл гість → офіціант працює на проді: виклик, замовлення, realtime-алерт зі звуком, обробка, закриття столу.
  - Невідомий піддомен → сторінка `domain-not-found`; фото меню віддаються з R2 по HTTPS і рендеряться.
  - Завантаження фото працює з прод-дашборда (CORS правильний).
  - Усі cookies на проді `Secure` + `HttpOnly`; TLS активний для apex, www і wildcard.

### Задача 4.4

- Title: Custom-домени ресторанів — Vercel Domains API, DNS-інструкції, верифікація

- Goal / Description:
  Як адміністратор закладу, я хочу підключити власний домен (наприклад, `bistro-name.com`) до свого меню, щоб гості бачили виключно мій бренд без згадки платформи.

- Technical Implementation Details for AI:
  - `src/lib/vercel.ts` (`import 'server-only'`; env `VERCEL_TOKEN`, `VERCEL_PROJECT_ID`, опц. `VERCEL_TEAM_ID` як query `?teamId=`):
    - `addDomain(name)` → `POST https://api.vercel.com/v10/projects/{PROJECT_ID}/domains`, body `{ name }`;
    - `getDomainStatus(name)` → `GET /v9/projects/{PROJECT_ID}/domains/{name}` (поле `verified`) + `GET /v6/domains/{name}/config` (поле `misconfigured`);
    - `removeDomain(name)` → `DELETE /v9/projects/{PROJECT_ID}/domains/{name}`.
  - Вкладка «Домен» у settings (розширення 3.9): список доменів ресторану (`restaurant_domains` + status badge «Очікує DNS» / «Активний» за `verified_at`); форма додавання: zod — lowercase, без протоколу/шляху/пробілів, містить крапку; блокувати `{ROOT_DOMAIN}` і будь-який `*.{ROOT_DOMAIN}`; ліміт 3 домени на ресторан.
  - Server Action додавання: `requireAdmin` → `addDomain` у Vercel → INSERT `restaurant_domains` admin-клієнтом (`23505` → «Домен уже використовується іншим закладом»); при збої INSERT — компенсаційний `removeDomain` (без розсинхрону Vercel ↔ БД).
  - DNS-інструкції в UI після додавання: apex-домен → `A 76.76.21.21`; субдомен (≥ 3 лейбли) → `CNAME cname.vercel-dns.com`; показувати конкретний варіант за формою домену.
  - Кнопка «Перевірити»: Server Action → `getDomainStatus` → `verified && !misconfigured` → `UPDATE restaurant_domains SET verified_at = now()`; інакше — актуальна підказка.
  - Видалення (confirm): `removeDomain` + DELETE рядка.
  - Middleware (1.6) вже резолвить `restaurant_domains` з кешем 60 с — новий верифікований домен починає працювати без деплою.
  - `VERCEL_TOKEN` ніколи не потрапляє на клієнт; усі виклики Vercel API — лише в Server Actions.

- Acceptance Criteria:
  - Додавання домену створює його у Vercel-проєкті та в `restaurant_domains`; збій на будь-якому кроці не залишає часткового стану.
  - UI показує коректні DNS-записи для apex і субдомена.
  - Після налаштування DNS «Перевірити» ставить `verified_at`, і домен віддає брендоване меню саме цього ресторану (e2e з реальним тестовим доменом).
  - Спроба додати зайнятий домен або `*.{ROOT_DOMAIN}` → зрозуміла помилка.
  - Видалення прибирає домен і з Vercel, і з БД; він перестає резолвитись після TTL кешу (≤ 60 с). Waiter доступу не має.

### Задача 4.5

- Title: Security-верифікація і production hardening (реліз-gate)

- Goal / Description:
  Як система, я хочу автоматизовану перевірку tenant-ізоляції, RLS-політик, табличних токенів і webhook-автентифікації, щоб перед запуском гарантувати, що жоден критичний захист не був послаблений під час розробки.

- Technical Implementation Details for AI:
  - `scripts/security-check.ts` (`pnpm security:check`; env-параметри: base URL + креденшали seed-акаунтів обох ресторанів; працює і локально, і проти staging/prod). Кожна перевірка — assert очікуваної ВІДМОВИ або успіху; вивід — таблиця PASS/FAIL; ненульовий exit code при будь-якому FAIL:
    1. **anon**: `select` з `tables`, `service_requests`, `orders`, `order_items`, `staff_users`, `hardware_buttons` → 0 рядків; прямий `insert` у `service_requests`/`orders` → RLS-помилка; `select restaurants/menu_items` → лише активні/доступні.
    2. **waiter A проти tenant B**: `select`/`update` чужих `service_requests`/`orders`/`order_items`/`tables` → 0 рядків / 0 affected; `rpc close_table` зі столом B → `forbidden`.
    3. **ролі**: waiter A `insert/update` у `menu_items`/`menu_categories`/`hardware_buttons` свого ресторану → RLS-помилка (лише admin); admin A не може писати `restaurant_branding`/`staff_users` ресторану B.
    4. **guest API**: POST `/api/service-requests` і `/api/orders` без cookie → 401; з cookie з підробленим підписом → 401; з простроченим `exp` → 401; `/api/orders` з `menu_item_id` чужого ресторану → 400.
    5. **table URL**: невірний token → відповідь без `Set-Cookie`; неактивний стіл → відмова.
    6. **hardware**: невірний секрет / невідомий `button_id` → 401 (ідентичні); неактивна кнопка → 403; повторний виклик → без другого відкритого рядка.
    7. **uploads**: `/api/uploads/presign` без сесії → 401, під waiter → 403.
    8. **dashboard**: GET `/dashboard` без сесії → redirect на `/login`.
  - Hardening-чекліст (виконати в межах задачі, зафіксувати результати в описі PR/коміта):
    - `SUPABASE_SERVICE_ROLE_KEY`, `TABLE_SESSION_SECRET`, `VERCEL_TOKEN`, R2-ключі відсутні в клієнтському бандлі (grep по `.next/static`);
    - `admin.ts`, `r2.ts`, `vercel.ts`, `table-session.ts` мають `import 'server-only'`;
    - усі route handlers валідують метод і zod-схему; помилки не витікають stack trace назовні;
    - cookies: `HttpOnly`, `Secure` (prod), `SameSite=Lax`; security headers з 1.1 присутні;
    - Supabase Auth: signups вимкнені, leaked password protection увімкнено.
  - Знайдені FAIL виправляються в межах цієї задачі; реліз MVP дозволений лише при 100% PASS.

- Acceptance Criteria:
  - `pnpm security:check` дає 100% PASS локально і на production-оточенні.
  - Hardening-чекліст повністю виконаний і зафіксований.
  - Виявлені під час перевірки дірки виправлені, скрипт перезапущений до зеленого стану.
  - Скрипт задокументований (як запускати, які env потрібні) у README-блоці репозиторію.

---

## Definition of Done для MVP

MVP вважається завершеним, коли на production одночасно виконується:

- [ ] Гість сканує QR/NFC → бачить брендоване меню свого закладу → кличе офіціанта / просить рахунок → надсилає замовлення (без оплати онлайн).
- [ ] Офіціант на дашборді отримує звуковий і візуальний алерт < 2 с, бачить активні столи, обробляє запити, коригує ціни позицій, закриває стіл із способом оплати (cash / terminal / paid_manually).
- [ ] Фізична кнопка через `POST /api/hardware-call` створює той самий сервісний запит, що й цифрова.
- [ ] Ресторани працюють на `{slug}.platform.com` і на власних custom-доменах; брендинг (колір, лого, шрифт) — індивідуальний для кожного tenant'а.
- [ ] Адмін закладу самостійно керує меню (з фото на R2), столами (QR/токени), персоналом, брендингом, hardware-кнопками і доменом.
- [ ] `pnpm security:check` — 100% PASS: tenant-ізоляція, RLS, токени столів, auth-захист і webhook-валідація підтверджені.
