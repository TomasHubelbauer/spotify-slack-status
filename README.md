# Spotify Slack Status

This script will set your Slack status to the current song playing in Spotify
and update it every 10 seconds as long as your Slack status is empty or set to
`:headphones:`.
It will not overwrite your Slack status if you set your own status.

```sh
# Make sure these prerequisites are met in order for this script to work
# - macOS
# - jq (`brew install jq`)
# - Spotify

# Get this token by:
# - Creating a Slack app https://api.slack.com/apps
# - Go to Permissions > Scopes > User Token Scopes
# - Add `users.profile:write` for the status update Slack API call
# - Add `users.profile:read` for the :headphones: current status emoji check
# - Scroll up and select Install to Workspace under OAuth Tokens for Your Workspace
# - Copy User OAuth Token under OAuth Tokens for Your Workspace
TOKEN=$(cat slack.pat)

# Configure the delay between the Spotify checks and Slack status updates
DELAY=10

while true
do
  # Bail if Slack status is already set and is not `:headphones:`
  EMOJI=$(curl -H "Authorization: Bearer $TOKEN" --no-progress-meter https://slack.com/api/users.profile.get | jq --raw-output '.profile.status_emoji')
  if [ -n "$EMOJI" ];
  then
    if [ "$EMOJI" != ":headphones:" ];
    then
      echo "Slack status is $EMOJI, not :headphones: or empty, waiting a minute…"
      sleep $DELAY
      continue
    fi
  fi

  RUNNING=$(pgrep -lf Spotify)
  # Bail if Spotify is not running to prevent `osascript` from launching it
  if [ -z "$RUNNING" ];
  then
    echo "Spotify is off, waiting a minute…"
    sleep $DELAY
    continue
  fi

  # See `/Applications/Spotify.app/Contents/Resources/Spotify.sdef`
  ARTIST=$(osascript -e 'tell application "Spotify" to artist of current track')
  SONG=$(osascript -e 'tell application "Spotify" to name of current track')

  # Bail if Spotify is not playing anything (but is running)
  if [ -z "$ARTIST" ] || [ -z "$SONG" ];
  then
    echo "Spotify is not playing anything, waiting a minute…"
    sleep $DELAY
    continue
  fi

  # Schedule status expiration for 5 minutes from now to clear if not replaced
  STAMP=$(date -v"+1M" +%s)
  REQUEST_JSON=$(echo '{}' | jq --arg SONG "$SONG" --arg ARTIST "$ARTIST" --arg STAMP $STAMP '.profile.status_text=$ARTIST+" - "+$SONG | .profile.status_emoji=":headphones:" | .profile.status_expiration=($STAMP|tonumber)')

  # See https://api.slack.com/docs/presence-and-status#custom_status
  RESPONSE_JSON=$(curl -X POST --data "$REQUEST_JSON" -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json; charset=utf-8" --no-progress-meter https://slack.com/api/users.profile.set | jq 'del(.profile)')

  echo "$(date -Iseconds) $(echo "$REQUEST_JSON" | jq '.profile.status_text') $(echo "$RESPONSE_JSON" | jq '.ok')"
  
  # Wait for a minute in between the status updates
  sleep $DELAY
done
```

To use, copy-paste it to a file named `spotify-slack-status.sh` and
run `chmod +x spotify-slack-status.sh` and then `./spotify-slack-status.sh`.

The script will set the status to expire in a minute so that if either the script
or Spotify quit abruptly, the status will be cleared and won't linger around.

Note that sometimes due to sheer coincidence the script might pick up new song
name but old artist name.
If this happens, it will be fixed in the next loop run in 10 seconds, so it is no
biggie.

The script that gave me the push to attempt my own version of this is here:
https://gist.github.com/jgamblin/9701ed50398d138c65ead316b5d11b26

It helped me figure out how to use `osascript` to talk to Spotify, but it is
using the legacy token type which is no longer supported and I replaced it with
a Slack app and it also uses a Slack API endpoint which no longer exists and I
replaced it with a correct one based on the current Slack API documentation.

## To-Do

### Install this as a login item

I would like for the script to start running automatically on macOS logon.
However, I don't want to be blind to its output so I need to find a way to attach
a console to it when started as a login item.

### Skip the status update if it would have resulted in the same status text

Slack seems to animate the status updates in the people roster and the status
dropdown so potentially it would pulse every 10 seconds even when there is a need
to update the status every 3-5 minutes depending on the songs.

I will introduce a new check to bail if the existing status and new status have
the same text.

Althought I need to think about how this will interact with the short status
expiration.
Maybe I can fetch the song length from Spotify and set the status expiration to
that but that will re-introduce the problem with mismatched artist and song in
case of that race condition.
