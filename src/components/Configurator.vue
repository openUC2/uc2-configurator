<script>
 import axios from 'axios';
 import rateLimit from 'axios-rate-limit';
 
 export default {
     name: "Configurator",
     data: function() {
         return {
             repo: "AlecVercruysse/UC2-GIT",
             modules: [],
             applications: [],
             rateLimitedAxios: rateLimit(axios.create(), { maxRequests: 60, perMilliseconds: 60*60*1000}),
             modulesInUse: [],
             selectedAppName: "None"
         }
     },
     computed: {
         selectedApp: function() {
             const selectedApp = this.applications.find(x => x.name == this.selectedAppName)
             return selectedApp
         },
         selectedFilePaths: function () {
             /* TODO: implement this to read the config and return a list of file paths */
             return ["/CAD/ASSEMBLY_CUBE_TEST/test1", "/CAD/ASSEMBLY_CUBE_TEST/test2", "/CAD/ASSEMBLY_CUBE_TEST/test1"]
         }
     },
     methods: {
         getItems(that) {
             /* get all applications and modules without loading configs */
             const url = "https://api.github.com/repos/" + this.repo + "/git/trees/master?recursive=1"
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
             })
         },
         getConfig(item) {
             /* we prefer to grab directly from raw.githubusercontent.com as not to use up our rate limits with the api */
             const url = "https://raw.githubusercontent.com/" + this.repo + "/master/" + item.path
             axios.get(url).then(function (response) {
                 item.config = response.data
             })
         },
         checkConfigLoaded() {
             console.log("checking if config is loaded...")
             console.log(this.selectedApp)
             if (this.selectedApp.config.loaded === false) {
                 console.log("no. loading")
                 this.getConfig(this.selectedApp)
             }
             
         },
         testreq(url) {
             /* for testing only */
             let resp = {}
             axios.get(url).then(function (response) {
                 resp = response
             })
             return resp
         }
     },
     created: function() {
         this.getItems(this)
     },
     watch: {
         selectedAppName: function() {
             this.checkConfigLoaded()
         }
     }
 }
                 
</script>

<template>
    <div>
        <div class="container card-body"><h3>UC2-Configurator <br><small class="text-muted"> Configure and download all needed STL files </small></h3></div>
        <div class="row card-body">
            <div class="col-md-8 p-4"> <!-- ALL SELECTION ITEMS -->
                <form>
                    <div class="form-group">
                        <label for="applicationSelect"> Select Application </label>
                        <select class="form-control" id="applicationSelect" v-model="selectedAppName">
                            <option v-for="application in applications" :key="application.name"> {{ application.name }}
                            </option>
                        </select>
                        <small class="form-text text-muted" v-if="selectedApp"> {{ selectedApp.config.description }} </small>
                    </div>
                    <hr/>
                    <p> Configure Modules: </p>
                </form>
            </div>
            <div class="col-md-4 card p-4"> <!-- A LIST OF INCLUDED FILES -->
                <p class="font-weight-bold">Files Needed:</p>
                <ul class="list-group">
                    <li class="list-group-item" v-for="item in selectedFilePaths" v-bind:key="item"> {{ item }} </li>
                </ul>
                <button type="button" class="btn btn-primary">Download ZIP</button>
            </div>
        </div>
    </div>
</template>


<style>
 @import'~bootstrap/dist/css/bootstrap.css'
</style>
