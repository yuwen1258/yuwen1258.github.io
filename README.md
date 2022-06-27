<!--

To run this demo, you need to replace 'YOUR_API_KEY' with an API key from the ArcGIS Developers dashboard.

Sign up for a free account and get an API key.

https://developers.arcgis.com/documentation/mapping-apis-and-services/get-started/

-->

<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="initial-scale=1, maximum-scale=1, user-scalable=no">
  <title>ArcGIS Developer Guide: Closest facility routing</title>
  <style>
    html, body, #viewDiv {
      padding: 0;
      margin: 0;
      height: 100%;
      width: 100%;
    }
  </style>

  <link rel="stylesheet" href="https://js.arcgis.com/4.23/esri/themes/light/main.css">
  <script src="https://js.arcgis.com/4.23/"></script>

  <script>
    require([
      "esri/config",
      "esri/Map",
      "esri/views/MapView",
      "esri/Graphic",
      "esri/layers/GraphicsLayer",
      "esri/rest/closestFacility",
      "esri/rest/support/ClosestFacilityParameters",
      "esri/rest/support/FeatureSet",
      "esri/geometry/Point",
      "esri/symbols/Symbol",
      "esri/rest/locator"
    ], (
      esriConfig,
      Map,
      MapView,
      Graphic,
      GraphicsLayer,
      closestFacility,
      ClosestFacilityParameters,
      FeatureSet,
      Point,
      Symbol,
      locator,
    )=> {

      esriConfig.apiKey = "AAPK1525711ac0e04c1c80f5ae77b75f89446lk_-Ce631ezZc8s7Sqy9netFXFTY6yaX_UesmIi5IGAR3mJvJ85R1a0kPth5JA4";

      let placeCategory = "Gas station";

      let startSymbol = {
        type: "simple-marker",
        color: "white",
        size: "10px",
        outline: {
          color: "black",
          width: "1px",
        },
      };

      let facilitySymbol = {
        type: "simple-marker",
        color: "black",
        size: "11px",
        outline: {
          color: "white",
          width: "1px",
        },
      };

      let routeSymbol = {
        type: "simple-line",
        color: [50, 150, 255, 0.75],
        width: "5",
      };

      const locatorUrl = "http://geocode-api.arcgis.com/arcgis/rest/services/World/GeocodeServer";

      const closestFacilityUrl = "https://route-api.arcgis.com/arcgis/rest/services/World/ClosestFacility/NAServer/ClosestFacility_World/solveClosestFacility/";


      const routeLayer = new GraphicsLayer();
      const facilitiesLayer = new GraphicsLayer();
      const selectedFacilitiesLayer = new GraphicsLayer();
      const startLayer = new GraphicsLayer();

      let map = new Map({
        basemap: "arcgis-navigation",
        layers: [routeLayer, facilitiesLayer, selectedFacilitiesLayer, startLayer],
      });

      let view = new MapView({
        container: "viewDiv",
        map: map,
        center: [-123.18586,49.24824],
        zoom: 12,
        constraints: {
          snapToZoom: false
        }
      });

      view.popup.actions = [];

      const places = [
        ["Gas station", "gas-station"],
        ["College", "school"],
        ["Grocery", "grocery-store"],
        ["Hotel", "hotel"],
        ["Hospital", "hospital"]
      ];

      const select = document.createElement("select", "");
      select.setAttribute("class", "esri-widget esri-select");
      select.setAttribute(
        "style",
        "width: 175px; font-family: 'Avenir Next'; font-size: 1em"
      );

      places.forEach((p) => {
        const option = document.createElement("option");
        option.value = p[0];
        option.innerHTML = p[0];
        select.appendChild(option);
      });

      view.ui.add(select, "top-right");

      view.when(() => {
        addStart(view.center);
        findFacilities(view.center, true);
      });

      view.on("click", (event)=> {
        view.hitTest(event).then((response)=> {
          if (response.results.length === 1) {
            findFacilities(event.mapPoint, false);
          }
        });
      });

      select.addEventListener("change", (event) => {
        placeCategory = event.target.value;
        findFacilities(startLayer.graphics.getItemAt(0).geometry, true);
      });

      // Find places and add them to the map
      function findFacilities(pt, refresh) {
        view.popup.close();
        //startLayer.graphics.removeAll();
        addStart(pt);
        if (refresh) {
          // Add facilities
          locator
            .addressToLocations(locatorUrl,{
              location: pt,
              searchExtent: view.extent,
              categories: [placeCategory],
              maxLocations: 25,
              outFields: ["Place_addr", "PlaceName"],
              outSpatialReference: view.spatialReference
            })
            .then((results)=> {
              facilitiesLayer.removeAll();
              // Add graphics
              showFacilities(results);
              // Find closest place
              findClosestFacility(startLayer.graphics.getItemAt(0), facilitiesLayer.graphics);
            });
          } else {
            findClosestFacility(startLayer.graphics.getItemAt(0), facilitiesLayer.graphics);
          }
      }

      function addStart(pt) {
        startLayer.graphics.removeAll();
        startLayer.add(new Graphic({
          geometry: pt,
          symbol: startSymbol
        }));
      }

      function findClosestFacility(startGraphic, facilityGraphics) {
        routeLayer.removeAll();
        selectedFacilitiesLayer.removeAll();
        let params = new ClosestFacilityParameters({
          incidents: new FeatureSet({
            features: [startGraphic],
          }),
          facilities: new FeatureSet({
            features: facilityGraphics.toArray(),
          }),
          returnRoutes: true,
          returnFacilities: true,
          defaultTargetFacilityCount: 3,
        });

        closestFacility.solve(closestFacilityUrl, params).then(
          (results) => {
            results.routes.forEach((route, i)=> {
              // Add closest route
              route.symbol = routeSymbol;
              routeLayer.add(route);
              // Add closest facility
              const facility = results.facilities[route.attributes.FacilityID - 1];
              addSelectedFacility(i + 1, facility.latitude, facility.longitude, route.attributes);
            });
          },
          (error) => {
            console.log(error.details);
          }
        );
      }

      function showFacilities(results) {
        results.forEach((result,i)=> {
          facilitiesLayer.add(
            new Graphic({
              attributes: result.attributes,
              geometry: result.location,
              symbol: {
                type: "web-style",
                name: getIconName(placeCategory),
                styleName: "Esri2DPointSymbolsStyle",
              },
              popupTemplate: {
                title: "{PlaceName}",
                content: "{Place_addr}" +
                  "<br><br>" +
                  result.location.longitude.toFixed(5) +
                  "," +
                  result.location.latitude.toFixed(5),
              },
            })
          );
        });
      }

      function addSelectedFacility(number, latitude, longitude, attributes) {
        selectedFacilitiesLayer.add(new Graphic({
          symbol: {
            type: "simple-marker",
            color: [255, 255, 255,1.0],
            size: 18,
            outline: {
              color: [50,50,50],
              width: 1
            }
          },
          geometry: {
            type: "point",
            latitude: latitude,
            longitude: longitude
          },
          attributes: attributes
        }));
        selectedFacilitiesLayer.add(new Graphic({
          symbol: {
            type: "text",
            text: number,
            font: { size: 11, weight: "bold" },
            yoffset: -4,
            color: [50,50,50]
          },
        geometry: {
          type: "point",
          latitude: latitude,
          longitude: longitude
        },
        attributes: attributes
        }));
      }

      function getIconName(category) {
        let iconName;
        switch (category) {
          case "Grocery":
            iconName = "grocery-store";
            break;
          case "College":
            iconName = "school";
            break;
          case "Gas station":
            iconName = "gas-station";
            break;
          case "Hospital":
            iconName = "hospital";
            break;
          case "Hotel":
            iconName = "hotel";
            break;
        }
        return iconName;
      }
    });

</script>
</head>
<body>
  <div id="viewDiv"></div>
</body>
</html>
