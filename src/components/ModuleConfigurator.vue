
<script>
 import axios from 'axios';
 import Vue from 'vue'

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
         /*          todo move getConfig and checkConfigLoaded to another js file since they basically reuse the same code */
         getConfig(item) {
             /* we prefer to grab directly from raw.githubusercontent.com as not to use up our rate limits with the api */
             const url = "https://raw.githubusercontent.com/" + this.repo + "/master/" + item.path
             axios.get(url).then(function (response) {
                 item.config = response.data
                 item.config.loaded = true
                 Object.entries(item.config.options).forEach(
                     function([key, value]) {
                         Vue.set(item.config.options[key], 'selected', value.choices[0])
                         /*                          console.log(item.config.options[key], value) */
                     }
                 )
             })
         },
         checkConfigLoaded() {
             if (this.selectedModule.config.loaded === false) {
                 this.getConfig(this.selectedModule)
             }
         },
     },
     computed: {
         selectedModule: function () {
             let m = this.modules.find(x => x.name == this.selectedModuleName)
             m.fixedOptions = this.providedModule.fixedOptions
             m.applicationSpecific = this.providedModule.applicationSpecific
             /* todo consider refactoring to automatically bring any extra parameters from the initial passed object to this new one*/
             m.key = this.providedModule.key
             return m
         }
     },
     watch: {
         selectedModuleName: function() {
             this.checkConfigLoaded()
         },
         selectedModule: {
             deep: true,
             handler() {
                 this.$emit('update-selected-module', this.selectedModule)
             }
         }
     },
     created: function () {
         this.checkConfigLoaded() /* load config of default object on creation */
     }
 }
</script>
   

<template>
    <div class="container card p-3 my-1">
        <form>
            <div class="form-group">
                <label for="moduleTypeSelect"> Module: </label>

                <!-- delete button -->
                <button class="btn btn-outline-danger float-right mb-2" type="button" v-on:click="$emit('delete-selected-module', selectedModule)">delete</button>

                <!-- module select -->
                <select class="form-control" v-bind:class="{'form-control-plaintext': providedModule.applicationSpecific}" id="moduleTypeSelect" v-model="selectedModuleName" v-bind:disabled="providedModule.applicationSpecific">
                    <option v-for="module in modules" :key="module.name"> {{ module.name }} </option>
                </select>
                <small class="form-text text-muted" v-if="selectedModule"> {{ selectedModule.config.description }} </small>
                
                <!-- module-level config options -->
                <div class="container my-2 ml-4 pr-4" v-for="option in selectedModule.config.options" :key="option.displayName">
                    <label for="appSpecificSelect"> {{ option.displayName }}: </label>
                    <select id="appSpecificSelect" class="form-control sm" v-model="option.selected">
                        <option v-for="choice in option.choices" :key="choice" > {{ choice }} </option>
                    </select>
                </div>
            </div>
        </form>
    </div>
</template>

