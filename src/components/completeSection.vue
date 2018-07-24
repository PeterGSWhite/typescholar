<template>
    <div id="completeSection">
        <div>
            oi m8
        </div>
        <div id="feedback-display">
            <span id="complete-stopwatch-display">{{ completeStopwatch }} s</span>
            <br>
            <span id="complete-wpm-display">{{ completeWPM }} WPM</span>
            <br>
            <span id="complete-accuracy-display">{{ completeAccuracy }}% Accuracy</span>
        </div> 
    </div>

</template>

<script>
// Imports
export default {
    props: ['html', 'completeStopwatch', 'completeWPM', 'completeAccuracy'],
    components: {
      
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
}
</style>