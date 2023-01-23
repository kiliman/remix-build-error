# remix-build-error

This sample project includes a couple of patches to render the build error
directly in the browser. It supports `<LiveReload>` so the browser will automatically
refresh once the build error has been fixed.

To try it out, clone this repo and run the web app.

Edit any file and include an error, like a syntax error, then save the file. Remix
will automatically rebuild your app. If there's a build error, it will be displayed
in the browser. Fix the error and save again. Remix will build again and this time
it will refresh the browser with a working app.

## How it works

The patch simply creates a file `.cache/build-error.log` when there's a build error.
The request handler will look for this file and if present, will generate an HTML response
with the error as well as the `<LiveReload>` support, so the page will refresh after each
build.

If the build is successful, it will remove the `build-error.log` file and Remix will send
the actual route response.

<img style="max-width:300px;" src="https://cdn.loom.com/sessions/thumbnails/39c6bd8ad3a94e2bbb9888b158f35c27-with-play.gif">

<a href="https://www.loom.com/share/39c6bd8ad3a94e2bbb9888b158f35c27">
    Show Remix build errors with support for LiveReload - Watch Video
  </a>

## Adding to your project

Simply copy the patches to your project and install `patch-package`. Update your _package.json_
to add a `postinstall` script to run `patch-package`.

```json
"scripts": {
  "postinstall": "patch-package"
}
```

This patches Remix App Server out of the box. If you are using the Express adapter,
include the following code to your _server.js_ file.

```js
const fs = require("fs");

app.all(
  "*",
  process.env.NODE_ENV === "development"
    ? (req, res, next) => {
        // ------------- Add this block --------------
        // check if there was a build error and show it if present
        if (fs.existsSync(`.cache/build-error.log`)) {
          let buildError = fs.readFileSync(`.cache/build-error.log`, "utf-8");
          let html = `<html><head><title>Build Error</title></head>
          <body style="color:red;"><pre>${buildError}</pre>${liveReloadScript}</body></html>`;
          res.status(500).send(html);
          return;
        }
        // ------------- End block -------------------
        purgeRequireCache();

        return createRequestHandler({
          build: require(BUILD_DIR),
          mode: process.env.NODE_ENV,
        })(req, res, next);
      }
    : createRequestHandler({
        build: require(BUILD_DIR),
        mode: process.env.NODE_ENV,
      })
);

let liveReloadScript = `<script>
function remixLiveReloadConnect(config) {
  let protocol = location.protocol === "https:" ? "wss:" : "ws:";
  let host = location.hostname;
  let socketPath = protocol + "//" + host + ":" + ${
    process.env.REMIX_DEV_SERVER_WS_PORT || 8002
  } + "/socket";
  let ws = new WebSocket(socketPath);
  ws.onmessage = (message) => {
    let event = JSON.parse(message.data);
    if (event.type === "LOG") {
      console.log(event.message);
    }
    if (event.type === "RELOAD") {
      console.log("💿 Reloading window ...");
      window.location.reload();
    }
  };
  ws.onopen = () => {
    if (config && typeof config.onOpen === "function") {
      config.onOpen();
    }
  };
  ws.onclose = (event) => {
    if (event.code === 1006) {
      console.log("Remix dev asset server web socket closed. Reconnecting...");
      setTimeout(
        () =>
          remixLiveReloadConnect({
            onOpen: () => window.location.reload(),
          }),
        1000
      );
    }
  };
  ws.onerror = (error) => {
    console.log("Remix dev asset server web socket error:");
    console.error(error);
  };
}
remixLiveReloadConnect();
</script>`;
```
