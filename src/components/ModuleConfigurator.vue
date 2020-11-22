
<script>
 import axios from 'axios';
 import Vue from 'vue'

 export default {
     name: "ModuleConfigurator",
     data: function() {
         return {
             selectedModuleName: this.providedModule.name,
             selectedModule: {}
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
                 response.data.loaded = true
                 item.config = response.data
                 Object.entries(item.config.options).forEach(
                     function([key, value]) {
                            Vue.set(item.config.options[key], 'selected', value.choices[0])
                           item.config.options[key].fixed = false
                     }
                 )
                 /* update our master modules list for future instances of this object */
                 this.modules.find(x => x.name == this.selectedModuleName).config = response.data
             }.bind(this))
         },
         checkConfigLoaded() {
             /*              console.log("checking if config is loaded. selectedModule:", this.selectedModule) */
             if (this.selectedModule.config.loaded === false) {
                 this.getConfig(this.selectedModule)
             }
         },
         getSelectedModule: function () {
             /*              console.log("getting selected module") */
             let m = JSON.parse(JSON.stringify(this.modules.find(x => x.name == this.selectedModuleName)))
             m.applicationSpecific = this.providedModule.applicationSpecific
             /* todo consider refactoring to automatically bring any extra parameters from the initial passed object to this new one*/
             m.key = this.providedModule.key
             this.selectedModule = m
         },
     },
     watch: {
         selectedModuleName: function() {
             /*              console.log('selectedModuleName changed. in watch handler:') */
             this.getSelectedModule()
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
         this.getSelectedModule()
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
                <select class="form-control" v-bind:class="{'form-control-plaintext custom-disabled': providedModule.applicationSpecific}" id="moduleTypeSelect" v-model="selectedModuleName" v-bind:disabled="providedModule.applicationSpecific">
                    <option v-for="module in modules" :key="module.name"> {{ module.name }} </option>
                </select>
                <small class="form-text text-muted" v-if="selectedModule"> {{ selectedModule.config.description }} </small>

            </div>
        </form>
    </div>
</template>

<style scoped>
 .custom-disabled {
     background-color: #dfdfdf !important;
     opacity: 0.7 !important;
     -moz-appearance: none !important;
     -webkit-appearance: none !important;
     appearance: none !important;

 }

</style>
