<template>
    <div class="fluid-container">
    <div id="app">
        
	<Configurator> </Configurator>
    </div>
    <footer class="page-footer"> <div class="container text-center"><span class="text-muted">
        Alec Vercruysse &middot; <a href="https://github.com/AlecVercruysse/uc2-configurator">github</a>
    </span></div></footer>
    </div>
</template>

<script>
 import Configurator from "./components/Configurator.vue"
 /*  import HelloWorld from "./components/HelloWorld.vue" */

 export default {
     name: 'App',
     components: {
         Configurator
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
