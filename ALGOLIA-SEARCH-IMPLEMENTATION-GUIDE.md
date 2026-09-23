# Implementing Algolia Search in a ShopWired Theme

This is an implementation brief for an AI coding agent (Claude Code or similar) to port
the Algolia-powered header search + `/search/products` results page into a **different**
ShopWired Twig theme. It was extracted from a working implementation built in `bsv-theme`
(themeId `147899`), itself adapted from a newer sibling theme (`karema-theme` /
`banana-cph-theme`). Read this whole document before touching any files — several steps
depend on decisions made earlier in the doc.

## What you're building

Two independent, cooperating pieces:

1. **Header search overlay** (`#header-search`) — a full-width panel that drops down from
   the top of the page, opened by a nav button. As the visitor types, it calls the Algolia
   Search API directly (via the `algoliasearch` JS client — no InstantSearch) and renders a
   responsive image/title/description grid. Desktop shows windowed numbered pagination;
   mobile hides the page numbers and appends a "Vis flere" (load more) button instead.
2. **`/search/products` results page** — the theme's existing search results route,
   re-rendered with Algolia's `instantsearch.js` + `@algolia/autocomplete-js` widgets
   instead of ShopWired's native product listing, when Algolia is configured.

Both are **entirely optional at runtime**: everything is gated behind theme settings
(App ID + API Key present). If they're blank, the theme behaves exactly as it did before —
native ShopWired search, no extra scripts loaded. This matters because you're often
shipping this to a theme that's already live; it must never regress default behavior.

## Prerequisites

- An Algolia application with a **search-only API key** (never the admin key) and an
  index already populated with product records. The templates below assume each record
  has at minimum: `title`, `url`, `images` (array of `{ url }`), `description` (HTML or
  plain text), `price` (number). If the target index uses different field names, update
  `attributesToRetrieve` and the hit templates accordingly — don't assume the schema
  matches without checking with whoever built the index.
- Read access to the target theme's files, and a way to deploy them. ShopWired themes are
  typically synced with the `shopwired-theme` CLI (npm package `shopwired-theme`,
  installed globally, invoked as `shopwired-theme`). It reads `apiKey` / `apiSecret` /
  `themeId` from a `config.json` in the theme's root (gitignored — never commit it).
  Relevant commands: `shopwired-theme download [files...]`, `upload <files...>`,
  `remove <files...>`. **Do not trust an "Uploaded" message at face value** — see the
  Deployment Gotchas section below; verify with a round-trip.

## First: figure out which "flavor" of theme you're porting into

ShopWired themes in this family come in at least two shapes. Check the target theme
before writing code:

| Check | Older theme (like `bsv-theme`) | Newer theme (like `karema-theme`) |
|---|---|---|
| CSS framework | Foundation only, custom SCSS, plain classes | Foundation + Tailwind utility classes mixed in |
| Search markup already present | A plain `<form action="/search/products">` inside a Foundation Reveal modal (`data-reveal`) and/or a separate mobile search bar | Often *already* has a `#header-search` div with `data-toggler=".hide"` and even unused `$('#header-search').on('off.zf.toggler', ...)` JS wired up but unconnected to any markup |
| `header_search` block called | Twice (once for desktop reveal, once for a mobile bar) | Once, or the block is literally named `algolia_search` |

**Grep for `#header-search` in `application.js` before writing any JS.** In both
`bsv-theme` and `karema-theme` we found the theme's *stock* `application.js` already had
an unused `$('#header-search').on('off.zf.toggler', function() { ... .focus(); })` handler
sitting dormant — a leftover from a shared theme template. If you find one, **reuse it,
don't duplicate it** — it already does the "focus the input on open" job.

If the theme uses Foundation's `Reveal` plugin (`data-reveal`, `data-open="x"`,
`data-close`) for its current search modal, you'll be **replacing that mechanism** with
Foundation's lighter `Toggler` plugin (`data-toggler=".hide"`, `data-toggle="x"`) — see
Step 4. Toggler doesn't give you a backdrop, scroll lock, or open/close events for free
the way Reveal does, so those are hand-rolled in the CSS/JS below.

## Step 1 — Theme settings

Add a new settings section (adjust the icon/insertion point to fit the target theme's
existing `settings.json` sections — anywhere with `"group": 3` alongside other
technical/utility sections is fine):

```json
{
    "title": "Algolia Search",
    "group": 3,
    "icon": "text",
    "description": "Connect Algolia to power the header autocomplete and the /search/products results page. Leave the App ID and API Key blank to fall back to the default search.",
    "resettable": true,
    "settings": [
        {
            "name": "text_algolia_app_id",
            "type": "text",
            "label": "Algolia App ID",
            "defaultValue": "",
            "livePreview": false,
            "separator": true
        },
        {
            "name": "text_algolia_api_key",
            "type": "text",
            "label": "Algolia API Key",
            "note": "Use a search-only API key, not your admin key.",
            "defaultValue": "",
            "livePreview": false,
            "separator": true
        },
        {
            "name": "text_algolia_index_name",
            "type": "text",
            "label": "Algolia Index Name",
            "defaultValue": "",
            "livePreview": false,
            "separator": true
        },
        {
            "name": "text_algolia_filters",
            "type": "text",
            "label": "Algolia Filters Logic",
            "note": "Optional filter expression applied to every search, e.g. active:true AND hasImage:true",
            "defaultValue": "",
            "livePreview": false,
            "separator": true
        }
    ]
}
```

Keep the setting **names** (`text_algolia_app_id` etc.) exactly as above — the JS below
reads them off `gts` (the theme's settings object) by these names.

If the target theme doesn't already have an `items_per_search_page` setting (check the
"Items Per Page" section), the results-page widget below falls back to `12` — fine either
way, add one if you want it configurable.

## Step 2 — `master.twig`: load the libraries, gated on config

Inside the stylesheets block, alongside the theme's other CDN stylesheets:

```twig
{% if gts.text_algolia_app_id and gts.text_algolia_api_key %}
    {{ html.stylesheet('//cdn.jsdelivr.net/npm/@algolia/autocomplete-theme-classic@1.19.2/dist/theme.min.css') }}
{% endif %}
```

Inside the scripts block, **before** the theme's own `application.js` include:

```twig
{% if gts.text_algolia_app_id and gts.text_algolia_api_key %}
    {{ html.script('//cdn.jsdelivr.net/npm/algoliasearch@4.24.0/dist/algoliasearch-lite.umd.js') }}
    {{ html.script('//cdn.jsdelivr.net/npm/instantsearch.js@4/dist/instantsearch.production.min.js') }}
    {{ html.script('//cdn.jsdelivr.net/npm/@algolia/autocomplete-js') }}
    <script>
        const { autocomplete, getAlgoliaResults } = window['@algolia/autocomplete-js'];
        const { gts } = {{ _context|json_encode|raw }};
    </script>
{% endif %}
{{ html.script(asset_url('js/application.js')) }}
```

That inline `<script>` is what makes `gts` (and `autocomplete`/`getAlgoliaResults`)
available as **globals** to `application.js`, which loads as a separate, non-module
`<script>` tag right after. This works because classic (non-module) `<script>` tags on a
page share one global lexical scope — a `const` declared in one tag is visible to code in
a later tag, in execution order. It only works if `master.twig` itself has
`{% set gts = global.theme.settings %}` at its own top level (nearly every ShopWired
master template already does, for other reasons — confirm it's there).

## Step 3 — Header search markup (`header.twig`)

This is the part most likely to need real adaptation, not copy-paste, depending on what
the theme's `header_search`/`algolia_search` block currently looks like. The target
structure, regardless of what's there now:

```twig
{% block header_search %}
    <div class="header-search algolia hide" id="header-search" data-toggler=".hide" aria-expanded="false">
        <div class="search-container">
            <button class="close-button absolute" data-toggle="header-search" aria-controls="header-search" aria-label="Close modal" type="button">
                <span aria-hidden="true" class="text-black text-4xl">&times;</span>
            </button>
            <div class="input-container">
                <form class="input-group mod-search" action="/search/products" method="get">
                    <input type="search" name="keywords" placeholder="Søg" id="header-search-input" class="header-search-input input-group-field js-algolia-search-input" autocomplete="off">
                    <button type="submit" class="search-icon">
                        {{ theme.sw_icon('glass-2') }} {# swap for whatever this theme's search icon macro call is #}
                        <span class="show-for-sr">Søg</span>
                    </button>
                </form>
            </div>
            {% if gts.text_algolia_app_id and gts.text_algolia_api_key %}
                <div class="algolia-pagination"></div>
                <div id="algolia-search-results" class="js-algolia-search-results" data-equalizer="algolia-results-equalizer" data-equalize-by-row="true"></div>
                <div id="no-results" class="js-algolia-no-results"></div>
            {% endif %}
        </div>
    </div>
{% endblock %}
```

Notes:

- `name="keywords"` on the input (not the `{{gts.text_algolia_index_name}}[query]`
  bracket-notation you may see in some sibling themes) — this keeps the form a working
  **native fallback**: if JS fails, or the visitor presses Enter before the Algolia client
  has loaded, it still submits to `/search/products?keywords=...` and the theme's normal
  search runs. Don't disable the native form submit entirely (some reference
  implementations do this with a `keydown` handler on Enter and a `submit` preventDefault
  — that removes the graceful-degradation path; there's no reason to trade that away here).
- The `js-algolia-search-input` class (in addition to the `id`) matters — the
  results-page script listens for `change` events on it to keep an already-open results
  page's URL in sync if the visitor edits the header box while on that page.
- Only render the three results containers when Algolia is actually configured
  (`{% if gts.text_algolia_app_id and gts.text_algolia_api_key %}`) — otherwise you're
  shipping empty dead divs to every visitor of a shop that hasn't set this up.

Then, **remove** whatever previously wrapped the search form (a Foundation `Reveal`
`data-reveal` div, a separate mobile-only `#mobileSearch` div, etc.) and call the block
exactly once, near the top of `<header>`:

```twig
{{ block('header_search') }}
```

Point **every** trigger button (desktop nav "Søg" button, mobile search icon button) at
the same target:

```twig
<button class="header-search-toggle" data-toggle="header-search">...</button>
```

If the theme's mobile search button previously had its own `data-toggler=".active"` /
companion class behavior, drop that — it's now just `data-toggle="header-search"`, same
as the desktop button, since there's only one overlay.

## Step 4 — `items.twig`, `search.twig`, `templates/collection.twig`

These three cooperate to swap the results-page rendering from native ShopWired listing to
the Algolia `#hits`/`#pagination` widgets, **only** on `/search/products` and **only**
when Algolia is configured. `category.twig` and any other template that also extends
`collection.twig` is unaffected (the `algoliaSearch` variable is simply undefined/false
there, which Twig treats as falsy).

**`views/partials/items.twig`** — add a branch:

```twig
{% if type == 'algolia' %}
    <div class="row column algolia-search-page" data-initial-query="{{ initial_query|default('') }}">
        <div id="autocomplete-input" class="algolia-autocomplete-input"></div>
        <div id="hits" class="products items-container row large-up-4"></div>
        <div id="pagination"></div>
    </div>
{% else %}
    {# ...existing branch unchanged... #}
{% endif %}
```

**`views/search.twig`**:

```twig
{% extends 'templates/collection.twig' %}

{# IMPORTANT: re-declare gts here. See "Twig gotcha" below. #}
{% set gts = global.theme.settings %}
{% set algoliaSearch = gts.text_algolia_app_id and gts.text_algolia_api_key ? true : false %}

{# ...existing title/filter_product_count/etc. setup... #}

{% block items %}
    {% if algoliaSearch %}
        {% include 'partials/items.twig' with { 'item_type': 'algolia', 'initial_query': keywords|default('') } %}
    {% else %}
        {# ...existing native include, unchanged... #}
    {% endif %}
{% endblock %}
```

**`views/templates/collection.twig`** — wrap the native sort/filter UI and pagination
includes so they don't render (and conflict visually) when Algolia is driving the page:

```twig
{% if not algoliaSearch %}
    {# ...existing sort-form / filter row / top paginator... #}
{% endif %}

{# ...items block call, unchanged... #}

{% if not algoliaSearch %}
    {# ...existing bottom paginator include... #}
{% endif %}
```

### Twig gotcha you will hit if you skip this

`{% set gts = global.theme.settings %}` normally lives at the top of `master.twig`. When
Twig renders a chain of `{% extends %}` (`search.twig` → `collection.twig` → `page.twig`
→ `master.twig`), **each template's own top-level code runs, leaf-first**, before the base
template starts producing output. So a top-level `{% set algoliaSearch = gts.xxx %}` in
`search.twig` runs *before* `master.twig`'s `{% set gts = ... %}` has executed — `gts`
would silently be `null` there and `algoliaSearch` would always be `false`. The fix, shown
above, is to redeclare `{% set gts = global.theme.settings %}` locally at the top of
`search.twig` too (it's a real global object either way, so this is safe and just
redundant once `master.twig` also sets it later in the same render). This bit us once
already — don't skip it.

## Step 5 — `application.js`

Two call sites in the theme's `$(function() { ... })` ready block, near wherever other
`init*()` calls live:

```js
algoliaHeaderSearch();
algoliaSearchPage();
```

And these functions, appended anywhere at file scope (adjust `algoliaFormatPrice`'s
locale/currency if the shop isn't Danish/DKK):

```js
function algoliaCreditsPresent() {
    return typeof algoliasearch !== 'undefined' && typeof gts !== 'undefined' && gts.text_algolia_app_id && gts.text_algolia_api_key;
}

function algoliaFormatPrice(price) {
    return new Intl.NumberFormat('da-DK', {
        style: 'currency',
        currency: 'DKK'
    }).format(price || 0).replace('kr.', 'DKK');
}

// Used by the /search/products results page (has price + basket icon)
function algoliaHitTemplate(item) {
    var image = (item.images && item.images[0] && item.images[0].url) || '/fallback.jpg';
    var description = (item.description || '').replace(/<[^>]+>/g, '').slice(0, 150);

    return '' +
        '<article class="card item-box product-box">' +
            '<div class="item-image">' +
                '<a href="' + item.url + '" class="image-container" data-fit="1">' +
                    '<img src="' + image + '" alt="' + item.title + '" class="item-img">' +
                '</a>' +
            '</div>' +
            '<div class="card-section box-data product-box-data">' +
                '<h2 class="product-box-title">' +
                    '<a href="' + item.url + '">' + item.title + '</a>' +
                '</h2>' +
                '<div class="item-description">' + description + '</div>' +
                '<div class="product-box-basket">' +
                    '<a href="' + item.url + '" class="quick-view">' +
                        '<i class="sw-icon-basket" aria-hidden="true"></i>' +
                        '<span class="show-for-sr">View product</span>' +
                    '</a>' +
                    '<span class="price">' + algoliaFormatPrice(item.price) + '</span>' +
                '</div>' +
            '</div>' +
        '</article>';
}

// Used by the header overlay (image + title + description only, no price/basket)
function algoliaHeaderHitTemplate(item) {
    var image = (item.images && item.images[0] && item.images[0].url) || '/fallback.jpg';
    var description = (item.description || '').replace(/<[^>]+>/g, '').slice(0, 150);

    return '' +
        '<article class="item-box search-item-box">' +
            '<div class="item-image">' +
                '<a href="' + item.url + '" class="image-container">' +
                    '<img src="' + image + '" alt="' + item.title + '" class="item-img">' +
                '</a>' +
            '</div>' +
            '<h3 class="item-title">' +
                '<a href="' + item.url + '">' + item.title + '</a>' +
            '</h3>' +
            '<div class="item-description">' + description + '</div>' +
        '</article>';
}

// Header overlay: plain algoliasearch client + hand-rolled pagination.
// Desktop = windowed numbered pages. Mobile = "Vis flere" append-in-place button.
function algoliaHeaderSearch() {
    if (!algoliaCreditsPresent()) {
        return;
    }

    var $input = $('#header-search-input');
    var $results = $('#algolia-search-results');
    var $noResults = $('#no-results');
    var $pagination = $('.algolia-pagination');
    var debounceTimeout;
    var currentQuery = '';
    var currentPage = 0;
    var totalPages = 0;
    var HITS_PER_PAGE = 6;
    var MIN_CHARS = 2;
    var MAX_PAGE_BUTTONS = 5;
    var mobileMq = window.matchMedia('(max-width: 1023px)'); // match the CSS grid's mobile breakpoint
    var isMobile = function() {
        return mobileMq.matches;
    };

    if (!$input.length || !$results.length) {
        return;
    }

    var searchClient = algoliasearch(gts.text_algolia_app_id, gts.text_algolia_api_key);
    var index = searchClient.initIndex(gts.text_algolia_index_name);

    if (mobileMq.addEventListener) {
        mobileMq.addEventListener('change', function() {
            renderPagination();
        });
    }

    $input.on('keydown', function(e) {
        if (e.key === 'Escape') {
            $(this).val('');
            clearResults();
        }
    });

    $input.on('input', function() {
        var query = $(this).val().trim();
        clearTimeout(debounceTimeout);

        currentQuery = query;
        currentPage = 0;
        totalPages = 0;

        if (!query || query.length < MIN_CHARS) {
            clearResults();
            return;
        }

        debounceTimeout = setTimeout(function() {
            performSearch(0, false);
        }, 150);
    });

    $pagination.on('click', 'button[data-page]', function(e) {
        e.preventDefault();

        if (isMobile() || $(this).is('[disabled]')) {
            return;
        }

        var page = parseInt($(this).attr('data-page'), 10);
        if (isNaN(page) || page === currentPage) {
            return;
        }

        performSearch(page, false);
    });

    $results.on('click', '[data-load-more]', function(e) {
        e.preventDefault();

        if (!isMobile() || !currentQuery || currentPage >= totalPages - 1) {
            return;
        }

        performSearch(currentPage + 1, true);
    });

    function clearResults() {
        $results.removeClass('active').empty();
        $noResults.removeClass('active').empty();
        $pagination.empty();
    }

    function performSearch(page, append) {
        index.search(currentQuery, {
            page: page,
            hitsPerPage: HITS_PER_PAGE,
            attributesToRetrieve: ['title', 'url', 'images', 'description', 'price'],
            filters: gts.text_algolia_filters || ''
        }).then(function(res) {
            currentPage = page;
            totalPages = res.nbPages || 0;
            renderHeaderResults(res.hits || [], append);
        }).catch(function(err) {
            console.error('Algolia search error:', err);
        });
    }

    function renderHeaderResults(hits, append) {
        if (append) {
            removeLoadMoreRow();
        } else {
            $results.empty();
        }

        if (!hits.length) {
            if (!append) {
                $results.removeClass('active');
                $noResults.addClass('active').html('<p>Ingen resultater fundet</p>');
            }
            renderPagination();
            $(window).trigger('resize'); // nudges Foundation's Equalizer plugin to recalc
            return;
        }

        $noResults.removeClass('active').empty();
        $results.addClass('active');
        hits.forEach(function(item) {
            $results.append(algoliaHeaderHitTemplate(item));
        });

        renderPagination();
        $(window).trigger('resize');
    }

    function removeLoadMoreRow() {
        $results.find('.algolia-load-more-row').remove();
    }

    function renderLoadMoreRow() {
        removeLoadMoreRow();

        if (currentPage >= totalPages - 1) {
            return;
        }

        $results.append('<div class="algolia-load-more-row"><button type="button" class="algolia-load-more" data-load-more="1">Vis flere</button></div>');
    }

    function renderPagination() {
        if (!totalPages || totalPages <= 1) {
            $pagination.empty();
            removeLoadMoreRow();
            return;
        }

        if (isMobile()) {
            $pagination.empty(); // hide numbered pagination on mobile
            renderLoadMoreRow();
            return;
        }

        removeLoadMoreRow();

        var maxButtons = MAX_PAGE_BUTTONS;
        var start = Math.max(0, currentPage - Math.floor(maxButtons / 2));
        var end = start + maxButtons - 1;

        if (end > totalPages - 1) {
            end = totalPages - 1;
            start = Math.max(0, end - (maxButtons - 1));
        }

        var html = '<button type="button" class="algolia-page-btn" data-page="' + (currentPage - 1) + '"' + (currentPage === 0 ? ' disabled' : '') + '>&lsaquo;</button>';

        if (start > 0) {
            html += pageButton(0);
            if (start > 1) {
                html += '<span class="algolia-ellipsis">&hellip;</span>';
            }
        }

        for (var i = start; i <= end; i++) {
            html += pageButton(i);
        }

        if (end < totalPages - 1) {
            if (end < totalPages - 2) {
                html += '<span class="algolia-ellipsis">&hellip;</span>';
            }
            html += pageButton(totalPages - 1);
        }

        html += '<button type="button" class="algolia-page-btn" data-page="' + (currentPage + 1) + '"' + (currentPage === totalPages - 1 ? ' disabled' : '') + '>&rsaquo;</button>';

        $pagination.html(html);
    }

    function pageButton(page) {
        return '<button type="button" class="algolia-page-btn' + (page === currentPage ? ' current' : '') + '" data-page="' + page + '">' + (page + 1) + '</button>';
    }
}

// /search/products results page: InstantSearch.js hits/pagination widgets + an
// autocomplete-js query box that stays in sync with the URL via InstantSearch's history router.
function algoliaSearchPage() {
    if (!algoliaCreditsPresent() || $('#hits').length === 0) {
        return;
    }

    var indexName = gts.text_algolia_index_name;
    var searchClient = algoliasearch(gts.text_algolia_app_id, gts.text_algolia_api_key);
    var router = instantsearch.routers.history();
    var filters = gts.text_algolia_filters || '';
    var initialQuery = $('.algolia-search-page').data('initial-query') || '';

    var search = instantsearch({
        indexName: indexName,
        searchClient: searchClient,
        routing: { router: router }
    });

    var virtualSearchBox = instantsearch.connectors.connectSearchBox(function() {});

    search.addWidgets([
        virtualSearchBox({}),
        instantsearch.widgets.hits({
            container: '#hits',
            templates: {
                item: function(hit) {
                    return algoliaHitTemplate(hit);
                }
            }
        }),
        instantsearch.widgets.pagination({ container: '#pagination' }),
        instantsearch.widgets.configure({
            filters: filters,
            hitsPerPage: parseInt(gts.items_per_search_page, 10) || 12
        })
    ]);

    search.start();

    function setInstantSearchUiState(params) {
        search.setUiState(function(uiState) {
            var currentState = uiState[indexName] || {};
            return Object.assign({}, uiState, {
                [indexName]: Object.assign({}, currentState, { page: 1 }, params)
            });
        });
    }

    // Seed the initial query from the classic ShopWired ?keywords= search when arriving fresh
    var urlState = router.read();
    var routerQuery = (urlState && urlState[indexName] && urlState[indexName].query) || '';
    var startingQuery = routerQuery || initialQuery;
    if (!routerQuery && initialQuery) {
        setInstantSearchUiState({ query: initialQuery });
    }

    if (typeof autocomplete === 'undefined' || $('#autocomplete-input').length === 0) {
        return;
    }

    var skipRouterUpdate = false;
    var autocompleteInstance = autocomplete({
        container: '#autocomplete-input',
        placeholder: 'Søg efter produkter',
        initialState: { query: startingQuery },
        onSubmit: function(params) {
            setInstantSearchUiState({ query: params.state.query });
        },
        onReset: function() {
            setInstantSearchUiState({ query: '' });
        },
        onStateChange: function(params) {
            if (!skipRouterUpdate && params.prevState.query !== params.state.query) {
                setInstantSearchUiState({ query: params.state.query });
            }
            skipRouterUpdate = false;
        },
        getSources: function(params) {
            var query = params.query;
            if (!query) {
                return [];
            }
            return [{
                sourceId: 'instant_search',
                getItems: function() {
                    return getAlgoliaResults({
                        searchClient: searchClient,
                        queries: [{ indexName: indexName, query: query, params: { hitsPerPage: 5, filters: filters } }]
                    });
                },
                templates: {
                    item: function(params) {
                        return params.html`<div>${params.components.Highlight({ attribute: 'title', hit: params.item })}</div>`;
                    }
                },
                onSelect: function(params) {
                    params.event.preventDefault();
                    params.setQuery(params.item.title);
                    setInstantSearchUiState({ query: params.item.title });
                    params.setIsOpen(false);
                }
            }];
        }
    });

    window.addEventListener('popstate', function() {
        skipRouterUpdate = true;
        var currentQuery = search.helper ? search.helper.state.query : '';
        autocompleteInstance.setQuery(currentQuery);
    });

    // Keep the header search box in sync with InstantSearch once the results page is active
    $('.js-algolia-search-input').on('change', function() {
        setInstantSearchUiState({ query: $(this).val().trim() });
    });
}
```

Also, in the ready block, wire the overlay's open/close behavior onto whatever handler
already exists for `#header-search` (reuse it if the theme has one — see the "figure out
which flavor" section above):

```js
$('#header-search').on('off.zf.toggler', function() {
    // "off.zf.toggler" fires when Foundation's Toggler plugin REMOVES the toggle class
    // (.hide here) — i.e. the panel just became visible. Confusing name, but that's Foundation.
    $(this).find('.header-search-input').focus();
    $('body').addClass('search-open');
}).on('on.zf.toggler', function() {
    $('body').removeClass('search-open');
    // reset search state on close so it opens fresh next time
    $(this).find('.header-search-input').val('');
    $('#algolia-search-results').removeClass('active').empty();
    $('#no-results').removeClass('active').empty();
    $('.algolia-pagination').empty();
});
```

If you removed a Foundation `Reveal`-specific handler (e.g. something listening for
`open.zf.reveal` on `.header-search`) because you dropped the Reveal wrapper in Step 3,
delete it — it's now dead code that will never fire.

## Step 6 — SCSS (new `_algolia.scss` partial + one import line)

Create `assets/scss/_algolia.scss`:

```scss
// Algolia Search — header search overlay (#header-search)
body.search-open {
    overflow: hidden;

    &:before {
        content: '';
        position: fixed;
        inset: 0;
        z-index: 998;
        background: rgba(0, 0, 0, 0.8);
    }
}

#header-search {
    display: none;
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    z-index: 999;
    background: #fff;
    max-height: 90vh;
    overflow-y: auto;
    box-shadow: 0 12px 30px rgba(0, 0, 0, 0.18);

    &:not(.hide) {
        display: block;
    }
}

@media (max-width: 1023px) {
    #header-search {
        height: 100vh;
        max-height: 100vh;
    }
}

#header-search .search-container {
    position: relative;
    max-width: 1400px;
    margin: 0 auto;
    padding: 40px 1.5rem 3rem;
}

#header-search .close-button {
    position: absolute;
    top: 0px;
    right: 1rem;
    background: transparent;
    color: #000;
    font-size: 2em !important;
    line-height: 1;
    width: auto;
    height: auto;
    opacity: 0.5;
    transition: opacity 150ms ease-in-out;

    &:hover {
        opacity: 1;
    }
}

#header-search .input-container {
    max-width: 720px;
    margin: 0 auto 1.5rem;
}

#header-search .input-group.mod-search {
    display: flex;
    align-items: center;
    border: 2px solid #333; // swap for the theme's brand border color
    border-radius: 40px;
    overflow: hidden;
    margin: 0 auto;
    margin-bottom: 0;
}

#header-search .header-search-input {
    border: none;
    box-shadow: none;
    flex: 1;
    font-size: 16px !important;
    height: 48px !important;
    padding: 0 1.5rem;
    font-weight: normal;
}

#header-search .search-icon {
    border: none;
    background: transparent;
    padding: 0 1.5rem;
    display: flex;
    align-items: center;
    cursor: pointer;

    i {
        font-size: 1.5rem;
    }
}

// Results grid
#algolia-search-results {
    display: none;
    max-width: 1400px;
    margin: 0 auto;
    grid-template-columns: repeat(2, 1fr);
    gap: 2.5rem 1.5rem;

    &.active {
        display: grid;
    }
}

@media (min-width: 640px) {
    #algolia-search-results.active {
        grid-template-columns: repeat(3, 1fr);
    }
}

@media (min-width: 1024px) {
    #algolia-search-results.active {
        grid-template-columns: repeat(6, 1fr); // keep in sync with HITS_PER_PAGE in the JS
        gap: 2.5rem 1.5rem;
    }
}

.algolia-pagination {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 0.35rem;
    max-width: 1400px;
    margin: 0 auto 1.5rem;
    min-height: 1.75rem;

    .algolia-page-btn {
        appearance: none;
        background: transparent;
        border: 1px solid #ddd;
        border-radius: 4px;
        color: #111;
        cursor: pointer;
        font-size: 12px;
        line-height: 1;
        padding: 6px 8px;
        transition: all .15s ease-in-out;

        &:hover:not([disabled]) {
            border-color: #000;
        }

        &.current {
            background: #000;
            color: #fff;
            border-color: #000;
        }

        &[disabled] {
            opacity: 0.4;
            cursor: default;
        }
    }

    .algolia-ellipsis {
        padding: 0 0.25rem;
        color: #888;
        user-select: none;
    }
}

.algolia-load-more-row {
    grid-column: 1 / -1;
    text-align: center;
    margin: 1rem 0 1.5rem;
}

.algolia-load-more {
    appearance: none;
    display: inline-block;
    color: #fff;
    background-color: #fec60e; // swap for the theme's brand accent color
    border: 1px solid #fec60e;
    padding: 0.65rem 2rem;
    font-size: 0.9rem;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    border-radius: 2px;
    cursor: pointer;
    transition: all 150ms ease-in-out;

    &:hover {
        background-color: #000;
        border-color: #000;
    }
}

.search-item-box {
    text-align: left;

    .item-image {
        margin-bottom: 1rem;

        .item-img {
            width: 100%;
            aspect-ratio: 1 / 1;
            object-fit: cover;
            display: block;
        }
    }

    .item-title {
        text-transform: uppercase;
        font-size: 0.95rem;
        font-weight: 700;
        margin-bottom: 0.35rem;
        line-height: 1.3;
        display: -webkit-box;
        -webkit-box-orient: vertical;
        -webkit-line-clamp: 2;
        overflow: hidden;
        text-overflow: ellipsis;

        a {
            color: inherit;

            &:hover {
                color: #fec60e; // swap for the theme's brand accent color
            }
        }
    }

    .item-description {
        font-size: 0.85rem;
        color: #666;
        line-height: 1.5;
        display: -webkit-box;
        -webkit-box-orient: vertical;
        -webkit-line-clamp: 2;
        overflow: hidden;
        text-overflow: ellipsis;
    }
}

#no-results {
    display: none;
    max-width: 1400px;
    margin: 0 auto;
    padding: 2rem 0;
    text-align: center;

    &.active {
        display: block;
    }
}

// /search/products results page (autocomplete-js + instantsearch.js)
.algolia-autocomplete-input {
    max-width: 640px;
    margin: 0 auto 2rem;

    .aa-Form {
        border-radius: 30px;
    }

    .aa-Form:focus-within {
        border-color: #fec60e; // swap for the theme's brand accent color
        box-shadow: none;
    }
}

.aa-Input {
    margin: 0;
}

.ais-Hits-list,
#hits.products {
    margin-left: 0;
}

.ais-Pagination-list {
    justify-content: center;
    display: flex;
    list-style: none;
    margin: 2rem 0;
    padding: 0;
}

.ais-Pagination-item {
    margin: 0 0.25rem;
}

.ais-Pagination-item--selected .ais-Pagination-link {
    background: #fec60e; // swap for the theme's brand accent color
    color: #fff;
    border-color: #fec60e;
}

.ais-Pagination-link {
    display: block;
    padding: 0.5rem 0.875rem;
    border: 1px solid #ddd;
}
```

Then add one import line to the theme's `theme.scss` (or wherever the other partials are
imported), near the end, before any final "customisations/overrides" partial so those can
still override this if needed:

```scss
@import '_algolia';
```

Every color in this file is BSV's brand accent (`#fec60e`) — **grep the target theme's own
SCSS for its accent color variable/hex and swap it in**, don't ship BSV's yellow into
someone else's theme.

## Deployment: `shopwired-theme` CLI gotchas learned the hard way

These cost real time in the original build — don't repeat them:

1. **An "Uploaded X" message does not prove the upload reached the theme you think it
   did.** If there's any doubt about which `config.json` / API key / theme ID is active
   (e.g. you just changed credentials, or inherited someone else's `config.json`), do a
   throwaway round-trip before trusting anything:
   ```
   cp assets/scss/_algolia.scss /tmp/verify.scss
   shopwired-theme upload assets/scss/_algolia.scss
   shopwired-theme download assets/scss/_algolia.scss
   diff /tmp/verify.scss assets/scss/_algolia.scss && echo VERIFIED
   ```
   If the diff isn't clean, or the download silently returns old content, the credentials
   are pointed at the wrong theme (or are stale/invalid) — stop and confirm before doing
   any more work.
2. `config.json` (`apiKey`, `apiSecret`, `themeId`) must be `.gitignore`d — it's a live
   credential file, not theme content.
3. Occasional single-file `upload`/`download` calls fail with `ECONNRESET` /
   "socket hang up" — this is transient network flakiness in the CLI's HTTP layer, not a
   real error. Just retry the same command for the specific file(s) that failed.
4. `upload`/`download`/`remove` take literal relative file paths as arguments — no globs,
   no directories. If a file is deleted locally, `upload` can't remove it remotely; use
   `shopwired-theme remove <path>` for that.
5. Validate before uploading, every time — a broken deploy to a live theme is worse than a
   slow one:
   ```
   node --check assets/js/application.js
   npx --yes sass --no-source-map assets/scss/_algolia.scss /tmp/out.css
   ```

## QA checklist before calling it done

- [ ] With Algolia settings **blank**: header search button still opens *something*
      reasonable (native fallback), `/search/products` still shows native results, no
      Algolia scripts appear in page source, no console errors.
- [ ] With Algolia settings **filled in**: typing 2+ characters in the header box shows a
      results grid within ~150ms of the last keystroke; typing 1 character shows nothing
      (respects `MIN_CHARS`).
- [ ] Desktop width (≥1024px): numbered pagination appears when results exceed
      `HITS_PER_PAGE` (6); clicking a page number swaps results without a page reload;
      first/last page buttons are correctly disabled.
- [ ] Mobile width (<1024px): no numbered pagination is visible; a "Vis flere" button
      appears at the end of the results grid when more pages exist; clicking it **appends**
      results below the existing ones and the button moves to the new bottom; it disappears
      entirely once the last page is reached.
- [ ] Resizing the browser across the 1024px breakpoint while results are showing
      re-renders the correct pagination UI for the new width (the `matchMedia` listener).
- [ ] Pressing Escape while the input is focused clears the query and the results.
- [ ] Closing the overlay (X button, or reopening) resets the input and results — it
      doesn't show stale results from the last search.
- [ ] `/search/products?keywords=foo` (a direct link, e.g. from the native fallback form)
      lands with that query pre-populated in both the on-page autocomplete input and the
      results grid.
- [ ] No JS console errors on any page when Algolia is configured, especially check pages
      that don't have `#header-search` in the DOM at all (shouldn't matter, but confirm the
      early `if (!$input.length...) return;` guards actually prevent errors).

## Known simplifications (intentional, don't "fix" these without being asked)

- `HITS_PER_PAGE` is a fixed `6` for the header overlay, matching the desktop 6-column
  grid — it does not adapt live to the responsive column count, so at the 3-column tablet
  breakpoint a "row" is visually 2 rows of 3. This was an accepted tradeoff, not a bug.
- The header overlay's Algolia query has no faceting/filtering UI — `text_algolia_filters`
  is a single static filter string set once in theme settings, applied to every search.
- `algoliaFormatPrice` hardcodes `da-DK` / `DKK` — change this if the target shop isn't
  Danish.
- Only the top `MAX_PAGE_BUTTONS` (5) page numbers are shown around the current page, with
  `…` ellipses and always-visible first/last page buttons — not every page is enumerated
  for large result sets.
