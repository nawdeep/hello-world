<template>
    <div class="container">
        <div class="infoHead">
            <div :class="[{'warn-msg' : nameError}]">
                <label> Name : </label>
                <input type="text" v-model="mappingTemplate.output.catalogName" placeholder="mapping name"  @blur="nameError"/>
            </div>
            <div>
                <label> Product : </label>
                <v-select :options="prodTemplates" label="productName" placeholder="Select output catalog"
                  v-model="mappingTemplate.prodTemplate" @input="setProdTemplate">
                    <template slot="option" slot-scope="ops">
                      <div class="row"> <span class="col"> {{ops.productName}} </span> </div>
                    </template>
                </v-select>
            </div>
            <div>
                <label> Source(s) : </label>
                <v-select multiple :options="setSource" v-model="selectedSource" :no-drop="true" :disabled='true' ></v-select>
            </div>
            <DS-data-loader ref="_dsData" v-if="isDS" :connProps="selectedDSProps" @setCatalog="setCatalogs"></DS-data-loader>
        </div>
        <div class="catalogHead">
            <div>
                <label> Catalogs </label>
                <v-select multiple label="title" :options="catalogs" v-model="selectedCatalogs" @input="clearSelectionOnClose"></v-select>
            </div>
            <div v-if = "selectedCatalogs.length > 0">
               <label> Load layers for catalog </label>
               <label class="infomsg">{{selectedCatalogs.length - mappedCatalog.length}} Catalog(s) pending.</label>
               <v-select :options="selectedCatalogs" label="title"  v-model="selectedCatalog" @input="setLayers">
                   <template slot="option" slot-scope="ops">
                       <div class="row" :class="isMapped(ops.title)?'fileMapped':'fileUnmapped'"> <span class="col"></span>
                           <span class="col"> {{ops.title}} </span></div>
                   </template>
               </v-select>
            </div>
        </div>
        <div v-if="showSchema" class="schemaDetails">
           <catalogPublicationView
              class="item"
              :items="getCatalogDetails"
              :parent="parent" >
            </catalogPublicationView>
          <span v-if="!allLayersLoaded" class="loadLayers pointer" @click="loadSchemaForLayers" >load more layers ...</span>
          <lui-button v-show="allLayersLoaded" class="lui--small" @click="saveCatalog"> Save </lui-button>
        </div>
        <lui-button v-show="selectedCatalogs.length > 0 && mappingTemplate.sources.length-selectedCatalogs.length==0" class="lui--small" @click="saveCatalogMapping"> Save Catalog Mapping</lui-button>
        <lui-button v-show="selectedCatalogs.length > 0 && mappingTemplate.sources.length-selectedCatalogs.length==0" class="lui--small" @click="saveAndBuildCatalogMapping"> Save And Build </lui-button>

        <div id="errorBlock" :style="{display : hasError? 'block' : 'none'}">
            <p v-if="mappingNameError" id = "errors">
                <img src='../assets/loginError.gif' alt='Error'>
                <span class='errormsg'>Please provide the mapping name! </span>
            </p>
            <lui-button class="lui--small" @click="clearErrorBlock" style="top: 70%; left:40%">OK</lui-button>
        </div>
    </div>
</template>

<script>
import "vue-awesome/icons/times-rectangle";
import "@lui/init/dist/lui-init.min";
import "@lui/button/dist/lui-button.min";
var CatalogMappingJson = function() {
  this.inputSourceHRN = "";
  this.name = "";
  this.layers = [];
};
var CatalogMapping = function() {
  this.inputSourceHRN = "";
  this.name = "";
  this.include = true;
  this.attributes = []; //list of layers
};

var LayerMapping = function(name, hrn, schemaHRN) {
  this.name = name;
  this.layerHRN = hrn;
  this.schemaHRN = schemaHRN;
  this.condition = null;
  this.transformationMap = null;
  this.attributes = [];
  this.include = true;
  this.createChild = function(n, fN, t, children) {
    return {
      name: n,
      fullName: fN,
      dataType: t,
      include: true,
      condition: null,
      transformationMap: null,
      attributes: children
    };
  };
};

export default {
  name: "create-catalog-to-catalog-mapping",
  components: {
    vSelect: () => import("vue-select"),
    DSDataLoader: () => import("@/components/DSDataLoader.vue"),
    catalogPublicationView: () => import("@/components/TreeView.vue")
  },
  data() {
    return {
      // defaultOutputProduct: {
      //   productName: "Select output catalog"
      // },
      mappingTemplate: {
        prodTemplateId: 0,
        mappingInfo: {
          mappingType: "C2C",
          version: 1,
          specVersion: "1.0",
          mappingSchemaUrl: "http://hellas/catalog2catalog.schema.json"
        },
        output: {
          catalogName: ""
        },
        sources: [],
        description: "catalog mapping file from ui"
      },
      prodTemplates: [],
      mappedCatalog: [],
      parent: "root",
      selectedSource: [],
      catalogs: [],
      selectedCatalog: null,
      selectedCatalogs: [],
      dsType: "",
      connProps: {},
      layers: [],
      fetchingData: false,
      showSchema: false,
      catalogLayers: [],
      catalogDetails: null,
      layerIndex: 0,
      allLayersLoaded: false,
      nameError: false,
      mappingNameError: false,
      hasError: false,
      selectedProductTemplate: {}
    };
  },
  created: function() {
  //  this.mappingTemplate.prodTemplate = this.defaultOutputProduct;
  //  this.prodTemplates.push(this.defaultOutputProduct);
  //  this.selectedProductTemplate = this.defaultOutputProduct;
    this.$axios
      .get(this.$root.urlFor("listTemplates"),
        { params: {
            type: 'CSV'
          }
        })
      .then(res => {
        console.debug("fetched product names");
        this.prodTemplates.push(...res.data);
      })
      .catch(e => {
        console.error("error while fetching product names");
        console.error(e.statusText);
      });
  },
  computed: {
    setSource: function() {
      var sources = this.$root.getConfigFor("catalogSource");
      if (sources.includes("OLP")) {
        this.setDefaultSource();
      }
      return sources;
    },
    isDS: function() {
      this.setDStype();
      return this.selectedSource.includes("OLP");
    },
    selectedDSProps: function() {
      return this.getDSProps();
    },
    getCatalogDetails: function() {
      return this.catalogDetails;
    }
  },
  methods: {
    isMapped: function(fn) {
      return this.mappedCatalog.indexOf(fn) != -1;
    },
    getDSProps: function() {
      this.connProps = this.$root.getConfigFor(this.dsType + ".connectionInfo");
      this.connProps.catalogCachekey = this.dsType + "_catalogList";
      this.connProps.dsType = this.dsType;
      return this.connProps;
    },

    setCatalogs: function() {
      var lookupkey = this.selectedDSProps.catalogCachekey;
      if (sessionStorage.getItem(lookupkey)) {
        this.catalogs = JSON.parse(sessionStorage.getItem(lookupkey));
      } else {
        setTimeout(this.setCatalogs, 1500);
      }
    },
    setDStype: function() {
      this.dsType = this.selectedSource.includes("OLP") ? "OLP" : "";
    },
    setDefaultSource: function() {
      this.selectedSource = ["OLP"];
    },

    setLayers: function() {
      if (this.selectedCatalog != null) {
        this.catalogDetails = new CatalogMapping();
        if (this.isMapped(this.selectedCatalog.title)) {
          this.mappingTemplate.sources.forEach(ctlog => {
            if (
              ctlog.inputSourceHRN == this.selectedCatalog.hrn &&
              ctlog.name == this.selectedCatalog.title
            ) {
              this.catalogDetails.inputSourceHRN = this.selectedCatalog.hrn;
              this.catalogDetails.name = this.selectedCatalog.title;
              this.catalogDetails.attributes = ctlog.layers;
              this.showSchema = true;
              this.allLayersLoaded = true;
            }
          });
        } else {
          this.catalogDetails.inputSourceHRN = this.selectedCatalog.hrn;
          this.catalogDetails.name = this.selectedCatalog.title;
          this.catalogDetails.attributes = this.catalogLayers;
          if (this.selectedCatalog) {
            var lookupkey = this.getValUsingMetadata(
              "catalog",
              "key",
              this.selectedCatalog
            );
            if (sessionStorage.getItem(lookupkey)) {
              this.layers = JSON.parse(sessionStorage.getItem(lookupkey));

              this.loadSchemaForLayers();
              this.fetchingData = false;
            } else {
              this.layers = [];
              if (!this.fetchingData) {
                this.$refs._dsData.loadLayersForCatalog(this.selectedCatalog);
                this.fetchingData = true;
              }
              setTimeout(this.setLayers, 1500);
            } //end else
          }
        }
      }
    },
    clearSelectionOnClose: function() {
      if (this.selectedCatalogs && this.selectedCatalogs.length == 0) {
        this.clearSchema();
      }
    },
    loadSchemaForLayers: function() {
      var layerObjs = [];
      var layerSchemaObjs = [];
      var startIndex = this.layerIndex;
      var limit = this.layerIndex + 5;
      if (limit >= this.layers.length) {
        limit = this.layers.length;
        this.allLayersLoaded = true;
      }
      for (var i = startIndex; i < limit; i++) {
        if (this.layers[i].schema) {
          var lookupkey = this.getValUsingMetadata(
            "layer",
            "key",
            this.layers[i]
          );
          if (sessionStorage.getItem(lookupkey)) {
            var layerSchemaObj = {};
            layerSchemaObj.name = this.layers[i].name;
            layerSchemaObj.layerHRN = this.layers[i].hrn;
            layerSchemaObj.schemaHRN = this.layers[i].schema.hrn;
            layerSchemaObj.schema = JSON.parse(
              sessionStorage.getItem(lookupkey)
            );

            layerSchemaObjs.push(layerSchemaObj);
          } else {
            layerObjs.push(this.layers[i]);
          }
        }
        this.layerIndex = i + 1;
      }
      var that = this;
      var schemaFetchPromise = new Promise(function(resolve, reject) {
        that.$refs._dsData.loadSchemaInBatch(
          that.selectedCatalog,
          layerObjs,
          resolve,
          reject
        );
      });
      schemaFetchPromise.then(
        result => {
          var ii = 0;
          result.forEach(schema => {
            var lookupkey = this.getValUsingMetadata(
              "layer",
              "key",
              layerObjs[ii]
            );
            sessionStorage.setItem(lookupkey, schema);
            var layerSchemaObj = {};
            layerSchemaObj.name = layerObjs[ii].name;
            layerSchemaObj.layerHRN = layerObjs[ii].hrn;
            layerSchemaObj.schema = JSON.parse(schema);
            layerSchemaObj.schemaHRN = layerObjs[ii].schema.hrn;
            layerSchemaObjs.push(layerSchemaObj);
            ii++;
          });
          that.setLayerSchemaMapping(layerSchemaObjs);
        },
        error => {
          if (error.response) {
            console.error("error: " + error.response);
          }
        }
      );
    },
    setLayerSchemaMapping: function(layerSchemaObjs) {
      layerSchemaObjs.forEach(layerSchemaObj => {
        var schema = layerSchemaObj.schema;
        var layerDetails = new LayerMapping(
          layerSchemaObj.name,
          layerSchemaObj.layerHRN,
          layerSchemaObj.schemaHRN
        );
        var attrObj = schema.definitions[schema.entrypoint];
        for (var attribute in attrObj.properties) {
          var child = this.createChild(
            layerDetails,
            attribute,
            attrObj.properties[attribute],
            schema,
            "",
            schema.entrypoint
          );

          layerDetails.attributes.push(child);
        }
        this.catalogLayers.push(layerDetails);
      });
      this.showSchema = true;
    },

    createChild: function(
      layer,
      attributeName,
      attribute,
      schema,
      parentName,
      entryPoint
    ) {
      if (parentName == "") {
        var n = entryPoint.lastIndexOf(".");
        if (n > -1) parentName = entryPoint.substring(n + 1);
      }
      var fullName = parentName + ".." + attributeName;
      var children = null;
      if (attribute.$ref) {
        var isRef = true;
        var ref = attribute.$ref.replace("#/", "").split("/");
        attribute = schema[ref[0]];
        for (var ii = 1; ii < ref.length; ii++) {
          attribute = attribute[ref[ii]];
        }
      }
      if (attribute.type == "array" || isRef) {
        children = [];
        var props = [];
        if (isRef) {
          props = attribute.properties;
        } else {
          if (attribute.items.$ref) {
            ref = attribute.items.$ref.replace("#/", "").split("/");
            var beforeType = attribute.type;
            attribute = schema[ref[0]];
            for (ii = 1; ii < ref.length; ii++) {
              attribute = attribute[ref[ii]];
              if (beforeType != attribute.type) {
                attribute.type = beforeType;
              }
            }
            props = attribute.properties;
          } else {
            props = attribute.items.properties;
          }
        }
        fullName = parentName + ".." + attributeName;
        for (var subAttr in props) {
          var child = this.createChild(
            layer,
            subAttr,
            props[subAttr],
            schema,
            fullName,
            entryPoint
          );
          children.push(child);
        }
      }
      return layer.createChild(
        attributeName,
        fullName,
        attribute.type,
        children
      );
    },

    getValUsingMetadata: function(objType_, propName_, obj_) {
      var prop = this.connProps.dsType + "." + objType_ + "metadata";
      var mtdt = this.$root.getConfigFor(prop);
      return this.$root.extractValueFor(mtdt, propName_, obj_);
    },

    verify: function() {
      if (this.mappingTemplate.output.catalogName.trim().length === 0) {
        this.nameError = true;
        this.mappingNameError = true;
        this.hasError = true;
        return false;
      }
      return true;
    },
    saveCatalog: function() {
      var ifUpdate = false;
      this.mappingTemplate.sources.forEach(catalog => {
        if (
          catalog.inputSourceHRN == this.catalogDetails.inputSourceHRN &&
          catalog.name == this.catalogDetails.name
        ) {
          catalog.layers = this.catalogDetails.attributes;
          ifUpdate = true;
        }
      });
      if (!ifUpdate) {
        var catalogMappingJson = new CatalogMappingJson();
        catalogMappingJson.name = this.catalogDetails.name;
        catalogMappingJson.inputSourceHRN = this.catalogDetails.inputSourceHRN;
        catalogMappingJson.layers = this.catalogDetails.attributes;
        this.mappingTemplate.sources.push(catalogMappingJson);
        this.mappedCatalog.push(this.catalogDetails.name);
      }
      this.clearSchema();
    },

    saveCatalogMapping: function() {
      if (this.verify()) {
        this.$axios({
          method: "post",
          url: this.$root.urlFor("saveCatalogMappings"),
          data: this.mappingTemplate,
          config: { headers: { "Content-Type": "application/json" } }
        })
          .then(resp => {
            this.mappingTemplate = resp.data;
            var msg =
              "saved catalog mapping file '" +
              this.mappingTemplate.output.catalogName +
              "' with id : " +
              this.mappingTemplate.uuid;
            this.$root.showModal("success", msg, "home");
          })
          .catch(e => {
            var msg = "Error while triggering build:";
            if (e.response) {
              msg = msg + e.response.status;
            }
            this.$root.showModal("failure", msg);
          });
      }
    },
    saveAndBuildCatalogMapping: function() {
      if (this.verify()) {
        this.$axios({
          method: "post",
          url: this.$root.urlFor("saveAndBuildCatalogMapping"),
          data: this.mappingTemplate,
          config: { headers: { "Content-Type": "application/json" } }
        })
          .then(resp => {
            var pipelineResp = resp.data;
            var msg = "";
            if (
              "null" != pipelineResp.pipelineId &&
              null != pipelineResp.pipelineId &&
              "" != pipelineResp.pipelineId
            ) {
              msg =
                "deployed catalog mapping file '" +
                this.mappingTemplate.output.catalogName +
                "' with pipleine id : " +
                pipelineResp.pipelineId;
            } else {
              msg =
                "Some error in building the pipeline, catalog mapping '" +
                this.mappingTemplate.output.catalogName +
                "' saved with id : " +
                pipelineResp.mapping.uuid;
            }
            if (
              "null" != pipelineResp.platformUrl &&
              null != pipelineResp.platformUrl &&
              "" != pipelineResp.platformUrl &&
              pipelineResp.platformUrl.includes("http")
            ) {
              window.location = pipelineResp.platformUrl;
            } else {
              this.$root.showModal("success", msg, "home");
            }
          })
          .catch(e => {
            var msg = "Error while triggering build:";
            if (e.response) {
              msg = msg + e.response.status;
            }
            this.$root.showModal("failure", msg);
          });
      }
    },

    clearSchema: function() {
      this.selectedCatalog = null;
      this.catalogLayers = [];
      this.catalogDetails = null;
      this.showSchema = false;
      (this.layerIndex = 0), (this.allLayersLoaded = false);
    },
    clearErrorBlock: function(event) {
      console.debug(event);
      this.hasError = false;
      this.mappingNameError = false;
    },
    setProdTemplate: function() {
      // if(!this.mappingTemplate.prodTemplate){
      //   this.mappingTemplate.prodTemplate = this.defaultOutputProduct;
      // }
      this.selectedProductTemplate = this.mappingTemplate.prodTemplate;
    }
  }
};
</script>

<style scoped>
.container {
  width: 100%;
  text-align: left;
  margin-left: 2em;
}

.infoHead {
  padding: 10px;
  display: block;
  border-bottom: 1px solid white;
}

.infoHead > div {
  display: table-cell;
  padding: 2px;
}

.infoHead label {
  display: block;
}

.infoHead input {
  height: 36px;
}

.catalogHead {
  padding: 10px;
  display: block;
}

.catalogHead > div {
  display: table-cell;
  padding: 2px;
}

.schemaDetails {
  width: 90vw;
  background: rgba(158, 158, 158, 0.26);
  padding: 3px;
  height: 200%;
  overflow: scroll;
}

span.loadLayers {
  text-align: center;
}

.pointer {
  color: blue;
  cursor: pointer;
}

.fileMapped {
  background: rgba(0, 200, 0, 0.3);
}

.fileUnmapped {
  background: rgba(200, 0, 0, 0.3);
}

.warn-msg {
  border: 1px solid red;
  border-radius: 4px;
}

#errorBlock {
  border: 1px solid black;
  border-radius: 10px;
  background: #e8ae55b5;
  position: absolute;
  z-index: 9;
  top: 40%;
  left: 37%;
  width: 400px;
  height: 200px;
}

#errors {
  margin: 0;
  position: absolute;
  top: 40%;
  left: 50%;
  margin-right: -50%;
  transform: translate(-50%, -50%);
}
</style>

<docs>
    documentation for this component
</docs>
