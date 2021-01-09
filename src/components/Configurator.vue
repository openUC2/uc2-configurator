<script>
 import axios from 'axios';
 import rateLimit from 'axios-rate-limit';
 import JSZip from 'jszip';
 import Promise from 'promise';
 import FileSaver from 'file-saver';
 import ModuleConfigurator from "./ModuleConfigurator.vue"

 export default {
     name: "ConfiguratorPanel",
     data: function() {
         return {
             repo: "bionanoimaging/UC2-GIT",// "bionanoimaging/UC2-GIT", //
             branch: "v3", //"master"
             modules: [],
             applications: [],
             rateLimitedAxios: rateLimit(axios.create(), { maxRequests: 60, perMilliseconds: 60*60*1000}),
             modulesInUse: [],
             selectedAppName: "None",
             selectedFilePaths: []
         }
     },
     computed: {
         selectedApp: function() {
           // We will first get the configuration for the selected APP - selectAPP is the one from the drop-down menu.
             const selectedApp = this.applications.find(x => x.name == this.selectedAppName)
             return selectedApp
         },
         containsDuplicateFiles: function() {
             return this.selectedFilePaths.some(x => x.count > 1)
         }
     },
     methods: {
       // This is called the very first time the browser loads the APP!
         getItems(that) {
             /* get all applications and modules without loading configs */
             const url = "https://api.github.com/repos/" + this.repo + "/git/trees/" + this.branch + "?recursive=1"
             this.rateLimitedAxios.get(url).then(function (response) {
                 const modules = response.data.tree.filter(function (item) {
                     /* look for all ASSEMBLY folders which contain config.json, without loading the config */
                     const path = item.path.split('/')
                     return (path[path.length - 1] == "config.json" && path[path.length - 2].includes("ASSEMBLY"))
                 })
                 const applications = response.data.tree.filter(function (item) {
                     /* look for all ASSEMBLY folders which contain config.json, without loading the config */
                     const path = item.path.split('/')
                     return (path[path.length - 1] == "config.json" && path[path.length - 2].includes("APP"))
                 })
                 var computeProperties = function (list) {
                     /* just computes the name, and adds a skeleton "config" that will get filled out by loading later */
                     list.forEach(function (item) {
                         const path = item.path.split('/')
                         item.name = path[path.length - 2]
                         item.config = { description:"",
                                         loaded: false
                         }
                     })
                 }
                 computeProperties(modules)
                 computeProperties(applications)
                 applications.unshift({
                     name: "None",
                     config: {
                         type: "application",
                         description: "No Application. Add all modules yourself.",
                         modules: []
                     }
                 })
                 that.modules = modules
                 that.applications = applications
                 that.$emit('loading-done')
             })
         },
         generateID() {
             /*https://gist.github.com/gordonbrander/2230317 */
             return '_' + Math.random().toString(36).substr(2, 9);
         },
         constructModulesInUse() {
           // Construct the modules we want to use - get those properties from the JSON we want to propagate through the code..
             this.modulesInUse = []
             for(let i = 0; i < this.selectedApp.config.modules.length; i++) {
                 let newModule = JSON.parse(JSON.stringify({name: this.selectedApp.config.modules[i].name}))
                 newModule.key = this.generateID()
                 newModule.price = this.selectedApp.config.modules[i].price
                 newModule.applicationSpecific = true
                 newModule.partslist = this.selectedApp.config.modules[i].partslist
                 console.log("created app specific (fixed) module in constructModulesInUse(), pushing to modulesInUse", newModule)
                 this.modulesInUse.push(newModule)
             }
         },
         getAppConfig(item) {
             /* we prefer to grab directly from raw.githubusercontent.com as not to use up our rate limits with the api */
             const url = "https://raw.githubusercontent.com/" + this.repo + "/" + this.branch + "/" + item.path
             axios.get(url).then(function (response) { // here we have all the modules!
                 item.config = response.data
                 item.config.loaded = true
                 /* app specific stuff that belongs in the handler: */
                 this.constructModulesInUse()
             }.bind(this))
         },
         checkAppConfigLoaded() {
             if (this.selectedApp.config.loaded === false) {
                 this.getAppConfig(this.selectedApp)
             } else {
                 this.constructModulesInUse()
             }
         },
         addModule() {
             let newModule = JSON.parse(JSON.stringify(this.modules[0]))
             newModule.key = this.generateID()
             newModule.applicationSpecific = false
             //newModule.price = this.selectedApp.config.modules[i].price
             newModule.applicationSpecific = true
             //newModule.partslist = this.selectedApp.config.modules[i].partslist
             this.modulesInUse.push(newModule)
         },
         updateSelectedModule(mymodule) {
             let oldModuleIndex = this.modulesInUse.findIndex(x => x.key === module.key)
             /* vue wont detect Object.assign since it changes the array at an index: https://vuejs.org/v2/guide/reactivity.html#For-Arrays */
             this.modulesInUse.splice(oldModuleIndex, 1, mymodule)
         },
         updateSTLFileList() {
             this.selectedFilePaths = []
             console.log("Number of modules in use: ", this.modulesInUse.length)
             for(var modIdx = 0; modIdx < this.modulesInUse.length; modIdx++) {
                 const module = this.modulesInUse[modIdx]
                 console.log("Adding module:  ", module)
                   /* now load in dynamic files */
                  module.partslist.forEach(function (part) {
                  console.log(part.githublink)
                     if (part.is_printable) {
                         this.addFileToSTLFileList(part.name)
                     }
                  }.bind(this))
             }
         },
         shouldIncludeFile(file, module) {
             for (let [option, choices] of Object.entries(file.conditions)) {
                 if (choices.includes(module.config.options[option].selected) === false) {
                     return false
                 }
             }
             return true
         },
         addFileToSTLFileList(fname) {
            this.filetype = '.stl'
            this.prefix = 'UC2_'             
            this.selectedFilePaths.push({
                path: this.prefix+fname+this.filetype,
                count: 1,
                displayName: fname
            })
         },
         deleteModule(module) {
             this.modulesInUse = this.modulesInUse.filter(x => x.key !== module.key)
         },
         downloadZIP() {
             this.idx=0
             this.$emit('loading')
             const zip = new JSZip()
             let promises = []
             this.selectedFilePaths.forEach(file => {
                 const path = file.path.split('/')[file.path.split('/').length-1]
                 const url = "https://raw.githubusercontent.com/" + this.repo + "/"+ this.branch +"/CAD/RAW/STL/" + path
                 try {
                   promises.push(axios.get(url, {
                   responseType: 'blob'
                 }
                 ).then(response => {
                     console.log(response)
                     const content = new Blob([response.data])
                     zip.folder("UC2-Print_all_STLs_once").file(this.idx+"_"+file.displayName+".stl", content, {binary: true})
                     this.idx++
                 }))
                  }//end try
               catch (e) {
                 console.log(e)
               }//end catch
             })
             Promise.all(promises).then(() => zip.generateAsync({type:"blob"}).then(content => {
                 FileSaver.saveAs(content, "UC2-STL.zip")
                 this.$emit('loading-done')
             }))
         }
     },
     created: function() {
         this.getItems(this)
     },
     watch: {
         selectedAppName: function() {
             this.checkAppConfigLoaded()
         },
         modulesInUse: {
             deep: true,
             handler() {
                 this.updateSTLFileList()
             }
         }
     },
     components: {
         ModuleConfigurator
     }
 }

</script>

<template>
    <div>
        <div class="container card-body"><h3>UC2-Configurator <br><small class="text-muted"> Configure and download all required STL files </small></h3></div>
        <div class="row card-body">
            <div class="col-md-8 p-4"> <!-- ALL SELECTION ITEMS -->
                <form>
                    <div class="form-group">
                        <label for="applicationSelect" class="font-weight-bold"> Select Application </label>
                        <select class="form-control" id="applicationSelect" v-model="selectedAppName">
                            <option v-for="application in applications" :key="application.name"> {{ application.name }}
                            </option>
                        </select>
                        <small class="form-text text-muted" v-if="selectedApp"> {{ selectedApp.config.description }} </small>
                    </div>
                    <hr/>
                    <p class="font-weight-bold"> Configure Modules: <br>
                        <small class="text-muted"> Options in gray are fixed and required by the application's configuration. </small>
                    </p>
                    <module-configurator v-for="module in modulesInUse" :key="module.key" v-bind:providedModule="module" v-bind:modules="modules" v-bind:repo="repo" v-bind:branch="branch" v-on:update-selected-module="updateSelectedModule($event)" v-on:delete-selected-module="deleteModule($event)"></module-configurator>
                    <hr>
                    <button type="button" class="btn btn-outline-primary"  v-on:click="addModule">Add Module</button>
                </form>
            </div>
            <div class="col-md-4 card p-4"> <!-- A LIST OF INCLUDED FILES -->
                <p class="font-weight-bold">Files Needed:</p>
                <ul class="list-group">
                    <li class="list-group-item small" v-for="item in selectedFilePaths" v-bind:key="item.path"> {{ item.count }}x {{ item.displayName }} </li>
                </ul>
                <hr>

                <button type="button" class="btn btn-primary" v-on:click="downloadZIP">Download ZIP</button>
                <p class="small text-danger my-4" v-if="containsDuplicateFiles">Warning: Your configure requires multiple copies of a file. The ZIP comes with a single copy of each. Please remember to print all files! </p>
            </div>
        </div>
    </div>
</template>


<style>
 @import'~bootstrap/dist/css/bootstrap.css'
</style>
