<template>
    <div class="fluid-container">
    <div id="app">
        
	<ConfiguratorPanel v-on:loading-done="loading = false" v-on:loading="loading = true"> </ConfiguratorPanel>
    </div>
     <div class="container text-center"><span class="text-muted">
        Alec Vercruysse &middot; <a href="https://github.com/AlecVercruysse/uc2-configurator">github</a>
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
