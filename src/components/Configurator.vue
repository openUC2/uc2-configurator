<script>
import axios from 'axios';

 export default {
     name: "Configurator",
     data: function() {
         return {
             repo: "AlecVercruysse/UC2-GIT",
             contents: {}
         }
     },
     methods: {
         getConfig() {
             var that = this;
             const url = "https://api.github.com/repos/" + this.repo + "/contents"
             axios.get(url).then(function (response) {
                 console.log(response.data)
                 that.contents = response.data
                 
             })
         }
     },
     created: function() {
         this.getConfig()
         }
 }
 
</script>

<template>
    <div style="margin: 20px; justify-content:space-around;">
        <div style="display:flex"><h3> Config: </h3></div>
        <div style="display:flex">
            <ul>
                <li v-for="item in contents" :key="item.name"> {{ item.name }} </li>
            </ul>
        </div>
    </div>
</template>


