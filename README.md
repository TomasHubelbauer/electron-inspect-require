# Electron `--inspect` `require()`

I have been at trying to figure out how to capture a screenshot of an Electron window "from within" for a while now.

This problem occured to me when thinking about how I could generate screenshots of the VS Code window with my VS Code
extensions in action in an automated manner. Right now, I create the screenshots manually and place them to the repo,
but this is unwieldy and as a software developer, I should be able to make my own life easier, right?

So I figured, I could capture these screenshots in CI. I use Azure Pipelines for my extensions' CI and I have control
over the OS matrix, so in the Windows build for example, I could have conditional logic, which in each unit test, at
a designated point, would call out to an external script and wait for it to finish, the continue the test.

The script would capture a screenshot of the VS Code window and return so that the unit test execution could continue.

This would slow the tests down, but I could have specific test functions just for setting up the scene and capturing
the screenshot, which could be excluded from the test report so their slow times would not be a problem. The captured
screenshots would then become just another set of build artifacts of the CI pipeline.

I explored this some, it's not so easy, you have to drop down to the Windows API, enumerate the windows, find the VS
Code one (which is opened by the VS Code test runner), capture it's drawing context, save that to a file… Or at least
find its bounds, then make a screenshot of the whole desktop and cut the window bounds out of that. But if you're
going through all the trouble of finding the window handle, you might as well capture it specifically then.

I didn't like this, so my thoughts turned to Electron itself next. Surely there must be a way to capture a screenshot
of the Electron window "from within". The Electron window has a main process and a rendered process which for the
purposes of capturing a screenshot of the window's contents collide. The renderer process is what actually generates
the graphics of the page being displayed, but the main process is the Electron API.

There _is_ a method for capturing the renderer process output from the main process: `webContents.captureImage`. This
method is exactly what I need. However, in a context of a VS Code extensions, you don't get access to either the main
process or the renderer process. And the rendered process doesn't have access to the main process either, so really,
I need to get access to the main process _somehow_.

I know Electron allows having a debugger attached. Since it is based on Chrome, it uses the Chrome debug inspector
protocol. Attaching a debugger to Electron is quite simple. I will demonstrate this in a development scenario first:

- Create a new Node project: `npm init`
- Create an `index.js` file which is going to be what the main process runs
- Install Electron as a development dependency: `npm install electron --save-dev`
- Set up a `start` NPM command which runs Electron: `electron index.js`

Now you can run the app by running `npm start`, but we haven't set up any code for displaying the main window, so you
will only be able to tell the app runs by going to the process explorer.

However, we can already start experimenting with the debugger in this setup. We just need to add the `--inspect` flag
as Electron is being invoked. We set up the `npm start` command so we just need to add the extra arguments using the
double dash notation: `npm start -- --inspect`.

When issuing the above command, you should see something like this in the terminal:

```
Debugger listening on ws://127.0.0.1:9229/00000000-0000-0000-0000-000000000000
For help, see: https://nodejs.org/en/docs/inspector
```

It is now possible to go to `chrome://inspect` and under the default Devices tab find the inspector instance to connect
to.

If it doesn't show up, try opening the Chrome tab first and then running Electron with the inspector flag attached. Also,
make sure that whatever flag you are runnin the inspector on, default or custom, it is listed in the prompt you get when
clicking Configure next to Explore network targets in the Devices tab.

Upon clicking *inspect*, a Chrome DevTools window will open, connected to the Electron Main Context JavaScript context.

In here, you can issue `require('electron')` and get access to the main context APIs. From here, it is quite easy to make
the screenshot logic work:

```js
const electron = require('electron');
const webContents = electron.webContents.getAllWebContents()[0];
// This will resolve to `undefined` since we are not showing a window, but would work if we were
webContents.capturePage
```

All of this so far is leading up to a pretty promising solution. To get a screenshot of the VS Code test runner window,
I'd just have to make sure `npm test` in the extension repo starts the VS Code instance up with a debugger attached,
which I am not sure how to do yet, but I think it would be doable, because `npm test` from the Yeoman VS Code extension
scaffold calls some Batch file in `node_modules/vscode` which I could either patch or see if it would relay command line
arguments to the VS Code instance it starts up.

With that out of the way, I could just implement a debug protocol client, connect to the VS Code extension host process
from within itself as it runs the unit test which is actually a screenshot cpaturing hack, capture an image of the window,
save it and let the test runner move onto other test functions.

My switch to use of conditional statements probably already hints that there are trouble with this. Let's try by hand
first: go to your terminal, start VS Code with a debugger attached `code --inspect` and then go to Chrome and connect
DevTools to the instance. It will work. VS Code will report the web socket URL, Chrome will see it, the DevTools will
open and you will see the Electron Main Context JavaScript context attached. You will even see the correct globals,
like `process`! But you won't see `require`.

I don't know why this is, but I can tell you this is not a VS Code hardening measure. I figured it might be, so I went
ahead and downloaded prebuilt Electron binaries off GitHub. These binaries work in such a way that you can copy the
whole directory you download, place your `main.js` for the main process and `index.html` for the rendered process in
`resources/app` in that directory and have an Electron application running and ready to distribute in no time. Of course
the source code is out there in the open, so people usually use builders to build binaries which contain the compiled
source in an ASAR file or maybe even pack it into the executable itself, I don't know.

In any case, the default prebuilt Electron binary, when set up with `resources/app` and started with a debugger attached,
will not export `require` either. So VS Code probably just inherits this behavior. For some reason, the release build
and the development build you get with `electron index.js` are just different and the former seems to strip that symbol
out, leaving you with no way to obtain the JavaScript context even if you have a debugger attached to the binary.

A rollercoasted experience. I don't know why this symbol gets stripped, it may be a technical limitation. Maybe it has
to do with Node.

I decided to figure that out by creating a simple Node script which stalls at a standard input line read (to give me time
to attach the debugger and test things out in the developer tools):

```js
const readline = require('readline');
const rl = readline.createInterface({ input: process.stdin, output: process.stdout });
// Hold the process so it doesn't exit before we attach the debugger and have a go
rl.question('', () => rl.close());
```

This script uses `require`, the Node runtime provides that global. This script compiles and runs perfectly when run using
`node --inspect index.js` and the `require` global is there.

Maybe Electron bundles Node in some special way causing it to not export this particular global. No idea.

All in all, this is not a VS Code issue, because it reproduces in Electron proper, but it is not a Node issue, because in
Node this behavior is not present.

I asked for more information in [a Stack Overflow question](https://stackoverflow.com/q/56182168/2715716).

I filed [a GitHub issue](https://github.com/electron/electron/issues/18334) as well.
