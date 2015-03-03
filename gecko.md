## guide to relevant parts of gecko

`/browser/base/content/browser.js`
  - assembles the modules
  - `gURLBar` is defined here on the global window object

### Views

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
