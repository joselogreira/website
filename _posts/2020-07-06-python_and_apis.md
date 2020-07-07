---
layout: post
title: "Syncing YouTube and Spotify with Python"
author: "Jose Logreira"
categories: python
tags: [api, python]
image: youtube_spotify_python/cover.jpg
---


I started learning [Python](https://www.python.org/) until recent months (pandemic-given time). I gotta say it's been fun and very interesting, considering my background in C language. One of the best ways to understand technology (or so I've been told) is by projects-based learning. So this is my first take with Python and [REST APIs](https://en.wikipedia.org/wiki/Representational_state_transfer).

I saw a nice project video in [The Come Up](https://www.youtube.com/channel/UC-bFgwL_kFKLZA60AiB-CCQ) YouTube channel. The idea is simple: Over time, you hit ___Like___ on many YouTube music videos, so after running a __Python__ script, you'd be able to automatically create a __Spotify__ playlist containing all the liked songs from YouTube. Below is the original project video. It also contains links to the original [GitHub project](https://github.com/TheComeUpCode/SpotifyGeneratePlaylist).

<iframe width="560" height="315" src="https://www.youtube.com/embed/7J_qcttfnJA" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

The project was very appealing to me since it covers [web APIs](https://en.wikipedia.org/wiki/Web_API), a subject I knew nothing about, yet it is one of the most basic building blocks of web applications. 

I decided to make [my own version](https://github.com/joselogreira/playlist_creator). The codebase is completely new, although both projects use the same libraries (or Python modules). So the idea is for you to go watch the video, understand what's going on, and then come here to understand what are the tweaks and additions I've made. Some of these are:

* ___Using [SQLite Databases](https://www.sqlite.org/index.html):___ SQL-based databases are a whole world in themselves, but I wanted to implement some basic database functionality. The database contains information about the YouTube _"liked"_ playlist contents, the extracted video Artists and Tracks, and the Spotify IDs for each track. All the database manipulation is performed using the [SQLite python module](https://docs.python.org/2/library/sqlite3.html).

* ___[Spotify OAuth2.0](https://developer.spotify.com/documentation/general/guides/authorization-guide/) Authentication:___ The _TheComeUp_ project doesn't implement _OAuth2.0_ authorization, but it uses a manual process (using the [Spotify Developer's Console](https://developer.spotify.com/console/post-playlist-tracks/)) to grant access to a user account, where a valid _OAuth_ token is generated. My project includes the required "[Authorization Code Flow](https://developer.spotify.com/documentation/general/guides/authorization-guide/#authorization-code-flow)" for this step to be automated.

* ___Improved [YouTube Data API](https://developers.google.com/youtube/v3/docs/playlists/list) implementation:___ Both projects use the same [library](https://developers.google.com/youtube/v3/quickstart/python) for interaction with YouTube, but mine makes use of some extra API parameters that allow for more flexibility when requesting data, while saving on network traffic (after all, the YouTube Data API is NOT free). More specifically: no unused information is retrieved from YouTube, and the user can select the number of _liked_ videos to download and sync (user can even download the whole _"liked"_ history, which in my case is more than 800 videos).

* ___Execution switches:___ This just means that the user chooses whether to skip or execute certain steps. This greatly helps when debugging and also when testing different sections of the program.

* ___Support for subsequent script executions___: When executing the script once, you'd expect to sync YouTube and Spotify playlists. When executing the script twice (or some time in the future) you'd expect the previous data not to be overwritten or downloaded again, but only to apply the new changes made between script executions. This project handles that.

* ___Error Handling:___ I've tried to handle most of the API-related formatting or data errors, by printing descriptive messages of the reasons the script might have failed. This is not the focus of the project, but certainly it is a good addition.

## Script Steps

Below is a general program flow. The script is executed on a command line application, and the user input is also read from the command line.

### STEP 1: Request user to download _"liked"_ videos playlist from Youtube.

The script opens a new browser window and the user is redirected to the Google _OAuth_ authentication page. The user needs to grant access to the application (the script) so that it can read user data. Since this application is not verified by Google, it flags it as a safety risk. Nevertheless, the user can click on _"Advanced"_ -> _"Go to Playlist(unsafe)"_. 

![Google warning](/assets/img/youtube_spotify_python/google_warning.jpg)

Then, the user is asked whether to grant permissions to view the YouTube account. Upon allowance, the browser window can now be closed, and program execution continues in the command line. Depending on the amount of videos to be retrieved, the application performs multiple requests to the API until receiving all the videos' info (each response includes 50 videos maximum). In this case, 883 results were retrieved.

![Step 1](/assets/img/youtube_spotify_python/step1.jpg)

Each [JSON](https://www.json.org/json-en.html)-formatted response is individually saved in a temporary file with ___.json___ extention

### STEP 2: Create database and store videos

Database object is immediately created (if non-existent) and initialized with three tables:

* __videos:__ It stores the title string, video length, YouTube ID, and database ID fields.
* __tracks:__ It stores the songs' names, spotify IDs and database ID fields.
* __artists:__ It stores the artists' names, spotify IDs and database ID fields.

Each temporary JSON file is read and the videos are stored in the database in the table called ___"videos"___. These videos include music and non-music videos, that will later be filtered. Individual videos extracted from the JSON files are displayed in the console with its YouTube ID. Whenever a video is already stored in the database, it shows the _"Already Exists"_ message.

![Step 2](/assets/img/youtube_spotify_python/step2.jpg)

### STEP 3: Filter videos to extract only songs

The YouTube video ID is used to search through the [YoutubeDL](https://pypi.org/project/youtube_dl/) API, to extract the video __artist__ and __track__ name. If no artist nor track are found, the video is assumed not to be a song. Otherwise, the artist and song names are stored in the database tables ___"artist"___ and ___"tracks"___ respectively, and linked to the respective video in the ___"video"___ table using _primary_ and _foreing_ keys (in database jargon).

In this step, messages from the API as well as from the script are printed in console.

![Step 3](/assets/img/youtube_spotify_python/step3.jpg)

### STEP 4: Search in Spotify catalog

Here, connection with the [Spotify API](https://developer.spotify.com/documentation/web-api/reference/playlists/) is performed. Just as Google requested permissions for the application, in this step the user is redirected to a Spotify _OAuth_ Authentication page, where user grants access for the application to read and write playlists. The API is used to search in the Spotify catalog and retrieve _Spotify IDs_ as results of the query for each individual song. These IDs are also stored in the database.

![Step 4](/assets/img/youtube_spotify_python/step4.jpg)

Some songs may not be found, just because the artist or song strings found with YoutubeDL do not match. This is one of the drawbacks of this approach.

### STEP 5: Insert all songs found in Spotify into a new playlist

All user playlists are downloaded and the script looks for the existence of a playlist named ___"Youtube Liked Vids"___. If it exists, all the songs that were found to have a valid Spotify ID are now added to that playlist. If it doesn't, the script creates it using the API, and adds all songs to the newly created playlist.

![Step 5](/assets/img/youtube_spotify_python/step5.jpg)

## YouTubeDL is not perfect

You can imagine that many YouTube videos have odd names, special characters or strange symbols and emojis, thus, the weakest part of the whole chain is the YoutubeDL search engine. In fact, mistakes do happen: either it gives wrong song names and artists, or it does not find song name nor artist for a given video. It all depends on how well formatted the video title is (and how the YoutubeDL algorithm works, which I have not stopped to evaluate).

So the sad story is that these errors are to be fixed manually by deleting songs or adding missing ones. But hey! I just have to do it for a few of them, not for hundreds!. Besides, when running the script again in the future, the errors fixed are not lost. Data is not overwritten (unless the user deletes the database file).

## Learning Resources:

For Python, I went through all the [Python For Everybody](https://www.py4e.com/) series. It's free, it's all in YouTube, it makes part of a [Coursera specialization](https://www.coursera.org/specializations/python). 

Here's the [Youtube Data API documentation](https://developers.google.com/youtube/v3/docs/playlists).

Here's the [Spotify API documentation](https://developer.spotify.com/documentation/web-api/reference/playlists/)

For OAuth2.0 Authentication with Spotify, I took the [Spotipy](https://spotipy.readthedocs.io/en/2.13.0/) Python module as a reference.

This project is [hosted on GitHub](https://github.com/joselogreira/playlist_creator) under MIT License.

---

Finally, thanks to [The Come Up](https://www.youtube.com/channel/UC-bFgwL_kFKLZA60AiB-CCQ) channel. It was great to find this project.