<template>
  <div id="mapwrapper">
    <div id="presentation">
      <h1>MapAbility</h1>
    </div>
    <div id="navbar">
      <div id="searchbar-div">
        <input
          id="location-search-bar"
          type="text"
          v-model="locationSearch"
          @keyup.enter="updateView"
        />
        <button id="search-button" @click="updateView">search</button>
      </div>
      <div id="nav-bar-buttons">
        <button id="refresh-button" @click="updateOSMData">refresh</button>
        <button id="download-button" @click="downloadCSV">Download</button>
        <button id="upload-button" @click="showUpload = !showUpload">Upload</button>
        <input
          type="number"
          id="polyline-weight"
          min="1"
          max="15"
          step="1"
          value="5"
          v-model="polylineWeight"
        />
      </div>
      <br />
      <div id="upload-div" v-show="showUpload">
        <input
          type="file"
          name="File Upload"
          id="txtFileUpload"
          accept=".csv"
          @change="readParseCSV"
        />
        <p id="upload-count">
          Rendered {{ uploadedEdgeLoaded }}/{{
          Object.keys(uploadedData).length
          }}
          edges from file
        </p>
      </div>
      <br />
    </div>
    <div id="osmmap"></div>
    <div id="color-picker">
      <div id="green-color" @click="changeColorToGreen" :style="{ borderColor: greenBorderColor }">
        <p id="green-color-value">Accessible for everyone</p>
      </div>
      <div
        id="orange-color"
        @click="changeColorToOrange"
        :style="{ borderColor: orangeBorderColor }"
      >
        <p id="orange-color-value">Inaccessible with a wheelchair</p>
      </div>
      <div id="red-color" @click="changeColorToRed" :style="{ borderColor: redBorderColor }">
        <p id="red-color-value">Inaccessible</p>
      </div>
    </div>
  </div>
</template>

<script>
var leaflet = require("leaflet");
export default {
  name: "maposm",
  data() {
    return {
      data: "",
      mymap: null,
      currentNodeList: JSON.parse(window.localStorage.getItem("edges")) || {},
      currentEdgeList: JSON.parse(window.localStorage.getItem("nodes")) || {},
      locationSearch: "New York City",
      currentColor: { color: "#D9042B", value: 2 },
      savedEdges: JSON.parse(window.localStorage.getItem("savedEdges")) || {},
      defaultColor: "#3388ff",
      showUpload: false,
      uploadedData: {},
      uploadedEdgeLoaded: 0,
      polylineWeight: 5,
    };
  },
  computed: {
    greenBorderColor: function () {
      if (this.currentColor.value == 0) {
        return "#04af1d";
      }
      return "#04D924";
    },
    orangeBorderColor: function () {
      if (this.currentColor.value == 1) {
        return "#c89704";
      }
      return "#F2B705";
    },
    redBorderColor: function () {
      if (this.currentColor.value == 2) {
        return "#af0423";
      }
      return "#D9042B";
    },
  },
  methods: {
    updateOSMData: function () {
      this.clearMap();
      let request = new XMLHttpRequest();
      let bounds = this.mymap.getBounds();
      let apiUrl =
        "http://overpass-api.de/api/interpreter?data=[bbox][out:json];(node;way;);out;&bbox=" +
        bounds.getWest() +
        "," +
        bounds.getSouth() +
        "," +
        bounds.getEast() +
        "," +
        bounds.getNorth();
      let nodeList = {};
      let edgeList = {};
      request.open("GET", apiUrl, true);
      request.onload = function (vueMap) {
        let osmResponse = JSON.parse(this.response).elements;
        for (let i = 0; i < osmResponse.length; i++) {
          if (osmResponse[i].type == "node") {
            nodeList[osmResponse[i].id] = {
              lat: osmResponse[i].lat,
              lon: osmResponse[i].lon,
            };
          }
        }
        vueMap.uploadedEdgeLoaded = 0;
        for (let i = 0; i < osmResponse.length; i++) {
          if (
            osmResponse[i].type == "way" &&
            osmResponse[i].tags &&
            !osmResponse[i].tags.building &&
            osmResponse[i].tags.highway &&
            osmResponse[i].tags.access != "private"
          ) {
            edgeList[osmResponse[i].id] = osmResponse[i].nodes;
            let edgePolyline = [];
            for (let j = 1; j < osmResponse[i].nodes.length; j++) {
              if (
                osmResponse[i].nodes[j] in nodeList &&
                osmResponse[i].nodes[j - 1] in nodeList
              ) {
                let polyline = leaflet
                  .polyline(
                    [
                      [
                        nodeList[osmResponse[i].nodes[j]].lat,
                        nodeList[osmResponse[i].nodes[j]].lon,
                      ],
                      [
                        nodeList[osmResponse[i].nodes[j - 1]].lat,
                        nodeList[osmResponse[i].nodes[j - 1]].lon,
                      ],
                    ],
                    { weight: vueMap.polylineWeight }
                  )
                  .addTo(vueMap.mymap);
                if (
                  [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]] in
                  vueMap.uploadedData
                ) {
                  vueMap.uploadedEdgeLoaded++;
                  vueMap.savedEdges[
                    [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]]
                  ] =
                    vueMap.uploadedData[
                      [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]]
                    ];
                }
                if (
                  [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]] in
                  vueMap.savedEdges
                ) {
                  switch (
                    vueMap.savedEdges[
                      [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]]
                    ]
                  ) {
                    case 0:
                      polyline.setStyle({ color: "#04D924" });
                      break;
                    case 1:
                      polyline.setStyle({ color: "#F2B705" });
                      break;
                    case 2:
                      polyline.setStyle({ color: "#D9042B" });
                      break;
                  }
                }
                polyline.on("click", (e) => {
                  if (
                    [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]] in
                    vueMap.savedEdges
                  ) {
                    delete vueMap.savedEdges[
                      [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]]
                    ];
                    e.target.setStyle({ color: vueMap.defaultColor });
                  } else {
                    vueMap.savedEdges[
                      [osmResponse[i].nodes[j], osmResponse[i].nodes[j - 1]]
                    ] = vueMap.currentColor.value;
                    e.target.setStyle({ color: vueMap.currentColor.color });
                  }
                  window.localStorage.setItem(
                    "savedEdges",
                    JSON.stringify(vueMap.savedEdges)
                  );
                });
              }
            }
          }
        }
        vueMap.currentNodeList = nodeList;
        vueMap.currentEdgeList = edgeList;
      }.bind(request, this);

      request.send();
    },
    updateView() {
      let request = new XMLHttpRequest();
      this.clearMap();
      let apiUrl =
        "https://nominatim.openstreetmap.org/search/" +
        this.locationSearch +
        "?format=json";
      request.open("GET", apiUrl, true);
      request.onload = function (vueMap) {
        let osmRequest = JSON.parse(this.response);
        if (osmRequest.length > 0) {
          vueMap.mymap.setView([osmRequest[0].lat, osmRequest[0].lon], 16);
          vueMap.updateOSMData();
        } else {
          vueMap.locationSearch = "Unable to find location";
        }
      }.bind(request, this);
      request.send();
    },
    clearMap() {
      for (let i in this.mymap._layers) {
        if (this.mymap._layers[i].options.format == undefined) {
          try {
            this.mymap.removeLayer(this.mymap._layers[i]);
          } catch (e) {
            console.log("problem with " + e + this.mymap._layers[i]);
          }
        }
      }
      leaflet
        .tileLayer("https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png", {
          attribution:
            'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
          maxZoom: 18,
          id: "mapbox.streets",
          accessToken:
            "pk.eyJ1Ijoic3BpaWxncmlpbSIsImEiOiJjanppZWgxN2gwODNzM2NvOHlmNTl4NDB6In0.7gxi13vVRZI5UMJR8ktV6A",
        })
        .addTo(this.mymap);
    },
    changeColorToGreen() {
      this.currentColor.color = "#04D924";
      this.currentColor.value = 0;
    },
    changeColorToOrange() {
      this.currentColor.color = "#F2B705";
      this.currentColor.value = 1;
    },
    changeColorToRed() {
      this.currentColor.color = "#D9042B";
      this.currentColor.value = 2;
    },
    getIdFromString(idString) {
      let index = idString.indexOf(",");
      return [idString.slice(0, index), idString.slice(index + 1)];
    },
    downloadCSV() {
      var csv = "Node1;Node2;Difficulty\n";
      for (let edge in this.savedEdges) {
        let edgeId = this.getIdFromString(edge);
        csv += [edgeId[0], edgeId[1], this.savedEdges[edge]].join(";");
        csv += "\n";
      }

      var downloadLink = document.createElement("a");
      var blob = new Blob(["\ufeff", csv]);
      var url = URL.createObjectURL(blob);
      downloadLink.href = url;
      let dateStamp = new Date(Date.now()).toISOString();
      downloadLink.download = this.locationSearch + dateStamp + ".csv";
      document.body.appendChild(downloadLink);
      downloadLink.click();
      document.body.removeChild(downloadLink);
    },
    readParseCSV(event) {
      let file = event.target.files[0];
      let reader = new FileReader();
      reader.readAsText(file);
      reader.onload = function (vueMap) {
        vueMap.uploadedData = {};
        let csvData = reader.result;
        let temp = ["", "", ""];
        let separator = ";";
        if (csvData.indexOf(",") > csvData.indexOf(";")) {
          separator = ",";
        }
        while (csvData.length > 1) {
          temp = ["", "", ""];
          let nextCut = csvData.indexOf(separator);
          temp[0] = csvData.slice(0, nextCut);
          csvData = csvData.slice(nextCut + 1);
          nextCut = csvData.indexOf(separator);
          temp[1] = csvData.slice(0, nextCut);
          csvData = csvData.slice(nextCut + 1);
          nextCut = csvData.indexOf("\n");
          temp[2] = csvData.slice(0, nextCut);
          csvData = csvData.slice(nextCut + 1);
          if (
            !isNaN(parseInt(temp[0])) &&
            !isNaN(parseInt(temp[1])) &&
            !isNaN(parseInt(temp[2]))
          ) {
            vueMap.uploadedData[
              [parseInt(temp[0]), parseInt(temp[1])]
            ] = parseInt(temp[2]);
          }
        }
        vueMap.updateOSMData();
      }.bind(reader, this);
    },
    upPolylineSize() {
      this.polylineWeight++;
      this.updateOSMData();
    },
    downPolylineSize() {
      this.polylineWeight--;
      this.updateOSMData();
    },
  },
  mounted() {
    this.mymap = leaflet.map("osmmap").setView([45.4241297, 4.4077427], 16);
    leaflet.tileLayer(
      "https://api.tiles.mapbox.com/v4/{id}/{z}/{x}/{y}.png?access_token={accessToken}",
      {
        attribution:
          'Map data &copy; <a href="https://www.openstreetmap.org/">OpenStreetMap</a> contributors, <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, Imagery © <a href="https://www.mapbox.com/">Mapbox</a>',
        maxZoom: 18,
        id: "mapbox.streets",
        accessToken:
          "pk.eyJ1Ijoic3BpaWxncmlpbSIsImEiOiJjanppZWgxN2gwODNzM2NvOHlmNTl4NDB6In0.7gxi13vVRZI5UMJR8ktV6A",
      }
    );
    this.updateOSMData();
  },
  watch: {
    currentNodeList: (newValue) => {
      window.localStorage.setItem("nodes", JSON.stringify(newValue));
    },
    currentEdgeList: (newValue) => {
      window.localStorage.setItem("edges", JSON.stringify(newValue));
    },
    savedEdges: (newValue) => {
      window.localStorage.setItem("savedEdges", JSON.stringify(newValue));
    },
  },
};
</script>

<style scoped>
#presentation {
  background-color: #2d2d2d;
  color: white;
  width: 100%;
  position: absolute;
  top: 0;
  left: 0;
  font-family: "Ubuntu Mono", monospace;
}

#presentation h1 {
  margin-left: 10px;
  display: inline-block;
}

#presentation a {
  float: right;
  margin-top: 10px;
  margin-right: 10px;
  color: white;
}
#navbar {
  margin-bottom: 10px;
  margin-top: 100px;
}

#nav-bar-buttons {
  display: inline-block;
  float: right;
}

#searchbar-div {
  display: inline-block;
  margin-bottom: 0;
}

#location-search-bar {
  width: 500px;
  border: 1px solid rgb(226, 226, 226);
  padding: 5px;
}

#search-button {
  background-color: #3388ff;
  color: white;
  border: none;
  border-radius: 3px;
  height: 25px;
}

#polyline-weight {
  float: right;
  margin-right: 5px;
  width: 70px;
  padding-top: 5px;
  padding-bottom: 5px;
}

#upload-button {
  float: right;
  margin-right: 5px;
  background-color: #f2b705;
  color: white;
  border: none;
  border-radius: 3px;
  height: 25px;
}

#download-button {
  float: right;
  margin-right: 5px;
  background-color: #d93d04;
  color: white;
  border: none;
  border-radius: 3px;
  height: 25px;
}

#refresh-button {
  float: right;
  background-color: #04d924;
  color: white;
  border: none;
  border-radius: 3px;
  height: 25px;
  margin-right: 5px;
}

#upload-div {
  position: absolute;
  right: 5px;
  top: 130px;
  border: 1px solid #f2b705;
  z-index: 10;
  background-color: white;
  padding: 10px;
  border-radius: 10px;
  font-family: "Ubuntu Mono", monospace;
}

#osmmap {
  height: 75vh;
  z-index: 1;
}

#color-picker {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  float: center;
  align-items: center;
  justify-content: center;
}

#color-picker p {
  display: inline-block;
  font-family: "Ubuntu Mono", monospace;
  color: white;
}

#green-color {
  grid-column: 1/2;
  background-color: #04d924;
  text-align: center;
  border: 5px solid;
}

#orange-color {
  grid-column: 2/3;
  background-color: #f2b705;
  text-align: center;
  border: 5px solid;
}

#red-color {
  grid-column: 3/4;
  background-color: #d9042b;
  text-align: center;
  border: 5px solid;
}

@media only screen and (max-width: 914px) and (min-width: 677px) {
  #upload-div {
    top: 160px;
  }

  #searchbar-div {
    margin-bottom: 10px;
  }
}

@media only screen and (max-width: 676px) and (min-width: 607px) {
  #navbar {
    margin-top: 170px;
  }

  #upload-div {
    top: 230px;
  }

  #location-search-bar {
    width: 400px;
  }

  #searchbar-div {
    margin-bottom: 10px;
  }
}

@media only screen and (max-width: 607px) and (min-width: 593px) {
  #navbar {
    margin-top: 250px;
  }

  #upload-div {
    top: 310px;
  }

  #location-search-bar {
    width: 450px;
  }

  #searchbar-div {
    margin-bottom: 10px;
  }
}

@media only screen and (max-width: 593px) and (min-width: 500px) {
  #navbar {
    margin-top: 250px;
  }

  #upload-div {
    top: 335px;
  }

  #location-search-bar {
    width: 400px;
  }

  #searchbar-div {
    margin-bottom: 10px;
  }
}

@media only screen and (max-width: 500px) {
  #navbar {
    margin-top: 250px;
  }

  #upload-div {
    top: 335px;
  }

  #location-search-bar {
    width: 300px;
  }

  #searchbar-div {
    margin-bottom: 10px;
  }
}
</style>
