# Special use cases and how-to's

1.  [Browserify hyphenopoly.module.js](#browserify-hyphenopolymodulejs)
2.  [Webpack, using hyphenopoly.module.js](#webpack-hyphenopoly-module)
3.  [Webpack, using Hyphenopoly_Loader.js](#webpack-hyphenopoly-loader)
4.  [Hyphenate depending on media queries](#hyphenate-depending-on-media-queries)
5.  [Set .focus() while Hyphenopoly is running](#set-focus-while-hyphenopoly-is-running)
6.  [Words containing special format characters](#format-chars)
7.  [Hyphenate HTML-Strings using using hyphenopoly.module.js](#hyphenate-html-strings-using-hyphenopolymodulejs)

**Note: It's not recommended to use `hyphenopoly.module.js` in a browser environment. See e.g. [this guide](./Hyphenators.md#use-case-hyphenopoly-in-react) on how to use Hyphenopoly in react.**

## Browserify hyphenopoly.module.js

**Note: A browserifyed hyphenopoly.module.js is by far larger then the Hyphenopoly_Loader.js and Hyphenopoly.js scripts which are optimized for usage in browsers.**

### Basic setup

Create a npm project:

```Shell
npm init
```

Install browserify as devDependency

```Shell
npm install --save-dev browserify
```

Install hyphenopoly

```Shell
npm install hyphenopoly
```

Setup hyphenopoly in main.js. Make sure to set the loader to "http" since browserify will not shim the "fs" module:

```javascript
"use strict";

const hyphenopoly = require("hyphenopoly");

const hyphenator = hyphenopoly.config({
  require: ["de", "en-us"],
  paths: {
    maindir: "./node_modules/hyphenopoly/",
    patterndir: "./node_modules/hyphenopoly/patterns/"
  },
  hyphen: "•",
  loader: "https"
});

async function hyphenate_en(text) {
  const hyphenateText = await hyphenator.get("en-us");
  console.log(hyphenateText(text));
}

async function hyphenate_de(text) {
  const hyphenateText = await hyphenator.get("de");
  console.log(hyphenateText(text));
}

hyphenate_en("hyphenation enhances justification.");
hyphenate_de("Silbentrennung verbessert den Blocksatz.");
```

Transform the module

```Shell
browserify main.js -o bundle.js
```

This will generate the script-file `bundle.js`. Usage of a minifying tool (e.g. [tinyify](https://github.com/browserify/tinyify)) is recommended.

_Note:_ Make sure the directories referenced in `paths` are available.

## Webpack, using hyphenopoly.module.js {#webpack-hyphenopoly-module}

**Note: A webpacked hyphenopoly.module.js is by far larger then the Hyphenopoly_Loader.js and Hyphenopoly.js scripts which are optimized for usage in browsers.**

**Note2: I am not a webpack expert. The following works for Webpack 5.4.0, but may not be the best way to use hyphenopoly.module.js in webpack.**

Like `browserify`, `webpack` will not shim "fs". Thus we have to tell `webpack` to shim the "fs" module with an empty object and how to polyfill other node-specific functions. And we configure `hyphenopoly` to use the "https"-loader.

webpack.config.js

```javascript
module.exports = {
  const path = require("path");
  const webpack = require("webpack");
  module.exports = {
    mode: "production",
    entry: "./src/index.js",
    output: {
      filename: "main.js",
      path: path.resolve(__dirname, "dist")
    },
    node: {
      global: true
    },
    plugins: [
      new webpack.ProvidePlugin({
        process: "process/browser",
        Buffer: ["buffer", "Buffer"],
      })
    ],
    resolve: {
      fallback: { 
        "fs": false,
        "https": require.resolve("https-browserify"),
        "http": require.resolve("stream-http")
      }
    }
  };
};
```

index.js

```javascript
"use strict";

const hyphenopoly = require("hyphenopoly");

const hyphenator = hyphenopoly.config({
    "require": ["de", "en-us"],
    "paths": {
        "maindir": "../node_modules/hyphenopoly/",
        "patterndir": "../node_modules/hyphenopoly/patterns/"
    },
    "hyphen": "•",
    "loader": "https"
});

async function addDiv(lang, text) {
    const hyphenateText = await hyphenator.get(lang);
    const element = document.createElement('div');
    element.innerHTML = hyphenateText(text);
    document.body.appendChild(element);
}

(async function () {
    await addDiv("de", "Silbentrennung verbessert den Blocksatz.");
    await addDiv("en-us", "hyphenation enhances justification.");
})();
```

## Webpack, using Hyphenopoly_Loader.js {#webpack-hyphenopoly-loader}

If you’re working in a browser environment you can add the required files, such as Hyphenopoly.js and the essential patterns, by copying them with the [copy-webpack-plugin](https://www.npmjs.com/package/copy-webpack-plugin) into your distribution folder.

_webpack.config.js_
```javascript
"use strict";
const path = require("path");
const TerserPlugin = require("terser-webpack-plugin");
const HtmlWebpackPlugin = require("html-webpack-plugin");
const CopyPlugin = require("copy-webpack-plugin");
const {CleanWebpackPlugin} = require("clean-webpack-plugin");
const HtmlWebpackInjector = require("html-webpack-injector");

module.exports = {
    "entry": {
        "main": "./src/index.js",
        "vendor_head": "./src/vendor_head.js"
    },
    "mode": "production",
    "module": {
        "rules": [
            {
                "loader": "html-loader",
                "test": /\.html$/i
            }
        ]
    },
    "optimization": {
        "minimizer": [new TerserPlugin()],
        "runtimeChunk": "single"
    },
    "output": {
        "filename": "js/[name].[contenthash].bundle.js",
        "path": path.resolve(__dirname, "dist")
    },
    "performance": {
        "hints": false
    },
    "plugins": [
        new CleanWebpackPlugin(), new CopyPlugin({
            "patterns": [
                {
                    "context": "./",
                    "from": "node_modules/hyphenopoly/min/Hyphenopoly.js",
                    "to": "./js/hyphenopoly/"
                }, {
                    "context": "./",
                    "from": "node_modules/hyphenopoly/min/patterns/{es,it,de,en-us}.wasm",
                    "to": "./js/hyphenopoly/patterns/[name].[ext]"
                }
            ]
        }), new HtmlWebpackPlugin({
            "template": "./src/index.html"
        }), new HtmlWebpackInjector()
    ]
};
```

Then, inside the vendor_head.js create the proper Hyphenopoly object describing the directories where the files are copied and finally import Hyphenopoly_Loader.js.

_vendor_head.js_
```javascript
"use strict";

const Hyphenopoly = {
    "require": {
        "es": "anticonstitucionalmente",
        "it": "precipitevolissimevolmente",
        "de": "Silbentrennungsalgorithmus",
        "en-us": "antidisestablishmentarianism"
    },
    "paths": {
        // Path to the directory of pattern files
        "patterndir": "./js/hyphenopoly/patterns/",
        // Path to the directory where the other ressources are stored
        "maindir": "./js/hyphenopoly/"
    }
};
window.Hyphenopoly = Hyphenopoly;
require("hyphenopoly/Hyphenopoly_Loader");
```

A demo can be found at _/examples/webpack_. [Live preview.](./dist/index.html)

## Hyphenate depending on media queries

In CSS hyphenation can be restricted to special media-queries. If hyphenation on a website must be dependent of e.g. the width of the window and only active on small screens, you'd do something like this:

```css
@media (max-width: 600px) {
  .hyphenate {
    hyphens: auto;
    -ms-hyphens: auto;
    -moz-hyphens: auto;
    -webkit-hyphens: auto;
  }
}
@media (min-width: 601px) {
  .hyphenate {
    hyphens: none;
    -ms-hyphens: none;
    -moz-hyphens: none;
    -webkit-hyphens: none;
  }
}
```

To polyfill hyphenation for browsers that don't support hyphenation (or don't support the required language) we'll have to tell Hyphenopoly to behave the same.

The standard way to enable Hyphenopoly would just hyphenate, regardless of the screen-width. Well have to tell the browser to run Hyphenopoly_Loader.js only for small screens and react to changes of the screen width (e.g. when rotating a mobile device). Therefor, instead of including Hyphenopoly the standard way

```html
<script>
  var startTime;
  var Hyphenopoly = {
    require: {
      "en-us": "FORCEHYPHENOPOLY"
    },
    paths: {
      maindir: "../",
      patterndir: "../patterns/"
    }
  };
</script>
<script src="../Hyphenopoly_Loader.js"></script>
```

we'll define a `selectiveLoad` IIFE:

```html
<script>
  (function selectiveLoad() {
    let H9YLisLoaded = false;
    let elements = null;
    function handleSize(mql) {
      if (mql.matches) {
        //i.e. if width <= 600px
        if (H9YLisLoaded) {
          window.Hyphenopoly.hyphenators["en-us"].then(deh => {
            elements.list.get("en-us").forEach(elo => {
              deh(elo.element, elo.selector);
            });
          });
        } else {
          // Hyphenopoly isn't loaded yet, so load the Loader
          // with the following settings:
          window.Hyphenopoly = {
            require: {
              "en-us": "supercalifragilisticexpialidocious"
            },
            paths: {
              maindir: "../",
              patterndir: "../patterns/"
            },
            setup: {
              selectors: {
                ".hyphenate": {}
              }
            }
          };
          const loaderScript = document.createElement("script");
          loaderScript.src = "../Hyphenopoly_Loader.js";
          document.head.appendChild(loaderScript);
          H9YLisLoaded = true;
        }
      } else {
        //i.e. if width > 600px
        if (H9YLisLoaded) {
          //remove hyphenation previously applied by Hyphenopoly
          window.Hyphenopoly.unhyphenate().then(els => {
            elements = els;
          });
        }
      }
    }
    // Create a Media-Query-List
    const mql = window.matchMedia("(max-width: 600px)");
    // Listen to changes
    mql.addListener(handleSize);
    // call handleSize on init
    handleSize(mql);
  })();
</script>
```

## Set .focus() while Hyphenopoly is running

By default `Hyphenopoly_Loader.js` hides the whole document to prevent a "Flash of unhyphenated content" (FOUHC) until hyphenation has finished. If `focus()` is called while the document is hidden the focus will not change.

To prevent this behavior experiment with [different settings for hiding](./Setup.md#hide). Using "element" or "text" should work in most cases.

## Format chars

Hyphenopoly does NOT hyphenate words that contain one of the following special format characters:

*   SOFT HYPHEN (\u00AD)
*   ZERO WIDTH SPACE (\u200B)
*   ZERO WIDTH NON-JOINER (\u200C)
*   ZERO WIDTH JOINER (\u200D)

## Hyphenate HTML-Strings using hyphenopoly.module.js

`hyphenopoly.module.js` only hyphenates plain text strings. If the string contains HTML tags, it must first be parsed. The textContent of the nodes may then be hyphenated using hyphenopoly:

````javascript
const { JSDOM } = require("jsdom")

const hyphenator = require("hyphenopoly").config({
  sync: true,
  require: ["de"],
  defaultLanguage: "de",
  minWordLength: 6,
  leftmin: 4,
  rightmin: 4,
})

function hyphenateText(text) {
  if (typeof text === "string") {
    return hyphenator(text)
  } else {
    return undefined
  }
}

function hyphenateHtml(html) {
  if (typeof html === "string") {
    if (html.trim().startsWith("<")) {
      const fragment = JSDOM.fragment(html)
      const hyphenateNode = (node) => {
        for (node = node.firstChild; node; node = node.nextSibling) {
          if (node.nodeType == 3) {
            node.textContent = hyphenator(node.textContent)
          } else {
            hyphenateNode(node)
          }
        }
      }
      hyphenateNode(fragment)
      return fragment.firstChild["outerHTML"]
    } else {
      return hyphenator(html)
    }
  } else {
    return undefined
  }
}

module.exports = { text: hyphenateText, html: hyphenateHtml }
````
