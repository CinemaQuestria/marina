# marina

![](http://i.imgur.com/U15Zpfv.jpg)

A multiple video stream control bot and stream viewer for stream viewers.

## Rationale

As of the writing of this document, CinemaQuestria currently uses a Google Docs
spreadsheet to control which one of many streams is actively shown to the user.
This allows for us to have multiple variants of the same stream (an HD and SD
stream for example), but we would like the ability to tile multiple streams and
control everything via a Discord bot instead of via Google Docs.

## Architecture

Data is stored in an [rqlite][rqlite] database. The
Discord bot will have a few unauthenticated HTTP API calls for browsers to 
request the current list of streams and how they should be displayed, etc.

## Configuration

| Envvar | "Sane" default value | Description |
|:------ |:-------------------- |:----------- |
| `PORT` | `5000` | HTTP port to listen on |
| `DATABASE_URL` | `http://` | [rqlite][rqlite] database to use as persistence layer |
| `MIGRATE_ON_START` | `true` | If set, run database migrations every time the app starts |
| `NOTIFICATION_INTERVAL` | `5m` | A [Go time Duration][go-duration] for how often Marina should check for notifications to send |
| `GOOGLE_API_KEY` | | API key from google to use for authentication |
| `API_CACHE_LIFETIME` | `30s` | A [Go time Duration][go-duration] for how long API results should be cached |

## Discord bot Commands

All Discord bot commands will have flags using [Go package flag][go-flag].


### `queuestream`

`queuestream` will append to an in-memory list of streams that are set to happen 
after the current stream. This will also update `#announcements` on Discord with
information about the upcoming stream and a unique ID to use for configuring
stream notifications.

#### Flags

| flag name | example value | description |
|:--- |:--- |:--- |
| `streamer`  | `"RainShadow"`  | Who will be providing a stream source |
| `stream` | `rainshadow-yt` | What stream will be embedded |
| `border` | `splatoon_2` | Which border CSS file to load |
| `title` | `"Splatoon 2 Gameplay with CQ Friends"` | Stream title |
| `kind` | `gameplay` | What "kind" of stream will be shown |
| `tab-title` | `HD` | What title this stream variant should use for the tab |
| `series` | `"Friday Game Night"` | What series this stream is a member of |

These flags will set the various flags to the stream that will be appended to 
the list of queued streams.

### `nextstream`

`nextstream` will signal all connected clients to change stream to the next set of
values that were configured with `queuestream`.

### `setborder`

`setborder` will signal all connected clients to change the border CSS around 
the current stream.

### `setmode`

`setmode <grid|tabs>` will change if the current set of active streams is displayed
as a grid of stream embeds or a tabset of variants of the primary stream embed.

### `info`

`info` will display information about who is live and up next. If there are 
multiple streams in grid mode, it will display information about who is streaming
in which section of the grid.

### `addstream`

`addstream` will add an additional stream variant or alternate angle for grid
mode. It has the same flags as `queuestream`. This will only append streams for
the currently active stream, not for any stream in the future.

### `notify`

#### `notify next`

`notify next` configures Marina to notify the user of this chat command when the
`nextstream` command is used as a Discord PM.

### `subscribe <series>`

`subscribe <series>` configures Maina to notify the user of this chat command
whenever a stream of a given series (eg: `"CQRiffs"`) is started by `nextstream`.

## API

All API requests should be made over HTTPS to `api.cinemaquestria.com`. Any plain 
HTTP requests will be rejected.

These will be cached heavily and the number of hits for this API call every
minute will allow us to gauge how many active viewers a given stream is
attracting.

### `GET /v1/stream/info`

This unauthenticated API request will return stream information in the following
schema:

```json
{
  "title":  "CQ Clan Training",
  "border": "splatoon_2",
  "kind":   "gameplay|community-gameplay|tv|chat",
  "up_next": {
    "streamer":  "matttheshadowman", 
    "title":     "Random British Games", 
    "when":      1500488079, 
    "kind":      "gameplay|community-gameplay|tv|chat"
  },
  "streams": [
    {
      "streamer":  "RainShadow",
      "game":      "Splatoon 2",
      "kind":      "youtube",
      "argument":  "dQw4w9WgXcQ",
      "tab_title": "HD"
    }
  ],
  "mode":        "grid|tabs",
  "invite_link": "https://discord.gg/g32cpqd",
  "viewers":     1337
}
```

### `GET /v1/stream/schedule`

This unauthenticated API request will return the stream schedule as recorded by 
Google Calendar. Clients should allow people to "subscribe" to notifications,
allowing them to either get a browser notification or a PM on Discord.

```json
{
  events: [
    {
      "id":       "google-calendar-id",
      "title":    "Games that Yanks can't Wank",
      "streamer": "matttheshadowman",
      "kind":     "gameplay|community-gameplay|tv|chat",
      "when":     1500488079
    }
  ]
}
```

### `POST /v1/notifications/submit`

This authenticated API request will let a user submit a request to be notified
when a given stream (by google calendar ID) is about to start.

Input:

```json
{
  "user_id":       72838115944828928,
  "event_id":      "google-calendar-id",
  "minutes_prior": 5
}
```

Output:

```json
{
  "message": "Notification set up"
}
```

### `/css/borders/`

This will allow clients to download the relevant CSS for CinemaQuestria stream borders.
This information will be stored in the database and will be controlled with the `setborder`
bot command.

### `/img/borders/`

Border images will live here as flat files.

### `/img/backgrounds/`

Background images will live here as flat files.

## Frontend

The Frontend should be a simple HTML + JS page using mithril and making API calls 
to control which stream is displayed.

### Components

#### Stream Viewer

This component will query the CQ API and use the results in order to display 
either a grid of stream embeds or a single set of named tabs.

#### Calendar Feed

This component will query the CQ calendar API and display the set of upcoming 
events with a countdown for each of the scheduled events.

---

[rqlite]: https://github.com/rqlite/rqlite
[go-duration]: https://godoc.org/time#ParseDuration
[go-flag]: https://godoc.org/flag
