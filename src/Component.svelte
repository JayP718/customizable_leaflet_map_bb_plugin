<script>
  import { onMount, getContext } from "svelte";
  import sanitizeHtml from "sanitize-html";
  import {
    FullScreenControl,
    LocationControl,
    initMapControls,
  } from "./EmbeddedMapControls.js";
  import L from "leaflet";

  initMapControls();

  export let dataProvider;
  export let error;
  export let zoomLevel;
  export let zoomEnabled = true;
  export let latitudeKey = null;
  export let longitudeKey = null;
  export let titleKey = null;
  export let fullScreenEnabled = true;
  export let locationEnabled = true;
  export let defaultLocation;
  export let tileURL = "https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png";
  export let mapAttribution;
  export let creationEnabled = false;
  export let onClickMarker;
  export let onCreateMarker;
  export let markerIcon;
  export let colorIcon;
  export let customIconOptions;
  export let customKeyField;
  export let customizationBoolean;
  export let iconType;

  const { styleable, notificationStore } = getContext("sdk");
  const component = getContext("component");
  const embeddedMapId = uuid();

  $: customIconOptions = customIconOptions ? customIconOptions : "bleh";
  function uuid() {
    return "cxxxxxxxxxxxx4xxxyxxxxxxxxxxxxxxx".replace(/[xy]/g, (c) => {
      const r = (Math.random() * 16) | 0;
      const v = c === "x" ? r : (r & 0x3) | 0x8;
      return v.toString(16);
    });
  }
  // Map Button Controls
  const locationControl = new LocationControl({
    position: "bottomright",
    onLocationFail: (err) => {
      if (err.code === GeolocationPositionError.PERMISSION_DENIED) {
        notificationStore.actions.error(
          "Location requests not permitted. Ensure location is enabled"
        );
      } else if (err.code === GeolocationPositionError.POSITION_UNAVAILABLE) {
        notificationStore.actions.warning(
          "Location could not be retrieved. Try again"
        );
      } else if (err.code === GeolocationPositionError.TIMEOUT) {
        notificationStore.actions.warning(
          "Location request timed out. Try again"
        );
      } else {
        notificationStore.actions.error("Unknown location error");
      }
    },
    onLocationSuccess: (pos) => {
      cachedDeviceCoordinates = pos;
      if (typeof mapInstance === "object") {
        mapInstance.setView(cachedDeviceCoordinates, 15);
      }
    },
  });
  const fullScreenControl = new FullScreenControl({
    position: "topright",
  });
  const zoomControl = L.control.zoom({
    position: "bottomright",
  });

  // Map and marker configuration
  $: defaultMarkerOptions = {
    html:
      '<div><i class="' +
      markerIcon +
      ' ri-2x" draggable="false" style="color:' +
      colorIcon +
      '"></i></div>',
    className: "embedded-map-marker",
    iconSize: [26, 26],
    iconAnchor: [13, 26],
    popupAnchor: [0, -13],
  };

  $: mapMarkerOptions = {
    icon: L.divIcon(defaultMarkerOptions),
    className: "embedded-map-marker",
    draggable: false,
    alt: "Location Marker",
  };

  const candidateMarkerOptions = {
    mapMarkerOptions,
  };
  const mapOptions = {
    fullScreen: false,
    zoomControl: false,
    scrollWheelZoom: zoomEnabled,
    minZoomLevel,
    maxZoomLevel,
  };
  const fallbackCoordinates = [40.73061, -73.935242]; // NYC ¯\_(ツ)_/¯

  let mapInstance;
  let mapMarkerGroup = new L.FeatureGroup();
  let candidateMarkerGroup = new L.FeatureGroup();
  let candidateMarkerPosition;
  let mounted = false;
  let initialMarkerZoomCompleted = false;
  let minZoomLevel = 0;
  let maxZoomLevel = 18;
  let cachedDeviceCoordinates;

  $: validRows = getValidRows(dataProvider?.rows, latitudeKey, longitudeKey);
  $: safeZoomLevel = parseZoomLevel(zoomLevel);
  $: defaultCoordinates = parseDefaultLocation(defaultLocation);
  $: initMap(tileURL, mapAttribution, safeZoomLevel);
  $: zoomControlUpdated(mapInstance, zoomEnabled);
  $: locationControlUpdated(mapInstance, locationEnabled);
  $: fullScreenControlUpdated(mapInstance, fullScreenEnabled);
  $: width = $component.styles.normal.width;
  $: height = $component.styles.normal.height;
  $: width, height, mapInstance?.invalidateSize();
  $: defaultCoordinates, resetView();
  $: addMapMarkers(
    mapInstance,
    validRows,
    latitudeKey,
    longitudeKey,
    titleKey,
    onClickMarker,
    defaultMarkerOptions,
    colorIcon
  );

  const isValidLatitude = (value) => {
    return !isNaN(value) && value > -90 && value < 90;
  };

  const isValidLongitude = (value) => {
    return !isNaN(value) && value > -180 && value < 180;
  };

  const getValidRows = (rows, latKey, lngKey) => {
    if (!rows?.length || !latKey || !lngKey) {
      return [];
    }
    return rows.filter((row) => {
      return isValidLatitude(row[latKey]) && isValidLongitude(row[lngKey]);
    });
  };

  const parseZoomLevel = (zoomLevel) => {
    let zoom = zoomLevel;
    if (zoom == null || isNaN(zoom)) {
      zoom = 50;
    } else {
      zoom = parseFloat(zoom);
      zoom = Math.max(0, Math.min(100, zoom));
    }
    return Math.round((zoom * maxZoomLevel) / 100);
  };

  const parseDefaultLocation = (defaultLocation) => {
    if (typeof defaultLocation !== "string") {
      return fallbackCoordinates;
    }
    let defaultLocationParts = defaultLocation.split(",");
    if (defaultLocationParts.length !== 2) {
      return fallbackCoordinates;
    }

    let parsedDefaultLatitude = parseFloat(defaultLocationParts[0].trim());
    let parsedDefaultLongitude = parseFloat(defaultLocationParts[1].trim());

    return isValidLatitude(parsedDefaultLatitude) === true &&
      isValidLongitude(parsedDefaultLongitude) === true
      ? [parsedDefaultLatitude, parsedDefaultLongitude]
      : fallbackCoordinates;
  };

  const resetView = () => {
    if (!mapInstance) {
      return;
    }
    if (mapMarkerGroup.getLayers().length) {
      mapInstance.setZoom(0);
      mapInstance.fitBounds(mapMarkerGroup.getBounds(), {
        paddingTopLeft: [0, 24],
      });
    } else {
      mapInstance.setView(defaultCoordinates, safeZoomLevel);
    }
  };

  const locationControlUpdated = (mapInstance, locationEnabled) => {
    if (typeof mapInstance !== "object") {
      return;
    }
    if (locationEnabled) {
      locationControl.addTo(mapInstance);
    } else {
      mapInstance.removeControl(locationControl);
    }
  };

  const fullScreenControlUpdated = (mapInstance, fullScreenEnabled) => {
    if (typeof mapInstance !== "object") {
      return;
    }
    if (fullScreenEnabled) {
      fullScreenControl.addTo(mapInstance);
    } else {
      mapInstance.removeControl(fullScreenControl);
    }
  };

  const zoomControlUpdated = (mapInstance, zoomEnabled) => {
    if (typeof mapInstance !== "object") {
      return;
    }
    if (zoomEnabled) {
      zoomControl.addTo(mapInstance);
      mapInstance.scrollWheelZoom.enable();
    } else {
      mapInstance.removeControl(zoomControl);
      mapInstance.scrollWheelZoom.disable();
    }
  };


const isValidColor = (strColor) => {
  const s = new Option().style;
  s.color = strColor;
  return s.color !== '';
}





  const addMapMarkers = (
    mapInstance,
    validRows,
    latKey,
    lngKey,
    titleKey,
    onClick,
    defaultMarkerOptions,
    colorIcon
  ) => {
    if (!mapInstance) {
      return;
    }
    mapMarkerGroup.clearLayers();
    if (!validRows?.length) {
      return;
    }
    validRows.forEach((row) => {
      var customMapMarkerOptions = Object.assign({}, mapMarkerOptions);
      let markerCoords = [row[latKey], row[lngKey]];
      if (customizationBoolean) {
        let lookup_element =
          customIconOptions.find(
            (element) => element.label == row[customKeyField]
          ) ?? false;
        if (iconType ? true : false && lookup_element) {
          if (iconType == "faCustIcon") {
            customMapMarkerOptions.icon.options.html =
              '<div><i class="' +
              lookup_element.value + ' ' + markerIcon +
              ' ri-2x" draggable="false" style="color:' +
              colorIcon +
              '"></i></div>';
          } else if (iconType == "colorCustIcon") {
            let validColor = isValidColor(lookup_element.value)?lookup_element.value:colorIcon
            customMapMarkerOptions.icon.options.html =
              '<div><i class="' +
              markerIcon +
              ' ri-2x" draggable="false" style="color:' +
              validColor +
              '"></i></div>';
          } else if (iconType == "urlCustIcon") {
            customMapMarkerOptions.icon = L.icon({
              iconUrl: lookup_element.value,
              className: "embedded-map-marker",
              iconSize: [26, 26],
              iconAnchor: [13, 26],
              popupAnchor: [0, -13],
            });
          } else {
            customMapMarkerOptions = Object.assign({}, mapMarkerOptions);
          }
        } else {
          customMapMarkerOptions = Object.assign({}, mapMarkerOptions);
        }
      } else {
        customMapMarkerOptions = Object.assign({}, mapMarkerOptions);
      }

      let marker = L.marker(markerCoords, customMapMarkerOptions).addTo(
        mapInstance
      );
      let markerContent = generateMarkerPopupContent(
        row[latKey],
        row[lngKey],
        row[titleKey]
      );

      marker
        .bindTooltip(markerContent, {
          direction: "top",
          offset: [0, -25],
        })
        .addTo(mapMarkerGroup);

      if (onClick) {
        marker.on("click", () => {
          onClick({
            marker: row,
          });
        });
      }
    });

    // Zoom to markers if this is the first time
    if (!initialMarkerZoomCompleted) {
      resetView();
      initialMarkerZoomCompleted = true;
    }
  };

  const generateMarkerPopupContent = (latitude, longitude, text) => {
    return text || latitude + "," + longitude;
  };

  const initMap = (tileURL, attribution, zoom) => {
    if (!mounted) {
      return;
    }
    if (mapInstance) {
      mapInstance.remove();
    }
    mapInstance = L.map(embeddedMapId, mapOptions);
    mapMarkerGroup.addTo(mapInstance);
    candidateMarkerGroup.addTo(mapInstance);

    // Add attribution
    const cleanAttribution = sanitizeHtml(attribution, {
      allowedTags: ["a"],
      allowedAttributes: {
        a: ["href", "target"],
      },
    });
    L.tileLayer(tileURL, {
      attribution: "&copy; " + cleanAttribution,
      zoom,
    }).addTo(mapInstance);

    // Add click handler
    mapInstance.on("click", handleMapClick);

    // Reset view
    resetView();
  };

  const handleMapClick = (e) => {
    if (!creationEnabled) {
      return;
    }
    candidateMarkerGroup.clearLayers();
    candidateMarkerPosition = [e.latlng.lat, e.latlng.lng];
    let candidateMarker = L.marker(candidateMarkerPosition, mapMarkerOptions);
    candidateMarker
      .bindTooltip("New marker", {
        permanent: true,
        direction: "top",
        offset: [0, -25],
      })
      .addTo(candidateMarkerGroup)
      .on("click", clearCandidateMarker);
  };

  const createMarker = async () => {
    if (!onCreateMarker) {
      return;
    }
    const res = await onCreateMarker({
      lat: candidateMarkerPosition[0],
      lng: candidateMarkerPosition[1],
    });
    if (res !== false) {
      clearCandidateMarker();
    }
  };

  const clearCandidateMarker = () => {
    candidateMarkerGroup.clearLayers();
    candidateMarkerPosition = null;
  };

  onMount(() => {
    mounted = true;
    initMap(tileURL, mapAttribution, safeZoomLevel);
  });
</script>

<div class="embedded-map-wrapper map-default" use:styleable={$component.styles}>
  {#if error}
    <div>{error}</div>
  {/if}
  <p>{colorIcon} {markerIcon}</p>

  <div id={embeddedMapId} class="embedded embedded-map" />
  {#if candidateMarkerPosition}
    <div class="button-container" />
  {/if}
</div>

<style>
  .embedded-map-wrapper {
    background-color: #f1f1f1;
    height: 320px;
  }
  .map-default {
    min-height: 180px;
    min-width: 200px;
  }
  .embedded-map :global(a.map-svg-button) {
    display: flex;
    justify-content: center;
    align-items: center;
  }
  .embedded-map :global(.leaflet-top),
  .embedded-map :global(.leaflet-bottom) {
    z-index: 998;
  }
  .embedded-map :global(.embedded-map-marker) {
    color: #ee3b35;
  }
  .embedded-map :global(.embedded-map-marker--candidate) {
    color: var(--primaryColor);
  }
  .embedded-map :global(.embedded-map-control) {
    font-size: 22px;
  }
  .embedded-map {
    height: 100%;
    width: 100%;
  }
  .button-container {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    gap: var(--spacing-xl);
    margin-top: var(--spacing-xl);
  }
</style>
