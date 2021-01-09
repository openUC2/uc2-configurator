<template>
    <div class="fluid-container">
    <div id="app">
        
	<ConfiguratorPanel v-on:loading-done="loading = false" v-on:loading="loading = true"> </ConfiguratorPanel>
    </div>
     <div class="container text-center"><span class="text-muted">
        <a href="https://github.com/beniroquai">@beniroquai</a> &middot; <a href="https://github.com/bionanoimaging/uc2-configurator">Code on Github</a> <br> Coded by Alec Vercruysse and Inspired by Kaspar Emanuel's <a href="https://gitlab.com/openflexure/ofm-stl-selector/">ofm-stl-selector</a>
    </span></div>
    <footer class="page-foooter fixed-bottom">
        <div class="alert alert-primary" role="alert" v-if="loading">
            Fetching data from GitHub...
        </div>
    </footer>
    </div>
</template>

<script>
 import ConfiguratorPanel from "./components/Configurator.vue"
 /*  import HelloWorld from "./components/HelloWorld.vue" */

 export default {
     name: 'App',
     data: function () {
         return {
             loading: true
         }
     },
     components: {
         ConfiguratorPanel
     }
 }
</script>

<style>
 #app {
     font-family: Avenir, Helvetica, Arial, sans-serif;
     -webkit-font-smoothing: antialiased;
     -moz-osx-font-smoothing: grayscale;
     margin-top: 60px;
 }

@import'~bootstrap/dist/css/bootstrap.css'

</style>
