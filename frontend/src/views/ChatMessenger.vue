<template>
  <div id="chat-bot">
    <div id="chat-header">
      <h5 id="chat-header-text">Not Your Average Life Coach</h5>
      <div>
        <b-form-select v-model="selected" :options="options"></b-form-select>
      </div>
      <button
        :disabled="userMessages.length === 0 || typingEnabled === false"
        @click="goToChatAnalysisRoute"
        class="button-link"
      >
        Analyze
      </button>
    </div>
    <div class="chat-messages" v-chat-scroll>
      <div class="filler"></div>
      <div v-for="(message, index) in conversation" :key="message.index">
        <Bot :conversation="conversation" :index="index" />
        <User :conversation="conversation" :index="index" />
      </div>
    </div>
    <div class="user-input">
      <input
        :disabled="!typingEnabled"
        ref="textinput"
        class="text-input"
        type="text"
        @keyup.enter="sendMessage"
        v-model="userMessage"
        placeholder="Type a message..."
      />
      <button @click="sendMessage" class="send-message">
        <img id="send-icon" src="../assets/sendIcon.png" />
      </button>
    </div>
  </div>
</template>

<script>
import Vue from "vue";
import { makeHandshake, postMessage, getBotReply } from "@/services/axios.js";
import { FormSelectPlugin } from "bootstrap-vue";
import VueChatScroll from "vue-chat-scroll";
import Bot from "@/components/Bot.vue";
import User from "../components/User.vue";
import translate from "translate";
import wikipedia from "wikipedia";
Vue.use(VueChatScroll);
Vue.use(FormSelectPlugin);

translate.engine = "libre";

export default {
  components: {
    Bot,
    User,
  },
  created() {
    this.nlpHandshake();
    this.initialMessage();
    this.$nextTick(() => {
      this.$refs["textinput"].focus();
    });
  },
  data() {
    return {
      nlpRestToken: "",
      userMessage: "",
      reply: "",
      allConvoData: undefined,
      botMessages: [],
      userMessages: [],
      botMessageCount: -1,
      conversation: [],
      typingEnabled: true,
      error: "",
      selected: "en",
      options: [
        { value: "ar", text: "Arabic" },
        { value: "zh", text: "Chinese" },
        { value: "en", text: "English" },
        { value: "fr", text: "French" },
        { value: "de", text: "German" },
        { value: "hi", text: "Hindi" },
        { value: "it", text: "Italian" },
        { value: "ja", text: "Japanese" },
        { value: "ko", text: "Korean" },
        { value: "pt", text: "Portuguese" },
        { value: "ru", text: "Russain" },
        { value: "es", text: "Spanish" },
      ],
    };
  },

  name: "ChatBot",
  props: {
    msg: String,
  },
  methods: {
    goToChatAnalysisRoute() {
      this.$store.commit("setAllConvoData", this.allConvoData);
      this.$store.commit("setConversation", this.conversation);
      this.$router.push("chat-analysis");
    },
    initialMessage() {
      this.conversation.push({
        chatStyle: "bot",
        text:
          'Hello, I am your Motivational Lifecoach, ask me anything! I can speak in different langauges!! If you want me to wiki something just type the word "wiki <topic>"',
      });
    },
    nlpHandshake() {
      makeHandshake()
        .then((dataId) => {
          this.nlpRestToken = dataId;
        })
        .catch((error) => {
          this.error = "handshake api call is unsuccessful";
        });
    },
    sendMessage() {
      if (this.userMessage != "") {
        this.userMessages.push(this.userMessage);
        this.conversation.push({
          chatStyle: "user",
          text: this.userMessage,
        });
        if (this.userMessage.substring(0, 4) == "wiki") {
          wikipedia
            .page(this.userMessage.replace("wiki ", ""))
            .then((data) => {
              data.summary().then((data) =>
                translate(data.extract, this.selected).then((data) => {
                  this.conversation.push({
                    chatStyle: "bot",
                    text: data,
                  });
                })
              );
            })
            // clears chat message when you done
            .then((this.userMessage = ""))
            .catch((error) => {
              this.conversation.push({
                chatStyle: "bot",
                text: "Sorry dude, can't find that article on wikipedia :/",
              });
            });
        } else if (this.userMessage.substring(0, 16) == "translate this: ") {
          this.userMessage = this.userMessage.substring(15)
          translate(this.userMessage, this.selected).then((data) => {
                  this.conversation.push({
                    chatStyle: "bot",
                    text: data,
                  }); 
                }).then((this.userMessage = ""))
        }else {
          postMessage(this.userMessage, this.nlpRestToken)
            .then(() => {
              this.typingEnabled = false;
              setTimeout(() => {
                this.typingEnabled = true;
                this.$nextTick(() => {
                  this.$refs["textinput"].focus();
                });
                this.getReply();
              }, Math.random() * 1500 + 500);
            })
            .catch((error) => {
              console.log(error);
            })
            .finally(() => {
              this.userMessage = "";
            });
        }
      }
    },
    getReply() {
      getBotReply(this.nlpRestToken)
        .then((response) => {
          this.reply = response.data.activities[(this.botMessageCount += 2)];
          this.allConvoData = response.data;
          this.botMessages.push(this.reply.text);
          translate(this.reply.text, this.selected).then((data) => {
            this.conversation.push({
              chatStyle: "bot",
              text: data,
            });
          });
        })
        .finally(() => {});
    },
    updateMessage(currentMessage) {
      this.userMessage = currentMessage;
    },
  },
};
</script>

<style lang="scss">
@import "@/scss/theme.scss";

@media only screen and (max-width: 600px) {
  #chat-bot {
    width: 100vw !important;
    height: 100vh !important;
    margin: 0 !important;
  }
}

.button-link {
  height: fit-content;
  background-color: #333333;
  border-radius: 8px;
  border-width: 1px;
  border-color: #0389c5;
  text-shadow: 1px 1px 1px #444444;
  color: $light;
  padding: 3px 10px;
  transition-duration: 0.4s;
  text-align: center;
  font-size: 16px;
  margin-left: auto;
  margin-right: 5px;
}

.button-link:hover {
  background-color: #0389c5;
  color: #ffffff;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}

#chat-header-text {
  margin-left: auto;
  margin-right: auto;
}

#chat-header {
  margin: 0em 0em auto 0em;
  background-color: #1b1b1b;
  justify-content: center;
  align-items: center;
  height: fit-content;
  display: flex;
  box-shadow: 0px 2px 5px 0px $neutral;
  z-index: 100;
}
#chat-header h5 {
  color: $light;
  margin: 3% 2%;
}

.send-message {
  background-color: #1b1b1b;
  z-index: 1;
  border: none;
  width: 3em;
}
.user-input {
  background-color: $light;
  border-top: $neutral solid 2px;
  margin: 0em 0em 0em 0em;
  display: flex;
}
.text-input {
  color: white;
  padding: 1em;
  border-right: none;
  border-bottom: none;
  border-left: none;
  border-top: none;
  width: -webkit-fill-available;
  width: -moz-available;
  background-color: #1b1b1b
}

.filler {
  height: -webkit-fill-available;
  height: -moz-available;
  height: inherit;
}

.chat-messages {
  background: #1b1b1b;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  overflow-y: scroll;
  height: -webkit-fill-available;
  height: -moz-available;
  height: 100%;
}

.chat-message {
  color: white;
  display: flex;
  width: 50%;
  padding: 0.2em;
  margin-top: 0.5em;
  margin-bottom: 0.5em;
}
.botMessage {
  background-color: #333333;
  margin-right: auto;
  margin-left: 5px;
  border-radius: 5px;
  box-shadow: 0px 2px 5px $neutral;
  padding-left: 15px;
  padding-top: 15px;
  padding-bottom: 15px;
  padding-right: 15px;
}

.botPic {
  width: 3em;
  height: 3em;
}

.bot-flexbox {
  display: flex;
  justify-content: center;
  align-items: center;
}

.user-flexbox {
  display: flex;
  justify-content: center;
  align-items: center;
}

.userMessage {
  background-color: #0389c5;
  margin-left: auto;
  margin-right: 5px;
  border-radius: 5px;
  margin-bottom: 1em;
  box-shadow: 0px 2px 5px $neutral;
  padding-left: 15px;
  padding-top: 15px;
  padding-bottom: 15px;
  padding-right: 15px;
}
.userPic {
  width: 3em;
  height: 3em;
}

header .filler {
  flex: 0 10000 100%;
}
#send-icon {
  width: 1.75rem;
}

#chat-bot {
  padding: 0px;
  background-color: $light;
  border: #0389c5 solid 4px;
  width: 500px;
  height: 80vh;
  display: flex;
  flex-direction: column;
  margin: 30px auto;
  border-radius: 5px;
}
textarea:focus,
input:focus,
button:focus {
  outline: none;
}
</style>
