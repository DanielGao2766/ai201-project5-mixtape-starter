``models.py`` 

defines 5 SQLAlchemy models: User, Song, Playlist, PlaylistSong, and Notification. The PlaylistSong table is a join table that adds an order column — songs in a playlist have an explicit position, not just insertion order.

``streak_service.py``

increments the streak each day a user listens
Resets back to 1 if a day is skipped
There are three functions in streak_service.py: 
record_listening_event, update_listening_streak, and get_streak
The update function is the logic behind the class while the record and get function are the input and output respectively 




Data flow — user rates a song: POST /songs/<id>/rate in routes/songs.py calls notification_service.notify_song_rated(). That function creates a Notification record for the song's original sharer. There's no separate rating model — the rating is stored directly on the Song.

Pattern I noticed: every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in services/."