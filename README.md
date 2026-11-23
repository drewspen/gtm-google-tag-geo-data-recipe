# Google Tag Built-in Geolocation for GTM: Templates and Sandboxed JavaScript Only

Read the country, region and DMA flag that the Google tag already holds in the page, and use them in Google Tag Manager with **no Custom JavaScript variables and no inline JavaScript**. Everything is a GTM custom template, a built-in variable type, or a standard tag/trigger, so the solution adds nothing custom to review against a strict Content Security Policy.

This is an implementation variation on David Vallejo's excellent work: [Using Google Tag's Built-in Geolocation Data](https://www.thyngster.com/leveraging-google-tag-built-in-geolocation-data). Thank you, David, for finding and sharing a clean, free way to get location in GTM.

> \*\*Tested:\*\* Verified in GTM Preview. The getter resolves and country, region and DMA values are returned through the templates alone.

> \*\*Warning:\*\* This uses undocumented Google tag internals (`google\_tag\_data.xcd.ip\_geo\_data\_cache`). Google can change it without notice. Always handle `undefined`.

## Download

* [Download the container JSON from Google Drive](https://drive.google.com/uc?export=download&id=1drIxf3HY_4b0JRPUfysibp1C6e3qvTi4) ([view in Drive](https://drive.google.com/file/d/1drIxf3HY_4b0JRPUfysibp1C6e3qvTi4/view?usp=drivesdk))
* Or use `gtm-google-tag-geo-data-recipe.json` in this repository.

## What's in this repository

|File|Description|
|-|-|
|`gtm-google-tag-geo-data-recipe.json`|GTM container export, ready to import|
|`README.md`|This file|

## How it works

```
Google tag loads -> fills google\_tag\_data.xcd.ip\_geo\_data\_cache
Window Loaded trigger -> "Geo Data - dataLayer Push" tag
JavaScript Variable "Google Tag Geo Getter" supplies the .get function
Sandboxed template calls it, parses JSON -> index 0 / 1 / 6
dataLayer.push({ event: 'geo\_data\_ready', geo\_country, geo\_region, geo\_dma\_region })
Trigger "geo\_data\_ready" -> your tags use the Geo / DLV variables
```

Data indices, as documented in David's post:

|Index|Meaning|Example|
|-|-|-|
|`0`|ISO 3166-1 country code|`ES`, `JP`|
|`1`|Region code|`ES-MC`, `JP-13`|
|`6`|In a DMA-affected region|`true`, `false`|

## What changes from the original post

|Aspect|Original|This variation|
|-|-|-|
|Reading the data|Custom JavaScript variable reaching into `window.google\_tag\_data`|Built-in **JavaScript Variable** with name `google\_tag\_data.xcd.ip\_geo\_data\_cache.get`|
|Parsing|`JSON.parse` inside `try/catch`|Sandboxed template using `getType` and the sandbox `JSON` API|
|Variables|Three functions|One template with an output selector (country, region, DMA, JSON)|
|Inline / custom JS|Yes|None|
|Template permissions|n/a|None for the variable; `dataLayer` read/write for the tag|
|Delivery|Variables only|Adds a `geo\_data\_ready` data layer event, trigger and Data Layer variables|

David's post notes that Custom Templates can't access `window.google\_tag\_data`. Instead, GTM's own JavaScript Variable type reads the path, and the sandboxed template receives the function as a field value.

Original, for reference:

```javascript
function get\_country() {
  try {
    var geo = JSON.parse(window.google\_tag\_data.xcd.ip\_geo\_data\_cache.get());
    return geo\['0'];
  } catch (e) {
    return undefined;
  }
}
```

## Components (imported into a folder named "Google Tag Geo Data")

|Type|Name|Purpose|
|-|-|-|
|Variable (JavaScript Variable)|`Google Tag Geo Getter`|Global variable name `google\_tag\_data.xcd.ip\_geo\_data\_cache.get`|
|Variable template|`Google Tag Geo Data`|Returns country, region, DMA flag or JSON|
|Variables|`Geo - Country`, `Geo - Region`, `Geo - Is DMA Region`|Instances of the variable template|
|Tag template|`Google Tag Geo Data - dataLayer Push`|Pushes geo values to the data layer|
|Tag|`Geo Data - dataLayer Push`|Fires on Window Loaded; requires `analytics\_storage` consent|
|Triggers|`Geo Loader Window Loaded`, `geo\_data\_ready`|Starts the loader; fires after the push|
|Variables (Data Layer)|`geo\_country`, `geo\_region`, `geo\_dma\_region`|Read pushed values|

### Variable template (sandboxed JavaScript)

```javascript
const JSON = require('JSON');
const getType = require('getType');

const readGeo = () => {
  const getter = data.geoGetter;
  if (getType(getter) !== 'function') return undefined;
  const raw = getter();
  if (!raw) return undefined;
  const geo = getType(raw) === 'string' ? JSON.parse(raw) : raw;
  if (!geo) return undefined;
  const dma = geo\['6'];
  return {
    country: geo\['0'],
    region: geo\['1'],
    isDma: dma === undefined ? undefined : (dma === true || dma === 1 || dma === 'true')
  };
};

const geo = readGeo();
if (!geo) return undefined;

const out = data.output;
if (out === 'country') return geo.country;
if (out === 'region') return geo.region;
if (out === 'dma') return geo.isDma;
if (out === 'json') return JSON.stringify(geo);
return undefined;
```

### Tag template (sandboxed JavaScript)

Uses the same `readGeo` function, then:

```javascript
const createQueue = require('createQueue');
const geo = readGeo();
if (!geo) {
  data.gtmOnFailure();
  return;
}
createQueue('dataLayer')({
  event: data.eventName || 'geo\_data\_ready',
  geo\_country: geo.country,
  geo\_region: geo.region,
  geo\_dma\_region: geo.isDma
});
data.gtmOnSuccess();
```

## Import

1. In Google Tag Manager go to **Admin > Import Container**.
2. Choose `gtm-google-tag-geo-data-recipe.json`.
3. Pick an existing or new workspace.
4. Choose **Merge**. Use **Rename conflicting tags, triggers, and variables** if your container may have clashing names. Click **Confirm**.
5. Open **Preview** and load your site. Verify:

   * `Google Tag Geo Getter` is a function
   * `Geo - Country` returns a country code
   * a `geo\_data\_ready` event appears in the data layer
6. Adjust consent and the trigger to suit your site, then publish.

Account and container IDs in the file belong to the container it was exported from; GTM remaps them on import.

## Notes and troubleshooting

* **Consent:** the exported tag requires `analytics\_storage`. Remove that in the tag's Advanced Settings if you need location before consent.
* **Timing:** the data exists only once the Google tag has loaded. If Window Loaded is too early, fire from a later trigger.
* **Getter is a function but country is empty:** check that the Google tag has loaded first. If Google changes the structure, the getter may no longer return a function, in which case the template simply returns `undefined`.
* **Everything is `undefined`:** the Google tag has not populated the cache yet, or Google changed the structure.
* **Preview won't connect:** disable browser extensions and ad blockers.

## Credits

All credit for the discovery goes to [David Vallejo (thyngster)](https://www.thyngster.com/leveraging-google-tag-built-in-geolocation-data).

## License

MIT
