# Spotify Slack Status

This script will set your Slack status to the current song playing in Spotify.

```sh
# Get this token by:
# - Creating a Slack app https://api.slack.com/apps
# - Select Permissions > Scopes > User Token Scopes and add `users.profile:write`
# - Scroll up and select Install to Workspace under OAuth Tokens for Your Workspace
# - Copy User OAuth Token under OAuth Tokens for Your Workspace
TOKEN=$(cat slack.pat)

while true
do
  echo "Stamp:"
  date -Iseconds

  RUNNING=$(pgrep -lf Spotify)
  # Bail if Spotify is not running to prevent `osascript` from launching it
  if [ -z "$RUNNING" ];
  then
    echo "Spotify is off, waiting a minute…"
    sleep 60
    continue
  fi

  # See `/Applications/Spotify.app/Contents/Resources/Spotify.sdef`
  ARTIST=$(osascript -e 'tell application "Spotify" to artist of current track')
  SONG=$(osascript -e 'tell application "Spotify" to name of current track')

  # Bail if Spotify is not playing anything (but is running)
  if [ -z "$ARTIST" ] || [ -z "$SONG" ];
  then
    echo "Spotify is not playing anything, waiting a minute…"
    sleep 60
    continue
  fi

  STAMP=$(date -v"+5M" +%s)
  JSON=$(echo '{}' | jq --arg SONG "$SONG" --arg ARTIST "$ARTIST" --arg STAMP $STAMP '.profile.status_text=$ARTIST+" - "+$SONG | .profile.status_emoji=":headphones:" | .profile.status_expiration=($STAMP|tonumber)')
  echo "Request:"
  echo "$JSON" | jq '.'

  # See https://api.slack.com/docs/presence-and-status#custom_status
  echo "Response:"
  curl -X POST --data "$JSON" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json; charset=utf-8" --no-progress-meter https://slack.com/api/users.profile.set | jq 'del(.profile)'

  # Wait for a minute in between the status updates
  sleep 60
  echo ""
done
```

To use, copy-paste it to a file named `spotify-slack-status.sh` and
run `chmod +x spotify-slack-status.sh` and then `./spotify-slack-status.sh`.

It will update the status every minute and will set its expiration to five minutes.

The script that gave me the push to attempt my own version of this is here:
https://gist.github.com/jgamblin/9701ed50398d138c65ead316b5d11b26

It helped me figure out how to use `osascript` to talk to Spotify, but it is
using the legacy token type which is no longer supported and I replaced it with
a Slack app and it also uses a Slack API endpoint which no longer exists and I
replaced it with a correct one based on the current Slack API documentation.

## To-Do

### Add a Bash trap to react to Enter key press to force-reload while waiting

I noticed sometimes the 1-minute refresh interval falls right between when the
artist is still old but the song is already new resulting in non-sensical names.

For debugging purposes I would like to add an Enter-trap and force-refresh even
when in the midst of the 1-minute wait between updates.

For the real fix, update the logic to read the artist and song twice and only
update when the values still agree 1 second or 10 seconds apart.

### Install this as a login items

I would like for the script to start running automatically on macOS logon.
However, I don't want to be blind to its output so I need to find a way to attach
a console to it when started as a login item.

### Skip resetting the status if not empty of `:headphones:`

Make the script respect custom statuses by not replacing them.
