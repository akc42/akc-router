#akc-router

##The Web Site Developers Perspective

This repository has one key file, `akc-router.html` which contains two elements and a behaviour, which when used as described below will provide routing capability for a Polymer app.

Before we start, it is worth defining the requirements we are trying to achieve with a router.  This set of components assume they are the following:-

1. When ever the application is visually in a particular state (ie a particular page is selected, or a dialog presented), then this state is represented by a URL, extended from the base URL of the application, shown in the address bar of the browser.
2. When the address bar of the browser changes as a result of naviagation, it is possible for the user to go back to the previous point in the navigation using the browsers back button.
3. Having used the back button, the user can then use the forward button to return to where he went back from.
4. If the user copies the address seen in the browser address bar and passes it on to someone else, or re-uses it himself at a later time, then when it is used again (with limitations based on security, or changes to the underlying database that drives the application) this use will return the user to the same place in the application.
5. The application could be nested. This means that although at the top level of the application there might be a selection of different pages which could be displayed, some of those pages my have sub areas where again a selection might need to be made.  It is important to note, that one page may have multiple sub areas.
6. As well as the possibility of selection of one page (and within that one or more sub pages), it should also be possible to represent an open modal dialog box within the URL.
7. Each page, sub page, or dialog can have one or more parameters which effect the exact data on display and the value of these parameters should be reflected in the URL.

It is worth just discussing what routing capability means, because in general terms, Polymer based apps are quite capable of managing themselves, through he use of attributes and databinding, or through the use of element method calls. We could therefore regard routing as a passive capability, in that the app decides where to go, and then generates a URL that specifies where it is, saving it to the browser history so that user then sees the address bar change. This would meet the requirements

This router takes a different view.  Because of two factors:-
1. The application is going to have to react to a URL when presented as the first address the application sees when it starts, or when the user hits the back button, and
2. The complexity of passing many parameters, particularly optional ones to various elements in the element hierarchy.
We will drive the application routing from the URL.  If a part of the application wishes to navigate to a new place, it will fire an event with the new URL in it and an element (which is called `<akc-router>`) will listen for it and use it to make the Polymer App switch to the correct place.

In a Polymer app, the main view on the screen normally driven by an `<iron-pages>` element (or a derivative such as `neon-animated-pages`). A `selector` attribute on this element can be given a new value and it will determine which of its top level children to display. Menus and tabs, similarly have `selector` attibutes to indicate which of several options is selected.  The routing system must be capable of targetting these selector attributes and setting their value.

These requirements are handled by three components:-

* `AKC.Route` Is a behaviour that should be added to any element that should be selected if an appropriate URL is used.
*  `<akc-router>` is an element that should be placed near the top of the application and will be responsible for loading the correct state when the user actions described above take place.
*  `<akc-route-selector>` can be used (if needed) inside a custom element which contains an `<iron-pages>` style selector, to reflect the appropriate value it (the selector).

### The `AKC.Route` Behavior

`AKC.Route` behavior is added to two types of custom element:-

1. Those that sit (once the elements have been stamped out into the DOM) as direct children of an `<iron-pages>` (or derivative) element, or
2. An element controlling a modal dialog.

The behavior defines a `path` property which should be configured on the element to specify which URLs it will respond to. It should be statically defined because it is only analysed on the Polymer **Ready** callback. Because elements with this behaviour can be nested, the `path` only maps on to part of the URL. 

Each portion of the URL is called a **model** and consists of a *name* and a number of values acting as parameters to the model separated with the "/".  Actually there is one other special type of `model` which has the name "/".  This may not have any parameters, or children elements with an `AKC.Route` behavior. It is only needed if there is a need to sit alongside sibling `AKC.Route` elements, but still have something to display when the URL is missing the model portion of the others.

This `path` attribute should contain a pattern which consists of a series of segments separated by the "/" character (The initial "/" character is optional, except of course the when it the only character). The first segment is the "name" of the model. Following the name can be zero or more segments which start with a "+" character. Each one of those specifies a *definite* parameter, with the remainder of the segment being the name of that parameter. Following definite parameters are zero or more *optional* ones.  These are specified in a similar way, except that the "-" character is used instead of the plus. Finally, optionally, there may be an action list.  This is a set of names separated by ":" characters.

Because of the way the URL to model matching mechanism has to work, there are some constraints on model and action names.
* Model names are case insensitive. We recommend only using lowercase characters and avoiding using the '-' character within them.
* The model name will always be matched in preference to an optional parameter value in a url (assuming no action names) and so should be chosen such that it is unlikely to clash.  For example, a model pattern '/user/+uid/-country' means it would be extremely unwise to have a model name of 'france' that might follow it.
* An action name will always be matched in preference to an optional parameter value in a url, For example it would not be a good idea to have a model pattern '/user/+uid/-sport/fight:stop'
* To support multiple areas of the screen, the models that match them can come in any order in the url. But this is regarded as an unusual use case. So the matching algorithm will favour a child model over a sibling, and will favour a sibling over an element which is the sibling of the parent.  So use with caution. For example the URL '/user/5/edit' will match a 'user' model with a sub model of 'edit' over a pair of areas with 'user' and 'edit' capability.

When the element containing this behavior is selected, it will expose the actual values of its parameters in a `parameters` property. This property will be an Object which contains keys based on the names of the parameters. Only the keys representing paramters that are actually present in the URL will be present in the properties object.
It will expose an `action` property with a string value of the option used, unless there is no specification, when the `action` property will be the empty string.

The behaviour also defines a read/write `state` Object property with a notify:true flag and a `replay` Boolean readOnly property, also with the notify:true flag.  If the URL is being switched to as a result of the initial start up to this URL, or as a result of being switched to from another part of the application, the  `replay` property will be false, and the state object will be invalid. If the user switches to the URL through the back or forward button on the browser, any previously saved state will be presented on the `state` property and the `replay` property will be true. If the state property is changed. it will replace any state already saved (using `history.replaceState()`), so should be reflected back on the next replay.

Finally, when this element is selected as a result of a URL change, an abstract function "_routeSelected()" is called. The host element should implement this function and return false if it doesn't intend to update state state in the near future, and true if it does. (it doesn't have to wait for the state to actually change, its just an indication to it expects to change shortly.)   

### The `<akc-router>` Element

This element provides the control of routing within the application, and should be placed near, but within, the `<template is="dom-bind">` in the main index.html file.

It has the following properties
* `base` is used to define the prefix to any URL being used.  So if the application is sitting at (for example) http://example.com/blog, then `base` should be set to '/blog'
* `selected`  is a numeric property, giving the index amongst all the top level elements with `AKC.Route` behavior that has been selected as a result of the current URL.
* `404` is used to define a URL (relative or absolute) where a 404 page can be found should the current URL not be matched. It not present it will default to '/404.html'
* `bind` should be used to define a selector for the `dom-bind` template at application layer.  The default is '#app'

It will listen to `akc-route-change` events fired from elements below as they bubble up to the window, or to `popstate` events generated by the brower, and respond to them by parsing the URL, unpacking state (where appropriate) and initiating the state change.


### The `<akc-router-selector>` Element

This element is designed to site close to an `<iron-pages>` or derivative element below the first level in the hierarchy to provided the `iron-pages` `selector` property with the correct value.

It has the following properties
* `path` A list of model names, each proceeded by a '/' character, which represent models higher up the DOM hierarchy that have to be matched from a URL.
* `selected` is a numeric property that gives the 0 based index

### Examples of Use

Firstly, lets look at the overall shape of our application

Assuming `<x-page>`, `<y-page>` and `<z-dialog>` are custom elements which all have `AKC.Route` behavior, then the following example is how `<akc-router>` might be used.

```
<body unresolved class="fullbleed layout verticle">
  <template id="app" is="dom-bind">
    <akc-router base="/blog" selected="{{index}}"></akc-router>
    <iron-pages selected="[[index]]">
      <x-page path="/"></x-page>
      <y-page path="user/+id"></y-page>
    </iron-pages>
    <z-dialog path="/search"></z-dialog>
  </template>
</body>
```

The main issue is showing how an selected property from the router is being used to select a page based in whether the URL is '/base' or starts with /base/user/4 (it may have further components - see below).


Then we come to define one of custom elements `<y-page>`

```
<link rel="import" href="../polymer/polymer.html">
<link rel="import" href="../iron-pages/iron-pages.html">
<link rel="import" href="../iron-ajax/iron-ajax.html">
<link rel="import" href="../akc-router/akc-router.html">

<dom-module is="y-page">
  <template>
    <iron-ajax
      id="getuser"
      url="[[url]]"
      last-response="{{state}}"
      ><iron-ajax>
    <akc-route-selector path="/user" selected="{{index}}"></akc-route-selector>
    <iron-pages selected=[index]>
      <a-page path="/" user="[[state]]"></a-page>
      <b-page path="edit/profile:roles" user="[[state]]"></b-page>
    </iron-pages>
  </template>
</dom-module>
<script>
  Polymer({
      is:'y-page',
      behaviors:['AKC.Route'],
      properties:{
        url: {
          computed:'_computerURL(params)'
        },
        index:{
          type:Number,
          notify:true,
          value:0
        }
      },
      _computeURL:function(params) {
        return ['/api/user',params.id,'get'].join('/');
      },
      _routeSelected:function(e){
        if(!this.replay) {
          this.$.generateRequest();
          return true;
        }
        return false;
      }

  });
</script>
```


This is quite a complicated example, but is showing several things.  Firstly when the y-page element is first selected (along with its `id` parameter), its _routeSelected() function will be called. It will retrieve (partially by databinding) the id, and whether this is a replay or not.  If its not a replay, it uses `<iron-ajax>` to retrieve the `state` and set this as a property. Doing so will push the state into the browser history, so it can be retrieved if its a replay.

Secondly, the `akc-route-selector` element is used to select from a number of children of this element based on the URL. '/blog/user/4' will load the `a-page` element, a URL of '/blog/user/4/edit' will load the `b-page` element. NOTE: `a-page` and `b-page` would both have to included `AKC.Route` behavior for this to work. If they had not both done so then at least the second URL would cause 404 error.

NOTE: The use of "action" parameter in the `b-page` `path` attribute. 

##How the elements work

### The `model` Object

The elements work together by sharing a few simple data structures as element static variables.  The often have a single Object structure in common, the **model** so lets define its keys:-

* `name` The name of this model
* `element` The element that has the `AKC.Route` behavior that defines this model
* `regexp` A string, containing the regexp snippet that will be used to match the appropriate portion of the URL
* `parameters` an Object, containing a hash key for each parameter, and a string with its last value (the empty string is used of there was no last value)
* `action` A string containing the last value of an action.  An empty string will be used if the element doesn't have any action parameters, or they have not been set yet.
* `state` An Object holding the state for this model if one hasn't been supplied yet, it will be an empty object({}).
* `children` An Object Array, keyed on model name of `model` entries representing the children in the hierarchy.
* `index` The index of this element within the DOM in relation to other models
* `replay` A boolean value indicating if this model is being replayed or not.

###At the ready stage

As each element with the `AKC.Route` behavior becomes *ready* it creates a `model` object to hold the information about itself. Its adds the element itself as a parameter of this object.
It analyses its path attribute and generates part of a regular expression to match an actual URL being used, adding its name and the regular expression into the model object. It also prepares dummy `parameters`,`action` and `state` elements for the model.

In addition it scans a "RouteList" of items being prepared but not yet consolidated, looking for elements that are descendants (in the distributed nodes).  If it finds any, it removes them from the RouteList and adds them as a hashed list of children, adding the `index` field as it goes. If it finds more than one with the same name it outputs a warning to the console, but adds them anyway as an Array of entries under that particular name.

As it adds children to its own entry, it also expands the `Regexp` field, so as to add a new group with an option for each of the children (other than a "home" entry).  I home entry then causes a ? to follow the group, otherwise it is left off.

Finally, it adds itself back into the RouteList so that higher up elements will be able to find it.

`<akc-route-selector>` elements also build them self into a "Selectors" list, although this time they are only interested in being informed about changes so do no further processing.

The `<akc-router>` element needs to assure itself that it is the only router element being declared, but assuming it does, adds a listener of the `dom-ready` event on the dom-bind template specified in its `bind` property. It will use this event to complete it's setup. 

When the `dom-ready` event fires, the router needs to add the `index` field to the top level of RouteList entries as well as generate the final Rexexp from these children and the `base` parameter.  It can check the URL the application was started with as one it can handle, redirecting to the `404` page if it can't.

Finally by firing the `akc-route-change` event with this URL it causes it to be dispatched.

###Dispatching a route.

Routes are dispatched in a very similar manner whether they are requested via `popstate` or `akc-route-change` events.
The only difference is in the use of the "replay" flag, and in that case obtaining the saved `history.state` object and breaking amongst the models on the path. More on that below.

Parsing the URL with the combined regExp can split all the various parts into an Array, which then can be used update the model's entries for their paramters. For each model the router will fire a non bubbling event `akc-route-selected` with the parsed model entry on the element itself. It **must** do this in order down the model heirarchy<sup name="f1">[1](#footnote1)</sup>.

An element receiving this `akc-route-selected` event will (done by the behavior) set new parameters for the event, and will inform the host element through the _routeSelected() call._

If the state is expected to change then the behavior creates one or two promises.  One waits on the update of the `state` property of the element and is always created, the other (to wait for a `akc-route-save` event bubbling up for its children) is only created if there are children to create the event.  If that event does bubble up it is cancelled and then recreated when both promises resolve.

The `akc-route-save` is eventually propagated upwards, with a merged state object.  this routes portion of the entire state object is keyed of the model name.

Eventually this `akc-route-save` event will bubble all the way to the dom-bind template, were a listener set by the router, will collect it, and write the value into the state history using `history.replaceState()`

<a name="footnote1"><strong>1</strong></a>: It is important that elements higher up the hierarchy create a Promise to go and get data as a result of this `akc-route-newp` event before they receive the bubbling up of an `akc-route-save` event from one of their children so that they know whether to cancel it or pass it on. [↩](#f1)
