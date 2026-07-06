# Song Soliloquy Patch v3.7
## Delete / Permanent Delete Confirmation Action Fix

## Purpose
Fix the Admin Library delete and permanent-delete flows so that songs can actually be removed, archived, soft-deleted, or permanently deleted after confirmation.

## Diagnosis
The current modal accepts typed confirmation text, but the destructive action button remains disabled even after the user types `DELETE`.

Likely causes:

1. Confirmation comparison is too strict or mismatched.
   - Expected value may be `delete`, `Delete`, song title, slug, or another token.
   - User interface says type `DELETE`, but button logic may check something else.

2. Input state is not bound correctly.
   - The visible input updates, but the modal action button reads stale or different state.

3. Whitespace/case handling is missing.
   - `DELETE`, `DELETE `, or pasted text should normalize before comparison.

4. Button disabled state is checking the wrong condition.
   - Example: disabled until `selectedSong.status === "deleted"`, which would never be true before the action.

5. The soft-delete and permanent-delete flows are sharing modal state incorrectly.
   - One modal may expect `DELETE`; the other may expect a different confirmation phrase.

6. The destructive action handler is not wired to the modal button.
   - The button may visually render but never call `softDeleteSong`, `permanentlyDeleteSong`, or `removeUploadedSongAnalysis`.

## Product Rule
Deleting must be administrative, reversible by default, and clear.

Recommended lifecycle:

- `Archive` removes from public Library but can be restored.
- `Delete` moves the song to Trash with `status: "deleted"` and `deletedAt`.
- `Permanent Delete` removes the song record entirely after typed confirmation.

## Required UX Behavior

### Delete / Move to Trash
The normal Delete action should show a confirmation modal:

Title:
`Move this song to Trash?`

Copy:
`This removes the song from the Library, but it can be restored from Trash.`

Buttons:
- Cancel
- Move to Trash

This should not require typing `DELETE` unless the product owner wants extra friction. Soft delete is reversible.

### Permanent Delete
The permanent delete action should require typed confirmation.

Title:
`Permanently delete this song?`

Copy:
`This removes the song analysis, parsed content, Library entry, and associated admin data. This cannot be undone.`

Input label:
`Type DELETE to confirm`

Validation:
- Normalize the input with `.trim().toUpperCase()`.
- Enable the destructive button only when normalized input equals `DELETE`.

Button:
`Permanently Delete`

## Implementation Requirements

### 1. Centralize confirmation logic
Create a helper:

```ts
export function normalizeDeleteConfirmation(value: string): string {
  return value.trim().toUpperCase();
}

export function canConfirmPermanentDelete(value: string): boolean {
  return normalizeDeleteConfirmation(value) === "DELETE";
}
```

### 2. Fix modal state binding
The input field must be controlled by the same state used by the button disabled condition.

```tsx
const [confirmationText, setConfirmationText] = useState("");
const canDelete = canConfirmPermanentDelete(confirmationText);
```

The button should use:

```tsx
<button
  type="button"
  disabled={!canDelete || isDeleting}
  onClick={handlePermanentDelete}
>
  Permanently Delete
</button>
```

### 3. Reset state on modal open/close
When opening or closing any delete modal:

```ts
setConfirmationText("");
setDeleteError(null);
setIsDeleting(false);
```

### 4. Wire the action handler
Permanent delete must actually call the store mutation.

Preview/local mode:

```ts
permanentlyRemoveUploadedSongAnalysis(song.id);
```

or:

```ts
removeUploadedSongAnalysis(song.id, { permanent: true });
```

Soft delete:

```ts
updateUploadedSongAnalysis(song.id, {
  status: "deleted",
  deletedAt: new Date().toISOString(),
  updatedAt: new Date().toISOString(),
});
```

Archive:

```ts
updateUploadedSongAnalysis(song.id, {
  status: "archived",
  archivedAt: new Date().toISOString(),
  updatedAt: new Date().toISOString(),
});
```

### 5. Update UI after mutation
After delete/permanent delete:

- Close modal.
- Refresh Library state from `getAllSongAnalyses()` or the uploaded-song store.
- Remove the song from the current visible list.
- Show a toast:
  - `Song moved to Trash.`
  - `Song permanently deleted.`

### 6. Make deleted songs re-uploadable
When uploading a file whose canonical slug matches a deleted song:

Show conflict options:

- Restore and Replace Existing
- Save as New Version
- Permanently Remove Old Record and Import Fresh
- Cancel

If the song was permanently deleted, normal upload should create a clean new record.

### 7. Protect demo songs
Do not permanently delete demo/static songs unless developer mode is enabled.

If the selected song is demo/static:

- Disable permanent delete.
- Show: `Demo songs cannot be permanently deleted in preview mode.`

Uploaded songs are allowed to be permanently deleted.

## ACTION — Paste into Figma Make / Builder

```text
Fix the Admin Library delete and permanent-delete confirmation flow.

Current issue: the Permanent Delete modal says “Type DELETE to confirm,” and the user typed DELETE, but the Permanently Delete button stays disabled. This prevents songs from being deleted or re-uploaded cleanly.

Treat this as an admin action-state bug, not a visual design issue, parser issue, upload issue, or routing issue.

Do not change the approved layout, parser, SongDetail resolver, theme toggle, OBS integration, motion layer, Supabase plan, or Library design. Only fix the delete action logic.

Requirements:

1. Fix the permanent delete confirmation logic.
   The modal input should enable the Permanently Delete button when:
   confirmationText.trim().toUpperCase() === "DELETE"

2. Make sure the visible input field and the button disabled state use the same React state.
   Use a controlled input:
   value={confirmationText}
   onChange={(event) => setConfirmationText(event.target.value)}

3. Do not compare the input against the song title, slug, lowercase delete, or any hidden value if the UI says “Type DELETE to confirm.”

4. Reset confirmationText whenever the modal opens or closes.

5. Fix both flows:
   - Soft Delete / Move to Trash
   - Permanent Delete

6. Soft Delete should move the song to Trash:
   status = "deleted"
   deletedAt = current ISO timestamp
   updatedAt = current ISO timestamp
   It should not require typing DELETE unless explicitly configured.

7. Permanent Delete should remove the uploaded song record entirely from the local preview store:
   song-soliloquy.uploadedSongs.v1
   It should also remove any legacy copy from:
   ss_uploaded_analyses
   if present.

8. After either delete action, refresh the Admin Library list immediately.

9. After soft delete, show toast:
   “Song moved to Trash.”

10. After permanent delete, show toast:
   “Song permanently deleted.”

11. Close the modal after successful action.

12. If delete fails, keep the modal open and show an error message.

13. Prevent permanent deletion of demo/static songs unless developer mode is enabled.
   Uploaded songs should be deletable.

14. Make deleted songs re-uploadable.
   If a user uploads a file with the same canonicalSlug as a soft-deleted song, show a conflict modal with:
   - Restore and Replace Existing
   - Save as New Version
   - Permanently Remove Old Record and Import Fresh
   - Cancel

15. Verify dropdown Delete and modal Permanent Delete use the same underlying store actions. Do not leave one wired and the other unwired.

Acceptance criteria:
- Typing DELETE enables the Permanently Delete button.
- Typing delete, Delete, or DELETE with spaces also works after normalization.
- Clicking Permanently Delete removes the uploaded song from Admin Library.
- Clicking Delete moves the song to Trash.
- Trash filter shows soft-deleted songs.
- Permanent delete removes the song from Trash as well.
- The same song can be re-uploaded after deletion.
- Existing upload, Library, SongDetail, parser, theme toggle, OBS, Live Studio, and motion features remain unchanged.
```

## QA Checklist

- [ ] Open `/admin/library`.
- [ ] Choose an uploaded song.
- [ ] Click Delete.
- [ ] Confirm it moves to Trash.
- [ ] Restore the song from Trash.
- [ ] Open Permanent Delete.
- [ ] Type `DELETE`.
- [ ] Confirm the destructive button enables.
- [ ] Click Permanently Delete.
- [ ] Confirm song disappears from Admin Library.
- [ ] Re-upload the same song file.
- [ ] Confirm it imports as a fresh record or shows replacement options.

## Notes
Keep all destructive actions admin-only. Public users should never see delete controls.
