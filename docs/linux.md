# Сборка под linux

До версии 0.31 более или менее нормально собирается.
Начиная с версии 0.32 нужен полифил для [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy). [proxy-polyfill](https://www.npmjs.com/package/proxy-polyfill) нормально работает только с версией 0.32. В версии 0.33 уже не работает, поэтому объекты, созданные через прокси оказываются undefined и вызывают ошибку вроде `TypeError: undefined is not an object (evaluating 'languageFeaturesService.colorProvider.register')`

```js
// package.json
{
  "name": "bsl-console",
  "version": "1.0.0",
  "description": "",
  "main": "editor.js",
  "homepage": "https://github.com/salexdv/bsl_console",
  "scripts": {
    "test": "mocha",
    "debug": "webpack-dev-server --mode development --env lang=ru --progress",
    "build": "webpack --mode production --env lang=ru --progress",
    "build_en": "webpack --mode production --progress",
    "dev": "webpack --mode development --env lang=ru --progress"
  },
  "author": "Shkuraev Alexander",
  "license": "MIT",
  "devDependencies": {
    "@babel/core": "^7.26.10",
    "@babel/preset-env": "^7.26.9",
    "@codingame/monaco-vscode-api": "^15.0.3",
    "@codingame/monaco-vscode-language-pack-ru": "^15.0.3",
    "@ungap/global-this": "^0.4.4",
    "autoprefixer": "^9.8.6",
    "babel-loader": "^10.0.0",
    "chai": "^5.2.0",
    "clean-webpack-plugin": "^3.0.0",
    "copy-webpack-plugin": "^6.4.1",
    "core-js": "^3.41.0",
    "css-loader": "^7.1.2",
    "file-loader": "^6.1.0",
    "html-inline-script-webpack-plugin": "^3.2.1",
    "html-webpack-plugin": "^5.6.3",
    "mini-css-extract-plugin": "^2.9.2",
    "mocha": "^11.1.0",
    "performance-polyfill": "^0.0.3",
    "postcss-cli": "^8.2.0",
    "postcss-loader": "^3.0.0",
    "proxy-polyfill": "^0.3.2",
    "regenerator-runtime": "^0.14.1",
    "remove-files-webpack-plugin": "^1.4.5",
    "resize-observer-polyfill": "^1.5.1",
    "string-replace-loader": "^3.1.0",
    "style-loader": "^4.0.0",
    "terser-webpack-plugin": "^4.2.3",
    "url-loader": "^4.1.1",
    "webpack": "^5.98.0",
    "webpack-cli": "^6.0.1",
    "webpack-dev-server": "^5.2.0"
  },
  "dependencies": {
    "big-integer": "^1.6.52",
    "monaco-editor": "^0.33.0",
    "monaco-editor-webpack-plugin": "^7.1.0"
  }
}

```

```js
// webpack.config.js
const MonacoWebpackPlugin = require('monaco-editor-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const HtmlInlineScriptPlugin = require('html-inline-script-webpack-plugin');
const { CleanWebpackPlugin } = require("clean-webpack-plugin");
const RemovePlugin = require('remove-files-webpack-plugin');
const webpack = require('webpack');
const path = require('path');
const CopyWebpackPlugin = require('copy-webpack-plugin');
const nls = require.resolve('./src/nls.ru');

module.exports = (env, args) => {

  return {
    context: path.resolve(__dirname, 'src'),
    entry: Object.assign(
      {
        console: './editor'
      },
      args.mode == 'development' ?
        {
          test: './test',
          test_query: './test_query'
        }
        : {}
    ),
    output: {
      path: path.resolve(__dirname, 'dist'),
      publicPath: '',
      filename: '[name].js',
      environment: {
        arrowFunction: false,
        const: false,
        destructuring: false,
        forOf: false,
        dynamicImport: false,
        module: false
      }
    },
    resolveLoader: {
      alias: {
        'blob-url-loader': require.resolve('./tools/loaders/blobUrl'),
        'compile-loader': require.resolve('./tools/loaders/compile'),
      }
    },
    devtool: args.mode == 'development' ? "inline-source-map" : false,
    module: {
      rules: [
        args.customOptions ?
          {
            test: /src[\\/]editor\.js$/,
            loader: 'string-replace-loader',
            options: {
              search: 'customOptions: true',
              replace: args.customOptions + ', customOptions: true'
            }
          } : {},
        {
          test: /node_modules[\\/]monaco-editor[\\/].+actions\.js$/,
          loader: 'string-replace-loader',
          options: {
            search: '(this._menuItems.get(id) || []).slice(0);',
            replace: '(this._menuItems.get(id) || []).slice(0);result=result.filter(function(item){return isIMenuItem(item)&&item.command.id.indexOf("_bsl")>=0;});'
          }
        },
        {
          test: /node_modules[\\/]monaco-editor[\\/].+parameterHints\.js$/,
          loader: 'string-replace-loader',
          options: {
            multiple: [{
              search: '[512 /* KeyMod.Alt */ | 16 /* KeyCode.UpArrow */',
              replace: '[2048 /* KeyMod.CtrlCmd */ | 16 /* KeyCode.UpArrow */'
            },
            {
              search: '[512 /* KeyMod.Alt */ | 18 /* KeyCode.DownArrow */',
              replace: '[2048 /* KeyMod.CtrlCmd */ | 18 /* KeyCode.DownArrow */'
            }]

          }
        },
        {
          test: /node_modules[\\/]monaco-editor[\\/].+tfIdf\.js$/,
          loader: 'string-replace-loader',
          options: {
            search: "word.split(/(?<=[a-z])(?=[A-Z])/g)",
            replace: "word.replace(/([a-z])([A-Z])/g, '$1 $2').split(' ')"
          }
        },
        {
          test: /\.js$/,
          enforce: 'pre',
          include: function(modulePath) {
            return (modulePath.includes('monaco-editor') || modulePath.includes('src'))
          },
          exclude: /tingle/,
          use: {
            loader: 'babel-loader',
            options: {
              cacheDirectory: true,
              presets: [
                ["@babel/preset-env", {
                  targets: {
                    ie: "11"
                  },
                  useBuiltIns: "usage",
                  corejs: 3              
                }]
              ],
              plugins: [
                '@babel/plugin-transform-arrow-functions',
                '@babel/plugin-transform-block-scoping',
                '@babel/plugin-transform-modules-commonjs',
                '@babel/plugin-transform-block-scoped-functions'
              ]
            }
          }
        },
        {
          test: /\.(png|jpg|gif|svg|ttf)$/i,
          type: 'asset/inline',
        },
        {
          test: /\.css$/,
          use: [
            MiniCssExtractPlugin.loader,
            {
              loader: 'css-loader',
              options: {
                esModule: false
              }
            }
          ]
        },
        {
          test: /\.wasm$/,
          type: "asset/inline",
        },
      ]
    },
    optimization: {
      // minimize: args.mode === 'production',
      minimize: false,
      splitChunks: {
        chunks: 'all'
      }
    },
    plugins: [
      new MiniCssExtractPlugin({
        filename: '[name].css'
      }),
      env.lang == 'ru' ? new webpack.NormalModuleReplacementPlugin(/\/(vscode-)?nls\.js$/, function (resource) {
        resource.request = nls
        resource.resource = nls
      }) : false,
      args.mode == 'development' ? new CopyWebpackPlugin({
        patterns: [
          { from: path.join(__dirname, 'node_modules/mocha/mocha.js'), to: 'mocha.js' },
          { from: path.join(__dirname, 'node_modules/mocha/mocha.css'), to: 'mocha.css' },
          { from: path.join(__dirname, 'node_modules/chai/chai.js'), to: 'chai.js' },
        ]
      }) : false,
      new MonacoWebpackPlugin({
        languages: ['xml'],
      }),
      new HtmlInlineScriptPlugin(),
      new CopyWebpackPlugin({
        patterns: [
          { from: './tree/icons', to: 'tree/icons' }
        ]
      }),
      args.mode == 'production' ? new webpack.optimize.LimitChunkCountPlugin({
        maxChunks: 1
      }) : false,
      new CleanWebpackPlugin(),
      new HtmlWebpackPlugin({
        inject: 'body',
        chunks: ['console'],
        template: './index.html',
        filename: 'index.html',
        cache: false
      }),
      args.mode == 'development' ? new HtmlWebpackPlugin({
        inject: 'body',
        chunks: ['console', 'test'],
        template: './test.html',
        filename: 'test',
        cache: false
      }) : false,
      args.mode == 'development' ? new HtmlWebpackPlugin({
        inject: 'body',
        chunks: ['console', 'test_query'],
        template: './test_query.html',
        filename: 'test_query',
        cache: false
      }) : false,
      args.mode == 'production' ? new RemovePlugin({
        after: {
          include: [
            './dist/test.js',
            './dist/test_query.js',
            './dist/editor.worker.js'
          ]
        }
      }) : false
    ].filter(Boolean),
    devServer: {
      port: 9000,
      open: true
    }
  }
};
```