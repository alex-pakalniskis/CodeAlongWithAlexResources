## Recipe
1. Set up a new AssemblyScript project and open it with VS Code

    ``` bash
    mkdir as-mandelbrot && cd as-mandelbrot
    npm init -y
    npm install --save-dev assemblyscript
    npx asinit . -y
    code .
    ```

1. Populate `assembly/index.ts` with following contents

    ``` ts
      // see: https://en.wikipedia.org/wiki/Mandelbrot_set

      /** Number of discrete color values on the JS side. */
      const NUM_COLORS = 2048;

      /** Computes a single line in the rectangle `width` x `height`. */
      export function computeLine(y: u32, width: u32, height: u32, limit: u32): void {
        var translateX = width  * (1.0 / 1.6);
        var translateY = height * (1.0 / 2.0);
        var scale      = 10.0 / min(3 * width, 4 * height);
        var imaginary  = (y - translateY) * scale;
        var realOffset = translateX * scale;
        var stride     = (y * width) << 1;
        var invLimit   = 1.0 / limit;

        var minIterations = min(8, limit);

        for (let x: u32 = 0; x < width; ++x) {
          let real = x * scale - realOffset;

          // Iterate until either the escape radius or iteration limit is exceeded
          let ix = 0.0, iy = 0.0, ixSq: f64, iySq: f64;
          let iteration: u32 = 0;
          while ((ixSq = ix * ix) + (iySq = iy * iy) <= 4.0) {
            iy = 2.0 * ix * iy + imaginary;
            ix = ixSq - iySq + real;
            if (iteration >= limit) break;
            ++iteration;
          }

          // Do a few extra iterations for quick escapes to reduce error margin
          while (iteration < minIterations) {
            let ixNew = ix * ix - iy * iy + real;
            iy = 2.0 * ix * iy + imaginary;
            ix = ixNew;
            ++iteration;
          }

          // Iteration count is a discrete value in the range [0, limit] here, but we'd like it to be
          // normalized in the range [0, 2047] so it maps to the gradient computed in JS.
          // see also: http://linas.org/art-gallery/escape/escape.html
          let col = NUM_COLORS - 1;
          let sqd = ix * ix + iy * iy;
          if (sqd > 1.0) {
            let frac = Math.log2(0.5 * Math.log(sqd));
            col = <u32>((NUM_COLORS - 1) * clamp<f64>((iteration + 1 - frac) * invLimit, 0.0, 1.0));
          }
          store<u16>(stride + (x << 1), col);
        }
      }

      /** Clamps a value between the given minimum and maximum. */
      @inline
      function clamp<T extends number>(value: T, minValue: T, maxValue: T): T {
        return min(max(value, minValue), maxValue);
      }
    ```

1. Populate `index.html` with the following contents

    ``` html
      <!DOCTYPE html>
      <html>
      <head>
      <meta charset="utf-8">
      <meta name="viewport" content="user-scalable=0" />
      <title>Mandelbrot set - AssemblyScript</title>
      <link rel="icon" href="https://assemblyscript.org/favicon.ico" type="image/x-icon" />
      <style>
        html, body { height: 100%; margin: 0; overflow: hidden; color: #111; background: #fff; font-family: sans-serif; }
        h1 { padding: 20px; font-size: 12pt; margin: 0; }
        a { color: #111; text-decoration: none; }
        a:hover { color: #0074C1; text-decoration: underline; }
        canvas { position: absolute; top: 60px; left: 20px; width: calc(100% - 40px); height: calc(100% - 80px); background: #eee; }
        canvas.gradient { left: 0; top: 0px; height: 2px; width: 100%; }
      </style>
      </head>
      <body>
      <h1>
        <a href="https://en.wikipedia.org/wiki/Mandelbrot_set">Mandelbrot set</a> in
        <a href="http://assemblyscript.org">AssemblyScript</a>
        ( <a href="https://github.com/AssemblyScript/examples/blob/main/mandelbrot/assembly/index.ts">source</a> )
      </h1>
      <canvas></canvas>
      <script>"use strict";

      // Set this to false if you prefer a plain image instead.
      var animate = true;

      // Set up the canvas with a 2D rendering context
      var cnv = document.getElementsByTagName("canvas")[0];
      var ctx = cnv.getContext("2d");
      var bcr = cnv.getBoundingClientRect();

      // Compute the size of the viewport
      var width  = bcr.width  | 0;
      var height = bcr.height | 0;
      var ratio  = window.devicePixelRatio || 1;
      width  *= ratio;
      height *= ratio;
      var size = width * height;
      var byteSize = size << 1; // discrete color indices in range [0, 2047] (here: 2b per pixel)

      cnv.width  = width;
      cnv.height = height;

      ctx.scale(ratio, ratio);

      // Compute the size of and instantiate the module's memory
      const memory = new WebAssembly.Memory({
        initial: ((byteSize + 0xffff) & ~0xffff) >>> 16
      });
      const mem = new Uint16Array(memory.buffer);
      const imageData = ctx.createImageData(width, height);
      const argb = new Uint32Array(imageData.data.buffer);

      // Fetch and instantiate the module
      fetch("build/release.wasm")
      .then(response => response.arrayBuffer())
      .then(buffer => WebAssembly.instantiate(buffer, {
        env: {
          memory,
          "Math.log": Math.log,
          "Math.log2": Math.log2
        },
      }))
      .then(module => {
        const exports = module.instance.exports;
        const computeLine = exports.computeLine;
        const updateLine = function (y) {
          var yx = y * width;
          for (let x = 0; x < width; ++x) {
            argb[yx + x] = colors[mem[yx + x]];
          }
        };

        // Compute an initial balanced version of the set.
        const limit = 40;
        for (let y = 0; y < height; ++y) {
          computeLine(y, width, height, limit);
          updateLine(y);
        }

        // Keep rendering the image buffer.
        (function render() {
          if (animate) requestAnimationFrame(render);
          ctx.putImageData(imageData, 0, 0);
        })();

        if (animate) {

          // Let it glow a bit by occasionally shifting the limit...
          var currentLimit = limit;
          const shiftRange = 10;

          (function updateShift() {
            currentLimit = limit + (2 * Math.random() * shiftRange - shiftRange) | 0
            setTimeout(updateShift, 1000 + (1500 * Math.random()) | 0);
          })();

          // ...while continously recomputing a subset of it.
          const flickerRange = 3;
          (function updateFlicker() {
            for (let i = 0, k = (0.05 * height) | 0; i < k; ++i) {
              let ry = (Math.random() * height) | 0;
              let rl = (2 * Math.random() * flickerRange - flickerRange) | 0;
              computeLine(ry, width, height, currentLimit + rl);
              updateLine(ry);
            }
            setTimeout(updateFlicker, 1000 / 30);
          })();

        }
      }).catch(err => {
        alert("Failed to load WASM: " + err.message + " (ad blocker, maybe?)");
        console.log(err.stack);
      });

      // Compute a nice set of colors using a gradient
      const colors = (() => {
        const cnv  = document.createElement("canvas");
        cnv.width  = 2048;
        cnv.height = 1;
        const ctx = cnv.getContext("2d");
        const grd = ctx.createLinearGradient(0, 0, 2048, 0);
        grd.addColorStop(0.0000, "#000764");
        grd.addColorStop(0.1600, "#2068CB");
        grd.addColorStop(0.4200, "#EDFFFF");
        grd.addColorStop(0.6425, "#FFAA00");
        grd.addColorStop(0.8575, "#000200");
        ctx.fillStyle = grd;
        ctx.fillRect(0, 0, 2048, 1);
        cnv.className = "gradient";
        setTimeout(() => document.body.appendChild(cnv));
        return new Uint32Array(ctx.getImageData(0, 0, 2048, 1).data.buffer);
      })();
      </script>
      </body>
      </html>

    ```

1. Edit the build commands in package.json to include the following values

    ``` json
    "asbuild:debug": "asc assembly/index.ts --target debug --runtime stub --use Math=JSMath --importMemory",
    "asbuild:release": "asc assembly/index.ts --target release --runtime stub --use Math=JSMath --importMemory",
    ```

1. Compile the example then start the app

    ``` bash
    npm run asbuild
    npm start
    ```
    
1. Paste url into browser of your choice

1. If browser restricts fetching local resources when opening `index.html`, set up local server workaround

    ``` bash
    npm install --save-dev http-server
    http-server . -o -c-1
    ```

## Resources
* Tutorial link: https://www.assemblyscript.org/examples/mandelbrot.html
* Set up new AssymblyScript project: https://www.assemblyscript.org/getting-started.html#setting-up-a-new-project
