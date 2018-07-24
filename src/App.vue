<template>
    <div>
        <page-header></page-header>
        <typing-game></typing-game>
        <page-footer></page-footer>
        <!-- <router-view></router-view> -->
    </div>
</template>

<script>
// Imports
import typingGame from './components/typingGame.vue';
import pageHeader from './components/pageHeader.vue';
import pageFooter from './components/pageFooter.vue';
export default {
    components: {
      'page-footer': pageFooter, 
      'typing-game': typingGame,
      'page-header': pageHeader,
    },
    data () {
        return {
        }
    },
    methods: {
    }
}
</script>

<style>
body{
    margin: 0;
    font-family: 'Monospace';
}
</style>