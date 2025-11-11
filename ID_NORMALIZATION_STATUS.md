# ID Normalization Progress

## Completed ✅

### network.rs
- ✅ Updated imports to use `rspotify::model::idtypes::{AlbumId, ArtistId, PlaylistId, ShowId, TrackId, UserId}`
- ✅ Added `PlayableId` import (already in `model`, not `idtypes`)
- ✅ Changed `ContextId` → `PlayContextId` throughout
- ✅ IoEvent enum already uses typed IDs for all parameters
- ✅ Removed unused imports (SystemTime, EpisodeId - not needed yet)

### app.rs  
- ✅ Updated imports to use `idtypes` instead of `id` module
- ✅ Kept internal storage as `HashSet<String>` for liked_song_ids_set, etc. (simpler comparison logic)

## Remaining Work ❌

### Critical: Handler Dispatch Calls
All handlers that dispatch IoEvents need ID conversions. The pattern is:
```rust
// OLD (String)
app.dispatch(IoEvent::ToggleSaveTrack(track_id.to_string()));

// NEW (typed ID via helper)
if let Ok(id) = TrackId::from_id(&track_id.to_string()) {
    app.dispatch(IoEvent::ToggleSaveTrack(PlayableId::Track(id)));
}
```

Files needing updates:
- ❌ src/handlers/track_table.rs - ~15 dispatch calls
- ❌ src/handlers/album_tracks.rs - ~6 dispatch calls  
- ❌ src/handlers/recently_played.rs - ~3 dispatch calls
- ❌ src/handlers/playbar.rs - ~2 dispatch calls
- ❌ src/handlers/artist.rs - ~3 dispatch calls
- ❌ src/handlers/search_results.rs - ~5 dispatch calls
- ❌ src/handlers/input.rs - ~3 dispatch calls
- ❌ src/handlers/podcasts.rs - ~2 dispatch calls
- ❌ src/app.rs - Many get_recommendations, album_add/delete, artist_follow, etc.

### Critical: UI Duration + Frame Fixes
Replace `Frame<B>` signatures and `std::time::Duration` helpers with the ratatui/chrono equivalents:
```rust
// OLD signatures
pub fn draw_album_table<B>(f: &mut Frame<B>, ...) where B: Backend

// NEW
pub fn draw_album_table(f: &mut Frame<'_>, ...)

// OLD duration access
duration.as_millis()
resume_position_ms

// NEW  
duration.num_milliseconds() as u128
resume_position
```

Files:
- ❌ `src/ui/mod.rs` - Every draw helper still uses `Frame<B>` + `Backend` bounds, and duration/resume fields still call `as_millis()` / `resume_position_ms`. Need to update album/playlist/recommendation/song/recently-played tables plus episode lists to chrono APIs.

### Critical: Network.rs Method Calls
Several rspotify 0.12 methods have changed signatures or return types:
- ❌ `.playlist_items()` - now returns Stream, not Future
- ❌ `.artist_albums()` - now returns Stream
- ❌ `.current_user_playlists()` - now returns Stream  
- ❌ `.current_user_saved_shows()` - method may not exist in 0.12
- ❌ `.devices()` - check method name/signature
- ❌ Various `.get_id()` calls - this method doesn't exist, need manual ID extraction

### Medium Priority: App.rs Field Access
- ❌ `progress_ms` field renamed or removed from CurrentPlaybackContext
- ❌ Need to check what the new field name is (likely `progress` returning `chrono::Duration`)

## Strategy

### Phase 1: Make it compile (focus on E0412 errors)
1. Add ID conversion helpers to each handler file
2. Update all dispatch calls to use typed IDs
3. Fix duration conversions in UI

### Phase 2: Make it work (fix runtime API incompatibilities)
1. Update stream-based API calls to collect results
2. Fix missing/renamed API methods  
3. Handle ID extraction without `.get_id()`

## Helper Patterns

### String → Typed ID
```rust
use rspotify::model::idtypes::TrackId;
use rspotify::prelude::PlayableId;

// For tracks (to toggle save)
if let Some(id) = &track.id {
    if let Ok(track_id) = TrackId::from_id(&id.to_string()) {
        app.dispatch(IoEvent::ToggleSaveTrack(PlayableId::Track(track_id)));
    }
}

// For albums
if let Ok(album_id) = AlbumId::from_id(&album_id_string) {
    app.dispatch(IoEvent::GetAlbum(album_id));
}

// For playlists  
if let Ok(playlist_id) = PlaylistId::from_id(&playlist_id_string) {
    app.dispatch(IoEvent::GetPlaylistItems(playlist_id, offset));
}
```

### Typed ID → String (for storage/comparison)
```rust
// IDs have .id() method to get the String
let id_string = track_id.id().to_string();
app.liked_song_ids_set.insert(id_string);
```

### Duration Conversion
```rust
// chrono::TimeDelta
let millis = duration.num_milliseconds() as u128;
let seconds = duration.num_seconds();

// For resume point
if let Some(resume_point) = &episode.resume_point {
    let resume_ms = resume_point.resume_position; // Already u32 or similar
}
```

## Next Steps
1. Fix all handler dispatch calls (biggest impact on E0412 errors)
2. Fix UI duration conversions
3. Handle network.rs Stream API issues (may need to upgrade rspotify or rewrite Stream handling)
