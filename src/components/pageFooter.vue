<template>
    <div id="pageFooter">
        <h4>FAQs</h4>
        <h3>What is TypeScholar?</h3>
        <p>It's an app that queries Wikipedia's API for your search term and then you speed-type the result.</p>
        <h3>Why though?</h3>
        <p>I just wanted to make something small with VueJS...</p>
        <h3>Who are you?</h3>
        <p>I'm Peter, a developer from London, UK</p>
        <h3>Anything else?</h3>
        <p>Remember to press enter to load more text once you finish a section!</p>
        <p>If a search term gives an empty result without an error,<br> 
        you might need to capitalise each word in the term (I wanted to fix this,<br>
         and might at some point, but I intended to build this project in one weekend and ran out of time)</p>
        <p>Sometimes passages end in weird text, or normal typable text will be greyed out. Sorry about that.</p>
    </div>
</template>

<script>
// Imports
export default {
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
    font-size: 1.5em;
    font-family: "Monospace";
}
#pageFooter{
  text-align: center;
}
</style>