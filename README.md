# Cinema Finder

Cinema Finder is a React application for exploring cinemas in Australia and
New Zealand. It displays cinemas on an interactive map and lets users browse
locations by cinema franchise or by proximity.

## Requirements

- Node.js 16 or later
- npm

## Getting started

Install the dependencies and start the development server:

```bash
npm install
npm start
```

The application is then available at <http://localhost:3000>.

## Available scripts

| Command | Description |
| --- | --- |
| `npm start` | Start the development server |
| `npm test` | Run the test suite |
| `npm run build` | Create a production build |

Cinema data is stored in `src/data/auCinemas.json` and
`src/data/nzCinemas.json`.

## Datacom bug-fixing exercise

This repository was used for a Datacom software-development exercise. The
instructions were to read the project documentation, inspect and run the web
application, use browser developer tools to investigate a reproducible bug,
identify its root cause, implement a fix, and submit the updated code for
review as a ZIP archive.

The investigation focused on the cinema-list location button and the two map
libraries supported by the application: Leaflet and MapLibre. The code was
reviewed before editing, the cinema coordinate data was checked, and the
changes were validated with Git checks. The local environment used for the
exercise did not have Node.js/npm installed, so the React test suite and
production build could not be run there.

## Changes made

### 1. Correct MapLibre navigation coordinates

**File:** `src/components/Map/MaplibreMap.jsx`

**Before:**

```js
map.flyTo({
  center: [lat, lng],
  zoom: 14,
})
```

**After:**

```js
map.flyTo({
  center: [lng, lat],
  zoom: 14,
})
```

**Why this was a bug:** the shared `map.snapTo` event provides latitude and
longitude separately. Leaflet expects coordinates as `[latitude, longitude]`,
but MapLibre expects `[longitude, latitude]`. The MapLibre implementation used
the Leaflet order, so clicking a cinema's location button moved the map to the
wrong place.

**Method and reason for the change:** the event payload was traced from
`CinemaListItem.jsx` to both map implementations and compared with each
library's `flyTo` API. Only the MapLibre adapter was wrong, so the smallest and
safest fix was to swap the two values at that adapter boundary rather than
changing the shared event or the Leaflet implementation.

### 2. Display a zero-distance result

**File:** `src/components/CinemaList/CinemaListItem.jsx`

**Before:**

```jsx
{distance && <Chip label={`${format(',.1f')(distance)} km`} />}
```

**After:**

```jsx
{distance !== undefined && distance !== null && (
  <Chip label={`${format(',.1f')(distance)} km`} />
)}
```

**Why this was a bug:** JavaScript treats the number `0` as falsy. A cinema
with a calculated distance of exactly `0 km` therefore had no distance badge,
even though the value was valid.

**Method and reason for the change:** the conditional rendering was reviewed
against the distance calculation in `src/data/nearbyCinemas.js`. The intended
condition is “a distance value exists”, not “the distance is non-zero”, so the
fix explicitly excludes only `undefined` and `null` while preserving valid
zero values.

### 3. Configure snackbar auto-hide correctly

**File:** `src/components/Provider.jsx`

**Before:**

```jsx
<SnackbarProvider
  maxSnack={3}
  anchorOrigin={{ vertical: 'bottom', horizontal: 'right', timeout: 500 }}
>
```

**After:**

```jsx
<SnackbarProvider
  maxSnack={3}
  autoHideDuration={500}
  anchorOrigin={{ vertical: 'bottom', horizontal: 'right' }}
>
```

**Why this was a bug:** `timeout` is not an option of Material UI's
`anchorOrigin`. It describes the position configuration, while the display
duration must be configured with `autoHideDuration`. The previous setting
therefore did not reliably configure how long notifications remained visible.

**Method and reason for the change:** the provider configuration was compared
with the Snackbar API. The fix moves the existing intended value (`500 ms`)
to the correct property and leaves the notification position unchanged.

### 4. Remove the unbounded distance memoization

**File:** `src/data/nearbyCinemas.js`

**Before:**

```js
import { sortBy, memoize } from "lodash";

const computeCinemaDistance = memoize((lat, lng) => {
  // distance calculation
});
```

**After:**

```js
import { sortBy } from "lodash";

const computeCinemaDistance = (lat, lng) => {
  // distance calculation
};
```

**Why this was a bug:** Lodash's `memoize` keeps results for every distinct
coordinate pair and has no size limit here. Geolocation coordinates can change
over time, so the module-level cache could grow unnecessarily and retain stale
results.

**Method and reason for the change:** the lifetime of the cache was compared
with the component's existing `useMemo`. The hook already recalculates the
nearby list only when `coords` changes, so the second, unbounded cache added
complexity without providing useful reuse. Removing it avoids the retention
problem while preserving the same distance calculation and sorting behavior.

### 5. Add project documentation and repository metadata

The repository also includes:

- `README.md` with setup instructions, the exercise context, and the detailed
  change log above.
- `.gitignore` for dependencies, build output, test coverage, local
  environment files, logs, and editor files.
- `LICENSE` containing the MIT License.

These files do not change the application's runtime behavior; they make the
project easier to install, share, review, and maintain.

## License

This project is licensed under the [MIT License](LICENSE).
