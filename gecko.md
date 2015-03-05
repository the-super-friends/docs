# hacking on gecko

## links

### autocomplete

It looks like the autocomplete code loosely binds a dropdown box and search providers, as well as providing caching for that particular field. So, any XUL form field could provide autocomplete search of history or previously-entered strings. There is also a notion of a remote autocomplete service, so there's already some code to handle firing an XHR at a server and expecting a specific type of response. Rather than mess with that base autocomplete class (which is used everywhere), we will be better off to create our own custom autocomplete provider and our own custom autocomplete popup UI.

- MDN docs for using autocomplete within XUL
  - https://developer.mozilla.org/en-US/Add-ons/Code_snippets/Autocomplete
  - https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/Textbox_%28Toolkit_autocomplete%29

- Implementing a custom autocomplete search component: simple instructions with an example
  - It's basically just implementing the `nsIAutoCompleteSearch` interface correctly & registering it correctly as a service
  - https://developer.mozilla.org/en-US/docs/How_to_implement_custom_autocomplete_search_component

### general XUL-related links/info

- Working with XUL popups, like the autocomplete results dropdown: https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/PopupGuide

- Working with windows from XUL: opening windows, accessing content windows, passing data between windows (many approaches): https://developer.mozilla.org/en-US/docs/Working_with_windows_in_chrome_code

- Messaging between web content and chrome (code snippets): https://developer.mozilla.org/en-US/Add-ons/Code_snippets/Interaction_between_privileged_and_non-privileged_pages
  - basically, either custom DOM events or postMessage
  - if postMessage, the final arg to the event listener in privileged chrome code must be 'true'
  - a third option: Chromium-style JSON message passing

- Displaying HTML inside XUL using iframes:
  - iframe vs browser XUL element types: https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/Tutorial/Content_Panels
  - XUL iframe defn (!= HTML iframe defn) https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/iframe
  - how to reduce permissions granted to web content inside a XUL iframe (XUL iframes, even if loading untrusted content, have root-level permissions by default): https://developer.mozilla.org/en-US/docs/Displaying_web_content_in_an_extension_without_security_issues
    - we will be rendering external content in our results, so we want to treat it as untrusted

### OpenSearch / adding search providers

- adding search providers is ridiculously easy: https://developer.mozilla.org/en-US/Add-ons/Creating_OpenSearch_plugins_for_Firefox

- SUMO page explaining how to enable different kinds of content in urlbar autocomplete results: https://support.mozilla.org/en-US/kb/awesome-bar-find-your-bookmarks-history-and-tabs#w_how-can-i-control-what-results-the-location-bar-shows-me

- ancient mozillazine article includes links to Firefox 3-era urlbar modifier addons: http://kb.mozillazine.org/Disabling_autocomplete_%28Firefox%29

## Reading the Code / Orienteering

### autocomplete-related interfaces - based on MDN, which omits some of them

Beware: MDN doesn't distinguish between obsolete xpfe interfaces (like `nsIAutoCompleteItem`) and toolkit interfaces used in modern Firefox. Everything we care about is under the top-level `/toolkit` module.

TODO: this needs another pass, the code is in a semi-refactored state, MDN is slightly inaccurate, need to reconcile everything

- Major interfaces used by the autocomplete service:
  - `nsIAutoCompleteSearch` ([MDN docs](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Reference/Interface/nsIAutoCompleteSearch), [C++ interface source](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/autocomplete/nsIAutoCompleteSearch.idl)) - object representing autocomplete search provider as an XPCOM service. (Note, the `SearchSuggestionController` (`toolkit/components/search/SearchSuggestionController.jsm`) [code comments](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/search/SearchSuggestionController.jsm#31-37) say it was extracted from the `nsSearchSuggestions.jsm` file to remove the dependency on the `nsIAutoCompleteSearch` interface. Not yet sure why.)
    - Exposes a single method, `startSearch`, which is passed the search query, and an `nsIAutoCompleteObserver` that accepts results in the form of an `nsIAutoCompleteResult` object. (This is a simplified summary; see the MDN page for full details and optional parameters.)
  - `nsIAutoCompleteObserver` ([MDN docs](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Reference/Interface/nsIAutoCompleteObserver), [dxr code](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/autocomplete/nsIAutoCompleteSearch.idl#34)) - exposes `onSearchResult` and `onUpdateSearchResult` methods (updating allows for asynchronous insertion of values from a remote service, I think).
  - `nsIAutoCompleteResult` ([MDN docs](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Reference/Interface/nsIAutoCompleteResult), [dxr code](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/autocomplete/nsIAutoCompleteResult.idl)) - represents the result of a search. The interface name is a bit ambiguous - this isn't an individual result from a search, it's an individual response from the search service for a given query. Contains result status (did search find a match, not find a match, or timeout); result count; and the actual results.
    - `nsIAutoCompleteSimpleResult`, an interesting subclass ([C++ idl](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/autocomplete/nsIAutoCompleteSimpleResult.idl), [header file](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/autocomplete/nsAutoCompleteSimpleResult.h), [C++ impl](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/autocomplete/nsAutoCompleteSimpleResult.cpp)) - used by satchel's `nsFormFillController`, and the places `nsPlacesAutoComplete.js` and `UnifiedComplete.js` files, TODO explore further

- Major interfaces used by the autocomplete UI layer (TODO needs updating):
  - [`nsIAutoCompleteController`](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Reference/Interface/nsIAutoCompleteController) - it's a controller but doesn't directly observe DOM; instead, observes user input / key press events fired by a `nsIAutoCompleteInput`, which could either be a XUL autocomplete textbox or a form fill controller that handles website form history
  - [`nsIAutoCompleteInput`](https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XPCOM/Reference/Interface/nsIAutoCompleteInput) - monitors the input in a text field and displays an autocomplete panel at the appropriate time
  - `nsIAutoCompletePopup` ([C++ interface](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/autocomplete/nsIAutoCompletePopup.idl), [XUL implementation](https://dxr.mozilla.org/mozilla-central/source/toolkit/content/widgets/autocomplete.xml#45)) - binds an `nsIAutoCompleteInput` object to an `nsIDOMElement`


### docs generated by actually reading the code, organized by filename:

`/browser/base/content/browser.js`
  - assembles the modules
  - `gURLBar` is defined here on the global window object

### Views

- `/browser/base/content/browser.xul`
  - defines an `autocomplete-richtextbox` entity (i.e. an xml tag name)
    - this corresponds to `PopupAutoCompleteRichResult`
    - which is defined in `components/search/content/search.xml`
  
- `/browser/base/content/urlbarBindings.xml`
  - urlbar UI handlers defined inside an XML file
  - urlbar extends `content/bindings/autocomplete.xml`, not sure if that really means
    `toolkit/content/widgets/autocomplete.xml` which seems to have similar stuff in it

- `/toolkit/content/widgets/autocomplete.xml`
  - UI definitions for the autocomplete UI that drops down

### Controllers

- `/toolkit/components/autocomplete/nsAutoCompleteController.cpp`
  - `HandleText` method contains logic around when to search & waiting for user to stop typing
  - `StartSearches` method - set a timer, call `StartSearch`
  - `StartSearch` - loop over `mSearches`, call `startSearch` on (some of) them
  - `SetInput` - assemble `mSearches` by getting pointers to search services, specifically, by getting a "contract id string" from an instance of the `nsIAutoCompleteInput` interface.

### Services

These are implementations of autocomplete interfaces I've found in the gecko codebase:

- `/toolkit/components/places`
  - TODO look at other files in this directory
  - `/toolkit/components/places/nsPlacesAutoComplete.js`
    - This file is incredibly helpful. We can lean on this implementation when defining our own autocomplete implementation (though we need to investigate how other search providers integrate into autocomplete. I've found an OpenSearch specification, need to explore further)
    - Autocomplete strings are sent here, to be queried against the Places SQLite DB
    - see "Smart Getters" section for SQL queries corresponding to keyword, bookmark, "frecency", etc searches
    - `startSearch` function is very helpful reading

- `/toolkit/components/filepicker`
  - Looks like an implementation of the autocomplete service interfaces:
    - [`nsFileComplete`](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/filepicker/nsFileView.cpp#188) - C++ code, implements `nsIAutoCompleteSearch`
    - [`nsFileResult`](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/filepicker/nsFileView.cpp#37) - C++ code, implements `nsIAutoCompleteResult`
  - I don't see autocomplete-specific UI elements. Instead, I see classic MVC:
    - [`/tookit/components/filepicker/content/filepicker.xul`](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/filepicker/content/filepicker.xul) - a very simple View: template for the file picker dialog 
    - [`nsFilePicker` (`/toolkit/components/filepicker/nsFilePicker.js`)](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/filepicker/nsFilePicker.js) - a large Model: operates on the underlying file system using various C++ iterators/enumerators; manages the results by setting properties on itself; its state is rendered into the `filepicker.xul` file picker dialog by the `filepicker.js` controller
    - `nsFileView`
      - ([`C++ interface`](), [`C++ class`](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/filepicker/nsFileView.cpp#219)) -  implements `nsITreeView` - looks like a base C++ class and interface, I'm not totally clear on the boundary between C++ and JS implementations of these interfaces
      - [`JS implementation` (`/toolkit/components/filepicker/content/filepicker.js`)](https://dxr.mozilla.org/mozilla-central/source/toolkit/components/filepicker/content/filepicker.js) - a Controller: binds handlers for click and other DOM events; updates the View (in this case, a dumb XUL template) with results; all the data seems to live in the `nsFilePicker.js` model.

- `toolkit/components/satchel`
  - need to read this and figure out what it does

### Digging into Search (search bar, not urlbar/awesomebar)

- OpenSearch is an XML spec describing search provider endpoints
  - https://developer.mozilla.org/en-US/Add-ons/Creating_OpenSearch_plugins_for_Firefox
  - examples are localized, see, eg, `browser/locales/en-US/searchplugins`

- `browser/base/content/searchSuggestionUI.js`
  - creates an xhtml table, inserts into DOM after a given textbox el, styles it to look like a dropdown
  - emits `GetSuggestions` signal with packet format `{engineName, searchString, remoteTimeout}`
  - listens for `ContentSearchService` events on `window`

- `netwerk/base/nsIBrowserSearchService.idl`
  - C++ interface defining a remote service facade

- `toolkit/components/search/nsSearchService.js`
  - basically a service facade, abstracts an OpenSearch endpoint
  - instantiates an Engine, sends requests, returns responses to caller
  - maintains a sorted array of Engines
  - also includes `engineMetadataService` (modifies & saves engine attributes to disk), `engineUpdateService` (checks for engine definition updates, modifies local Engines as needed)

- `toolkit/components/search/nsSearchSuggestionController.js`
  - helper module, code comments say it was factored out of `nsSearchSuggestions` to allow multiple consumers to request & display search suggestions from a given Engine (implementation of `nsISearchEngine`)
  - caches form history results
  - since it provides caching & recent search history, designed to use one of these per textbox
  - searches & returns both recent history and remote search suggestions
  - interesting methods:
    - `SearchSuggestionController::fetch` - for each search, look it up in form history cache & fetch from remote suggestion service, if enabled
    - `SearchSuggestionController::_fetchFormHistory` - get an `nsIAutoCompleteSearch` implementation, call `startSearch` on it, handle response
    - `SearchSuggestionController::_fetchRemote` - given an engine (`nsISearchEngine`), get the URL via `engine.getSubmission`, then create & send an XHR

- `toolkit/components/search/nsSearchSuggestions.js`
  - defines `SuggestAutoComplete`, base implementation of `nsIAutoCompleteSearch`
  - I think (not sure) input from the front-end is handled by `nsSearchSuggestionController`
  - sends output to front-end by:
    - creating & returning a `FormAutoCompleteResult` containing the result data
    - sending it to the caller (who must implement `nsIAutoCompleteObserver`)
  - fetches results from search service accessed via `Services.search`
  - search results are collected/aggregated by the `SearchSuggestionController`

### Misc

- places database schema: https://wiki.mozilla.org/images/0/08/Places.sqlite.schema.pdf
