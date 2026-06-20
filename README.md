# MOTUS — Real-Time Motion Gesture Classifier

Wave your phone around, and this app will tell you what gesture you just made. It watches your accelerometer in real time, draws what it's seeing as a live waveform, and runs it through a small machine learning model (trained in [Edge Impulse](https://edgeimpulse.com)) to guess whether you're drawing a **Circle**, doing a **Wave**, or just holding still (**Idle**).

🔗 Try it live: https://gesture-web-sensor-app.web.app

## What's actually happening under the hood

If you're curious how the pieces fit together, here's the short version:

1. **Reading the sensors.** The app tries a few different ways to get motion data from your device, falling back gracefully if one isn't available — starting with the newer `LinearAccelerationSensor` API, then `DeviceMotionEvent`, and finally a fake/simulated signal if you're testing on a desktop with no motion sensors at all.
2. **Collecting samples.** It grabs X, Y, and Z readings 62.5 times per second and keeps a rolling window of the most recent ones.
3. **Asking the model.** Once it has a full window of data, it hands that data to the Edge Impulse model and asks, "what does this look like?" The model replies with a confidence score for each gesture.
4. **Showing you the result.** The waveform on screen lights up in different colors per axis, and the gesture meters fill up live as predictions come in.

## What's in this folder

```
public/
├── index.html                   # the actual app — UI, sensors, waveform, predictions
├── run-impulse.js               # a small helper file the model needs (more on this below!)
├── edge-impulse-standalone.js   # exported straight from Edge Impulse
├── edge-impulse-standalone.wasm # the trained model itself
└── 404.html                     # the page Firebase shows if a link is broken
```

### Why does `run-impulse.js` exist?

When you export a model from Edge Impulse as WebAssembly, you get the `.js` and `.wasm` files — but they're written in a very low-level way that's awkward to call directly from regular JavaScript. The model genuinely expects you to hand it a raw memory address, not a normal array, and its answer comes back as a raw memory structure too, not a nice `{label, value}` object.

`run-impulse.js` is a small wrapper, written by Edge Impulse themselves, that handles all of that messy plumbing for you. Thanks to it, the rest of the app gets to write nice, simple code like this:

```js
const classifier = new EdgeImpulseClassifier();
await classifier.init();
const result = classifier.classify(features);
```

**Heads up:** all four files in `public/` need to stay together, with their names exactly right. If `run-impulse.js` ever goes missing or gets renamed, the app will show `model: wrapper not found` and every prediction will just sit at 0%.

## Running it on your own computer

You can't just double-click `index.html` and open it — browsers get suspicious of pages opened that way and block them from loading other files next to them. You need a real (if tiny) local server. From inside the `public` folder, run:

```bash
python -m http.server 8000
```

Then open this in your browser:

```
http://localhost:8000
```

## Putting it online

This project deploys through Firebase Hosting. From the main project folder (not `public`):

```bash
firebase deploy
```


## Tuning it to your model

Inside `index.html`, there's a setting called `WINDOW_SIZE_MS`. It controls how many samples get bundled together before asking the model for a prediction. This number has to match the window size you used when training your model in Edge Impulse — if they don't match, the data you're sending won't be shaped the way the model expects, and predictions will come out wrong or stuck.
