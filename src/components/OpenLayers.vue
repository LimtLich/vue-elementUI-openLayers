<template>
  <div>
    <div class="map" id="map"></div>
    <form class="form-inline">
      <label>Shape type &nbsp;</label>
      <select id="type">
        <option value="Square">Square</option>
        <option value="Box">Box</option>
        <option value="Star">Star</option>
        <option value="None">None</option>
      </select>
    </form>
    <div>矢量地图Feature总数：
      <span id="count"></span>
    </div>
    <div id="info">No boxes selected</div>
  </div>

</template>

<script>
import Map from "ol/map";
import Contorl from "ol/control";
import View from "ol/view";
import TileLayer from "ol/layer/tile";
import SourceLayer from "ol/source/tilewms";
import Source from "ol/source/vector";
import Vector from "ol/layer/vector";
import Draw from "ol/interaction/draw";
import Geom from "ol/geom/polygon";
import Select from "ol/interaction/select";
import DragBox from "ol/interaction/dragbox"
import Condition from "ol/events/condition"

export default {
  name: "HelloWorld",
  data() {
    return {
      msg: "Welcome to Your Vue.js App"
    };
  },
  mounted: function() {
    var raster = new TileLayer({
      source: new SourceLayer({
        url: "http://127.0.0.1/geoserver/hongwan/wms",
        params: {
          LAYERS: "hongwan:hongwan",
          TILED: true
        },
        serverType: "geoserver",
        transition: 0
      })
    });

    var source = new Source({ wrapX: false });

    var vector = new Vector({
      source: source
    });

    var map = new Map({
      target: "map",
      controls: Contorl.defaults({
        attributionOptions: {
          collapsible: false
        }
      }),
      layers: [raster, vector],
      view: new View({
        center: [113.45521, 22.16287],
        zoom: 15,
        projection: "EPSG:4326",
        rotation: Math.PI / 6
      })
    });
    // a normal select interaction to handle click
    var select = new Select();
    map.addInteraction(select);
    var selectedFeatures = select.getFeatures();

    // a DragBox interaction used to select features by drawing boxes
    var dragBox = new DragBox({
      condition: Condition.platformModifierKeyOnly
    });

    map.addInteraction(dragBox);

    dragBox.on("boxend", function() {
      // features that intersect the box are added to the collection of
      // selected features
      var extent = dragBox.getGeometry().getExtent();
      vectorSource.forEachFeatureIntersectingExtent(extent, function(feature) {
        selectedFeatures.push(feature);
      });
    });

    // clear selection when drawing a new box and when clicking on the map
    dragBox.on("boxstart", function() {
      selectedFeatures.clear();
    });

    var infoBox = document.getElementById("info");

    selectedFeatures.on(["add", "remove"], function() {
      var names = selectedFeatures.getArray().map(function(feature) {
        return feature.get("name");
      });
      if (names.length > 0) {
        infoBox.innerHTML = names.join(", ");
      } else {
        infoBox.innerHTML = "No countries selected";
      }
    });

    var typeSelect = document.getElementById("type");
    var draw; // global so we can remove it later
    function addInteraction() {
      var value = typeSelect.value;
      if (value !== "None") {
        var geometryFunction;
        if (value === "Square") {
          value = "Circle";
          geometryFunction = Draw.createRegularPolygon(4);
        } else if (value === "Box") {
          value = "Circle";
          geometryFunction = Draw.createBox();
        } else if (value === "Star") {
          value = "Circle";
          geometryFunction = function(coordinates, geometry) {
            if (!geometry) {
              geometry = new Geom(null);
            }
            var center = coordinates[0];
            var last = coordinates[1];
            var dx = center[0] - last[0];
            var dy = center[1] - last[1];
            var radius = Math.sqrt(dx * dx + dy * dy);
            var rotation = Math.atan2(dy, dx);
            var newCoordinates = [];
            var numPoints = 12;
            for (var i = 0; i < numPoints; ++i) {
              var angle = rotation + i * 2 * Math.PI / numPoints;
              var fraction = i % 2 === 0 ? 1 : 0.5;
              var offsetX = radius * fraction * Math.cos(angle);
              var offsetY = radius * fraction * Math.sin(angle);
              newCoordinates.push([center[0] + offsetX, center[1] + offsetY]);
            }
            newCoordinates.push(newCoordinates[0].slice());
            geometry.setCoordinates([newCoordinates]);
            return geometry;
          };
        }
        draw = new Draw({
          source: source,
          type: value,
          geometryFunction: geometryFunction
        });
        var listenerKey = vector.getSource().on("change", function() {
          if (vector.getSource().getState() === "ready") {
            // 判定是否加载完成
            document.getElementById(
              "count"
            ).innerHTML = vector.getSource().getFeatures().length;
            vector.getSource().unset(listenerKey); // 注销监听器
          }
        });
        map.addInteraction(draw);
      }
    }
    /**
       * Handle change event.
       */
    typeSelect.onchange = function() {
      map.removeInteraction(draw);
      addInteraction();
    };

    addInteraction();
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1,
h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
