# sdk-web

This repository contains examples and tutorials on how to use data from the Danish Roaddirectorate in your web application or on your website.

Available data
--------------

The following datatypes are available at the moment:

Datatype | Key | Description      
------- | ---------------- | ----------
Trafficevents|traffic|Map and lists
Roadworks|roadworks|Map and lists
Winter Deicing|winterdeicing|Map
Winter Condition|wintercondition|Map



How to get started 
------------------

In order to get started an API key is needed, the API key can be
obtained here: [https://nap.vd.dk/themes/1](https://nap.vd.dk/themes/1)

Using the SDK 
------------------------

This section contains a list of examples and use cases

### Make a list on your own website with roadworks in your area 

(see also the included example-list.html)


    const baseUrl = 'https://data.vd-nap.dk';
    // Add your NAP API key below.
    const apiKey = '[YOUR-API-KEY]';
    // And populate the array below with the datatypes you wish to load.
    // Supported types for the Traffic SDK: traffic, roadworks, wintercondition, winterdeicing
    const types = ['traffic'];
    const client = new marco.Client(new marco.EventSourceBackend());

    // The query parameters for our endpoints
    const search = 'api_key=' + apiKey + '&types=' + types.join(',')

    const snapshotPromise = fetch(baseUrl + '/api/v2/list/snapshot?' + search).then((response) => response.json())
    const session = client.listen(baseUrl + '/api/v2/list/updates?' + search, {initialValueSupplier: rxjs.from(snapshotPromise)});
    session.subscribe((data) => {
        const list = document.querySelector('#list');
        for (var element of data) {
            list.appendChild(buildListItem(element.heading, element.description));
        }
    });

    // Just a simple function to build our list items.
    function buildListItem(headerContent, bodyContent) {
        var item = document.createElement('li');
        var header = document.createElement('h5');
        header.innerHTML = headerContent;
        header.style.marginBottom = '0';
        var body = document.createElement('p');
        body.innerHTML = bodyContent;
        body.style.marginTop = '0';
        item.appendChild(header);
        item.appendChild(body);
        return item;
    }


### Make a map with different types of traffic data 
(see also the included example-map.html)

	  var baseUrl = 'https://data.vd-nap.dk';
    // Insert your NAP API key in this field
    const apiKey = '[YOUR-API-KEY]';
    // And populate the array below with the datatypes you wish to load.
    // Supported types for the Traffic SDK: traffic, roadworks, wintercondition, winterdeicing
    var types = ['traffic'];

    // The query parameters for our endpoints
    const search = 'api_key=' + apiKey + '&types=' + types.join(',')

    // Google Maps Callback
    function initMap() {
        var map = new google.maps.Map(document.getElementById('map'), {
            center: {lat: 55.906320, lng: 10.994602},
            zoom: 7
        });
        // We'll use an event-source backend here to use SSE based retrieval.
        const client = new marco.Client(new marco.EventSourceBackend());

        // Make sure to use the Google Maps snapshot manager. This greatly improves performance on the map loading.
        const manager = new GMapsDataLayerSnapshotManager(map, {
            keySelector: (val) => val.properties.tag,
            dataSelector: (it) => it['entity']
        });

        fetch(baseUrl + '/api/v2/map/snapshot?renderer=geojson&api_key=' + search)
            .then((response) => response.json())
            .then((data) => {
                manager.setInitialValue(data);
                const session = client.listen(baseUrl + '/api/v2/map/updates?renderer=geojson&' + search, {manager: manager});
                session.subscribe();
            });
    }


    // This is a custom snapshot manager for RTClient. It stores the current state directly in a Google Maps layer.
    class GMapsDataLayerSnapshotManager extends marco.SnapshotManager {
        constructor(map, options) {
            super(options);
            this.map = map;
            this.layer = new google.maps.Data({map: map});
        }

        setInitialValue(value) {
            this.clear();
            this.addValues(value);
        }

        onConnect(observer) {
            return new rxjs.Observable(() => {
                // Intentionally empty
            }).subscribe(observer);
        }

        clear() {
            this.layer.setMap(null);
            this.layer = new google.maps.Data({map: this.map});
        }

        addValues(values) {
            const collection = {type: 'FeatureCollection', features: values};
            this.layer.addGeoJson(collection, {});
        }

        removeKeys(keys) {
            this.layer.forEach((it) => {
                if (keys.some((k) => k === it.getProperty('tag'))) {
                    this.data.remove(it);
                }
            });
        }
    }

### Use traffic data in your own server 

This section is under development.

How to make your own traffic information map
-----------------------------------

This section is under development.




Detailed description of the datamodel and api's
=

Data Model 
==========

The structure of objects from the different APIs are presented below. We
assume the basic types of `String`, `Float`, `Int` - wheres a
question-mark suffix is used to denote a union of the type and `null`
value (`Int? = Int | null`). A literal value can be used for a type that
always has a known value (for tagged unions), e.g. `foo: “Bar”` denotes
a field `foo` that always has the string literal value of `Bar`.

Mapped key-value pair data is denoted with a `Map<Key,Value>` type, and
array types are denoted with a `[]` suffix (e.g. `Int[]`). Arrays of
known length or content are defined as array literals, e.g. `[Int, Int]`
denotes an array with two elements of types `Int`.

Union types or values are denoted with `|`, and will for the most part
refer to tagged unions or string enumerations. A wildcard type `*` is
used to denote the union of all types (i.e. any type).

Derived types are denoted as by either a series of fields or a union of
other types/values.

![](images/icons/grey_arrow_down.png)EBNF grammar for the type
definitions

``` 
<identifier> ::= [A-Za-z][0-9A-Za-z]*
<types> ::= <type> '|' <types> | <type>
<type> ::= ('String' | 'Int' | 'Float' | 'Map' <identifier> | <array_literal>) ['?'|<array_modifier>] [<type_param>)
<type_param> ::= '<' (<types> | <type_list>) '>'
<array_literal> ::= '[' <type_list> ']'
<type_list> ::= <type> ',' <type_list> | <type>
<array_modifier> ::= '[]' | '[]' <array_modifier>
<array_modifier> ::= 
<type_definition> ::= 'type' <identifier> (<type_literal>|<type_body>)
<type_literal> ::= '=' <types>
<type_body> ::= '{' <field_set> '}'
<field_set> ::= <field> <field_set> | <field>
<field> ::= <identifier> ':' <type>
```

Lastly, we’ll define a few aliases for known format strings for colors:

``` 
type Color = RGBAHex|ARGBHex|RGBHex
type RGBAHex = Regex: #([0-9A-F]{8})
type ARGBHex = Regex: #([0-9A-F]{8})
type RGBHex = Regex: #([0-9A-F]{6})
```

The values are hex strings where the ARGB format will include the alpha
value in the first byte, and the RGBA will include the alpha value in
the last byte. The color format used is largely dependent on what style
format is used on the API (configurable via the style\_format request
parameter where applicable).

LatLng 
------

``` 
type LatLng {
  lat: Float
  lng: Float
}
```

LatLngBounds 
------------

``` 
type LatLngBounds {
  northEast: LatLng
  southWest: LatLng
}
```

Style 
-----

``` 
type Style {
  id: String
  strokeColor: Color
  strokeWidth: Float
  fillColor: Color
  dashed: Boolean
  dashColor: Color
  zIndex: Int
}
```

Inline Styled Elements 
----------------------

`InlineStyledElement` is a tagged union type where the value of the
`type` field is used to denote what concrete type is to be used:

``` 
type InlineStyledElement = InlineStyledMarker | InlineStyledPolygon | InlineStyledPolyline

type InlineStyledMarker {
  type: "MARKER"
  center: LatLng
  entityType: String
  tag: String
  style: Style
  extras: Map<String, *>
}

type InlineStyledPolygon {
  type: "POLYGON"
  points: LatLng[]
  entityType: String
  tag: String
  style: Style
  extras: Map<String, *>
}

type InlineStyledPolyline {
  type: "POLYLINE"
  points: LatLng[]
  entityType: String
  tag: String
  style: Style
  extras: Map<String, *>
}
```

External Styled Element 
-----------------------

`ExternallyStyledElement` is a tagged union type where the value of the
`type` field is used to denote what concrete type is to be used. It
differs from the `InlineStyledElement` type in that the `style` field
refers to the id of the style to use from an external stylesheet.

``` 
type ExternallyStyledElement = ExternallyStyledMarker | ExternallyStyledPolygon | ExternallyStyledPolyline

type InlineStyledMarker {
  type: "MARKER"
  center: LatLng
  entityType: String
  tag: String
  style: String
  extras: Map<String, *>
}

type InlineStyledPolygon {
  type: "POLYGON"
  points: LatLng[]
  entityType: String
  tag: String
  style: String
  extras: Map<String, *>
}

type InlineStyledPolyline {
  type: "POLYLINE"
  points: LatLng[]
  entityType: String
  tag: String
  style: String
  extras: Map<String, *>
}
```

GeoJSON 
-------

The GeoJSON renderer produces GeoJSON Features with some of the
information added in the `properties` field of the standard GeoJSON
feature type. For full details of how the different `Geometry` types
work, see the [GeoJSON
specification](https://tools.ietf.org/html/rfc7946). The fields used
from GeoJSON are described below:

``` 
type Feature {
  type: "Feature"
  properties: Map<String, *> | FeatureProperties 
  geometry: Geometry
}

type FeatureProperties {
  tag: String
  entityType: String
  style: String
}

type Geometry = LineString | Polygon | Point

type Point {
  type: "Point"
  coordinates: [Float, Float]
}

type LineString {
  type: "LineString"
  coordinates: [Float, Float][]
}

type Polygon {
  type: "Polygon"
  coordinates: [Float, Float][][]
}
```

ListItem
--------

``` 
type ListItem {
   timestamp: String
   heading: String
   description: String
   tag: String
   entityType: String
   bounds: LatLngBounds?
}
```

Notification 
------------

```
type Notification {
   timestamp: String
   heading: String
   description: String
   tag: String
   entityType: String
}
```

RTEvents
========

The RTEvent delta protocol is used on SSE update endpoints. RTEvents are
used to synchronise data collections by notifying delta changes to the
client. There are four main RTEvent types: Change, Addition (New),
Deletion, and Clear. RTEvent updates use the synchronisation identifier
(`syncId`) to signal what entities are changed in a collection - i.e. a
change event for an entity with ID `x` should be treated such that all
local copies with ID `x` are replaced with the new entity.

RTEvent Data Model 
------------------

RTEvents are a tagged unions type with at `type` field. The New and
Change events include the entities to update, whereas the Deleted event
only contains the IDs to delete from the collection.

The model below only considers the fields useful to interpret incoming
events. Other fields may be present but are mostly reserved for
diagnostics.

``` 
type RTEvent<T> = RTNewEvent<T> | RTChangeEvent<T> | RTDeletedEvent | RTClearEvent

type RTNewEvent<T> {
  type: "NEW"
  new: T[]
}

type RTChangeEvent<T> {
  type: "CHANGED"
  changed: T[]
}

type RTDeletedEvent {
  type: "DELETED"
  deleted: String[]
}

type RTClearEvent {
  type: "CLEAR"
}
```

RTEvent Semantics 
-----------------

Upon receiving a `RTNewEvent`, the contents of the `new` field should be
concatenated to the local collection. When receiving an `RTDeletedEvent`
ALL local entities with an identifier in the `deleted` array should be
removed from the local collection. A `RTChangeEvent` signals that the
local copies of entities with an ID should be replaced with the incoming
entities with that ID (see warning below). A `RTChangeEvent` can, in
other words, be treated as an `RTDeleted` event followed by an
`RTNewEvent`. An RTClear event should clear the local collection in its
entirety.

The mapping from synchronisation identifiers (SyncIDs) to Entities is
not guaranteed to be one-to-one. Make sure to always delete ALL entities
with an ID when receiving a DELETED event, and always make sure to
replace ALL local copies with ALL the incoming entities on CHANGED
events - 2 incoming events can replace 3 local ones.


Map Drawing Endpoints 
=====================

The map drawing APIs expose functionality to retrieve entities in a
geo-representable form.

Retrieve a Snapshot for Map Drawing 
-----------------------------------

HTTP GET `api/v2/map/snapshot`

**Response Type**:
`(InlineStyledElement|ExternallyStyledElement|Feature)[]` (depending on
the value of the `renderer` parameter

![](images/icons/grey_arrow_down.png)Request Parameters

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
zoom  | A Google-maps style zoom level used on the map view. This controls the resolution of polygonal and polylineal type data.        | integer       | no
sw   | The south-western boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
ne   | The north-eastern boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
style\_format   | The style format to use. See the Style Format section. Only applicable when using 'inline' renderer.| 'rgb', 'rgba', 'argb', 'android', 'ios'| No
renderer   | The renderer for the server to use. See the Renderers section.| 'geojson', 'external' or 'inline'| No
api\_key | The request API key received| string| Yes


Get Map Drawing Updates 
-----------------------

SSE `api/v2/map/updates`

**SSE Body
Type:**`RTEvent<InlineStyledElement|ExternallyStyledElement|Feature>`
(depending on the value of the `renderer` parameter

![](images/icons/grey_arrow_down.png)Request Parameters

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
zoom  | A Google-maps style zoom level used on the map view. This controls the resolution of polygonal and polylineal type data.        | integer       | no
sw   | The south-western boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
ne   | The north-eastern boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
style\_format   | The style format to use. See the Style Format section. Only applicable when using 'inline' renderer.| 'rgb', 'rgba', 'argb', 'android', 'ios'| No
renderer   | The renderer for the server to use. See the Renderers section.| 'geojson', 'external' or 'inline'| No
api\_key | The request API key received| string| Yes


Get a Single Entity 
-------------------

The single entity lookup API's return the details for one entity with
the tag specified in the request parameters. If the search returns
multiple entities, the first entity is returned (the order of the types
argument specifies the priority). \
If the search returns zero results, the server responds with an HTTP-404
response.

HTTP GET `api/v2/map/entity`

**Response Type**:
`(InlineStyledElement|ExternallyStyledElement|Feature)[]` (depending on
the value of the `renderer` parameter

![](images/icons/grey_arrow_down.png)Request Parameters

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
tag  |  The tag of the entity to retrieve| string       | yes
style\_format   | The style format to use. See the Style Format section. Only applicable when using 'inline' renderer.| 'rgb', 'rgba', 'argb', 'android', 'ios'| No
renderer   | The renderer for the server to use. See the Renderers section.| 'geojson', 'external' or 'inline'| No
api\_key | The request API key received| string| Yes
zoom  | A Google-maps style zoom level used on the map view. This controls the resolution of polygonal and polylineal type data.        | integer       | no


Styling Endpoints
=================

Get a Stylesheet 
----------------

HTTP GET `api/v2/map/stylesheet`

**Response Type:**`Style[]`

![](images/icons/grey_arrow_down.png)Request Parameters
Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
style\_format   | The style format to use. See the Style Format section. Only applicable when using 'inline' renderer.| 'rgb', 'rgba', 'argb', 'android', 'ios'| No
api\_key | The request API key received| string| Yes


Style Formats 
-------------

Since different applications handle colors and files in different ways,
the 'format' parameter may be used to dictate how styles are resolved in
the response. The currently supported modes are:


Format | Description | Remarks | 
------- | ---------------- | ----------
rgb  | Resolves colors to an hex RGB string of the form \#RRGGBB | This mode is reserved for applications unable to handle alpha values  only. **Use with caution** as some elements may be transparent, and will  be drawn as being opaque with this mode!
rgba  | Resolves colors to an hex RGBA string of the form \#RRGGBBAA|
argb  | Resolves colors to an hex ARGB string of the form \#AARRGGBB|
android   | Resolves colors in the same manner as the argb mode, and transforms icon paths to the file base name only. This is useful to e.g. resolve them to Android drawable resource identifiers (i.e. /assets/some\_icon.png → some\_icon)| Should only be used on Android clients. Make sure that you have all the icons locally if using this mode.
ios  |Resolves colors in the same manner as the argb mode, and transforms icon  paths to the file base name only.| Should only be used on iOS clients. Make sure that you have all the  icons locally if using this mode.

List APIs 
=========

Get a List View Snapshot 
------------------------

HTTP GET `api/v2/list/snapshot`

**Response Type:**`ListItem[]`

![](images/icons/grey_arrow_down.png)Request Parameters

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
sw   | The south-western boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
ne   | The north-eastern boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
api\_key | The request API key received| string| Yes



Get List View Updates 
---------------------

SSE `api/v2/list/updates`

**SSE Body Type:**`RTEvent<ListItem>`

![](images/icons/grey_arrow_down.png)Request Parameters

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
sw   | The south-western boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
ne   | The north-eastern boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
api\_key | The request API key received| string| Yes



Get a Single List-Item 
----------------------

The single entity lookup API's return the details for one entity with
the tag specified in the request parameters. If the search returns
multiple entities, the first entity is returned (the order of the types
argument specifies the priority). \
If the search returns zero results, the server responds with an HTTP-404
response.

HTTP GET`api/v2/list/entity`

**Response Type:** `ListItem`

![](images/icons/grey_arrow_down.png)Request Parameters

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
tag  |  The tag of the entity to retrieve| string       | yes
api\_key | The request API key received| string| Yes



Notification Endpoints 
======================

Get Incoming Notifications 
--------------------------

SSE `api/v2/map/notifications`

**SSE Body Type:**`RTEvent<Notification>`

![](images/icons/grey_arrow_down.png)Request Parameters

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
types  | A list of types to include in the result. See the Supported Types section. | comma separated strings | yes
sw   | The south-western boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
ne   | The north-eastern boundary for the zone to retrieve data for | \\[latitude],\\[longitude], e.g. '57.3846,12.8630'      | No
api\_key | The request API key received| string| Yes

Entity APIs 
===========

The Entity APIs return a object of a Entity type

Entity Models 
-------------

``` 
type Entity {
   syncId: String;
}
```

Get entities
----------

HTTP GET `api/v2/entities`

**Response Type:**dynamic

![](images/icons/grey_arrow_down.png)Path Variables

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
type  |  See the Supported Types section. |  strings | yes
api\_key | The request API key received| string| Yes


Get entity 
----------

HTTP GET `api/realtime/snapshot/{type}/entity/{id}`

**Response Type:**dynamic

![](images/icons/grey_arrow_down.png)Path Variables

Parameter | Description | Type/Format      | Required?
------- | ---------------- | ---------- | ---------:
type  |  See the Supported Types section. |  strings | yes
id   |The entity id (tag from listview and map api's)| string | yes
api\_key | The request API key received| string| Yes


Entity Model - Traffic event and Roadworks
-------------
Field | Description | Type     
------- | ---------------- | ----------
syncId  |  Unique ID for this event |  string 
mcid  |  TrafikMan ID for this event |  string
created  |  Timestamp for the creation of this event | date
updated  |  Timestamp for the last update on this event. |  date
dominantCategory  |  Type of event |  DominantCategory
durationTimeFrom  |  Timestamp indicating when this event starts. Relevant for future events, such as scheduled roadworks, demonstrations or sport events.|  date
durationTimeTo  |  Timestamp indicating when this event ends. Relevant for future events, such as scheduled roadworks, demonstrations or sport events. Not necessarily equal to the expiration time. |  date
location  |  The name of the location of this event |  string
descriptions  |  Descriptions associated with the event |  Description
preferredMarkerLocation  |  Center coordinate |  Location
geometries  |  Geometry that describes the event | Array of Geometry
timeString  |  A time string describing when this event is relevant, e.g. during what hours in recurring periods.	| string

### DominantCategory (String Enum)

Name |  
:------- 
ROADWORK_CLOSED|
FUTURE_ROADWORK_CLOSED|
LOCAL_ROADWORK_CLOSED|
FUTURE_LOCAL_ROADWORK_CLOSED|
ROADWORK|
LOCAL_ROADWORK|
FUTURE_ROADWORK|
FUTURE_LOCAL_ROADWORK|
ROADBLOCK|
FUTURE_ROADBLOCK|
LOCAL_ROADBLOCK|
FUTURE_LOCAL_ROADBLOCK|
CONGESTION|
SLIPPERY|
WIND|
ICY|
SNOW|
PARKING|
ACTIVITY|
ALERT|
ACCIDENT|
ACCIDENT_BLOCKING|
FUTURE_ACTIVITY|
FUTURE_ALERT|
SPECIAL_EVENT|

### Description
Field | Description | Type
------- | ---------------- | ----------
heading  |  heading text for this event |  string
lead  |  lead text for this event |  string
body  |  body text for this event |  string

### Location
Field | Description | Type
------- | ---------------- | ----------
lat |  latitude |  double
lng  |  longitude |  double

### Geometry
Field | Description | Type
------- | ---------------- | ----------
type  |  Type of geometry |  POINT,POLYGON or POLYLINE
coordinates  |  Coordinates for this geometry |  Array of Location



