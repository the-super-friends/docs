# hacking on gecko

## links

- MDN article explaining how to work with (privileged) web content windows from browser chrome: https://developer.mozilla.org/en-US/docs/Working_with_windows_in_chrome_code

- MDN article explaining communication between privileged (browser chrome) and non-privileged (ordinary web) windows: https://developer.mozilla.org/en-US/Add-ons/Code_snippets/Interaction_between_privileged_and_non-privileged_pages

- adding search providers is ridiculously easy: https://developer.mozilla.org/en-US/Add-ons/Creating_OpenSearch_plugins_for_Firefox

- SUMO page explaining how to enable different kinds of content in urlbar autocomplete results: https://support.mozilla.org/en-US/kb/awesome-bar-find-your-bookmarks-history-and-tabs#w_how-can-i-control-what-results-the-location-bar-shows-me

- XUL docs: Content Panels, or, how to load remote web pages inside XUL chrome: https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/Tutorial/Content_Panels

- ancient mozillazine article includes links to Firefox 3-era urlbar modifier addons: http://kb.mozillazine.org/Disabling_autocomplete_%28Firefox%29

## Reading the Code / Orienteering

`/browser/base/content/browser.js`
  - assembles the modules
  - `gURLBar` is defined here on the global window object

### Views

`/base/content/browser.xul`
  - defines an `autocomplete-richtextbox` entity (i.e. an xml tag name)
    - this corresponds to `PopupAutoCompleteRichResult`
    - which is defined in `components/search/content/search.xml`
  
`/browser/base/content/urlbarBindings.xml`
  - urlbar UI handlers defined inside an XML file
  - urlbar extends `content/bindings/autocomplete.xml`, not sure if that really means
    `toolkit/content/widgets/autocomplete.xml` which seems to have similar stuff in it

`/toolkit/content/widgets/autocomplete.xml`
  - UI definitions for the autocomplete UI that drops down

### Controllers

`components/autocomplete/nsAutoCompleteController.cpp`
  - `HandleText` method contains logic around when to search & waiting for user to stop typing
  - `StartSearches` method - set a timer, call `StartSearch`
  - `StartSearch` - loop over `mSearches`, call `startSearch` on (some of) them
  - `SetInput` - assemble `mSearches` by getting pointers to search services, specifically, by getting a "contract id string" from an instance of the `nsIAutoCompleteInput` interface.

### Services

These are implementations of `nsIAutoCompleteInput` I've found in the gecko codebase:

`toolkit/components/places/nsPlacesAutoComplete.js`
  - This file is incredibly helpful. We can lean on this implementation when defining our own autocomplete implementation (though we need to investigate how other search providers integrate into autocomplete. I've found an OpenSearch specification, need to explore further)
  - Autocomplete strings are sent here, to be queried against the Places SQLite DB
  - see "Smart Getters" section for SQL queries corresponding to keyword, bookmark, "frecency", etc searches
  - `startSearch` function is very helpful reading

`toolkit/components/satchel`
  - need to read this and figure out what it does

### Digging into Search (search bar, not urlbar/awesomebar)

- OpenSearch is an XML spec describing search provider endpoints
  - https://developer.mozilla.org/en-US/Add-ons/Creating_OpenSearch_plugins_for_Firefox
  - examples are localized, see, eg, `browser/locales/en-US/searchplugins`

`browser/base/content/searchSuggestionUI.js`
  - creates an xhtml table, inserts into DOM after a given textbox el, styles it to look like a dropdown
  - emits `GetSuggestions` signal with packet format `{engineName, searchString, remoteTimeout}`
  - listens for `ContentSearchService` events on `window`

`netwerk/base/nsIBrowserSearchService.idl`
  - C++ interface defining a remote service facade

`toolkit/components/search/nsSearchService.js`
  - basically a service facade, abstracts an OpenSearch endpoint
  - instantiates an Engine, sends requests, returns responses to caller
  - maintains a sorted array of Engines
  - also includes `engineMetadataService` (modifies & saves engine attributes to disk), `engineUpdateService` (checks for engine definition updates, modifies local Engines as needed)

`toolkit/components/search/nsSearchSuggestionController.js`
  - helper module, code comments say it was factored out of `nsSearchSuggestions` to allow multiple consumers to request & display search suggestions from a given Engine (implementation of `nsISearchEngine`)
  - caches form history results
  - since it provides caching & recent search history, designed to use one of these per textbox
  - searches & returns both recent history and remote search suggestions
  - interesting methods:
    - `SearchSuggestionController::fetch` - for each search, look it up in form history cache & fetch from remote suggestion service, if enabled
    - `SearchSuggestionController::_fetchFormHistory` - get an `nsIAutoCompleteSearch` implementation, call `startSearch` on it, handle response
    - `SearchSuggestionController::_fetchRemote` - given an engine (`nsISearchEngine`), get the URL via `engine.getSubmission`, then create & send an XHR

`toolkit/components/search/nsSearchSuggestions.js`
  - defines `SuggestAutoComplete`, base implementation of `nsIAutoCompleteSearch`
  - I think (not sure) input from the front-end is handled by `nsSearchSuggestionController`
  - sends output to front-end by:
    - creating & returning a `FormAutoCompleteResult` containing the result data
    - sending it to the caller (who must implement `nsIAutoCompleteObserver`)
  - fetches results from search service accessed via `Services.search`
  - search results are collected/aggregated by the `SearchSuggestionController`

### Misc

- places database schema: https://wiki.mozilla.org/images/0/08/Places.sqlite.schema.pdf
