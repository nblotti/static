---
name: spotify
description: Manage Spotify playlists — list, read, classify, create, and replace tracks. Use whenever the user mentions playlists, songs, tracks, artists, music, Spotify, or classification.
allowed-tools: spotify_list_playlists spotify_get_playlist_tracks spotify_get_liked_tracks spotify_get_tracks spotify_get_artists spotify_get_audio_features spotify_create_playlist spotify_replace_playlist_tracks spotify_classify_tracks
---
# Spotify playlist management

## Available tools

| Tool | Purpose |
|------|---------|
| spotify_list_playlists | List all playlists (id, name, public, total_tracks) |
| spotify_get_playlist_tracks | Get tracks from a playlist (includes track_id, name, uri, artist_names) |
| spotify_get_liked_tracks | Get the Liked Songs (same fields as playlist tracks) |
| spotify_get_tracks | Get detailed metadata for tracks (duration, album, release date) — rate-limited |
| spotify_get_artists | Look up artist metadata — rate-limited, avoid if possible |
| spotify_classify_tracks | AI classification: genre, mood, energy, tempo (up to 50/call) |
| spotify_create_playlist | Create a new empty playlist |
| spotify_replace_playlist_tracks | Replace playlist contents with track URIs |

## Typical workflow

1. List playlists to find IDs: spotify_list_playlists()
2. Read tracks from a playlist: spotify_get_playlist_tracks(playlist_id)
   — returns track_id, name, uri, artist_names for each track
3. Pass those track dicts directly to classify (up to 50 per call):
   spotify_classify_tracks(tracks=<list of track dicts from step 2>)
4. Filter results by genre, mood, energy, tempo, artist uniqueness
5. Create a new playlist if needed: spotify_create_playlist(name)
6. Replace its contents: spotify_replace_playlist_tracks(playlist_id, uris)

IMPORTANT: spotify_classify_tracks accepts the track dicts returned by
spotify_get_playlist_tracks / spotify_get_liked_tracks directly. Do NOT
call spotify_get_tracks or spotify_get_artists first — they are rate-limited
and unnecessary for classification.

## Classification values

- Genre: pop, rock, electronic, classical, hip-hop, country, folk, latin, jazz, soundtrack, punk, metal, indie, french, brazilian, techno, trance, house, r-and-b, reggae, other
- Mood: happy, sad, melancholic, energetic, chill, dramatic, romantic, aggressive, dreamy, dark, uplifting
- Energy: low, medium, high
- Tempo: slow, moderate, fast

## Important notes

- Track URIs format: spotify:track:<id>
- spotify_classify_tracks calls GPT internally — limit 50 tracks per call
- For large playlists, classify in batches and merge results
- When filtering by artist uniqueness, use artist_names from playlist data
- Always use spotify tools directly. Do NOT use kubectl or execute for Spotify operations.
