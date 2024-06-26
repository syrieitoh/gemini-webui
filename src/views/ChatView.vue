<script setup>
import { onMounted, watch, ref, reactive } from 'vue';
import Message from '../components/Message.vue';
import PopupEditor from "../components/popupEditor.vue"
import Gemini from '../components/Gemini.vue';
import Common from '../components/Common.vue';
import { loadHistory,currentHistory } from '../components/History.vue';

const inputText = ref("")
let inputBox
const messages = reactive([])
const tokenUsage = ref(0)

const editedMsg = ref(-1)

const startEditMsg = (i) => {
    const text = messages[i].parts.map((part)=>{ return part.text }).join("").trim()
    messages[i].parts = [{text}]
    editedMsg.value = i
}
const saveEditMsg = (content) => {
    messages[editedMsg.value].parts[0].text = content
}

async function sendMessage(){
    if(inputText.value.trim() != "")
        messages.push({
            role: "user",
            parts: [{text: inputText.value}]
        })
    inputText.value = ""
    setTimeout(()=>{ inputBox.style.height = 'auto';inputBox.style.height = inputBox.scrollHeight + 'px' },30)

    const res = await Gemini.post(messages)
    messages.push(res.msg)
    tokenUsage.value = res.tokenUsage
}

// inputBox根据文本行数自动调整高度
watch(inputText, () => {
    inputBox.style.height = 'auto'
    inputBox.style.height = inputBox.scrollHeight + 'px'
})

onMounted(()=>{
    inputBox = document.getElementById("inputBox")
})

Common.bindEvent("onNewChat", ()=>{ 
    messages.splice(0,messages.length)
    tokenUsage.value = 0 
    currentHistory.value = ""
})
Common.bindEvent("getMsgs", () => { return { messages, tokenUsage:tokenUsage.value } })
Common.bindEvent("onLoadHistory", async (id) => {
    const historyData = await loadHistory(id)
    if(historyData){
        messages.splice(0, messages.length)
        historyData.messages.forEach((msg)=>{messages.push(msg)})
        tokenUsage.value = historyData.tokenUsage
        currentHistory.value = id
    }
})
</script>

<template>

<div id="chatView">
    <div id="chatContents">
        <Message v-if="messages.length > 0" :msg='{role: "model", parts: [{text: "Hello, What can I do for you?"}]}'></Message>
        <Message v-for="msg,i in messages" 
            :msg = "msg" 
            :onDelete = "() => {messages.splice(i,1)}" 
            :onEdit = "() => { startEditMsg(i) }" 
            :key = "i"/>
    </div>

    <div id="messageInput">
        <textarea id="inputBox" v-model="inputText" @keydown.enter.ctrl.prevent="sendMessage" placeholder="Enter Text" rows="1" cols="27"></textarea>
        <div style="display: flex;justify-content: space-between;align-items: flex-end;">
            <p style="font-size: 0.95rem;color: #444444C0">
                token: 
                <b style="color: rgb(202,148,138, 0.6)">{{ tokenUsage }}</b>
                ({{ (tokenUsage/Common.config.maxTokens * 100).toFixed(2) }}%)
            </p>
            <button @click="sendMessage"><img src="/public/icons/send.svg"></button>
        </div>
        
    </div>
    <PopupEditor v-if="editedMsg>=0" 
        title="编辑内容" 
        rows="20"
        :initContent="messages[editedMsg].parts[0].text" 
        :onSave="content => {messages[editedMsg].parts[0].text = content}" 
        :onDismiss="()=>{editedMsg = -1}"/>
</div>
</template>

<style>
#chatView {
    display: flex;
    flex: 1;
    flex-shrink: 0;
    flex-direction: column;
    height: 100%;
    padding: 20px 5px;
    box-sizing: border-box;
    align-items: center;
    background-color: rgb(254,244,238);
}

#messageInput {
    flex-shrink: 1;
    width:80%;
    max-height: 500px;
    box-sizing: border-box;
    text-align: right;
    border-radius: 20px;
    padding:15px 15px 10px 15px;
    background-color: rgb(252,233,218);
    box-shadow: 2px 1px 2px 4px rgb(252,198,188, 0.1);;
}

#inputBox {
    width:100%;
    height: auto;
    max-height: 400px;
    max-lines: 5;
    box-sizing: border-box;
    word-wrap: break-word;
    font-size: 1.1rem;
    overflow-y: auto;
    background: none;
    border: none;
    outline: none;
    resize: none;
}

#chatContents {
    display: block;
    flex-direction: column;
    align-items: center;
    flex: 1;
    width: 100%;
    max-width: 100%;
    padding: 0 10%;
    padding-bottom: 20px;
    overflow-y: auto;
    scroll-behavior: smooth;
    box-sizing: border-box;
}

@media screen and (max-width:500px){
    #messageInput {width: 95%;}
    #chatView {padding-top: 60px;}
    #chatContents { padding-left: 5px; padding-right: 5px;}
}

</style>