
<script>
 /* eslint-disable no-unused-labels */
 import axios from 'axios';
 
 export default {
     name: "ModuleConfigurator",
     data: function() {
         return { 
             selectedModuleName: this.providedModule.name
         }
     },
     props: {
         modules: Array,
         providedModule: Object,
         repo: String
     },
     methods: {
         /* todo move getConfig and checkConfigLoaded to another js file since they basically reuse the same code */
         getConfig(item) {
             /* we prefer to grab directly from raw.githubusercontent.com as not to use up our rate limits with the api */
             const url = "https://raw.githubusercontent.com/" + this.repo + "/master/" + item.path
             axios.get(url).then(function (response) {
                 item.config = response.data
             })
         },
         checkConfigLoaded() {
             console.log("checking if config for module is loaded...")
             console.log(this.selectedModule)
             if (this.selectedModule.config.loaded === false) {
                 console.log("no. loading")
                 this.getConfig(this.selectedModule)
             }
         },
     },
     computed: {
         selectedModule: function () {
             let m = this.modules.find(x => x.name == this.selectedModuleName)
             m.fixedOptions = this.providedModule.fixedOptions
             m.key = this.providedModule.key
             return m
         }
     },
     watch: {
         selectedModuleName: function() {
             this.checkConfigLoaded()
         },
         selectedModule: function() {
             /*              console.log("selectedModule watcher function fired") */
             this.$emit('update-selected-module', this.selectedModule)
         }
     }
 }
</script>
   

<template>
    <div class="container card p-3 my-1">
        <form>
            <div class="form-group">
                <label for="moduleTypeSelect"> Module: </label> <button class="btn btn-outline-danger float-right" type="button" v-on:click="$emit('delete-selected-module', selectedModule)">delete</button>
                <select class="form-control" v-bind:class="{'form-control-plaintext': providedModule.applicationSpecific}" id="moduleTypeSelect" v-model="selectedModuleName" v-bind:disabled="providedModule.applicationSpecific">
                    <option v-for="module in modules" :key="module.name"> {{ module.name }} </option>
                </select>
                <small class="form-text text-muted" v-if="selectedModule"> {{ selectedModule.config.description }} </small>
            </div>
        </form>
    </div>
</template>
