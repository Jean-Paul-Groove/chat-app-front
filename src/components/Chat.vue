<script setup lang="ts">
import { ref } from 'vue'
import kuzzle from '@/services/kuzzle';
import type { KDocument, KDocumentContent, KDocumentKuzzleInfo, KHit, Notification, SearchResult } from 'kuzzle-sdk';

type Message = {
  id: string,
  message: string,
  pseudo: string,
  createdAt: number,
  type: 'text' | 'alert'
}
interface KDocumentContentChat extends KDocumentContent {
  message: string,
  pseudo: string,
  type: 'text' | 'alert',
  _kuzzle_info: KDocumentKuzzleInfo
}

const connected = ref(false)
const messages = ref<Message[]>([])
const alert = ref()
const pseudo = ref()
const text = ref('')
const roomId = ref('')
function setAlert(message: string) {
  alert.value = "Alerte: " + message
  setTimeout(() => alert.value = undefined, 5000)
}
function addMessage(message: Message) {
  messages.value.push(message)
}
function getMessage(document: KDocument<KDocumentContentChat>) {
  const message: Message = {
    id: document._id,
    message: document._source.message,
    createdAt: document._source._kuzzle_info?.createdAt,
    pseudo: document._source.pseudo,
    type: document._source.type
  }
  return message
}
function validateText() {
  if (text.value.toLowerCase().includes('merde')) {
    throw new Error('IMPOLITE_MESSAGE')
  }
  if(text.value.length>255){
    throw new Error('MESSAGE_TOO_LONG')
  }
}
async function fetchMessages() {
  const results: SearchResult<KHit<KDocumentContentChat>> = await kuzzle.document.search("chat", "chat-messages", { sort: ["_kuzzle_info.createdAt"] }, { size: 100 })
  results.hits.map(hit => {
    addMessage(getMessage(hit))
  })
}
async function postMessage(e: Event) {
  e.preventDefault()
  try {
    validateText()
    if (text.value.length < 1) { return }
    await kuzzle.document.create("chat", "chat-messages", { pseudo: pseudo.value, message: text.value, type: 'text' })

  } catch (error) {
    if (error instanceof Error) {
      switch (error.message) {
        case "IMPOLITE_MESSAGE":
          setAlert('Vous avez été vulgaire !')
          const alertMessage = {
            pseudo: "Police des mots",
            message: pseudo.value + " a été impoli !",
            type: 'alert'
          }
          await kuzzle.document.create("chat", "chat-messages", alertMessage)
          break;
        case "MESSAGE_TOO_LONG":
          setAlert('Votre message ne doit pas dépasser 255 caractères') 
        default: console.log(error)
      }
    } else {
      console.log(error)
    }
  } finally {
    text.value = ''
  }

}
async function subscribeToChat() {
  function handleNotif(notif: Notification) {
    if (notif.type === "document") {
      if (notif.action === "create") {
        const content = notif.result as KDocument<KDocumentContentChat>
        const message = getMessage(content)
        addMessage(message)
      }
    }
  }
  roomId.value = await kuzzle.realtime.subscribe("chat", "chat-messages", {}, handleNotif)
}
async function connect(e: Event) {
  e.preventDefault()
  try {
    await kuzzle.connect()

    if (!await kuzzle.index.exists("chat")) {
      await kuzzle.index.create("chat")
    }
    if (!await kuzzle.collection.exists("chat", "chat-messages")) {
      await kuzzle.collection.create("chat", "chat-messages", {})
    }
    await fetchMessages()
    await subscribeToChat()
    connected.value = true
  } catch (error) {
    console.log(error)
  }

}

</script>

<template>
  <h1>Chat-app</h1>
  <ul class="chat-list">
    <li :class="message.type === 'alert' ? 'chat-message alert' : 'chat-message'" v-for="message in messages"
      :key="message.id">
      <p>De <span class="chat-message_sender">{{ message.pseudo }}</span> - {{ new
      Date(message.createdAt).toLocaleString('fr-FR') }}</p>
      <p>{{ message.message }}</p>
    </li>
  </ul>

  <div class="alert" v-if="alert">{{ alert }}</div>
  <form class="chat-box">
    <div class="chat-box-wrapper" v-if="!connected">
      <label for="pseudo"> Entrez votre nom:</label>
      <input type="text" name="pseudo" id="name" v-model="pseudo" required>
      <button :disabled="!pseudo" v-if="!connected" @click="connect">Connect</button>
    </div>
    <div class="chat-box-wrapper" v-if="connected">
      <label for="text"> Entrez votre message:</label>      
      <textarea v-model="text" name="text" id="text"></textarea required>
        <p :class="text.length>255 ?'chat-box-text-count alert' :'chat-box-text-count'">{{ text.length }} /255</p>
      
      <button  :disabled="!text || text.length>255" @click="postMessage">Envoyer</button>
    </div>
    
  </form>
  
</template>

<style scoped>
.chat-list {
  list-style: none;
  padding: 10px;
  background: rgb(247, 247, 247);
  border-radius: 10px;
  height: 50vh;
  width: 70vw;
  overflow-y: scroll;
}

.chat-message_sender {
  font-weight: bold;
}

.chat-message {
  background-color: white;
  padding: 2px;
  border-radius: 5px;
  margin-bottom: 10px;
}

.alert {
  background-color: rgb(238, 91, 91);
}

.chat-box {
  display: flex;
  flex-direction: column;
  place-items: center;
  justify-content: space-between;
  border: 1px solid lightgray;
  padding: 5px;
}
.chat-box-text-count{
  padding:3px;
  border-radius: 3px;
}
.chat-box-wrapper {
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  width: 100%;
  margin-bottom: 5px;
}
.chat-box-wrapper textarea {
  width: 60%;
}
.chat-box-wrapper button {
  height: fit-content;
  margin: auto 0;
}
</style>
