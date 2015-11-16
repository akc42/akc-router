#akc-router

##The Web Site Developers Perspective

This repository has one key file, `akc-router.html` which contains two elements and a behaviour,
which when used as described below will provide routing capability for a Polymer app.

Before we start, it is worth defining the requirements we are trying to achieve with a router.  This
set of components assume they are the following:-

1. When ever the application is visually in a particular state (ie a particular page is selected, or
a dialog presented), then this state is represented by a URL, extended from the base URL of the
application, shown in the address bar of the browser.

2. When the address bar of the browser changes as a result of naviagation, it is possible for the
user to go back to the previous point in the navigation using the browsers back button.

3. Having used the back button, the user can then use the forward button to return to where he went
back from.

4.If the user copies the address seen in the browser address bar and passes it on to someone else,
or re-uses it himself at a later time, then when it is used again (with limitations based on
security, or changes to the underlying database that drives the application) this use will return
the user to the same place in the application.

5. The application could be nested. This means that although at the top level of the application
there might be a selection of different pages which could be displayed, some of those pages my have
sub areas where again a selection might need to be made.  It is important to note, that one page may
have multiple sub areas.

6. As well as the possibility of selection of one page (and within that one or more sub pages), it
should also be possible to represent an open modal dialog box within the URL.

7. Each page, sub page, or dialog can have one or more parameters which effect the exact data on
display and the value of these parameters should be reflected in the URL.

It is worth just discussing what routing capability means, because in general terms, Polymer based
apps are quite capable of managing themselves, through he use of attributes and databinding, or
through the use of element method calls. We could regard routing as a passive capability, in that
the app decides where to go, and then generates a URL that specifies where it is, saving it to the
browser history so that user then sees the address bar change. This would meet the requirements

This router takes a different view.  Because of two factors:-

1. The application is going to have to react to a URL when presented as the first address the
application sees when it starts, or when the user hits the back button, and

2. The complexity of passing many parameters, particularly optional ones to various elements in the
element hierarchy. We will drive the application routing from the URL.  If a part of the application
wishes to navigate to a new place, it will fire an event with the new URL in it and an element
(which is called `<akc-router>`) will listen for it and use it to make the Polymer App switch to the
correct place.

In a Polymer app, the main view on the screen normally driven by an `<iron- pages>` element (or a
derivative such as `neon-animated-pages`). A `selector` attribute on this element can be given a new
value and it will determine which of its top level children to display. Menus and tabs, similarly
have `selector` attibutes to indicate which of several options is selected.  The routing system must
be capable of targetting these selector attributes and setting their value.

These requirements are handled by three components:-

* `AKC.Route` Is a behaviour that should be added to any element that should be selected if an
appropriate URL is used. If an element has this behavior it is said to be *routable* . These can be
nested in the DOM hierarchy. Routable elements at the same level in the hierarchy will be alternate
routes, whilst those with children will include those children as part of their alternate path.

*  `<akc-router>` is an element that should be placed near the top of the application and will be
responsible for loading the correct state when the user actions described above take place.

*  `<akc-route-selector>` can be used (if needed) either at the top level of the application or
inside a custom element. Its use is subtly different as to whether it has *routable* children or
not. With children, it will select only from its children, but more importantly, the segment of URL
it is going to generate a match for one of its children, will not be an alternate match with its
routable siblings, but an additional component in the URL. The siblings components expect to be
alternate matches at one further chunk down the url.  If it doesn't have children it will select
from its routable siblings.  To select an item it has a `selected` attribute on which it will output
an `<iron-pages>` style selector.

### The `AKC.Route` Behavior

`AKC.Route` behavior is *normally* added to two types of custom element:-

1. Those that sit (once the elements have been stamped out into the DOM) as direct children of an
`<iron-pages>` (or derivative) element, or

2. An element controlling a modal dialog.

The behavior defines a `path` property which should be configured on the element to specify which
URLs it will respond to. It should be statically defined because it is only analysed onced by the
router once all the elements have been added to the DOM. Because elements with this behaviour can
be nested, the `path` only maps on to part of the URL.  

Each portion of the URL is called a **model** and consists of a "/" followed by a *name* and a
number of values acting as parameters to the model separated with the "/". Actually there is one
other special type of `model` which has the name "/". This may not have any parameters, or
children elements with an `AKC.Route` behavior. It is needed  to sit alongside sibling `AKC.Route`
elements with names, but still have something to display when the URL is missing the model portion
of the.

This `path` attribute should contain a pattern which consists of a series of segments separated by
the "/" character (The initial "/" character is optional, except of course the when it the only
character). The first segment is the "name" of the model. Following the name can be zero or more
segments which start with a "+" character, followed by either "d" or "s". Each one of those
specifies a *definite* parameter, d specifying that it should only consists of digits (0-9) or s
specifying that the parameter should be a string (\w from RegExp). Following definite parameters are
zero or more *optional* ones.  These are specified in a similar way, except that the "-" character
is used instead of the plus. Finally, optionally, there may be an action list.  This is a set of
names separated by ":" characters.  If the first character of the action options is a ":" it means
that the action part of the model is optional (ie the URL may or may not contain it.)

Because of the way the URL to model matching mechanism has to work, there are some constraints on
model and action names.

* Model and Action names are case insensitive and must match the \w (from RegExp) characters. We
recommend only using lowercase characters. The same is true of the parameters after the "+" or "-"
signs, but of course they will not be represented literally in the URLs.

* Because the entire URL is matched in one go, a model name will be matched in preference to an
optional parameter value (assuming no action names). Care must be taken with the overall URL
strategy to avoid such ambiguities.

* An action name will always be matched in preference to an optional parameter value in a url, For
example it would not be a good idea to have a model pattern '/user/+d/-s/fight:stop' when the second
(optional) parameter might be the name of a sport.

When the element containing this behavior is selected, it will expose the actual values of its
parameters in a `parameters` property. This property will be an Array containing the values in the
order specified in the *path*. Only the values are actually present in the
URL will be present in the properties Array. The element will also expose an `action` property with
a string value of the option used, unless there is no specification, when the `action` property will
be the empty string.

The behaviour defines a read/write `state` Object property with a notify:true flag.  If the URL is
being switched to as a result of the initial start up to this URL, or as a result of being switched
to from another part of the application, the host element is responsible for getting the state based
on the `parameters`.  Since this is likely to be an asynchronous option, the `AKC.Route` module asks
the host element to get the state by calling an abstract routine called `_routeChange` from within a
Promise, with two (resolve and reject) callbacks.

A successful resolution will provide `AKC.Route` back with the state, which is then exposed on the
property, but perhaps more importantly, co-ordinates with the other elements also involved with this
URL so that the state can be passed back to the browsers history.  The reject option expects a new
URL to switch to.  This can be used if invalid parameters have been passed in a URL and the element
wants to redirect to somewhere to tell the user.

If the user switches to the URL through the back or forward button on the browser, any previously
saved state will seperated out from the other models and put back to the property. At that point the
behavior will call another abstract function, `_replayURL` which the host element can use (if it
needs to) to do any other operations it would normally undertake once the state had been aquired.

Elements may optionally include a `dialog` property.  When it does, it tells the router to exclude
this element from the list of elements to *select* when deciding what value to put out on the
`selected` attribute.  It's index will still be counted from the start of the level, so don't embed
one as a direct child of an `iron-pages` element.


### The `<akc-router>` Element

This element provides the control of routing within the application, and should be placed near, but
within, the top `<template is="dom-bind">` in the main index.html file.

It has the following properties

* `base` is used to define the prefix to any URL being used.  So if the `application is sitting at
(for example) http://example.com/blog, then `base` `should be set to '/blog'

* `selected`  is a numeric property, giving the index amongst all the top level elements with
```AKC.Route` behavior that has been selected as a result of the current URL.

* `url` the current url is being managed. If the last segment of the url had a "." character in it,
it is assumed this is a filename, and has been stripped off. This is provided primarily for
diagnostic and testing purposes, although of course any element can use it if they wish.

It will listen to `akc-route-change` events fired from elements below as they bubble up to the
window, or to `popstate` events generated by the brower, and respond to them by parsing the URL,
unpacking state (where appropriate) and initiating the state change.

This element works out which models to dispatch, but also works out which value to place in the
`selected` property based on its position amongst siblings at the same level.

If the URL passed does not match the ones it can handle, it will fire an `akc-route-404` event with
the passed url as its parameter. The app itself can decide on a policy to adopt in this case.

The element also has a single public function `DomOk` which returns a *Promise* which will be
fulfilled when the router has finished parsing the DOM.  It will be *Resolved* if the DOM was OK
(with the regex it is going to use  parse incoming URLs passed as a parameter to a *then* function),
it will be *Rejected* with an error containing the failure if the DOM wasn't correctly parseable.

### The `<akc-router-selector>` Element

This element is designed to site close to an `<iron-pages>` or derivative element below the first
level in the hierarchy to provided the `iron-pages` `selector` property with the correct value. If
it has routable children, it is expected that this `iron-pages` element should also one of its
children.

It has the following properties  

* `selected` is a numeric property that gives the value of its children within the overall DOM tree.

Its position in the DOM tree is worked out automatically by the router.

As indicated above it can be used with or without children.  Imagine a scenario where we are
developing an appointment booking system for a doctors surgery.  We might have a page where the top
of the page contains either a form to register a new patient, alternate details of an existing
patient.  Below that we might have an area which lists each doctor and his patients for the day, and
below that an area where if one of the doctors lists are clicked it brings up some details about the
doctor (which room he is in, when is is arriving and leaving, perhaps some notes).  We would build
the appointment element somthing like this (those elements with the path attribute include an
`AKC.route` behavior).

```
<akc-route-selector selected="{{segment}}">
  <iron-pages selected="[[segment]]">
    <new-patient path="new"></new-patient>
    <patient-summary path="patient/+d"></patient-summary>
  </iron-pages>
</akc-route-selector>
<template is="dom-repeat" items={{doctors}}>
  <appointment-list doctor="[[item]]"></appointment-list>
</template>
<doctor-detail path="doctor/+d"></doctor-detail>
```

This piece of the application would therefore respond to urls like this:-
* /appointment/20151115/new/doctor/4 
* /appointment/20151115/patient/238/doctor/2

See how in these cases the "doctor" chunk of the url is an additional element to the url, where as
"new" and "patient" elements are alternates.

### Examples of Use

Firstly, lets look at the overall shape of our application

Assuming `<x-page>`, `<y-page>` and `<z-dialog>` are custom elements which all have `AKC.Route`
behavior, then the following example is how `<akc-router>` might be used.

```
<body unresolved class="fullbleed layout verticle">
  <template id="app" is="dom-bind">
    <akc-router base="/blog" selected="{{index}}"></akc-router>
    <iron-pages selected="[[index]]">
      <x-page path="/"></x-page>
      <y-page path="user/+d"></y-page>
    </iron-pages>
    <z-dialog path="/search/+s" dialog></z-dialog>
  </template>
</body>
```

The main issue is showing how a selected property from the router is being used to select a page based on whether the URL is '/blog' or starts with
/blog/user/4 (it may have further components - see below).


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
    <akc-route-selector selected="{{index}}"></akc-route-selector>
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
        return ['/api/user',params[0],'get'].join('/');
      },
      _routeChange:function(resolve,reject){
        this.$.getuser.addEventListener('response',response) {
          //do something
          resolve(tresponse);
        }
      }

  });
</script>
```


This is quite a complicated example, but is showing several things.  Firstly when the y-page element
is first selected (along with its numeric `id` parameter), its `_routeChange()` function will be called. It will retrieve (partially by databinding) the id.  It uses `<iron-ajax>` to retrieve the `state` and set this as a property.
Doing so will push the state into the browser history, so it can be retrieved 
in the case of the user using the back button.

Secondly, the `akc-route-selector` element is used to select from a number of children of this element based on the URL. '/blog/user/4' will load the
`a-page` element, a URL of '/blog/user/4/edit' will load the `b-page` element.
NOTE: `a-page` and `b-page` would both have to have included `AKC.Route` behavior for this to work. If they had not both done so then at least the
second URL would cause the router to fire an `akc-route-404` event.

NOTE: The use of an "action" parameter in the `b-page` `path` attribute. 

##How the elements work

### The `model` Object

The elements work together primarily because the `akc-router` element waits for all the nodes to
have been stamped into the DOM, and then scans the DOM looking for its partner elements. It builds a
tree of **model** records which it can then use to match against incoming urls to work out which
elements to activate, and with what parameters.  Each **model** record has the following keys:-

* `isSelector` set to true if this model represents an `akc-route-selector` model,
* `isHome` set to true if this model represents an `AKC.Route` element with a *path* attribute of
'/'
* `isDialog` set to true if this element is an `AKC.Route` element with the *dialog* attribute set.
* `name` If this model record is not a selector or home element, this name is the first part of the
*path* attribute that will be matched against part of the url.  If it is a selector or home element
then it will be a '/' seperated list of names of the previous `AKC.Route` or `akc-route-selector`
elements higher up the DOM. `akc-route-selector` elements are given the name "*".
tree. 
* `element` The element that has the `AKC.Route` behavior that defines this model. When discovered,
the behavior's `route` property will be given a '/' separated list of names of the previous
`AKC.Route` elements higher up the DOM tree, with their own name as the last item on the list.
* `parameters` an Array, containing the name of each parameter (fixed or optional) in order parsed
from the path attribute.
* `fixed` the number of fixed parameters (ie ones that must be present in a url).  The number of
optional parameters can be determined as the difference between the length of the `parameters` Array
and `fixed`.
* `actions` An Array of action names found when parsing the path.
* `hasNullAction` is true if there is no action defined or one of the Actions supplied is null (ie
a URL without the action part).
* `children` An Array of **model** entries representing the children in the DOM hierarchy if order
of discovery.

###At the creation stage

The `akc-router` creates the promise which it exposes to the `DomOk` function which it will later
fulfill or not.

###At the attached stage

The `akc-router` will attempt to instantiate itself by checking to see if it is the first to do so. 
It does this by using a static variable to indicate whether another element of this type is already
active.  If there is already one, it exits, removing itself from the DOM in the process.

Assuming it is the active router it waits until the rest of the DOM has been attached (using the
Polymer Async function) and the proceeds to analyse the DOM, looking for `akc-route-selector`
elements and elements with `AKC.Route` behavior.  As it finds these it builds up the **model**
described above.

It does this in two passes.  In the first pass it builds the framework of the model tree, whilst in
the second pass it takes each element in turn and checks it in detail to develop the full **model**
record for each element.  At the same time it constructs a Regular Expression string which it can
use to match against incoming URLs.

Finally, it then resolves the `DomOk` promise.

###Dispatching a route.

Routes are dispatched in a very similar manner whether they are requested via `popstate` or `akc-route-change` 
events. The main difference is that `popstate` do not create a new history element, but instead
unpack the previously saved state and pass it to the event, see below, which is sent to the
appropriate `AKC.Route` elements. 

The URL we are dispatching to is checked with the Regular Expression built during the "attached"
stage and an Array (the **matches** array) of matching elements is created by the regular expression
parser. The **model** and **matches** arrays are then passed to a parser function.  This parser
function takes each of the model entries at that level and tries to match then with the first entry
in the matches array.  

If found we can parse the rest of the model record (looking for parameters and actions).  As we do,
we remove the matching entry from the **matches** array.  We can then try and parse the next level,
using the parser function recursively on the model record's children.

At any level the parser function may not find a match.  This is an error, and the web designer has
probably misconfigured the elements in some way.  The router will throw an error.

Eventually, other than in the error situation just described, the end of the URL will have been
reached and we will have reached the bottom level of the model hierarchy.  The router will send this
element a `akc-route-selected` event.  It will contain within it all the parameters for all the
elements down the path in the hierarchy (and possibly their state too, if this is a replay).  As the
event bubbles up the DOM tree, each routable element picks up their related information.

For the situation in which it is **not** a replay, each routable element **must** follow this event
with an `akc-route-save` event. Again this is started at the lowest level on the hierarchy and other
elements higher up, wait for it to bubble up to them.  Here, they cancel it, take the information
and add their own, then fire it again.  When it eventually reaches the top level (parent of the
`akc-router` element) the router sees it and uses it to replace the history state.

