# Shared posts setup

The community feed in Posts & Shorts needs a Supabase project and a public HTTPS
host for this app. Until configured, local videos continue to work and shared
posts remain disabled.

## 1. Configure Supabase

Create a Supabase project, then open its SQL Editor and run:

```sql
create extension if not exists pgcrypto;

create table if not exists public.social_posts (
    id uuid primary key default gen_random_uuid(),
    author text not null check (char_length(author) between 1 and 32),
    body text not null default '' check (char_length(body) <= 1000),
    media_path text,
    media_type text check (media_type in ('image', 'video') or media_type is null),
    created_at timestamptz not null default now(),
    constraint social_posts_body_or_media check (body <> '' or media_path is not null)
);

alter table public.social_posts enable row level security;

drop policy if exists "Anyone can read shared posts" on public.social_posts;
create policy "Anyone can read shared posts"
    on public.social_posts for select to anon, authenticated using (true);

drop policy if exists "Anyone can publish a post" on public.social_posts;
create policy "Anyone can publish a post"
    on public.social_posts for insert to anon, authenticated
    with check (
        char_length(author) between 1 and 32
        and char_length(body) <= 1000
        and (body <> '' or media_path is not null)
    );

insert into storage.buckets (id, name, public, file_size_limit, allowed_mime_types)
values (
    'slotphone-posts',
    'slotphone-posts',
    true,
    20971520,
    array['image/jpeg', 'image/png', 'image/webp', 'image/gif', 'video/mp4', 'video/webm', 'video/quicktime']
)
on conflict (id) do update set
    public = excluded.public,
    file_size_limit = excluded.file_size_limit,
    allowed_mime_types = excluded.allowed_mime_types;

drop policy if exists "Anyone can upload community media" on storage.objects;
create policy "Anyone can upload community media"
    on storage.objects for insert to anon, authenticated
    with check (bucket_id = 'slotphone-posts');
```

In the Supabase dashboard, copy the **Project URL** and the **anon/public key**
(or publishable key) into `supabase-config.js`:

```js
window.SLOTPHONE_SUPABASE = {
    url: "https://YOUR-PROJECT.supabase.co",
    anonKey: "YOUR-PUBLIC-KEY"
};
```

Never put a `service_role` or secret key in this file. Browser configuration is
public by design. Row-level security restricts this app to reading and adding
posts; it does not grant browser users permission to delete posts.

## 2. Host and share

Host the root `index.html`, `script.js`, `style.css`, `supabase-config.js`, and
the referenced assets on an HTTPS static host. Send your friend that hosted
URL. Both devices must use the same configured Supabase project. New posts are
visible to the community feed and refresh automatically while it is open.

Posts and uploaded media are public to anyone with access to the project. The
feed has no sign-in or moderation, so share only content you are comfortable
making public. Supabase enforces a 20 MB per-file upload limit.
