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
        <div id="typeArea" tabindex="0" @keydown="newKeyPress($event)" style="">
            <span v-for="(c, i) in currentText" :id="'char-' + i" :key="searchNum + '-' + i">{{ c }}</span>
        </div>
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
            currentText: '',
            emergencyBoolCosImRecursingInGetBriefText: false,
            allowedChars: 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ 1234567890!"£$%^&*()\'"[]-+_-\\,./',
            charIndex: 0,
            searchNum: 0,
        }
    },
    methods: {
        // API METHODS
        newSearch: function(e) {
            console.log('hi')
            console.log(e.target.value)
            this.redirectText = ''
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
            // Remove all tables because they cause trouble (!!if this happens again I make function for arbitrary tag!!)
            while(html.indexOf('</table>') !== -1){
                let tabl = html.substring(html.indexOf('<table'), html.indexOf('</table>') + 8);
                console.log(tabl);
                let tablIndex = html.indexOf(tabl);
                html = html.substring(0, tablIndex) + html.substring(tablIndex+tabl.length,);
            }
            html = html.substring(html.indexOf('<p>'), html.lastIndexOf('</p>'));
            // Remove all Reference tags including inner text
            while(html.indexOf('</sup>') !== -1){
                let refLink = html.substring(html.indexOf('<sup'), html.indexOf('</sup>') + 6);
                let refLinkIndex = html.indexOf(refLink);
                console.log(refLink);
                html = html.substring(0, refLinkIndex) + html.substring(refLinkIndex+refLink.length,);
            }

            // Remove remaining tags while preserving inner text
            html = this.strip(html);
            this.currentText = html;
            this.charIndex = 0;
            this.searchNum += 1;
        },
        strip: function(html) {
            var doc = new DOMParser().parseFromString(html, 'text/html');
            return doc.body.textContent || "";
        },

        // TYPING METHODS
        newKeyPress: function(e) {
            let typedChar = e['key']
            // Move back one char if backspace
            if(typedChar == 'Backspace' && this.charIndex > 0) {
                this.charIndex -= 1
                this.clearColorClass();
                return
            }
            // If not an allowed char, do nothing
            if(this.allowedChars.indexOf(typedChar) == -1) {
                return
            }
            // Any allowed char, check if it's correct and style currentChar to mark progress
            console.log(typedChar)
            console.log(this.currentChar)
            if(typedChar == this.currentChar) {
                this.alterCharColorClass('correct');
            }
            else {
                this.alterCharColorClass('incorrect');
            }
            this.charIndex += 1

        },
        // Skip untypable word method

        // Style current char
        alterCharColorClass: function(name) {
            let id = this.currentCharId;
            console.log(id)
            console.log(document.getElementById(id))
            document.getElementById(id).className = 'color-' + name;
        },
        clearColorClass: function() {
            let id = this.currentCharId;
            console.log(id)
            console.log(document.getElementById(id))
            document.getElementById(id).className = ''
        },
    },
    computed: {
        // API COMPUTED PROPERTIES
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
        
        currentCharId: function() {
            return 'char-' + this.charIndex
        },
        // TYPING COMPUTED PROPERTIES
        currentChar: function() {
            return document.getElementById(this.currentCharId).innerText
        },

    }
}
</script>

<style scoped>
body{
    margin: 0;
    font-family: 'Nunito SemiBold';
}
#type-challenge {
    margin: auto;
    width: 70%;
    padding: 10px;
}
#typeArea {
    border: 3px solid green;
}
#typeArea .color-correct {
    background: lightgreen;
}
#typeArea .color-incorrect {
    background: orangered;
}
#typeArea .color-invalid {
    background: lightslategrey;
}

</style>