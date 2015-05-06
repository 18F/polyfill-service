# Polyfill service

[![Build
Status](https://travis-ci.org/Financial-Times/polyfill-service.svg?branch=master)](https://travis-ci.org/Financial-Times/polyfill-service)

Makes web development less frustrating by selectively polyfilling just what the browser needs. Use it on your own site, or as a service.  For usage information see the [hosted service](https://cdn.polyfill.io), which formats and displays the service API documentation located in the [docs](docs/) folder.

## Deploy to Cloud Foundry

Polyfill service runs out of the box in Cloud Foundry with one caveat: The current node.js buildpack has no option to install without the `--production` flag. As a result, devDependencies including grunt are not installed.

A straightforward workaround is [bundling](http://docs.cloudfoundry.org/buildpacks/node/node-tips.html#nodemodules) prior to the push.

    npm install

With that done, a basic `manifest.yml` will suffice to describe the application.

	---
	applications:
	- name: polyfill
	  memory: 256M

With dependencies bundled and a manifest in place the app is ready to be pushed.

    cf push

## Installing as a service

1. Install [git](http://git-scm.com/downloads) and [Node](http://nodejs.org) on your system
2. Clone this repository to your system (`git clone git@github.com:Financial-Times/polyfill-service.git`)
3. Run `npm install` (this will also run `grunt build` which you will need to do again if you edit any polyfills)

To run the app for **development**:

Run `grunt dev` from the root of the working tree.  This will watch your filesystem and automatically rebuild sources and restart if you make any changes to any of the app source code or polyfills.  If you change the docs, library or service (`docs`, `lib` and `service` directories) the service will be restarted.  If you change the polyfills or polyfill configs (`polyfills` directory), the polyfill sources will be recompiled before the service restarts, which makes the restart slightly slower.

By default, `grunt dev` also *deletes historic polyfills*, for a faster build.  If you want to run the service with the historic polyfill collections installed, run `grunt installcollections buildsources service watch` instead.

To run the app for **production**:

Run `npm start`.  This will start the service using [forever](https://github.com/nodejitsu/forever), which runs the process in the background, monitors it and restarts it automatically if it dies.  It doesn't watch the filesystem for changes and you won't see any console output.

In either case, once the service is running, navigate to [http://localhost:3000](http://localhost:3000) in your browser (you can configure the port, see environment configuration below).

Alternatively, deploy straight to Heroku:

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy?template=https://github.com/Financial-Times/polyfill-service)

For an HTTP API reference, see the [hosted service documentation](http://polyfill.webservices.ft.com).

### Environment configuration

The service reads a number of environment variables when started as a service, all of which are optional:

* `PORT`: The port on which to listen for HTTP requests (default 3000)
* `FASTLY_SERVICE_ID`, `FASTLY_API_KEY`: Used to fetch and render cache hit stats on the [Usage](https://cdn.polyfill.io/v1/docs/usage) page of the hosted documentation.  If not specified, no stats will be shown.
* `PINGDOM_CHECK_ID`, `PINGDOM_API_KEY`, `PINGDOM_ACCOUNT`, `PINGDOM_USERNAME`, `PINGDOM_PASSWORD`: Used to fetch and render uptime and response time stats on the [Usage](https://cdn.polyfill.io/v1/docs/usage) page of the hosted documentation.  If not specified, no stats will be shown.
* `GRAPHITE_HOST`: Host to which to send Carbon metrics.  If not set, no metrics will be sent.
* `GRAPHITE_PORT`: Port on the `GRAPHITE_HOST` to which to send Carbon metrics (default 2002).
* `SAUCE_USER_NAME` and `SAUCE_API_KEY`: Sauce Labs credentials for grunt test tasks (not used by the service itself)

When running a grunt task, including running the service via `grunt dev` or `grunt service`, you can optionally read environment config from a file called `.env.json` in the root of the working tree.  This is a convenient way of maintaining the environment config that you need for development.  The `.env.json` file is gitignored so will not be accidentally committed.


## Using as a library

Polyfill service can also be used as a library in NodeJS projects.  To do this:

1. Add this repo as a dependency in your package.json
   e.g. `npm install polyfill-service --save`
2. Rebuild your project using `npm install`
3. Use the API from your code

### Library API reference

#### `getPolyfillString(options)` (method)

Returns a bundle of polyfills as a string.  Options is an object with the following keys:

* `uaString`: String, required. The user agent to evaluate for polyfills that should be included conditionally
* `minify`: Boolean, optional. Whether to minify the bundle
* `features`: Object, optional. An object with the features that are to be considered for polyfill inclusion. If not supplied, all default features will be considered. Each feature must be an entry in the features object with the key corresponding to the name of the feature and the value an object with the following properties:
	* `flags`: Array, optional. Array of flags to apply to this feature (see below)
* `libVersion`: String, optional. Version of the polyfill collection to use. Accepts any valid semver expression.  This is useful if you wish to synronise releases in the polyfill service with your own release process, to ensure that incompatibilities cannot cause errors in your applications without warning.  Intended for when deployed as a service.  Not generally useful in a library context.
* `unknown`: String, optional. What to do when the user agent is not recognised.  Set to `polyfill` to return default polyfill variants of all qualifying features, `ignore` to return nothing.  Defaults to `ignore`.

Flags that may be applied to polyfills are:

* `gated`: Wrap this polyfill in a feature-detect, to avoid overwriting the native implementation
* `always`: Include this polyfill regardless of the user-agent

Example:

```javascript
require('polyfill-service').getPolyfillString({
	uaString: 'Mozilla/5.0 (Windows; U; MSIE 7.0; Windows NT 6.0; en-US)',
	minify: true,
	features: {
		'Element.prototype.matches': { flags: ['always', 'gated'] },
		'modernizr:es5array': {}
	}
});
```

#### `getPolyfills(options)` (method)

Returns a list of features whose configuration matches the given user agent string.
Options is an object with the following keys:

* `uaString`: String, required. The user agent to evaluate for features that should be included conditionally
* `features`: Object, optional. An object with the features that are to be considered for polyfill inclusion. If not supplied, all default features will be considered. Each feature must be an entry in the features object with the key corresponding to the name of the feature and the value an object with the following properties:
	* `flags`: Array, optional. Array of flags to apply to this feature (see below)

Example:

```javascript
require('polyfill-service').getPolyfills({
	uaString: 'Mozilla/5.0 (Windows; U; MSIE 7.0; Windows NT 6.0; en-US)',
	features: {
		'Element.prototype.matches': { flags: ['always', 'gated'] },
		'modernizr:es5array': {}
	}
});
```

#### `getAllPolyfills()` (method)

Return a list of all the features with polyfills as an array of strings. This
list corresponds to directories in the `/polyfills` directory.

Example:

```javascript
require('polyfill-service').getAllPolyfills();
```
