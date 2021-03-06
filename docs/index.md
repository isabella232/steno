# Getting Started with Steno

Steno is your sidekick for developing tests for your Slack app. Use Steno to record and replay HTTP requests and responses to and from your app. Then, you'll be able to run tests on your app without worrying about manually recreating state in actual Slack workspaces or manually reproducing events. Hello, automation and continuous integation! :wave:

It doesn't matter which language you chose to program with or how your app is structured. Steno is a CLI tool that starts a server outside your process, so as long as your app speaks HTTP (all Slack apps do), you're ready to go!

## The Workflow

Steno is easiest to use if you follow this workflow. For many of you, this might look familiar -- it's based on tried-and-true integration testing patterns. But let's get out of the jargon and jump straight into it.

1. **Pick a behavior in your app that you want to test.**

   Start with something relatively small and self-contained. To illustrate, let's say your app will send a DM to the installing user as soon as the app gets installed on a team.

2. **It's time to record**

   In **record mode** Steno helps you build **scenarios**. Each scenario is a set of requests and responses that correspond to a behavior inside your app. A scenario is represented by a directory inside your project. That directory contains a set of text files; each text file describes one HTTP request and response. The first time you run Steno in record mode, it will create a `scenarios` folder in your working directory if one doesn't already exist (this can be customized with a command line option).

   Start by choosing a descriptive name for your scenario, e.g. `successful_installation_will_dm_installing_user`.

   Running `steno --record` will prompt Steno to start a local server listening on a URL -- by default, `http://localhost:3000/...`. You can change this using the `--out-port` option. Adjust your code to send requests Steno's URL instead of `https://slack.com/...`. Hint: if you're using the [Node Slack SDK](https://github.com/slackapi/node-slack-sdk), this can be done easily using the `slackApiUrl` option.

   Steno will also record requests that are coming **into** your app from the Slack Platform, such as those from interactive messages or the events API. By default, Steno will listen for these requests at `http://localhost:3010`. You can change this using the `--in-port` option. If you have a tunneling tool like [ngrok set up](https://api.slack.com/tutorials/tunneling-with-ngrok), follow the guide on [using Steno with tunneling in development](./guides/tunneling).

   Steno needs to be told where to forward the requests to reach your app. This is specified using the `--internal-url` option. For example, if the app is listening on port 5000, the `--internal-url` is `localhost:5000`.

   While recording, it's a good idea to keep sensitive data such as Slack API tokens off the record. You'll likely be committing the scenario directory to your source control. Steno can automatically replace API tokens with fake values by adding the `--slack-replace-tokens` option.

   Now we're ready to record. Open your terminal, navigate to a test directory in your project, and launch the tool in record mode:

   `steno --record --internal-url localhost:5000 --scenario-name successful_installation_will_dm_installing_user --slack-replace-tokens`

   Perform the behavior you'd like to test with your app. In our example, that means we install the app and observe a DM arrive. Steno will record the requests and responses that are sent and received into `scenarios/successful_installation_will_dm_installing_user`. Since the app worked, we now have hard proof that Steno can check against when we run it in replay mode later. Terminate the `steno` command in your terminal (<kbd>Ctrl</kbd> + <kbd>C</kbd>).

   **NOTE:** The first time you run `steno`, it will ask for your permission to report usage statistics. We highly recommend you answer `Y` to help the maintainers continually improve the tool.

3. With your sidekick Steno :couple: standing by, you can **write your first test.**

   Pick your favorite test runner and write a test case that simulates the behavior you chose. In the case of the DM on install example, you would write a case that completes the OAuth flow for installing your Slack app. The app will exchanging the code for an access token, store the token, and send a DM to the installing user. While recording, you should may have noticed a line like this:

   `Slack token replaced: TOKEN=xoxb-000000000000-I2XejP8axGr15Mz5JHFOKMCe REPLACEMENT=xoxf-I3JJhEJFMEBByPDykuinUQUErN3vRmBhpmH7k`

   Steno noticed an authentication token in a request and replaced it with a fake token for you automatically. Use this fake token in the test
   case in any place you'd expect the token to appear. In this example, we start with an unauthenticated client, and only use the token that is returned in Slack's responses, so we don't need to do any special replacement.

   Conclude your test case by asserting that your app is in the expected state. Namely, that the token has been stored, and that the DM was sent. Wait, how do we assert that the DM was sent containing the message we intended to be sent? Read on, and we'll find out.

4. Use Steno's **control API** to load scenarios :vhs:.

   We'll need to add some setup code to our test case so that Steno is prepared to replay this specific scenario. Steno allows the starting and
   stopping of scenarios from its control API. Steno is listening for these commands, by default, on the URL `http://localhost:4000`. To prepare for our example scenario manually you might use curl like so:

   `curl -X POST -H "Content-Type: application/json" -d "{ "name":"successfull_installation_will_dm_installing_user" }" http://localhost:4000/start`

   In your test runner, _before the test case is run_ you can perform the same request using an HTTP client of your choice.

5. Steno's control API equips you to answer our burning question :fire:, can we **make an assertion** that our DM was sent?

   In the test case, after the app has a chance to perform the behavior, you use the HTTP client once again to make a request to stop the replay of the scenario. In response you receive data about what actually happened during the test case and how it stacks up against our recorded scenario. For example, using curl:

   `curl -X POST http://localhost:4000/stop`.

   You should expect a response back that looks something like this:

   ```json
   {
     "interactions": [
       {
         "direction": "outgoing",
         "request": {
           "timestamp": 1502487343000,
           "method": "POST",
           "url": "/api/oauth.access",
           "headers": {
             "content-type": "application/x-www-form-urlencoded",
             "...": "..."
           },
           "body": "client_id=00000000000.999999999999&client_secret=e9eab23fd04e44c8d1b640e876d39d92&code=00000000000.111111111111.b5fc60d60ec8d65301fcda44028a135bc27cec7ac476938df3b2b15aac73af42&redirect_uri=http%3A%2F%2Fexample.com%2Fcallback"
         },
         "response": {
           "timestamp": 1502487343002,
           "statusCode": 200,
           "headers": {
             "content-type": "application/json; charset=utf-8",
             "...": "..."
           },
           "body": "{\"ok\":true,\"access_token\":\"xoxp-00000000000-11111111111-222222222222-900cf8de83f771f22932027dd9c36dc5\",\"scope\":\"identify,bot\",\"user_id\":\"U11111111\",\"\"bot\":{\"bot_user_id\":\"U00000000\",\"bot_access_token\":\"xoxf-I3JJhEJFMEBByPDykuinUQUErN3vRmBhpmH7k\"}}"
         }
       },
       {
         "direction": "outgoing",
         "request": {
           "timestamp": 1502487343005,
           "method": "POST",
           "url": "/api/chat.postMessage",
           "headers": {
             "content-type": "application/x-www-form-urlencoded",
             "...": "..."
           },
           "body": "token=xoxf-I3JJhEJFMEBByPDykuinUQUErN3vRmBhpmH7k&channel=U11111111&text=Hello%2C%20I%27m%20ExampleBot"
         },
         "response": {
           "timestamp": 1502487343006,
           "statusCode": 200,
           "headers": {
             "content-type": "application/json; charset=utf-8",
             "...": "..."
           },
           "body": "{\"ok\":true,\"channel\":\"D33333333\",\"ts\":\"0000000000.999999\",\"message\":{\"text\":\"Hello, I\'m ExampleBot\",\"username\":\"ExampleBot\",\"bot_id\":\"B44444444\",\"type\":\"message\",\"subtype\":\"bot_message\",\"ts\":\"0000000000.999999\"}}"
         }
       }
     ],
     "meta": {
       "durationMs": 12,
       "unmatchedCount": {
         "incoming": 0,
         "outgoing": 0
       }
     }
   }
   ```

   This represents a comprehensive history of what Steno witnessed since you loaded the scenario with the request to `/start`.

   In our example, in order to verify that the DM was sent, make assertions at the end of your test case that the last request has a `url` property equal to `/api/chat.postMessage`, and that the response's `body` property contains `"ok":true`.

6. Pull the plug :electric_plug: and let Steno handle the interactions in **replay mode**.

   Once the scenario is available in the directory, Steno can mimic Slack by responding to requests on its own. It also can send your app requests on its own. Let's go back to the command line and run with a slightly different command:

   `steno --replay --internal-url localhost:5000`

   Run your test case and watch those assertions succeed :white_check_mark:! (and, if not, maybe that's a good thing -- you just caught a bug). At the start of the test case, the request to the control API will load the interactions from the scenario directory. Then the app carries out the behavior. At the end the control API is used again to end the scenario and your test runner can report back to you.

7. Scrub the code and the scenarios of any other sensitive data :speak_no_evil:. In this case, you will want to manually replace the `client_id` and `client_secret`. **Commit the test code and scenario directory to your project, rinse and repeat**! Now you can add running Steno in replay mode as a step before starting your test runner in your testing scripts. If you have continuous integration set up, you can rest assured that with every commit along the way you haven't broken your existing behavior :massage:.

**Pro Tip**: This isn't the only way you can use Steno! Even if you don't have access to a specific team or have never been able to produce a specific event, you could write a scenario (using the anticipated request and response bodies), drop it into a scenario directory in your project,and run Steno in replay mode. This will make your app behave as if it magically was talking to the Slack Platform for real :sparkles:. You can hand-edit scenarios based on what's available in our documentation, or take a scenario from someone else who recorded them for you.

## Common Questions

* **Q**: I already use {VCR, node-nock, httpretty, or some other HTTP mocking library for tests}, why should I use Steno?

  **A**: You're a rockstar for thinking about testing from the get go! Those tools are great choices, but Steno still has some advantages for you:
  - Two-way proxying: Most other mocking tools are only concerned with *outgoing* requests from your app. But Slack
    also offers notifications using *incoming* requests (Events API, Interactive Messages, etc.). Steno can handle both.
  - Scenarios are a portable exchange format: No matter which language or toolkit you are using, Steno saves scenarios
    in a format we can all read and write: plain old HTTP. This means you can swap scenario files with other developers
    on your team or in the community, or even attach them when filing bugs.

## Documentation

### Command Line Interface

Steno comes with a `--help` option to help you navigate the command line interface.

```
$ ./steno --help
steno [command]

Commands:
  steno record <appBaseUrl>  (DEPRECATED: use --record) start recording scenarios
  steno replay <appBaseUrl>  (DEPRECATED: use --replay) start replaying scenarios

Options:
  --help                    Show help  [boolean]
  --version                 Show version number  [boolean]
  --record                  Start steno in record mode.  [boolean]
  --replay                  Start steno in replay mode.  [boolean]
  --internal-url, --app     The internal URL where your application is listening. In record mode, requests served from in-port are forwarded to this URL. In replay mode, incoming interactions' requests are sent to this URL.  [string] [default: "localhost:5000"]
  --external-url            The external URL to which steno will forward requests that are recieved on out-port. Only valid in recoed mode.  [string] [default: "https://slack.com"]
  --in-port, --in           The port where incoming requests are served by forwarding to the internal URL. Only valid in record mode.  [string] [default: "3010"]
  --out-port, --out         The port where outgoing requests are served either (in record mode) by forwarding to the external service (Slack API) or (in replay mode) by responding from matched interactions in the current scenario.  [string] [default: "3000"]
  --control-port, -c        The port where the control API is served  [string] [default: "4000"]
  --scenario-dir            The directory where all scenarios are recorded to or replayed from. Relative to current working directory.  [string] [default: "./scenarios"]
  --scenario-name           The initial scenario. This name is used for the subdirectory of scenario-dir where interactions will be recorded to or replayed from.  [string] [default: "untitled_scenario"]
  --slack-replace-tokens    Whether to replace Slack API tokens seen in request bodies. NOTE: When this option is set, sensitive data may appear on stdout. Only valid in record mode.  [boolean] [default: false]
  --slack-detect-subdomain  Whether to replace the subdomain in outgoing requests to Slack based on patterns in the path. This must be set in order for incoming webhooks, slash command request URLs, and interactive component request URLs to proxy correctly. Only valid in record mode.  [boolean] [default: true]

Examples:
  steno --record                                  Starts steno in record mode with defaults for the control-port, in-port, out-port, internal-url, scenario-dir, and scenario-name.
  steno --record --app localhost:3000 --out 5000  Starts steno in record mode and  customizes the internal-url and out-port
  steno --replay                                  Starts steno in replay mode with defaults for the control-port, out-port, internal-url, scenario-dir, and scenario-name.

for more information, visit https://slackapi.github.io/steno
```

### Control API

See the [control API documentation](./control) for details.

## Known Limitations

* Replay mode will attempt to fire as many incoming requests from the scenario as possible, even in parallel, if all chronologically previous outgoing requests have already been matched. We're investigating a better solution so that you can describe scenarios where those requests should occur serially. Feedback welcome!

* Request trailers are not currently supported.

* Replaying of chunked encoding requests has not been tested.

## Feedback

Steno is an open source project and we welcome feedback and new contributions. Please leave your comments in [our issue tracker on GitHub](https://github.com/slackapi/steno/issues).
