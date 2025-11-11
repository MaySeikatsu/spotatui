# ID Normalization & Duration Fix Summary

## Completed Work ✅

### 1. Network Module (`src/network.rs`)
**Fixed imports to use correct rspotify 0.12 structure:**
```rust
// BEFORE (broken)
use rspotify::model::id::{AlbumId, ...}
use rspotify::model::ContextId;

// AFTER (working)
use rspotify::model::idtypes::{AlbumId, ArtistId, PlaylistId, ShowId, TrackId, UserId};
use rspotify::model::PlayableId;  // Not in idtypes, directly in model
```

**Updated IoEvent and method signatures:**
- ✅ Changed `ContextId` → `PlayContextId` in `StartPlayback` event and `start_playback` method
- ✅ All IoEvent variants now correctly use typed IDs from `idtypes` module
- ✅ Removed unused imports (`SystemTime`, `EpisodeId`)

### 2. App Module (`src/app.rs`)  
**Fixed imports:**
```rust
// BEFORE (broken)
use rspotify::model::id::{AlbumId, ...}

// AFTER (working)
use rspotify::model::idtypes::{AlbumId, ArtistId, PlayableId, PlaylistId, ShowId, TrackId, UserId};
```

**Storage decision:**
- ✅ Kept `liked_song_ids_set`, `followed_artist_ids_set`, `saved_album_ids_set`, `saved_show_ids_set` as `HashSet<String>`
- This is simpler for comparison logic throughout the UI and handlers
- Typed IDs are converted to String via `.id().to_string()` when storing

### 3. UI Module (`src/ui/mod.rs`)
- ✅ Base `ratatui` migration: replaced `Spans` with `Line`, fixed `Text` double-move, and updated audio-analysis field access.
- ❌ Still pending: convert every draw helper from `Frame<B>`/`where B: Backend` to `Frame<'_>`, and replace `duration.as_millis()` / `resume_position_ms` with chrono `TimeDelta` accessors (`num_milliseconds`, `resume_position`).

## Remaining Work ❌

### Critical: Handler Dispatch Conversions
**Problem:** Handlers dispatch `String` IDs but IoEvents expect typed IDs.

**Solution Pattern:**
```rust
// Example: track_table.rs line 261
// BEFORE (broken)
if let Some(id) = &track.id {
  let id = id.to_string();
  app.dispatch(IoEvent::ToggleSaveTrack(id));  // ❌ Wrong type
}

// AFTER (correct)
if let Some(id) = &track.id {
  if let Ok(track_id) = TrackId::from_id(&id.to_string()) {
    app.dispatch(IoEvent::ToggleSaveTrack(PlayableId::Track(track_id)));  // ✅ Correct
  }
}
```

**Files Needing Updates (Priority Order):**
1. `src/handlers/track_table.rs` - ~15 dispatch calls (StartPlayback, AddItemToQueue, ToggleSaveTrack)
2. `src/handlers/album_tracks.rs` - ~6 dispatch calls
3. `src/handlers/playbar.rs` - ~2 dispatch calls (ToggleSaveTrack for track and episode)
4. `src/handlers/recently_played.rs` - ~3 dispatch calls
5. `src/handlers/artist.rs` - ~3 dispatch calls (StartPlayback, AddItemToQueue, GetAlbumTracks)
6. `src/handlers/search_results.rs` - ~5 dispatch calls
7. `src/handlers/input.rs` - ~3 dispatch calls (GetAlbum, GetAlbumForTrack, GetShow)
8. `src/handlers/podcasts.rs` - ~2 dispatch calls
9. `src/app.rs` - Many helper methods:
   - `get_recommendations_for_seed` - convert `Vec<String>` to `Vec<TrackId>` and `Vec<ArtistId>`
   - `current_user_saved_album_add/delete` - convert `String` to `AlbumId`
   - `user_follow/unfollow_artists` - convert `String` to `ArtistId`
   - `user_follow/unfollow_playlist` - convert `String` to `PlaylistId` and `UserId`
   - `user_follow/unfollow_show` - convert `String` to `ShowId`
   - `get_artist` - convert `String` to `ArtistId`

### Medium Priority: Network Module API Changes
**Problem:** rspotify 0.12 changed many methods to return Streams instead of Futures.

**Affected methods:**
- `playlist_items()` - Returns `Stream<PlaylistItem>` not `Future<Page<PlaylistItem>>`
- `artist_albums()` - Returns `Stream<SimplifiedAlbum>`
- `current_user_playlists()` - Returns `Stream<SimplifiedPlaylist>`  
- Several show-related methods may not exist or have different signatures

**Solution:** Either:
1. Collect streams into Pages/Vecs manually
2. Upgrade to rspotify 0.13+ which may have better API
3. Rewrite stream handling throughout

### Low Priority: Other Compatibility Issues
- Progress field renamed in `CurrentPlaybackContext` (`progress_ms` → `progress`?)
- Some API methods don't exist or are renamed (`devices()`, `current_user_saved_shows()`, etc.)

## Next Steps Recommendation

1. **Immediate (to reduce compiler errors):**
   - Finish the ratatui migration in `src/ui/mod.rs` (convert every `Frame<B>` signature to `Frame<'_>` and swap `as_millis()` / `resume_position_ms` for chrono APIs).
   - Add conversion helpers to each handler file and update all dispatch calls to use typed IDs (`.into_static()`); this eliminates the bulk of E0412/E0597 errors.

2. **Short-term (to get partial functionality):**
   - Fix app.rs helper methods so they construct typed IDs before dispatching.
   - Handle Stream-based API methods in network.rs.
   - Fix remaining field access issues and progress fields.

3. **Long-term (for full functionality):**
   - Consider upgrading to rspotify 0.13+ or 0.15
   - Complete ratatui migration (remove Frame<B> generics)
   - Test end-to-end with Spotify API

## Helper Code Snippets

### Import Pattern for Handlers
```rust
use rspotify::model::idtypes::{TrackId, AlbumId, ArtistId, PlaylistId, ShowId};
use rspotify::prelude::PlayableId;
```

### Conversion Patterns
```rust
// String → TrackId for ToggleSaveTrack
if let Ok(track_id) = TrackId::from_id(&id_string) {
    app.dispatch(IoEvent::ToggleSaveTrack(PlayableId::Track(track_id)));
}

// String → AlbumId for GetAlbum
if let Ok(album_id) = AlbumId::from_id(&id_string) {
    app.dispatch(IoEvent::GetAlbum(album_id));
}

// String → ArtistId for GetArtist
if let Ok(artist_id) = ArtistId::from_id(&id_string) {
    app.dispatch(IoEvent::GetArtist(artist_id, name, country));
}

// Vec<String> → Vec<TrackId> for recommendations
let track_ids: Option<Vec<TrackId>> = id_strings
    .and_then(|ids| ids.iter().map(|s| TrackId::from_id(s).ok()).collect());

// Typed ID → String for storage
let id_string = track_id.id().to_string();
app.liked_song_ids_set.insert(id_string);
```

## Files Modified
- ✅ `src/network.rs` - Imports and type signatures
- ✅ `src/app.rs` - Imports only
- 🔶 `src/ui/mod.rs` - Base ratatui updates; Frame signatures + chrono conversions still pending
- ✅ `AGENTS.md` - Updated status
- ✅ `ID_NORMALIZATION_STATUS.md` - Detailed tracking doc (new)
- ✅ `NORMALIZATION_SUMMARY.md` - This file (new)
