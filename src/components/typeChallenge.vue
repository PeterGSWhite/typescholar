<template>
    <div id="type-challenge">
        <!-- <form> -->
            <input @keyup.enter.prevent="newSearch($event)" placeholder="Enter a Topic"/>
        <!-- </form> -->
        <p>REDIRECT</p>
        <p><em>{{ redirectText }}</em></p>
        <p>ERROR</p>
        <p><b>{{ errorText }}</b></p>
        <p>TEST</p>
        <p>{{ test }}</p>
    </div>
</template>

<script>
// Imports
export default {
    components: {
      
    },
    data () {
        return {
            rawSearchTerm: '',
            briefHtml: '',
            sections: [],
            furtherReading: {},
            redirectText: '',
            errorText: '',
            test: '',
            emergencyBoolCosImRecursingInGetBriefText: false
        }
    },
    methods: {
        newSearch: function(e) {
            console.log('hi')
            console.log(e.target.value)
            this.rawSearchTerm = e.target.value;
            this.getBriefText();
        },
        getBriefText: function() {
            // Get Json and set Title and Html variables from it
            this.$http.get(this.articleBriefUrl).then(function(data){
                // Get body and deal with bad search term
                this.errorText = ''
                let bodyText = data['bodyText']
                if(bodyText.indexOf('"code":"missingtitle"') != -1){
                    this.errorText = 'No Article Found For ' + this.rawSearchTerm
                    return
                }
                // Deal with Redirects
                let redirectIndex = bodyText.indexOf('redirectText\\"><li><a href=\\"/wiki/')
                console.log('redirect index',redirectIndex)
                if (redirectIndex == -1){
                    console.log('good searr')
                    let html = JSON.parse(bodyText)['parse']['text']['*']
                    this.getSectionText(html)
                }
                else if (!this.emergencyBoolCosImRecursingInGetBriefText) {
                    console.log('REDIRECT')
                    this.redirectText = 'Redirected From ' + this.searchTerm
                    this.emergencyBoolCosImRecursingInGetBriefText = true
                    let redirectStringStart = bodyText.substring(redirectIndex + 35,)
                    this.rawSearchTerm = redirectStringStart.substring(0,redirectStringStart.indexOf('\\'))
                    this.getBriefText()
                }

            })
        },
        // What to do with redirects?
        // What to do with bad search term or failed request?
        getSectionsData: function() {
            this.$http.get(this.articleAllSectionsUrl).then(function(data){
                this.sections = JSON.parse(data['bodyText'])['parse']['sections'];
                // Move further reading to it's own variable
                this.furtherReading = this.sections.pop();
            })
        },
        // What to do with sections?
        getSectionText: function(html) {
            // Strip out all relevant, typable text from a section
            // Get everything that is between the first <p> and last </p>
            html = html.substring(html.indexOf('<p>'), html.lastIndexOf('</p>'))
            // Remove all Reference tags including inner text
            while(html.indexOf('</sup>') !== -1){
                let refLink = html.substring(html.indexOf('<sup'), html.indexOf('</sup>') + 6)
                let refLinkIndex = html.indexOf(refLink)
                console.log(refLink)
                html = html.substring(0, refLinkIndex) + html.substring(refLinkIndex+refLink.length,)
            }

            // Remove remaining tags while preserving inner text
            html = this.strip(html)
            this.test = html
            


        },
        strip: function(html) {
            var doc = new DOMParser().parseFromString(html, 'text/html');
            return doc.body.textContent || "";
        }
    },
    computed: {
        searchTerm: function() {
            // Spaces in the search term always results in redirects
            console.log(this.rawSearchTerm.replace(' ', '').toLowerCase())
            return this.rawSearchTerm.replace(' ', '').toLowerCase()
        },
        articleBriefUrl: function() {
            // API url to get the brief of the article, or the information needed for redirects
            return 'https://en.wikipedia.org/w/api.php?action=parse&section=0&prop=text&format=json&origin=*&page=' + this.searchTerm
        },

        articleAllSectionsUrl: function() {
            // API url to get sections of the article determined by the search term
            return 'https://en.wikipedia.org/w/api.php?action=parse&prop=sections&format=json&origin=*&page=' + this.searchTerm
        },

    }
}
</script>

<style>
body{
    margin: 0;
    font-family: 'Nunito SemiBold';
}
</style>