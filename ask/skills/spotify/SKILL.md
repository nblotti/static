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
| spotify_get_playlist_tracks | Get tracks from a playlist |
| spotify_get_liked_tracks | Get the Liked Songs |
| spotify_get_tracks | Get detailed metadata for tracks |
| spotify_get_artists | Look up artist metadata |
| spotify_classify_tracks | AI classification: genre, mood, energy, tempo (up to 50/call) |
| spotify_create_playlist | Create a new empty playlist |
| spotify_replace_playlist_tracks | Replace playlist contents with track URIs |

## Typical workflow

1. List playlists to find IDs
2. Read tracks from a playlist
3. Classify tracks in batches of up to 50
4. Filter by genre, mood, energy, tempo, artist uniqueness
5. Create a new playlist if needed
6. Replace its contents

## Classification values

- Genre: pop, rock, electronic, classical, hip-hop, country, folk, latin, jazz, soundtrack, punk, metal, indie, french, brazilian, techno, trance, house, r-and-b, reggae, other
- Mood: happy, sad, melancholic, energetic, chill, dramatic, romantic, aggressive, dreamy, dark, uplifting
- Energy: low, medium, high
- Tempo: slow, moderate, fast

## Important notes

- Track URIs format: spotify:track:<id>
- spotify_classify_tracks calls GPT internally — limit 50 tracks per call
- For large playlists, classify in batches and merge results
- When filtering by artist uniqueness, keep the first occurrence per artist
- Always use spotify tools directly. Do NOT use kubectl or execute for Spotify operations.
