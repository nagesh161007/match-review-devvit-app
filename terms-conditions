Match Review App uses HTTP Fetch to call the SportMonks Football API at 
api.sportmonks.com. This is required to load public football match data for 
the subreddit’s connected team: fixtures, scores, lineups, player names, and 
player images shown in match rating posts.

All SportMonks requests are made server-side from the Devvit app backend using 
a SportMonks API token stored in app settings. The client/webview does not call 
SportMonks directly.

We only fetch sports data needed to display matches and enable player ratings. 
We do not send Reddit user identifiers, usernames, or voting/rating data to 
SportMonks. User votes and aggregates are stored only on Reddit’s Devvit 
platform (Redis).

HTTP domain requested: api.sportmonks.com
Purpose: Football fixture, lineup, and match detail data for fan player-rating posts.
