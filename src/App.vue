<template>
    <div>
        <type-challenge></type-challenge>
        <!-- <router-view></router-view> -->
    </div>
</template>

<script>
// Imports
import typeChallenge from './components/typeChallenge.vue';
export default {
    components: {
      'type-challenge': typeChallenge
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
    font-family: 'Nunito SemiBold';
}
</style>