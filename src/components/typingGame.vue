<template>
 <div id="typingGame">
    <div id="controls">
      <span v-show="!gameStarted"><h4>Enter a Search term!</h4></span>
      <span v-show="gameStarted && !sectionComplete"><h4>Start Typing!</h4></span>
      <span v-show="gameStarted && sectionComplete"><h4>Press Enter To Load More Text...</h4></span>
      <input v-show="!gameStarted" v-model="searchTerm" @keyup.enter="startGame"/>
      <button v-show="!gameStarted" @click="startGame">START</button>
    </div>
    <div id="messages" v-show="gameStarted">
      <span>{{ errorText }}</span>
      <span><b>{{ articleTitle }}</b></span>
      <span>- {{ redirectText }}</span>
    </div>
    <div id="typeRow" v-show="gameStarted">
      <div id="typeArea" tabindex="0" @keydown.prevent="newKeyPress($event)" style="">
        <span v-for="(w, i) in sections[activeSection]" :id="'word-' + activeSection + '-' + i" :class="'index_'+w[1]" :key="activeSection + '-' + i">
            <span v-for="(c, j) in w[0]" :id="'char-' + activeSection + '-' + (w[2] + j)" :key="activeSection + '-' + i + '-' + j">{{ c }}</span>
        </span>
        <span v-show="!sections">Loading...</span>
      </div>
      <div id="feedback-display">
        <span id="stopwatch-display">Time: {{ activeStopwatch }} s</span>
        <br>
        <span id="wpm-display">Speed: {{ lastWPM ? lastWPM + ' WPM' : '-'}} </span>
        <br>
        <span id="accuracy-display">Accuracy: {{ activeAccuracy }}%</span>
      </div>
    </div> 
    <div id="resetControls" v-show="gameStarted">
        <input v-model="searchTerm" @keyup.enter="resetGame"/>
        <button @click="resetGame">CHOOSE A NEW TOPIC (RESET)</button>
    </div>
</div>
</template>

<script>
// Imports
import completeSection from "./completeSection.vue";
import Vue from 'vue';
export default {
  components: {
    completeSection: completeSection,
  },
  data() {
    return {
      searchTerm: "Fun",
      articleTitle: "",
      gameStarted: false,
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
        'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ 1234567890!?"£$%^&*()\'"[]-+_-=\\,./:;@#~{}<>',
      //
      stopwatch: null,
      activeSection: 0,
      charIndex: 0,
      lastWPM: 0,
      activeWPM: 0,
      activeStopwatchRaw: 0,
      activeStopwatch: 0,
      activeAccuracy: 0,
      sectionComplete: false,
      accuracyTuple: [0, 0],
    }
  },
  methods: {
    resetGame: function() {
      clearInterval(this.stopwatch);
      this.text = ""
      this.errorText =  ""
      this.redirectText = ""
      this.sections = []
      this.sectionLengths = []
      this.activeSection = 0
      this.charIndex = 0
      this.lastWPM = 0
      this.activeWPM = 0
      this.activeStopwatchRaw = 0
      this.activeStopwatch = 0
      this.activeAccuracy = 0
      this.sectionComplete = false
      this.accuracyTuple = [0, 0]
      this.startGame()
    },
    nextSection: async function() {
      // Move to next section
      console.log(this.sections[this.activeSection].length)
      this.lastWPM = Math.floor(60*this.sections[this.activeSection].length/this.activeStopwatchRaw);
      console.log('wpm ' + this.lastWPM)
      this.activeWPM = 0
      this.activeSection++;
      this.charIndex = 0;
      document.getElementById("typeArea").focus();
      this.activeStopwatchRaw = 0;
      this.activeStopwatch = 0;
      this.accuracyTuple = [0, 0];
      this.sectionComplete = false;
      this.runStopwatch();
      await this.populateExtraSections(this.activeSection+2,this.activeSection+2);
    },
    startGame: async function() {
      this.gameStarted = true;
      this.articleTitle = this.searchTerm;
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
        this.articleTitle = redirectSearchTerm;
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
      this.runStopwatch();
      // call function to populate other sections
      await this.populateExtraSections(1,3);
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
    populateExtraSections: async function(start, end) {
      let sectionNum = start;
      let bodyText = "";
      let url;
      while (
        bodyText.indexOf("nosuchsection") == -1 &&
        sectionNum < end
      ) {
        url = this.urlBase0 + sectionNum + this.urlBase1 + this.searchTerm;
        console.log(url)
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
      let n = 0
      while (html.indexOf("</table>") !== -1) {
        if(n > 20) {
          break
        }
        n++
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
      text = text.replace(/—/g, "-");
      
      return text;
    },

    // function to split text into sections of less than max size
    splitAndSectionText: function(inputtext) {
      let outputtext = "";
      let sentences = inputtext.split(".");
      for (let i in sentences) {
        let sentence = sentences[i].trim();
        sentence = sentence + ". " //sentence.indexOf(".") == -1 ? sentence + ". " : sentence;
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
      if(this.charIndex >= this.sectionLengths[this.activeSection] - 1) {
        console.log(this.charIndex)
        console.log('section', this.sections[this.activeSection]['words'])
        console.log('completed section')
        clearInterval(this.stopwatch);
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
            this.accuracyTuple[0]--;
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
    runStopwatch: function() {
      console.log('start timer')
      this.stopwatch = setInterval(this.addToStopwatch, 10);  
    },
    addToStopwatch: function() {
      this.activeStopwatchRaw += 0.01;
      this.activeStopwatch = Math.floor(this.activeStopwatchRaw)
      this.updateAccuracy();
    },
    updateWPM: function() {
      if(this.currentChar) {
        let parentSpanIdBits = this.currentCharEl.parentElement.id.split('-');
        let wordIndex = parseInt(parentSpanIdBits[parentSpanIdBits.length - 1]);
        let wpm = Math.ceil((60/this.activeStopwatch) * wordIndex);
        this.activeWPM = wpm;
      }
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
      if(this.currentCharEl) {
        return this.currentCharEl.innerText;
      }
      else {
        return ''
      }
    }
  }
};
</script>

<style>
body {
  margin: 0;
  font-size: 1.5em;
  font-family: "Monospace";
}
#typingGame {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  align-content: center;
  margin: auto;
}
#typingHistory {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin: auto;
}
#typeRow {
  display: flex;
  flex-direction: row;
  align-items: center;
  margin: auto;
}
#controls {
  display: flex;
  flex-direction: row;
  align-items: center;
  margin: auto;
}
#resetControls {
  display: flex;
  flex-direction: row;
  align-items: center;
  margin: auto;
}
#messages {
  display: flex;
  flex-direction: row;
  align-items: center;
  margin: auto;
}
#messages span {
  margin: 0.1em;
}
#controls input {
  margin: 1em;
}
#typeArea {
  font-size: 1.5em;
  min-width: 30em;
  max-width: 30em;
  border: 2px solid green;
}
#feedback-display {
  padding: 1%;
  flex-grow: 1;
  position: t;
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
</style>