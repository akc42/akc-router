#akc-router

##Overview

This repository has one key file, `akc-router.html` which contains an element and a behaviour, which when used as described below will provide comprehensive routing capability for a Polymer app.

The two components are:-

*  `<akc-router>` which provides routing for children elements.  It will listen to popstate events, and organise to select the approriate element from its children. It will utilise <neon-animated-pages> to achieve this.

* AKC.Route Is a behaviour that should be added to either a page element that is one of the children of `<akc-router>` or may be added to any element down the heirarchy of elements below the router that will be expected to throw up a modal dialog when activated (by the related url for its route being selected). There are a few instances where a dialog element needs to exist as a direct child of an `<akc-router>`.  In these cases, a `dialog` attribute may be added to the element to force it not to be treated as a page.

An important part of the whole concept is how the URL is interpreted. The URL is split into components each of which consist of a *model* name and a number of parameters, all sepaated by a "/" character. A parameter is indicated either by starting it with a ":" character, representing a named parameter, or a ";" character representing an optional named parameter.  Definite parameters must always preceed optional ones.

Each element containg the AKC.Route behavior should have a `path` attribute which specifies the model and its parameters.  

An element with AKC.Route behavior can have children nested within it which may well also have this behavior. Doing this effectively concatenates the url for the nested elements.  If the `<akc-router>`  element is also included in the nesting, it provides the ability for a page to have sub parts of the page to swap other parts in and out.

It is important that the top level `<akc-route>` element has the attribute `home` on it so we we know when we have finished concatenating urls.  In the explanations that follow we will refer to this element as the *Home Router*

Routing is not just about pages.  Its possible for a url to represent a dialog.   An element utilising AKC.Route that does not have a direct parent which is an `<akc-router>` will be assumed to be a dialog. When the Home Router hears a route change and this route represents that dialog element it will fire an event at it telling it to open its dialog.

##More on URLs

It is expected that the akc-router components that are discussed in this document will be used in a Single Page Application, where the majority of the application is loaded through the loading of a single url with the browser or the application fetching the rest via background requests.  It would be possible to forget entirely about URLs, and they would never need to change.

However, as is well recognised, this is not ideal from a user perspective.  They see what appears to be page changes as a result of clicking (or swiping or tapping or keyboard input) and have an expectation that the browser back button will work.  Also it may be useful to pass a URL to a colleague who enters it and gets to the same place. To support this is what these elements are all about.

The key therefore to everything is the URL.  These components in this repository and for interpreting the url and performing actions when it changes.  To that extend we make some assumptions about the form of the URLs that we are going to receive, specify a "template" for them, and then interpret the URLs as actually received against the template, in order to determine what to do.  So lets define the structure of the template first, and then define how we map a given url on to that template.

###The Structure of the URL template

The template is a string. We define the structure using a BNF notation

<template> ::=  <models> | <home model> | <models> <home-model>

<home model> ::= / 

<models> ::= <model> | <model> <models>

<model> ::= <model name> <fixed params> <optional params> | <model name> <fixed-params> | <model name> <optional params> | <model name>

<model name> :: = / <name>

<name> ::= <letter> | <letter> <name-remainder>

<name remainder> ::= <symbol> | <symbol> <name remainder>

<symbol> ::= <letter> | <digit> 

<fixed params> ::= <fixed param> | <fixed param> <fixed params>

<fixed params> ::= /:<name>

<optional params> ::= <optional param> | <optional param> <optional params>

<optional param> ::= /:<name> 

What this notation implies is that the template is split up into a series of Models (with a special case for the Home Model and given the special name '/') each of which can have a combination of fixed and optional parameters, provided all the optional parameters follow the fixed ones.

Each Model is represented by an `AKC.Route` module in the DOM element hierarchy, the more left the model in the template, the higher up the DOM it is.  Each unique template should represent one of the paths down the DOM element hierarchy starting at the Home Router element and ending at the deepest nested `AKC.Route` element.

The collection of possible templates made up from all the possible paths starting at the Home Router and is represented by a tree structure of Models.

###Mapping URL to the template.

When an actual URL is to be processed by the Home Router it needs to compare it against the set of templates to see which one is applicable.  With the exception of the special case of the url of '/' (where it matches a template that only has a home model in it), the url is split into strings separated by the '/' symbol. It tries to match the first string against the names of the top level of models. If it can't find it it fails to match.

If it finds a name, it tries to assign the subsequent segments to all of the fixed parameters.  Assuming this succeeds if says it is now working on the current Model. It then needs to work out how many, if any, of the optional parameters are represented within the URL. If the next string matches the name of any of the immediate children of Model it is working on then it assumes that no more of the optional parameters are present and moves on down the hierarchy.  If none match, it can assign the optional parameter a value of the string, and then it repeats the process for the next parameters.

If, in processing the URL as defined above, it reaches the end of the URL, then either it must be working on a Model that has no more children, or one of the children must be a Router with a "Home" Model sitting beneath it.

Any failure to find a match using the above rules will result in the Home Router assuming it can no longer schedule this URL. If the failure to match came from a `popstate` event or from the url that was being scheduled at the end of the **ready** stage (see below) then we redirect to the 404 page.  If the problem occurred after the `akc-route-change` event, then the router will throw an `akc-route-error` error.

###Redirecting to a 404 page.

We will assume that whatever server is configured to run this application, its configured to check if real files exist and serve them, else to actually serve the main index.html file of the application regardless of the URL.

The Home Router has an optional "404" attribute which can define the url of a 404 page.  It defaults to "/404.html". It will use this if it can't find a match

##How the elements work

###At the ready stage

As each element with the `AKC.Route` behavior becomes *ready* and adds itself to list of route nodes being prepared, along with its path attribute.  It will also analyse the model parameters that the path attribute implies and prepare properties to hold the actual values of those parameters when the url fires. 

When an `<akc router>` element becomes ready it just adds itself to the list of route nodes being prepared, as it does not have any model to worry about.

In addition, both of the above elements will scan the list of nodes being prepared, and check if they are a descendent of themselves.  For those that are:-

* It will remove them from the list of route nodes being preared and add them as a child element of its own entry 
* For an `<akc-router` element, it checks if is is the direct parent of the node, it which case :-
  * For `AKC.Route` nodes it marks them as a page unless they have the `dialog` attribute.
  * For `<akc-router>` nodes, it removes them and adds all its children to its own list of children.
* For an `<akc-router` which is the Home Router it will set up listeners for `windows.onpopstate`, some of the other custom events (see below).

Note it is possible that someone has created an element with these properties outside of the children of the Home Router.  When the Home Router's ready callback is called they may or may not already be in the route list.  If they are, the Home Router will throw a custom error `akc-route-error`.  If the route appears in the route-list after the Home Router is ready, then it will check it later, when responding to other events, and throw the error then.

Finally, the Home Router checks the current url and dispatches the appropriate Page or Dialog.

### When the Application or user wants to change route.

In general, we want to avoid allowing the user to click on something and have it dispatch a request to the server.  If they do, then the server should be configured to return the same resource (generally index.html).  The process in the previous section repeats itself as the page reloads and we end up were we intended to be, albeit at cost of performance.

A better approach will be for the application to fire an event and for the various elements in the hierarchy to listen for those events, and then perform the appropriate actions.

There are five such events that will be used:-

1. `akc-route-change` This will be when the application wants to request a change to a new url.
2. `akc-route-cancel` The application wishes to return to the previous page (dialog cancel is the most like source of this event)
3. `akc-route-save` This is when the application has obtained the data related to its model parameters and wants to save the data it might have obtained.
4. `akc-route-page` This is when the Home Router wants to tell a sub page
5. `akc-route-dialog` This is when a router wants to tell a dialog to open.

#### akc-route-change and popstate event

The `akc-route-change` event can be fired by anywhere in the application (provided it is a descendant of the Home Router), and will picked up by the Home Router.  It will call `history.pushState()` to save the change of route to the browser history.

The `popstate` event will be fired when the user has selected the back button on the browser. The difference in this case, is that the url, and by extrapolation, the models of all the `AKC.Route` elements it refers to will have recently retrieved key state data from the server.  We retrieve this as a result of this event, and include it in the notifications below. 

We fire either a `akc-route-page` event or a `akc-route-dialog` event at the `AKC.Route` element whose path is represented by the right hand end of the url.

#### akc-route-cancel event

This event will be fired by an element that has just closed its dialog box as a result of a cancel action.  The Home Router will listen for this and call `history.Back()` to return to the previous page.  Note, this should result in a `popstate` event firing.

#### akc-route-save event

This event is fired by an element when it has just retrieved key model data as a result of page change and it hasn't got any descendents who might be also wanting to save data.  It will be listened to by each `AKC-Route` element up the document hierarchy and possibly **cancelled**.  This receiving element will not cancel this save event if it has nothing of its own to add to the state (because the url change last seen didn't change its model parameters), but if it too is collecting data as a result of a page change it will delay the event until it also has its data, and will pass it on combined.

When the Home Router eventually receives this event it uses `history.replaceState()` to put the new state object against the current url.

#### akc-route-page event

The Home Router fires this event as a result of url change.  It will fire it directly<sup name="f1">[1](#footnote1)</sup> to the  `AKC.Route` page element at the leaf node of the hierarchy represented by the url.  We allow it to bubble up the hierarchy.  If this event is the result of a `popstate` event, it will include a marker to say so, and include the current `history.state`.

`AKC.Route` elements will use it to update their model (and then potentially save the state), `<akc-router>` elements will use it to inform their `<neon-animated-pages>` element to change page.

It is important that whenever it a element receives this event directly (ie it is the target) and it is **not** a `popstate` related event, it fires the `akc-route-save` event (asynchronously), even if there is nothing to save, because elements above it might be waiting for it. 

#### akc-route-dialog event

The Home Router fires this event directly on the Dialog element it wants to open. Other `AKC.Route` elements up the hierarchy will listen to for it bubbling up to decide if to update their own state.  As with `akc-route-page` events `popstate` handling and `akc-route-save` handling should be the same.

### Key Data Structures

The `RouteList` is used during the **ready** phase of the elements, and is used to hold a list of **model** elements as they are being formed into the full model hierarchy.  Each `model` contains the following fields:-
* `name` The name of this model
* `element` The element that processes this model
* `fixed` An Array of Strings, representing the names of the fixed parameters
* `optional` An array of Strings, representing the names of the optional parameters
* `values` A Object as a Key'ed array of values at last url change for each name in the `fixed` and `optional` Arrays
* `children` An Array of `model` entries representing the children in the hierarchy.  They are ordered to match the ordering in the DOM.
* `isPage` A Boolean flag to show that this is an element that should be displayed by a higher level `<akc-router` element (If this parameter is `false` then either the `element` field will say this is an `akc-router`, otherwise it is a dialog)

A `State` object is created as passed up the Model hierarchy (by event bubbling) when the `akc-route-save` event is fired.  The object is a keyed Array, which each key being the module name.  The state can be anything the element using the `AKC.Route` behavior thinks is appropriate in order to be able to reproduce its state when the browsers back button is used.  As this object makes its way up the Model hierarchy, each Model will add its own key to this object.


<a name="footnote1"><strong>1</strong></a>: It is important that elements higher up the hierarchy create a Promise to go and get data as a result of this `akc-route-page` event before they receive the bubling up `akc-route-save` event from one of their children so that they know whether to cancel it or pass it on. Until I get further into the implementation and testing it I don't know if my assumption that this will happen this way is true.  The alternative is have the Home Router fire events to all elements down the hierarchy in turn, starting at the highest first, and have each one cancel it, so it doesn't buble back up again.  Another alternative much be to make the events track down the hierarchy, but I am led to believe this doesn't work well. [↩](#f1)
