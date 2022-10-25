<script>
  import { onMount, getContext } from "svelte";
  import "leaflet/dist/leaflet.css";

  import sanitizeHtml from "sanitize-html";
  import {
    FullScreenControl,
    LocationControl,
    ShowLayerControl,
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
  export let showLayerControlEnabled = false;
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
  export let tileMultipleURL;

  const { styleable, notificationStore } = getContext("sdk");
  const component = getContext("component");
  const embeddedMapId = uuid();

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

  const showLayerControl = new ShowLayerControl(null, null, {
    position: "topleft",
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
  let mapMasterMarkerGroup = new L.FeatureGroup();

  let mapTileLayers = new L.layerGroup();

  let candidateMarkerGroup = new L.FeatureGroup();
  let candidateMarkerPosition;
  let mounted = false;
  let initialMarkerZoomCompleted = false;
  let minZoomLevel = 0;
  let maxZoomLevel = 18;
  let cachedDeviceCoordinates;
  let layerGroups = {};

  $: validRows = getValidRows(dataProvider?.rows, latitudeKey, longitudeKey);
  $: safeZoomLevel = parseZoomLevel(zoomLevel);
  $: defaultCoordinates = parseDefaultLocation(defaultLocation);
  $: initMap(tileURL, mapAttribution, safeZoomLevel);
  $: zoomControlUpdated(mapInstance, zoomEnabled);
  $: locationControlUpdated(mapInstance, locationEnabled);
  $: fullScreenControlUpdated(mapInstance, fullScreenEnabled);
  $: showLayerControlUpdated(mapInstance, showLayerControlEnabled);
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
    colorIcon,
    defaultMarkerOptions,
    customizationBoolean,
    customKeyField,
    iconType,
    customIconOptions
  );

  $: addMultipleTileLayers(mapInstance, tileMultipleURL);
  $: updateLayerGroup(showLayerControlEnabled);

  const updateLayerGroup = (showLayerControlEnabled) => {
    addLayerGroup(customIconOptions)
  }


  const addMultipleTileLayers = (mapInstance, tileMultipleURL) => {
    mapTileLayers.clearLayers();
    if (!mapInstance) {
      return;
    }
    if (!tileMultipleURL) {
      return;
    }
    for (let i = 0; i < tileMultipleURL.length; i++) {
      mapTileLayers.addLayer(L.tileLayer(tileMultipleURL[i].value));
    }
  };



  const addLayerGroup = (customIconOptions) => {
    if (!mapInstance) {
      return;
    }
    if(!showLayerControl){
      return;
    }

    for (let i = 0; i < customIconOptions.length; i++) {
      layerGroups[customIconOptions[i].label] = new L.FeatureGroup();
      layerGroups[customIconOptions[i].label].addTo(mapInstance);
      showLayerControl._addOverlay(
        layerGroups[customIconOptions[i].label],
        customIconOptions[i].label
      );
      mapMasterMarkerGroup.addLayer(layerGroups[customIconOptions[i].label]);
    }
    showLayerControl._addOverlay(mapMarkerGroup, "Other");
  };

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
    if (mapMasterMarkerGroup.getLayers().length) { //if custom markers
      mapInstance.setZoom(0);
      mapInstance.fitBounds(mapMasterMarkerGroup.getBounds(), {
        paddingTopLeft: [0, 24],
      });
    } 
    else if (mapMarkerGroup.getLayers().length){ //if default markers
      mapInstance.setZoom(0);
      mapInstance.fitBounds(mapMarkerGroup.getBounds(), {
        paddingTopLeft: [0, 24],
      });
    }
    else {
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

  const showLayerControlUpdated = (mapInstance, showLayerControlEnabled) => {
    if (typeof mapInstance !== "object") {
      return;
    }
    if (showLayerControlEnabled) {
      showLayerControl.addTo(mapInstance);
    } else {
      mapInstance.removeControl(showLayerControl);
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
    return s.color !== "";
  };

  const addMapMarkers = (
    mapInstance,
    validRows,
    latKey,
    lngKey,
    titleKey,
    onClick,
    colorIcon,
    defaultMarkerOptions,
    customizationBoolean,
    customKeyField,
    iconType,
    customIconOptions
  ) => {
    if (!mapInstance) {
      return;
    }
    if (!validRows?.length) {
      return;
    }
    mapMasterMarkerGroup.clearLayers()
    if (!initialMarkerZoomCompleted && customIconOptions) {
      addLayerGroup(customIconOptions);
    }

    let currentMapMarkerGroup;
    mapMarkerGroup.clearLayers();

    validRows.forEach((row) => {
      currentMapMarkerGroup = mapMarkerGroup;

      let markerCoords = [row[latKey], row[lngKey]];
      let key_field = row[customKeyField] ?? false;

      if (customizationBoolean && iconType && key_field && customIconOptions) {
        let lookup_element =
          customIconOptions.find((element) => element.label == key_field) ??
          false;
        currentMapMarkerGroup = layerGroups[key_field] ?? mapMarkerGroup;
        if (!lookup_element) {
        } else if (iconType == "faCustIcon") {
          mapMarkerOptions.icon.options.html =
            '<div><i class="' +
            lookup_element.value +
            " " +
            markerIcon +
            ' ri-2x" draggable="false" style="color:' +
            colorIcon +
            '"></i></div>';
        } else if (iconType == "colorCustIcon") {
          let validColor = isValidColor(lookup_element.value)
            ? lookup_element.value
            : colorIcon;
          mapMarkerOptions.icon.options.html =
            '<div><i class="' +
            markerIcon +
            ' ri-2x" draggable="false" style="color:' +
            validColor +
            '"></i></div>';
        } else if (iconType == "urlCustIcon") {
          mapMarkerOptions.icon = L.icon({
            iconUrl:
              lookup_element.value ??
              "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABgAAAAYCAYAAADgdz34AAAAAXNSR0IArs4c6QAAAERlWElmTU0AKgAAAAgAAYdpAAQAAAABAAAAGgAAAAAAA6ABAAMAAAABAAEAAKACAAQAAAABAAAAGKADAAQAAAABAAAAGAAAAADiNXWtAAAA7UlEQVRIDd1U2w2DMAyMmIM/5kBCYojuxiD8MwZL9Aepv/ROSiRw68RJk58inUjgfGc7D+f+/jmdm4EF2IHDg2N+m4sbgOAB2IAzAXKGLCMEjMAzIXw1Jnc0mYDIzHPEgxFj0pWAZGlLEJXvLVoFxLmgMih3flv4Tjg+xLxkqmsge24/LeMV/3oPjjXermaFIO5zLbAPgd5E4x2Bx7ds0fVf/TEya94iHn+tdOv3RS0d4tW36YcZTNodNLrBoO1V4U0mGL0Aa9/JnRhrfhDA9bCYkHO7HmqalIuHLCKV/C4eMakn/sWkvrgwyVrQN4qL5fShlLVMAAAAAElFTkSuQmCC",
            className: "embedded-map-marker",
            iconSize: [26, 26],
            iconAnchor: [13, 26],
            popupAnchor: [0, -13],
          });
        } else {
        }
      } else {
      }

      let marker = L.marker(markerCoords, mapMarkerOptions).addTo(mapInstance);
      mapMarkerOptions = {
        icon: L.divIcon(defaultMarkerOptions),
        className: "embedded-map-marker",
        draggable: false,
        alt: "Location Marker",
      };
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
        .addTo(currentMapMarkerGroup);

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

    mapInstance.addLayer(L.tileLayer(tileURL, {
        attribution: "&copy; " + cleanAttribution,
        zoom,
      }))

    mapTileLayers.addTo(mapInstance);

    
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
