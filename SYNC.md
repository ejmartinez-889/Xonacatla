# English translation sync (developer workflow)

Spanish is the source language of xonacatla.com. The client edits Spanish text
in `admin.html`; English lags behind by design. Whenever a Spanish block (`es`)
changes and its English (`en`) does not, a database trigger flips
`en_stale = true` on that row of the `content` table (Supabase project
**Xonacatla-Demo**, ref `vyarqorszxrsvpappcju`). The admin page shows those rows
with a "traducción pendiente" badge.

## The loop

1. **Find stale rows:**

   ```sql
   select key, es, en from public.content where en_stale = true order by key;
   ```

2. **Translate / adapt** each `es` value to English. Keep any inline HTML
   (e.g. `<em>…</em>`) and emoji intact. These are adaptations, not literal
   translations — match the tone of the existing `en` strings.

3. **Write back the translation and clear the flag** (one statement, so the
   trigger doesn't re-flag it):

   ```sql
   update public.content
   set en = 'New English text…', en_stale = false
   where key = 'the_key';
   ```

4. Verify nothing is left: the first query should return zero rows, and the
   badge disappears from `admin.html`.

Run the SQL from the Supabase dashboard SQL editor or via the Supabase MCP
tools. The public site picks up changes on the next page load — no deploy
needed, since `index.html` hydrates all text from the `content` table at
runtime (the strings hardcoded in the HTML are only offline fallbacks; keep
them roughly in sync when translations change significantly, but that is
optional).
