## AI Usage

This is a stub until I have real AI usage to put

## Codebase Map

``models.py`` 

defines 5 SQLAlchemy models: User, Song, Playlist, PlaylistSong, and Notification. The PlaylistSong table is a join table that adds an order column — songs in a playlist have an explicit position, not just insertion order.

``streak_service.py``

increments the streak each day a user listens
Resets back to 1 if a day is skipped
There are three functions in streak_service.py: 
record_listening_event, update_listening_streak, and get_streak
The update function is the logic behind the class while the record and get function are the input and output respectively 

``playlist_service.py``

Handles playlist creation and retrieval logic
File has four functions: 
create_playlist, get_playlist_songs, get_playlist, get_user_playlists

The database object creation is through create_playlist, while the other three functions run .get() commands to query the database for specific information

``notification_service.py``

Handles notification creation and receival 
Also handles interactions between friends and users and notifies users when friends interacts with users playlists

The file has 5 functions:
create_notification, add_to_playlist, rate_song, get_notification, mark_as_read

create_notification creates a db object
add_to_playlist and rate_song both follow similar logic of creating a notification to the user 
get_notification and mark_as_read query the db to both recieve any notifications and mark as True in the db when read

## Data Flow

Data flow — user rates a song: POST /songs/<id>/rate in routes/songs.py calls notification_service.notify_song_rated(). That function creates a Notification record for the song's original sharer. There's no separate rating model — the rating is stored directly on the Song.

Pattern I noticed: every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in services/."

## Bug Fixes

Check diagnosis.txt for the root cause analysis for now