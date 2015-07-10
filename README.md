# Meteor/React Example

This package contains an example app that uses Meteor and React together. It
demonstrates:

* Using the `react` package to integrate Meteor and React
* Fetching data from reactive Meteor data sources using the `ReactMeteorData`
  mixin and the `getMeteorData()` method
* Routing using `ReactRouter`
* A basic UI using `Twitter Bootstrap` and `ReactBootstrap`
* Login, change password, enrollment emails and user management using the Meteor
  accounts system
* ES6 syntax using the `Babel` transpiler (`.es6.js` files) and ReactJS'
  transpiler (`.jsx` files)
* Tests using `Velocity`.
* Typed collections / validation using `aldeed:collection2`.
* Client- and server-side dependencies loaded from NPM and served with
  `browserify`.
* JSHint configuration to allow editors like Sublime Text or Atom to warn
  about potential problems early

The app is very simple: it shows a ticking clock and allows the user to record
a timestamp with a name. Most of the sample app-specific logic can be found
in `client/components/timestamps.jsx` and `lib/models.es6.js`, with references
from `client/components/app.jsx`, `client/components/navigation.jsx`,
`client/js/routing.jsx` and `client/css/example.css`.

The various files have comments explaining their purpose and logic. At a high
level:

* `client/` contains various React UI components and other client-side-only
  code. Most of these are relatively re-usable, for example the login,
  change password, enrollment and user management components.
* `server/` contains server-only functionality such as application
  initialisation and methods.
* `lib/` contains shared code such as model/collection definitions and
  publications.

# Approach to dependencies and modularisation

Meteor does not (yet!) have a good approach to modularising applications. In
short, all of the JavaScript code intended for the client is loaded into the
browser as `<script />` tags, and so it is easy to end up with load-order
problems. Meteor does have a packaging system that supports dependency
management and imports, but it is quite ideosyncratic and requires that
any library found on e.g. NPMJS is wrapped into a Meteor package.

In this example, we have taken the following approach:

* We have created a "private" package called `packages/app-deps` (using
 `meteor create --package`).
* In the `package.js` file, we declare dependencies on published NPM modules
  using the standard Meteor API for this. When the package is built, this will
  cause Meteor to download the relevant modules.
* We use the `cosmos:browserify` package to run `Browserify` when the package is
  built to create a bundle that can be sent to the client. In the file
  `client.browserify.js` (plus some relevant options in
  `client.browserify.options.json`), we `require` the relevant modules and
  put them into a global object called `Dependencies`. This is then exposed
  to the client-side Meteor app via some configuration in `package.js`. We do
  something similar for server-side dependencies in `server.js` in the package.

This gives us a way to access NPM-published dependencies via the `Dependencies`
object.

The next problem is how to share components across files and make file lenghts
manageable, without too great a dependency on the load order of scripts or
the risk of variables leaking into the global namespace.

To address this, we use anonymous self-calling function blocks to wrap "module"
code, and export specific variables under a limited number of global "namespace"
objects.

We define these in `lib/_namespaces.es6.js`. By living in `lib/` and having
an early-sort name, we know it will load "early" and so its objects should
be defined in any other file where we use it:

    /* global Collections: true, Components: true, Utils: true */
    Collections = {};
    Components = {};
    Utils = {};

We then use these to export objects, and also import things from the global
`Dependencies` object. For example, in `client/components/loading.jsx`:

    /* jshint esnext:true */
    /* global Meteor, Dependencies, Components, React */
    "use strict";
    (function(NS) {
    var { _, ReactBootstrap } = Dependencies;
    var { Modal, ProgressBar } = ReactBootstrap;

    var Loading = React.createClass({
        displayName: 'Loading',

        render: function() {
            return (
                <div className="please-wait modal-open">
                    <Modal
                        title='Please wait...'
                        backdrop={false}
                        animation={false}
                        closeButton={false}
                        onRequestHide={() => {}}>
                        <div className='modal-body'>
                            <ProgressBar active now={100} />
                        </div>
                    </Modal>
                </div>
            );
        }

    });

    _.extend(NS, { Loading });
    })(Components);

The `(function(NS) { ... })(Components)` block makes a namespaced block. We
pass in the global `Components` object as the intended namespace (referenced
as `NS` in the block itself), and then use `_.extend()` from lodash to expose
it as `Components.Loading`.

At the top of the block, we import objects from the global `Dependencies`
object, here using fancy ES6 "destructuring" syntax.

In other components, we can do e.g.:

    var Timestamps = React.createClass({
        displayName: 'Timestamps',
        mixins: [ReactMeteorData],

        getMeteorData: function() {
            var user = Meteor.user();
            var subscriptionHandle = Meteor.subscribe("timestamps");

            return {
                loading: !subscriptionHandle.ready(),
                timestamps: Collections.Timestamps.find().fetch(),
                canWrite: user? Roles.userIsInRole(user, ['write', 'admin']) : false,
            };
        },

        render: function() {
            var { Loading } = Components;

            // Show loading indicator if subscriptions are still downloading

            if(this.data.loading) {
                return <Loading />;
            }

            return ( ... )
        }
    });

Note that here we import `Loading` from `Components` inside the `render()`
function rather than globally. That is because whilst we can be pretty sure
all of the namespaces and the `Dependencies` object are loaded "early", we
don't want to rely on the load order of the scripts that add objects to
`Components` and so we don't want to use it on load.

It is likely that in a future release of Meteor, all this madness goes away.
