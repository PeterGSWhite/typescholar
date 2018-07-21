<template>
 <div id="typingGame">
    <input v-model="searchTerm"/>
    <button @click="startGame">TEST</button>
    <button @click="nextSection">Next</button>
    <div id="typeArea" tabindex="0" @keydown.prevent="newKeyPress($event)" style="">
        <span v-for="(w, i) in sections[activeSection]" :id="'word-' + activeSection + '-' + i" :class="'index_'+w[1]" :key="activeSection + '-' + i">
            <span v-for="(c, j) in w[0]" :id="'char-' + activeSection + '-' + (w[2] + j)" :key="activeSection + '-' + i + '-' + j">{{ c }}</span>
        </span>
    </div>
    <div id="feedback-display">
      <span id="stopwatch-display">{{ activeStopwatch }} s</span>
      <br>
      <span id="wpm-display">{{ activeWPM }} WPM</span>
      <br>
      <span id="accuracy-display">{{ activeAccuracy }}% Accuracy</span>
    </div>
    {{ errorText }}
    {{ redirectText }}
    {{ text }}
</div>
</template>

<script>
// Imports
import completeSection from "./completeSection.vue";
export default {
  components: {
    completeSection: completeSection,
  },
  data() {
    return {
      searchTerm: "coffeehouse",
      text: "",
      errorText: "",
      redirectText: "",
      sections: [],
      sectionLengths: [],
      //abc: [['a', 'b', 'c'],['d', 'e', 'f']],
      urlBase0: "https://en.wikipedia.org/w/api.php?action=parse&section=",
      urlBase1: "&prop=text&format=json&origin=*&page=",
      maxSectionSize: 250,
      allowedChars:
        'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ 1234567890!?"£$%^&*()\'"[]-+_-\\,./:;@#~{}<>',
      //
      activeSection: 0,
      charIndex: 0,
      activeWPM: 0,
      activeStopwatch: 0,
      activeAccuracy: 0,
      sectionComplete: false,
      accuracyTuple: [0, 0],
    }
  },
  methods: {
    nextSection: function() {
      // spawn element for existing section as history
      this.activeSection++;
      this.charIndex = 0;
      document.getElementById("typeArea").focus();
      this.activeWPM = 0;
      this.activeStopwatch = 0;
      this.accuracyTuple = [0, 0];
      this.sectionComplete = false;
      this.runStopwatch(this.activeSection);
    },
    startGame: async function() {
      let url = this.urlBase0 + "0" + this.urlBase1 + this.searchTerm;
      let body = await this.getSectionBody(url);
      // If redirect, deal get the real body
      let redirectIndex = body.indexOf('redirectText\\"><li><a href=\\"/wiki/');
      if (redirectIndex != -1) {
        this.redirectText = "Redirected From " + this.searchTerm;
        let redirectStringStart = body.substring(redirectIndex + 35);
        let redirectSearchTerm = redirectStringStart.substring(
          0,
          redirectStringStart.indexOf("\\")
        );
        let redurectUrl =
          this.urlBase0 + "0" + this.urlBase1 + redirectSearchTerm;
        body = await this.getSectionBody(redurectUrl);
      }
      // If nothing found then error message and return
      if (
        body.indexOf('"code":"missingtitle"') != -1 ||
        body.indexOf('"title":"Undefined"') != -1
      ) {
        this.errorText = "No Article Found For " + searchTerm;
        return;
      }
      // function to parse useful text out of body
      let article_text = this.getTextFromResponse(body);
      // function to populate sections with body text
      this.splitAndSectionText(article_text);
      document.getElementById("typeArea").focus();
      this.runStopwatch(this.activeSection);
      // call function to populate other sections
      await this.populateExtraSections();
    },

    getSectionBody: function(url) {
      return new Promise(resolve => {
        this.$http.get(url).then(function(data) {
          resolve(data["bodyText"]);
        });
      });
    },

    // function to populate sections
    // call function to parse usable text out of the 0 response
    // call function to split text into more sections for 0 and add to sections
    // call until failure
    // sectionNum ++
    // get response for section
    // parse for failure
    // !!!!! DO SOMETHING FOR REFERENCES NOTES EXTERNALLINKS SEEALSO !!!!!
    // function to parse usable text out of response
    // function to split text into sections
    populateExtraSections: async function() {
      let sectionNum = 1;
      let emergencyBreakN = 100;
      let bodyText = "";
      let url;
      while (
        bodyText.indexOf("nosuchsection") == -1 &&
        sectionNum < emergencyBreakN
      ) {
        url = this.urlBase0 + sectionNum + this.urlBase1 + this.searchTerm;
        if (bodyText) {
          let article_text = this.getTextFromResponse(bodyText);

          // detect and call functions to deal with the end sections here

          this.splitAndSectionText(article_text);
        }
        bodyText = await this.getSectionBody(url);
        sectionNum++;
      }
    },

    getTextFromResponse: function(body) {
      let html = JSON.parse(body)["parse"]["text"]["*"];
      // Lose the tables
      while (html.indexOf("</table>") !== -1) {
        let tabl = html.substring(
          html.indexOf("<table"),
          html.indexOf("</table>") + 8
        );
        let tablIndex = html.indexOf(tabl);
        html =
          html.substring(0, tablIndex) +
          html.substring(tablIndex + tabl.length);
      }
      // Get the article text
      html = html.substring(html.indexOf("<p>"), html.lastIndexOf("</p>"));
      // Get rid of citations
      while (html.indexOf("</sup>") !== -1) {
        let refLink = html.substring(
          html.indexOf("<sup"),
          html.indexOf("</sup>") + 6
        );
        let refLinkIndex = html.indexOf(refLink);
        html =
          html.substring(0, refLinkIndex) +
          html.substring(refLinkIndex + refLink.length);
      }
      // Get rid of tags
      let text = this.strip(html);
      // Get rid of annoying characters (IS THIS SENSIBLE METHOD!?!?!?!?!?!)
      let i = 0;
      while (text.indexOf(String.fromCharCode(160) != -1) && i < 500) {
        text = text.replace(String.fromCharCode(160), " ");
        i += 1;
      }
      text = text.replace(/\s\s+/g, " ");
      text = text.replace(/–/g, "-");
      return text;
    },

    // function to split text into sections of less than max size
    splitAndSectionText: function(inputtext) {
      let outputtext = "";
      let sentences = inputtext.split(". ");
      for (let i in sentences) {
        let sentence = sentences[i];
        sentence = sentence.indexOf(".") == -1 ? sentence + ". " : sentence;
        if (outputtext.length + sentence.length < this.maxSectionSize) {
          outputtext += sentence;
        } else if (!outputtext) {
          // if not outputtext, this means the sentence exceeds the max!
          // Just make this it's own section
          this.addTextToSections(sentence);
        } else {
          outputtext = outputtext.trim();
          this.addTextToSections(outputtext);
          outputtext = "";
        }
      }
      if (outputtext) {
        this.addTextToSections(outputtext);
      }
    },

    // Keep index of valid words, invalid ones get index of -1
    addTextToSections: function(text) {
      let words = text.split(" ");
      let section = [];
      let validIndex = 0;
      let wordStartChar = 0;
      let preSpaceNextWord = false;
      for (let i in words) {
        let word = words[i];
        if (word) {
          if (preSpaceNextWord) {
            word = " " + word;
            preSpaceNextWord = false;
          }
          if (i < words.length - 1) {
            word += " ";
          }
          let re = new RegExp(this.escapeRegex(this.allowedChars));
          if (re.test(word)) {
            section.push([word, validIndex, wordStartChar]);
            wordStartChar += word.length;
            validIndex++;
          } else {
            section.push([word.trim(), -1]);
            preSpaceNextWord = true;
          }
        }
      }
      this
      this.sectionLengths.push(wordStartChar)
      this.sections.push(section);
    },
    // TYPING GAME METHODS
    newKeyPress: function(e) {
      console.log(e);
      // Is the section completed?
      if(this.charIndex >= this.sectionLengths[this.activeSection]) {
        console.log(this.charIndex)
        console.log('section', this.sections[this.activeSection]['words'])
        console.log('completed section')
        this.sectionComplete = true;
      }
      if(this.sectionComplete) {
        if (
          e["key"] == "Enter"
        ) {
          this.nextSection();
        }
      }
      else {
        let lastCharBad = false
        // Start Section if ctrl + enter
        if (e["key"] == "Enter" && e["ctrlKey"] && this.charIndex == 0) {
          console.log("STARTING ");
        }
        let typedChar = e["key"];
        // Move back one char if backspace
        if (typedChar == "Backspace" && this.charIndex > 0) {
          this.charIndex -= 1;
          if(this.currentCharEl.className == 'color-correct') {
            this.accuracyTuple[0] --;
            this.accuracyTuple[1]--;
          }
          this.clearColorClass();
          return;
        }
        // If not an allowed char, do nothing
        if (this.allowedChars.indexOf(typedChar) == -1) {
          return;
        }
        // Any allowed char, check if it's correct and style currentChar to mark progress
        console.log(typedChar);
        console.log(this.currentChar);
        let charIsSpace = false;
        if (typedChar == this.currentChar) {
          this.accuracyTuple[0]++ // Accuracy tuple keeps track of correct and total keystrokes
          this.accuracyTuple[1]++
          this.alterCharColorClass("correct");
          console.log("a", this.currentChar);
          if (this.currentChar == " ") {
            console.log("i fire");
            charIsSpace = true;
          }
        } else {
          this.accuracyTuple[1]++
          lastCharBad = true;
          this.alterCharColorClass("incorrect");
        }
        this.charIndex++;
        console.log("NEW CHAR", this.currentChar);
        // if 2 chars in a row are spaces auto fill the latter (more feedback for skipping illegals)
        if (this.currentChar == " " && charIsSpace) {
          console.log(" fire fhere");
          this.alterCharColorClass("correct");
          this.charIndex++;
        }
      }
    },
    // Style current char
    alterCharColorClass: function(name) {
      let id = this.currentCharId;
      console.log(id);
      console.log(document.getElementById(id));
      document.getElementById(id).className = "color-" + name;
    },
    clearColorClass: function() {
      let id = this.currentCharId;
      console.log(id);
      console.log(document.getElementById(id));
      document.getElementById(id).className = "";
    },
    runStopwatch: function(sectionIndex) {
      if (!this.sectionComplete) {
        let t = setTimeout(() => this.addToStopwatch(sectionIndex), 1000);
      }
    },
    addToStopwatch: function(sectionIndex) {
      this.activeStopwatch ++;
      this.updateWPM();
      this.updateAccuracy();
      this.runStopwatch(sectionIndex);
    },
    updateWPM: function() {
      let parentSpanIdBits = this.currentCharEl.parentElement.id.split('-');
      let wordIndex = parseInt(parentSpanIdBits[parentSpanIdBits.length - 1]);
      let wpm = Math.ceil((60/this.activeStopwatch) * wordIndex);
      this.activeWPM = wpm;
    },
    updateAccuracy: function() {
      let accuracy;
      if(this.accuracyTuple[1]) {
       accuracy  = (100*this.accuracyTuple[0] / this.accuracyTuple[1]).toFixed(2);
      }
      else {
        accuracy = 0;
      }
      this.activeAccuracy = accuracy
    },

    // Misc Methods
    strip: function(html) {
      var doc = new DOMParser().parseFromString(html, "text/html");
      return doc.body.textContent || "";
    },
    escapeRegex: function(str) {
      return (
        "^[" +
        str.replace(/[\-\[\]\/\{\}\(\)\*\+\?\.\\\^\$\|]/g, "\\$&") +
        "]+$"
      );
    }
  },
  computed: {
    currentCharId: function() {
      return "char-" + this.activeSection + "-" + this.charIndex;
    },
    // TYPING COMPUTED PROPERTIES
    currentCharEl: function() {
      return document.getElementById(this.currentCharId);
    },
    currentChar: function() {
      return this.currentCharEl.innerText;
    }
  }
};
</script>

<style>
body {
  margin: 0;
  font-family: "Monospace";
}
#typingGame {
  margin: auto;
  display: flex;
}
#typeArea {
  width: 70%;
  border: 2px solid green;
}
.prevType {
  border: 2px solid green;
}
#typeArea .color-correct {
  background: lightgreen;
}
#typeArea .color-incorrect {
  background: orangered;
}
.index_-1 {
  background: lightslategrey;
}
#feedback-display {
  padding: 1%;
  flex-grow: 1;
}
</style>