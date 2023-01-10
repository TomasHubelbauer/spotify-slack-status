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

# Find the directory the script is in for relative access to `slack.pat`
# Note that this is required when running as a login item which starts in `~`
DIR=$(dirname "$(readlink -f -- "$0")")

# Get this token by:
# - Creating a Slack app https://api.slack.com/apps
# - Go to Permissions > Scopes > User Token Scopes
# - Add `users.profile:write` for the status update Slack API call
# - Add `users.profile:read` for the :headphones: current status emoji check
# - Scroll up and select Install to Workspace under OAuth Tokens for Your Workspace
# - Copy User OAuth Token under OAuth Tokens for Your Workspace
TOKEN=$(cat $DIR/slack.pat)

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
      echo "Slack status is $EMOJI, not :headphones: or empty, waiting for $DELAY seconds…"
      sleep $DELAY
      continue
    fi
  fi

  RUNNING=$(pgrep -lf Spotify)
  # Bail if Spotify is not running to prevent `osascript` from launching it
  # Do not bail if Slack is not running as it might be running on the phone
  if [ -z "$RUNNING" ];
  then
    echo "Spotify is off, waiting for $DELAY seconds…"
    sleep $DELAY
    continue
  fi

  # See `/Applications/Spotify.app/Contents/Resources/Spotify.sdef`
  STATE=$(osascript -e 'tell application "Spotify" to player state')
  ARTIST=$(osascript -e 'tell application "Spotify" to artist of current track')
  SONG=$(osascript -e 'tell application "Spotify" to name of current track')

  # Bail if Spotify is not in playing state (paused, stopped) but is running
  if [ "$STATE" != "playing" ];
  then
    echo "Spotify is not playing anything, waiting for $DELAY seconds…"
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

## Login Item

To make this script run on macOS startup, do the following:

- Create a file named `spotify-slack-status.command` alongside the main script
- Make the file executable using `chmod +x spotify-slack-status.command`
- Put the following content into the file:
  ```sh
  screen -d -m ./Desktop/…/spotify-slack-status.sh -S spotify-slack-status
  exit
  ```
- Go to Apple > System Settings > General > Login Items > +
- Find the `spotify-slack-status.command` file and add it as a login item
- Go to Apple > Log Out and then log back in to a new macOS user session

You will be greeter with a Terminal window that says that the process exited.
This window will appear after each log in into macOS and is safe to close.
The Spotify Slack status updated script is now running in the background.
You can verify this by running `screen -ls` or this command:
`ps x | grep spotify-slack-status | grep SCREEN`

To monitor the script's output, run `screen -r $PID` where `$PID` is the number
shown when running the previous command.

To exit the output monitoring without killing the script, press Ctrl+a+d.
To kill the script, press Ctrl+C instead.

## To-Do

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

### Consider uploading the album art for the album as a custom status emoji

To get the album image, use:
`osascript -e 'tell application "Spotify" to artwork url of current track'`

Here's how you add emoji by hand:
https://slack.com/help/articles/206870177-Add-custom-emoji-and-aliases-to-your-workspace

Here's an API that can be used to do that programatically:
https://api.slack.com/methods/admin.emoji.add

I could upload a custom emoji for each album and use it as a status emoji.
I would need to adjust the check to check for a prefix/suffix not for the
`:headphones:` emoji.
