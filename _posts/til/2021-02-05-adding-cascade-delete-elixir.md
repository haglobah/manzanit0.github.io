---
layout: post
title: Adding DELETE CASCADE via Ecto Migrations
author: Javier Garcia
category: elixir
tags: elixir, ecto
---

Say you have a relationship in your PG database that doesn't have cascade
deletion, and you want to add it. This is how:

```elixir
defmodule MyApp.Repo.Migrations.CascadeDelete do
  use Ecto.Migration

  def change do
    drop constraint(:emails, :emails_timeline_event_id_fkey)

    alter table(:emails) do
      modify :timeline_event_id, references(:timeline_events, type: :uuid, on_delete: :delete_all)
    end
  end
end
```

You simply have to drop the foreign key constraint first, and then re-create it
with the `CASCADE DELETE` option.
