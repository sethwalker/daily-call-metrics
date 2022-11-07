# Daily Call Metrics

## In production

To initiate a video call, visit https://meet-remix.vercel.app/, enter a valid Daily room name and submit. You will be redirected to a url with that room name in the path, which can be copied and sent to others to join the call.

The call app is configured to send stats data to a collector running on fly.io.

I built the dashboard using Jupyter and Voila, which turned out to be rather difficult to deploy alongside the collector on fly.io. Here is a screencast of the dashboard running locally with data collected in production:

![Dashboard screen recording](dashboard_screen_recording.mov)

## The call app

The call app took design inspiration from Root Ventures [meet](https://github.com/rootvc/meet) app, using the room name in the url to find (or in this case, create) rooms. [Remix](https://remix.run/) was chosen to maintain the React frontend while adding a server-side component for API calls to create rooms. Fully porting meet to Remix, Next.js, or another modern React framework is an exercise for another time.

The stats beacon is sent using the navigator.sendBeacon web API, which ensures that it consumes minimal resources / is not blocking when collected and is guaranteed to fire before the page is unloaded. In a more mature implementation, one may choose to implement fallbacks in the rare case that sendBeacon is not available in the browser. Also, in a "real" implementation, likely this data should be rolled into an existing analytics beacon.

To run in development, clone [the repository](https://github.com/sethwalker/meet-remix). Find your Daily API token in the Developers section of your Daily Dashboard and add it to a .env file like:
```
DAILY_TOKEN=<your API key>
```
then:
```sh
npm install
npm run dev
```

By default it will run at http://localhost:3000

When deploying to Vercel (or similar hosts), set environment variables in the config.

## The beacon collector

The beacon collector is a minimal flask app that writes the JSON payload directly to a log file and returns a 204 No Content (presumably the client is not even listening, given the nature of sendBeacon).

Given that the sendBeacon api is nonblocking in the browser, geographic location of the collector in production is not a consideration. If the client was blocking in some way, the collector function should be run on an edge network to be located as close to the client as possible.

Writing to a log file has the benefits of being incredibly simple, post-processing the log can be done as a separate process, and the history of events is preserved. In a more robust implementation, the log file would not be the canonical record store for inspecting the data, but for small data and early exploratory work it does suffice.

To run in development, clone [the repo](https://github.com/sethwalker/daily-beacon-collector), then:
```
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
flask run
```

The collector will run on http://127.0.0.1:5000/, so adding
```
BEACON_URL=http://127.0.0.1:5000/
```
to your .env file in meet-remix and restarting will connect the call app with the beacon collector.

## The dashboard

The dashboard is implemented as a [Jupyter](https://jupyter.org/) interactive notebook using the Plotly graphing library. The notebook reads and parses the JSON log file directly into a dataframe for exploratory analysis. Initial graphs show the 4 key metrics as a timeseries over the course of the call as well as a histogram.

To run the dashboard in development, install the beacon collector as above, then run

```
jupyter-lab daily_network_stats.ipynb
```

To see the minimal Voila dashboard, run
```
voila daily_network_stats.ipynb
```

Both commands will launch the notebook in a browser. If in Jupyter Lab, click into the cell and click the "run" button (or command-enter).

To download beacon.log from fly.io to analyze locally, run
```
fly ssh sftp shell
```
and then in the shell, run
```
get /workspace/beacon.log
```

### Notes

The link to the getting started guide is broken, I believe it should go to https://docs.daily.co/get-started#main
