<script>
import axios from "axios";
import rateLimit from "axios-rate-limit";
import JSZip from "jszip";
import Promise from "promise";
import FileSaver from "file-saver";
import ModuleConfigurator from "./ModuleConfigurator.vue";

export default {
  name: "ConfiguratorPanel",
  data: function () {
    return {
      repo: "bionanoimaging/UC2-GIT", // "bionanoimaging/UC2-GIT", //
      branch: "master", //"master"
      modules: [],
      userProfilePic: "",
      applications: [],
      rateLimitedAxios: rateLimit(axios.create(), {
        maxRequests: 60,
        perMilliseconds: 60 * 60 * 1000,
      }),
      modulesInUse: [],
      selectedAppName: "None",
      selectedFilePaths: [],
    };
  },
  computed: {
    selectedApp: function () {
      // We will first get the configuration for the selected APP - selectAPP is the one from the drop-down menu.
      const selectedApp = this.applications.find(
        (x) => x.name == this.selectedAppName
      );
      return selectedApp;
    },
    containsDuplicateFiles: function () {
      return this.selectedFilePaths.some((x) => x.count > 1);
    },
  },
  methods: {
    // This is called the very first time the browser loads the APP!
    getItems(that) {
      /* get all applications and modules without loading configs */
      const url =
        "https://api.github.com/repos/" +
        this.repo +
        "/git/trees/" +
        this.branch +
        "?recursive=1";
      this.rateLimitedAxios.get(url).then(function (response) {
        const modules = response.data.tree.filter(function (item) {
          /* look for all ASSEMBLY folders which contain config.json, without loading the config */
          const path = item.path.split("/");
          return (
            path[path.length - 1] == "config.json" &&
            path[path.length - 2].includes("ASSEMBLY")
          );
        });
        const applications = response.data.tree.filter(function (item) {
          /* look for all APPLICATINO  and BOX folders which contain config.json, without loading the config */
          const path = item.path.split("/");
          return (
            path[path.length - 1] == "config.json" &&
            (path[path.length - 2].includes("APP") ||
              path[path.length - 2].includes("BOX"))
          );
        });
        var computeProperties = function (list) {
          /* just computes the name, and adds a skeleton "config" that will get filled out by loading later */
          list.forEach(function (item) {
            const path = item.path.split("/");
            item.name = path[path.length - 2];
            item.config = { description: "", loaded: false };
          });
        };
        computeProperties(modules);
        computeProperties(applications);
        applications.unshift({
          name: "None",
          config: {
            type: "application",
            description: "No Application. Add all modules yourself.",
            modules: [],
          },
        });
        that.modules = modules;
        that.applications = applications;
        that.$emit("loading-done");
      });
    },
    generateID() {
      /*https://gist.github.com/gordonbrander/2230317 */
      return "_" + Math.random().toString(36).substr(2, 9);
    },
    constructModulesInUse() {
      this.modulesInUse = [];
      // Construct the modules we want to use - get those properties from the JSON we want to propagate through the code..
      for (let i = 0; i < this.selectedApp.config.modules.length; i++) {
        let newModule = JSON.parse(
          JSON.stringify({ name: this.selectedApp.config.modules[i].name })
        );
        newModule.key = this.generateID();
        newModule.price = this.selectedApp.config.modules[i].price;
        newModule.applicationSpecific = true;
        newModule.partslist = this.selectedApp.config.modules[i].partslist;
        console.log(
          "created app specific (fixed) module in constructModulesInUse(), pushing to modulesInUse",
          newModule
        );
        this.modulesInUse.push(newModule);
      }
    },
    constructModuleInUse() {
      this.modulesInUse = [];
      // Construct the modules we want to use - get those properties from the JSON we want to propagate through the code..
      // Hacky, but should work for now
      let newModule = JSON.parse(JSON.stringify(this.selectedApp.config));
      newModule.key = this.generateID();
      newModule.price = this.selectedApp.config.config.price;
      newModule.applicationSpecific = false;
      newModule.partslist = this.selectedApp.config.config.partslist;
      console.log(
        "created app specific (fixed) module in constructModulesInUse(), pushing to modulesInUse",
        newModule
      );
      this.modulesInUse.push(JSON.parse(JSON.stringify(newModule)));
    },
    getAppConfig(item) {
      /* we prefer to grab directly from raw.githubusercontent.com as not to use up our rate limits with the api */
      const url = "https://github.com/"+
        this.repo +
        "/raw/" +
        this.branch +
        "/" +
        item.path;
      axios.get(url).then(
        function (response) {
          // here we have all the modules!
          item.config = response.data;
          item.config.loaded = true;
          /* app specific stuff that belongs in the handler: */
          //https://raw.githubusercontent.com/bionanoimaging/UC2-GIT/v3/APPLICATIONS/APP_Abbe_Setup/IMAGES/cover.png
          this.getImage(
            "https://github.com/" +
              this.repo +
              "/raw/" +
              this.branch +
              "/APPLICATIONS/" +
              item.name.split("/")[0] +
              "/IMAGES/cover.jpg"
          );
          this.constructModulesInUse();
        }.bind(this)
      );
    },
    checkAppConfigLoaded() {
      if (this.selectedApp.config.loaded === false) {
        this.getAppConfig(this.selectedApp);
      } else {
        this.constructModulesInUse();
      }
    },
    addModule() {
      let newModule = JSON.parse(JSON.stringify(this.modules[0]));
      newModule.key = this.generateID();
      newModule.applicationSpecific = false;
      newModule.fixedOptions = {}; /* should always be empty for a module created with this function */
      this.modulesInUse.push(newModule);
    },
    updateSelectedModule(mymodule) {
      let oldModuleIndex = this.modulesInUse.findIndex(
        (x) => x.key === module.key
      );
      /* vue wont detect Object.assign since it changes the array at an index: https://vuejs.org/v2/guide/reactivity.html#For-Arrays */
      this.modulesInUse.splice(oldModuleIndex, 1, mymodule);
    },
    updateSTLFileList() {
      this.selectedFilePaths = [];
      console.log("Number of modules in use: ", this.modulesInUse.length);
      for (var modIdx = 0; modIdx < this.modulesInUse.length; modIdx++) {
        let mymodule = this.modulesInUse[modIdx];
        console.log("Adding module:  ", mymodule.name);
        console.log("Adding module:  ", mymodule.partslist);
        // Add parts to the list if they are not none.. TODO: this is wrong behavious!
        if ('partslist' in mymodule) {
          mymodule.partslist.forEach(
            function (part) {
              if (part.is_printable) {
                    for (let i = 0; i < part.n_parts; i++){
                  this.addFileToSTLFileList(part.name);  
                }
                
              }
            }.bind(this)
          );
        }// TODO: Not sure why I have .config.partslist and .patslist
        else if ('partslist' in mymodule.config) {
          mymodule.config.partslist.forEach(
            function (part) {
              if (part.is_printable) {
                    for (let i = 0; i < part.n_parts; i++){
                  this.addFileToSTLFileList(part.name);  
                }
              }
            }.bind(this)
          );
        }
      }
    },
    shouldIncludeFile(file, module) {
      for (let [option, choices] of Object.entries(file.conditions)) {
        if (
          choices.includes(module.config.options[option].selected) === false
        ) {
          return false;
        }
      }
      return true;
    },
    addFileToSTLFileList(fpath) {
      this.filetype = ".stl";
      this.prefix = "UC2_";
      const splitFpath = fpath.split("/");
      const idx = this.selectedFilePaths.findIndex(
        (f) => f.displayName == splitFpath[splitFpath.length - 1]
      );
      if (idx != -1) {
        this.selectedFilePaths[idx].count++;
      } else {
        const path = fpath.split("/");
        this.selectedFilePaths.push({
          path: this.prefix + fpath + this.filetype, //'/' + fpath,
          count: 1,
          displayName: path[path.length - 1],
        });
      }
    },
    getImage(path) {
      this.userProfilePic = path;
    },
    deleteModule(module) {
      this.modulesInUse = this.modulesInUse.filter((x) => x.key !== module.key);
    },
    deleteAll() {
      this.modulesInUse = [];
      this.getImage("")
    },
    downloadZIP() {
      this.idx = 0;
      this.$emit("loading");
      const zip = new JSZip();
      let promises = [];
      this.selectedFilePaths.forEach((file) => {
        const path = file.path.split("/")[file.path.split("/").length - 1];
        const url = "https://raw.githubusercontent.com/" +this.repo +"/" +this.branch +"/CAD/RAW/STL/" +path;
        const count = file.count
        try {
          promises.push(axios
              .get(url, {
                responseType: "blob",
              })
              .then((response) => {console.log(response);
                const content = new Blob([response.data]);
                for(let i=0; i<count; i++){ // have multiple copies per file
                  zip.folder("UC2-Print_all_STLs_once").file(this.idx + "_" + file.displayName + ".stl", content, {binary: true,});
                  this.idx++;
                }
              })
          );
        } catch (e) {
          //end try
          console.log(e);
        } //end catch
      });
      Promise.all(promises).then(() =>
        zip.generateAsync({ type: "blob" }).then((content) => {
          FileSaver.saveAs(content, "UC2-Print_all_STLs_once.zip");
          this.$emit("loading-done");
        })
      );
    },
  },
  created: function () {
    this.getItems(this);
  },
  watch: {
    selectedAppName: function () {
      this.checkAppConfigLoaded();
    },
    modulesInUse: {
      deep: true,
      handler() {
        this.updateSTLFileList();
      },
    },
  },
  components: {
    ModuleConfigurator,
  },
};
</script>

<template>
  <div>
    <div class="row">
      <p style="width: 40px"></p>
      <img src="UC2_Logo-258x300.png" width="86" height="100" />
      <div class="container card-body">
        <h3>
          UC2-Configurator <br /><small class="text-muted">
            Configure and download all required STL files
          </small>
        </h3>
      </div>
    </div>
    <div class="row card-body">
      <div class="col-md-8 p-4">
        <!-- ALL SELECTION ITEMS -->
        <form>
          <div class="form-group">
            <label for="applicationSelect" class="font-weight-bold">
              Select Application or Module
            </label>
            <select
              class="form-control"
              id="applicationSelect"
              v-model="selectedAppName"
            >
              <option
                v-for="application in applications"
                :key="application.name"
              >
                {{ application.name }}
              </option>
            </select>
            <small class="form-text text-muted" v-if="selectedApp">
              {{ selectedApp.config.description }}
            </small>
          </div>
          <hr />
          <p class="font-weight-bold">
            Configure Modules: <br />
            <small class="text-muted">
              Options in gray are fixed and required by the application's
              configuration.
            </small>
          </p>
          <p v-if="userProfilePic">
            <img style="width: 40%" :src="userProfilePic" alt="" />
          </p>
          <p><a href="#" @click="getImage()"></a></p>
          <module-configurator
            v-for="module in modulesInUse"
            :key="module.key"
            v-bind:providedModule="module"
            v-bind:modules="modules"
            v-bind:repo="repo"
            v-bind:branch="branch"
            v-on:update-selected-module="updateSelectedModule($event)"
            v-on:delete-selected-module="deleteModule($event)"
          ></module-configurator>
          <hr />
          <button
            type="button"
            class="btn btn-outline-info"
            v-on:click="addModule"
          >
            Add Module
          </button>
        </form>
      </div>
      <div class="col-md-4 card p-4">
        <!-- A LIST OF INCLUDED FILES -->
        <p class="font-weight-bold">Files Needed:</p>
        <ul class="list-group">
          <li
            class="list-group-item small"
            v-for="item in selectedFilePaths"
            v-bind:key="item.path"
          >
            {{ item.count }}x {{ item.displayName }}
          </li>
        </ul>
        <hr />
        <button type="button" class="btn btn-danger" v-on:click="deleteAll">
          Delete all
        </button>
        <p style="height: 4px"></p>
        <button type="button" class="btn btn-info" v-on:click="downloadZIP">
          Download ZIP
        </button>
        <p class="small text-danger my-4" v-if="containsDuplicateFiles">
          Attention: The ZIP file contains an STL-file 
          for each printed part and can get large!!
        </p>
      </div>
    </div>
  </div>
</template>


<style>
@import "~bootstrap/dist/css/bootstrap.css";
</style>
