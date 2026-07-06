# Song Soliloquy Patch v3.4
## Library Admin Governance: Remove, Archive, Reorder, Replace, and Re-upload Songs

**Patch type:** Additive admin/library management patch  
**Status:** Ready for Figma Make / Builder and GitHub archive  
**Priority:** High after parser/content-integrity patch v3.3  
**Primary goal:** Give the owner/admin full control over songs in the Library without destabilizing upload, parsing, SongDetail, Supabase, theme toggle, OBS, Live Studio, motion, or overlay systems.

---

## Diagnosis

Song Soliloquy currently has a strong upload-to-library-to-song-page flow, but it needs a real administrative process for Library management.

The missing layer is not just a delete button. The site needs **Library Governance**:

- Remove a song from the public Library.
- Archive older songs without permanently deleting them.
- Restore archived songs.
- Permanently delete songs only with extra confirmation.
- Re-upload corrected song files without creating confusing duplicates.
- Replace an existing song analysis with a corrected version.
- Sort, pin, arrange, and filter the Library.
- Handle local preview storage now and Supabase storage later.

This is especially important after the content-integrity/parser fix, because older polluted records may need to be removed or replaced.

---

## Product Rule

```txt
The Library is not just a list of songs.
The Library is an admin-managed archive.
```

A song should have a lifecycle:

```txt
Draft -> Published -> Archived -> Restored or Permanently Deleted
```

Default removal should be **Archive**, not hard delete.

Permanent deletion should exist, but it should be protected.

---

# ACTION — Paste into Figma Make / Builder

```text
Add a Library Admin Governance system to Song Soliloquy.

The current app lets uploaded songs enter the Library, but there is no clear administrative process for removing songs, archiving old songs, arranging the Library, or replacing/re-uploading corrected files. Add this feature without changing the approved layout, song page design, parser architecture, slug resolver, Supabase plan, OBS integration, Live Studio, motion layer, or theme toggle.

Treat this as an additive admin/library management patch.

Core requirements:

1. Add song lifecycle fields to every song record:
   - status: "draft" | "published" | "archived" | "deleted"
   - visibility: "private" | "unlisted" | "public"
   - isPinned: boolean
   - sortOrder: number | null
   - archivedAt: string | null
   - deletedAt: string | null
   - updatedAt: string
   - createdAt: string
   - replacedById: string | null
   - previousVersionIds: string[]
   - adminNotes: string | null

2. Add Library admin actions:
   - Archive Song
   - Restore Song
   - Remove from Library
   - Permanently Delete Song
   - Replace with New Upload
   - Re-upload Corrected File
   - Edit Metadata
   - Pin / Unpin
   - Move Up
   - Move Down
   - Set Custom Order
   - Add to Collection
   - Remove from Collection

3. In the public Library view, show only songs where:
   - status is "published"
   - visibility is "public"
   - deletedAt is null

4. In Admin Library view, add filters:
   - All Songs
   - Published
   - Drafts
   - Archived
   - Deleted / Trash
   - Pinned
   - Needs Review
   - Parser Warnings
   - Duplicate Content Warnings
   - Recently Uploaded
   - Oldest Songs

5. Add an Admin Mode toggle or route:
   - /admin/library
   - or Settings -> Admin -> Library Management

6. Add row/card-level overflow menu on Library song cards:
   - Open Song Page
   - Open in Live Studio
   - Edit Metadata
   - Replace Analysis File
   - Archive
   - Delete
   - Pin / Unpin

7. Add a confirmation modal for Archive:
   Title: "Archive this song?"
   Copy: "Archived songs are removed from the public Library but can be restored later."
   Buttons:
   - Cancel
   - Archive Song

8. Add a stronger confirmation modal for Permanent Delete:
   Title: "Permanently delete this song?"
   Copy: "This removes the song analysis, parsed content, Library entry, and associated admin data. This cannot be undone."
   Required typed confirmation:
   - DELETE
   Buttons:
   - Cancel
   - Permanently Delete

9. Add a Replace Analysis workflow:
   If a user uploads a file with the same canonicalSlug or same artist+title as an existing song, show a conflict screen:
   - Existing song found
   - Replace existing analysis
   - Save as new version
   - Cancel

   Default recommended action: Replace existing analysis.

10. When replacing a song:
   - Keep the same canonicalSlug unless the admin changes it.
   - Preserve the public URL.
   - Update lyrics, bar analysis, key bars, hottest bar, NotebookLM pack, YouTube angle, figurative language, context cards, and parser hashes.
   - Add previous version to previousVersionIds.
   - Store replacedAt timestamp.
   - Show a success message: "Analysis replaced. Public song page updated."

11. Add local preview support now:
   For Figma/local preview, all admin actions must update localStorage key:
   - song-soliloquy.uploadedSongs.v1

   Archive should set status to "archived".
   Delete should set status to "deleted" and deletedAt.
   Permanent Delete should remove the record entirely.

12. Keep old compatibility key cleanup:
   If legacy key ss_uploaded_analyses exists, migration should either move records into song-soliloquy.uploadedSongs.v1 or clear it after confirmation.

13. Add Supabase-ready data model later:
   Add or prepare tables:
   - songs
   - song_versions
   - song_collections
   - song_collection_items
   - song_admin_events

14. Supabase deletion rules:
   - Archive: update status = archived, archived_at = now()
   - Soft delete: update status = deleted, deleted_at = now()
   - Permanent delete: admin-only action; delete database rows and storage files only after confirmation
   - Public users can only read published/public/non-deleted songs
   - Only admins can archive, restore, replace, or delete

15. Add Admin Activity Log:
   Each admin action should create an event:
   - song_created
   - song_updated
   - song_archived
   - song_restored
   - song_soft_deleted
   - song_permanently_deleted
   - song_replaced
   - song_pinned
   - song_unpinned
   - song_reordered

16. Add Library arrangement controls:
   - Sort by newest
   - Sort by oldest
   - Sort by artist
   - Sort by title
   - Sort by project/album
   - Sort by release date
   - Sort by last updated
   - Sort by custom order
   - Pinned first toggle

17. Add Collections as future-safe structure:
   Collections can include:
   - Ransom
   - Skyzoo
   - Best Hottest Bars
   - Spiritual Conflict
   - Street Wisdom
   - Live Show Queue
   - Needs Re-upload
   - Published Episodes

18. Add a Live Show Queue collection:
   This should support OBS/Live Studio later without changing the Library.
   Admin can add songs to Live Show Queue from Library or SongDetail.

19. Add bulk actions in Admin Library:
   - Select multiple songs
   - Archive selected
   - Restore selected
   - Delete selected
   - Add selected to collection
   - Remove selected from collection
   - Export selected

20. Add safety guardrails:
   - Do not permanently delete demo songs unless developer mode is enabled.
   - Do not delete Supabase files from Storage unless permanent delete is confirmed.
   - Do not remove a song while a Live Studio session is actively using it; show warning.
   - If a song is archived, keep existing shared URL but show "This analysis is archived" to admins.

21. Add UI labels:
   Public Library button: "Library"
   Admin route: "Admin Library"
   Archived filter: "Archived"
   Trash filter: "Trash"
   Replace button: "Replace Analysis File"
   Re-upload button: "Re-upload Corrected File"

22. Add SongDetail admin controls:
   On SongDetail, show admin-only buttons:
   - Archive
   - Replace File
   - Edit Metadata
   - Add to Live Queue
   - View Admin History

23. Add Upload Conflict/Replacement screen:
   When a new upload matches an existing song, show:
   - Existing title and artist
   - Existing canonicalSlug
   - Existing last updated date
   - New parsed title and artist
   - New parser warnings
   - Buttons: Replace Existing, Save as New, Cancel

24. Acceptance criteria:
   - Admin can archive a song from Library.
   - Archived songs disappear from public Library but appear in Admin Library -> Archived.
   - Admin can restore an archived song.
   - Admin can soft-delete a song into Trash.
   - Admin can permanently delete a song only after typing DELETE.
   - Admin can re-upload a corrected file and replace the existing song page without changing the public slug.
   - Library can be sorted and pinned.
   - Existing upload, Library click, SongDetail resolver, parser, theme toggle, OBS, Live Studio, and motion layer remain unaffected.
   - The public user never sees admin-only controls.
```

---

## Data Model Additions

### Song Record Fields

```ts
export type SongStatus = "draft" | "published" | "archived" | "deleted";
export type SongVisibility = "private" | "unlisted" | "public";

export interface SongAnalysis {
  id: string;
  title: string;
  artist: string;
  canonicalSlug: string;
  slugAliases: string[];

  status: SongStatus;
  visibility: SongVisibility;
  isPinned: boolean;
  sortOrder: number | null;

  createdAt: string;
  updatedAt: string;
  archivedAt: string | null;
  deletedAt: string | null;

  sourceFilename?: string;
  rawMarkdownHash?: string;
  contentHash?: string;

  replacedById: string | null;
  previousVersionIds: string[];
  adminNotes: string | null;
}
```

---

## Local Preview Store Functions

```ts
const UPLOADED_SONGS_KEY = "song-soliloquy.uploadedSongs.v1";

export function archiveSong(songId: string) {
  updateSong(songId, {
    status: "archived",
    archivedAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  });
}

export function restoreSong(songId: string) {
  updateSong(songId, {
    status: "published",
    archivedAt: null,
    deletedAt: null,
    updatedAt: new Date().toISOString(),
  });
}

export function softDeleteSong(songId: string) {
  updateSong(songId, {
    status: "deleted",
    deletedAt: new Date().toISOString(),
    updatedAt: new Date().toISOString(),
  });
}

export function permanentlyDeleteSong(songId: string) {
  const songs = getUploadedSongAnalyses();
  const next = songs.filter((song) => song.id !== songId);
  localStorage.setItem(UPLOADED_SONGS_KEY, JSON.stringify(next));
}
```

---

## Supabase Schema Patch

```sql
alter table public.songs
add column if not exists status text not null default 'draft',
add column if not exists visibility text not null default 'private',
add column if not exists is_pinned boolean not null default false,
add column if not exists sort_order integer,
add column if not exists archived_at timestamptz,
add column if not exists deleted_at timestamptz,
add column if not exists replaced_by_id uuid references public.songs(id),
add column if not exists previous_version_ids uuid[] not null default '{}',
add column if not exists admin_notes text;

create table if not exists public.song_admin_events (
  id uuid primary key default gen_random_uuid(),
  song_id uuid references public.songs(id) on delete cascade,
  event_type text not null,
  event_payload jsonb not null default '{}'::jsonb,
  created_by uuid,
  created_at timestamptz not null default now()
);

create table if not exists public.song_collections (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  slug text not null unique,
  description text,
  visibility text not null default 'private',
  sort_order integer,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now()
);

create table if not exists public.song_collection_items (
  id uuid primary key default gen_random_uuid(),
  collection_id uuid not null references public.song_collections(id) on delete cascade,
  song_id uuid not null references public.songs(id) on delete cascade,
  sort_order integer,
  created_at timestamptz not null default now(),
  unique(collection_id, song_id)
);
```

---

## Supabase RLS Policy Direction

Public reads should only see public, published, non-deleted songs:

```sql
create policy "Public can read published songs"
on public.songs
for select
using (
  status = 'published'
  and visibility = 'public'
  and deleted_at is null
);
```

Admin-only policies should control insert/update/delete. Exact implementation depends on the Homeland.ai auth/admin role model.

---

## UX Recommendation

Use **Archive** as the main removal action.

Use **Delete** as a secondary admin action.

Use **Permanent Delete** only inside Trash with typed confirmation.

This prevents accidental loss while still keeping the public Library clean.

---

## Placement in App

### Top Navigation

Add only if admin mode is available:

```txt
Admin
```

### Library Card Overflow Menu

```txt
Open Song Page
Open in Live Studio
Edit Metadata
Replace Analysis File
Archive
Delete
Pin / Unpin
```

### Admin Library Sidebar

```txt
All Songs
Published
Drafts
Archived
Trash
Needs Review
Parser Warnings
Duplicate Content
Pinned
Collections
Live Show Queue
```

---

## Re-upload Corrected File Flow

```txt
Upload file
-> Parse Preview
-> Existing song match detected
-> Admin chooses Replace Existing
-> Existing public slug preserved
-> Song content replaced
-> Previous version stored
-> SongDetail page updates
-> Admin event logged
```

This is essential for repairing songs affected by older parser bugs.

---

## Do Not Change

This patch must not alter:

- Song page layout
- Theme skins
- Motion layer
- OBS integration
- Live Studio routes
- Overlay routes
- SongDetail resolver contract
- Content-integrity parser patch
- Supabase connection strategy
- NotebookLM export
- YouTube angle module

---

## Definition of Done

This patch is complete when:

- Songs can be archived.
- Songs can be restored.
- Songs can be soft-deleted.
- Songs can be permanently deleted with typed confirmation.
- Corrected files can replace existing song analyses.
- Library can be sorted, pinned, and filtered.
- Admin Library separates active songs from archived/trash records.
- Public Library remains clean.
- Uploaded song data remains stable after refresh.
- SongDetail does not break when a song is archived or deleted.
- Supabase-ready schema exists for future persistence.

