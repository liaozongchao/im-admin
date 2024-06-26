<template>
    <div>
        <div class="chat-messages" id="chatMessages" ref="chatMessages">
            <div v-for="item in state.historyContentList[state.rkey]" :class="getClass(item)">
                <van-image width="50" height="50" round :src="item.avatar" />
                <div class="message-content">
                    <p v-if="item.msgMedia == 1">{{ getContent(item) }}</p>
                    <p v-if="item.msgMedia == 2">
                        <van-image width="100" height="100" :src="getContent(item)"
                            @click="previewPic(getContent(item))" />
                    </p>
                    <p v-if="item.msgMedia == 3">
                        <audio controls>
                            <source :src="getContent(item)" type="audio/mpeg">
                            <source :src="getContent(item)" type="audio/ogg">
                            <embed height="50" width="100" :src="getContent(item)">
                            Your browser does not support the audio element.
                        </audio>
                    </p>
                    <p v-if="item.msgMedia == 4">
                        <video width="100%" height="100%" controls>
                            <source :src="getContent(item)" type="video/mp4">
                            <source :src="getContent(item)" type="video/ogg">
                            Your browser does not support the video element.
                        </video>
                    </p>
                    <p v-if="item.msgMedia == 5">
                        <van-icon name="description-o" />
                        <a :href="item.content.url" download>{{ item.content.name }}</a>
                    </p>
                    <p v-if="item.msgMedia == 6">
                        <van-image width="40" height="40" round :src="getContent(item)"
                            @click="previewPic(getContent(item))" />
                    </p>
                    <p v-if="item.msgMedia == 10">
                        <van-icon name="phone-o" />{{ getContent(item) }}
                    </p>
                    <p v-if="item.msgMedia == 11">
                        <van-icon name="phone-o" />{{ getContent(item) }}
                    </p>
                    <p v-if="item.msgMedia == 12">
                        <van-icon name="phone-o" />通话时长: {{ formatSeconds(getContent(item)) }}
                    </p>
                    <p v-if="item.msgMedia == 13">
                        <van-icon name="phone-o" />{{ getContent(item) }}
                    </p>
                    <span class="message-time">{{ formatTime(item.createTime, "hh:mm") }}</span>
                    <div class="status-icon read"></div>
                </div>
            </div>
        </div>
        <div class="el-footer">
            <div class="input-container-inner">
                <van-field v-model="state.input" name="inputMsg" @update:model-value="changeInput"
                    placeholder="在输入消息" />
                <span :size="30" @click="showSmile" ref="showSmileRef"><van-icon name="smile-o" :size="30" /></span>
                <span :size="30" @click="showPlugin" v-show="state.isInputEmpty" ref="showPluginRef">
                    <van-icon name="add-o" :size="30" />
                </span>
                <van-icon name="guide-o" :size="30" @click="sendTextMessage" v-show="!state.isInputEmpty" />
            </div>
            <div class="chat-smile" v-if="state.isShowSmile" ref="smileRef">
                <van-image v-for="item in state.emojs" width="30" height="30" :src="item.newurl"
                    @click="sendEmojsMessage(item.url)" />
            </div>
            <div class="chat-plugins" v-if="state.isShowPlugin" ref="pluginRef">
                <div class="plugins-item">
                    <van-uploader accept="image/*" :after-read="handleSuccessImage">
                        <van-icon name="photo-o" class="icon" />
                        相册
                    </van-uploader>
                </div>
                <div class="plugins-item">
                    <van-uploader accept=".pdf,.doc,.docx,.zip,.rar,.7z,.csv" :after-read="handleSuccessFile">
                        <van-icon name="description-o" class="icon" />
                        文件
                    </van-uploader>
                </div>
                <div class="plugins-item">
                    <van-uploader accept="audio/*" :after-read="handleSuccessAudio">
                        <van-icon name="service-o" class="icon" />
                        音频
                    </van-uploader>
                </div>
                <div class="plugins-item">
                    <van-uploader accept="video/*" :after-read="handleSuccessVideo">
                        <van-icon name="video-o" class="icon" />
                        视频
                    </van-uploader>
                </div>
                <div class="plugins-item" v-if="props.talkData.msgType == 1">
                    <van-icon name="phone-o" class="icon" @click="goPhone" />
                    <div>电话</div>
                </div>
            </div>
        </div>
    </div>

</template>

<script setup lang="ts">
import { ref, reactive, onMounted, onUnmounted, watch, nextTick } from 'vue'
import { showImagePreview } from 'vant';
import { formatTime, formatSeconds } from '@/utils/formatTime';
import type { UserInfo, MsgData, Emojs } from '@/utils/schema';
import { Session } from '@/utils/storage';
import { inArray } from '@/utils/common';
import { upload } from '@/api/index';
import { addItem, deleteByMultipleIndexes, getByTimeIndex, getItemById } from '@/utils/indexedDB';
import { getImg, saveMessage } from '@/utils/dbsave';

const props = defineProps(['db', 'socket', 'talkData'])
const emit = defineEmits(['update-parameter-talk', 'update-parameter-talk-phone', 'update-parameter-talk-msg', 'update-parameter-talk-tips'])

const chatMessages = ref<HTMLElement | null>(null);
const pluginRef = ref<HTMLElement | null>(null);
const showPluginRef = ref<HTMLElement | null>(null);

const smileRef = ref<HTMLElement | null>(null);
const showSmileRef = ref<HTMLElement | null>(null);

const state = reactive({
    url: "/api/attach/upload",
    selftUserInfo: {} as UserInfo,
    isShowSmile: false,
    isShowPlugin: false,
    input: "",
    isInputEmpty: true,
    historyContentList: {} as { [key: string]: MsgData[] },
    rkey: "",
    emojs: [] as Emojs[],
});

onMounted(() => {
    console.log(props.talkData)
    // 在页面加载完成后，添加点击事件监听器
    document.addEventListener('click', handleOutsideClick);
    init()
});

onUnmounted(() => {
    document.removeEventListener('click', handleOutsideClick);
});

watch(props, () => {
    console.log("watch", props.talkData)
    init()
})

const handleOutsideClick = (event: any) => {
    if (pluginRef.value && showPluginRef.value) {
        if (!pluginRef.value.contains(event.target) && !showPluginRef.value.contains(event.target)) {
            state.isShowPlugin = false;
        }
    }
    if (smileRef.value && showSmileRef.value) {
        if (!smileRef.value.contains(event.target) && !showSmileRef.value.contains(event.target)) {
            state.isShowSmile = false;
        }
    }
}


const init = async () => {
    state.selftUserInfo = Session.get('userInfo')
    state.emojs = []
    for (let i = 0; i <= 134; i++) {
        const num = String(i).padStart(2, '0');
        const url = `/src/assets/images/emojs/${num}.gif`
        const newurl = await getImg(props.db, url)
        const temp: Emojs = { "url": url, "newurl": newurl }
        state.emojs.push(temp)
    }
    if (props.talkData.msgType && props.talkData.toId) {
        state.rkey = getKey(props.talkData.msgType, state.selftUserInfo.uid, props.talkData.toId)
        setChatList()
    }
};

const changeInput = () => {
    console.log(111, state.input)
    if (state.input == "") {
        state.isInputEmpty = true;
    } else {
        state.isInputEmpty = false;
        state.isShowPlugin = false;
    }
}

const previewPic = (pic: string) => {
    showImagePreview([pic]);
}

const showPlugin = () => {
    state.isShowPlugin = !state.isShowPlugin;
}

const showSmile = () => {
    state.isShowSmile = !state.isShowSmile;
}

const getKey = (msgType: number, fromId: number, toId: number) => {
    let rkey = ""
    if (msgType == 1) {
        rkey = fromId < toId ? `${msgType}_${fromId}_${toId}` : `${msgType}_${toId}_${fromId}`
    } else if (msgType == 2) {
        rkey = `${msgType}_${toId}`
    }
    return rkey
}

const getClass = (item: MsgData) => {
    const classstr = item.fromId === state.selftUserInfo.uid ? 'message user-message' : 'message other-message'
    if (item.msgType == 1 && item.msgMedia == 10) {
        return "message user-message"
    }
    return classstr
}

const setChatList = async () => {
    if (Object.keys(state.historyContentList).length === 0) {
        let nowtime = Math.floor(Date.now() / 1000)
        const temps = await getByTimeIndex(props.db, "message", "createTime", nowtime - 24 * 3600 * 30, nowtime)
        for (let temp of temps) {
            const msgData = temp as unknown as MsgData
            const rkey = getKey(msgData.msgType, msgData.fromId, msgData.toId)
            if (!state.historyContentList.hasOwnProperty(rkey)) {
                state.historyContentList[rkey] = []
            }
            msgData.avatar = await getImg(props.db, msgData.avatar)
            if (inArray(msgData.msgMedia, [2, 3, 4, 5, 6]) && msgData.content.url) {
                msgData.content.url = await getImg(props.db, msgData.content.url)
            }
            state.historyContentList[rkey].push(msgData)
        }
    }
    console.log("setChatList", state.historyContentList)

    setTimeout(setScroll, 1); // 1毫秒后滚动
}


const getContent = (item: MsgData) => {
    if (inArray(item.msgMedia, [1, 10, 11, 12, 13])) {
        return item.content.data
    } else {
        return item.content.url
    }
}

const sendEmojsMessage = (item: string) => {
    const content = { "url": item }
    sendMessage({ content: content, fromId: state.selftUserInfo.uid, toId: props.talkData.toId, msgMedia: 6, msgType: props.talkData.msgType });
    state.isShowSmile = !state.isShowSmile;
}

const sendTextMessage = () => {
    const content = { "data": state.input }
    sendMessage({ content: content, fromId: state.selftUserInfo.uid, toId: props.talkData.toId, msgMedia: 1, msgType: props.talkData.msgType });
    state.input = ""
    changeInput()
}


const handleSuccessImage = (file: any) => {
    const formData = new FormData()
    formData.append('file', file.file)
    upload(formData).then((response: any) => {
        const content = { "url": response.data, "name": file.file.name }
        sendMessage({ content: content, fromId: state.selftUserInfo.uid, toId: props.talkData.toId, msgMedia: 2, msgType: props.talkData.msgType });
        state.isShowPlugin = false
    });
}

const handleSuccessFile = (file: any) => {
    const formData = new FormData()
    formData.append('file', file.file)
    upload(formData).then((response: any) => {
        const content = { "url": response.data, "name": file.file.name }
        sendMessage({ content: content, fromId: state.selftUserInfo.uid, toId: props.talkData.toId, msgMedia: 5, msgType: props.talkData.msgType });
        state.isShowPlugin = false
    });
}

const handleSuccessAudio = (file: any) => {
    const formData = new FormData()
    formData.append('file', file.file)
    upload(formData).then((response: any) => {
        const content = { "url": response.data, "name": file.file.name }
        sendMessage({ content: content, fromId: state.selftUserInfo.uid, toId: props.talkData.toId, msgMedia: 3, msgType: props.talkData.msgType });
        state.isShowPlugin = false
    });
}
const handleSuccessVideo = (file: any) => {
    const formData = new FormData()
    formData.append('file', file.file)
    upload(formData).then((response: any) => {
        const content = { "url": response.data, "name": file.file.name }
        sendMessage({ content: content, fromId: state.selftUserInfo.uid, toId: props.talkData.toId, msgMedia: 4, msgType: props.talkData.msgType });
        state.isShowPlugin = false
    });
}


const sendMessage = async (data: any) => {
    const jsonString = JSON.stringify(data);
    if (!props.socket || props.socket.readyState !== WebSocket.OPEN) {
        emit("update-parameter-talk")
        return
    }
    if (data.msgType == 4) {
        console.log("sendMessage", JSON.parse(jsonString))
    }

    props.socket.send(jsonString);
    if (!inArray(data.msgType, [1, 2])) {
        return
    }

    data.avatar = state.selftUserInfo.avatar
    data.createTime = Math.floor(new Date().getTime() / 1000)

    saveMessage(props.db, data)

    const rkey = getKey(props.talkData.msgType, state.selftUserInfo.uid, props.talkData.toId)
    if (!state.historyContentList.hasOwnProperty(rkey)) {
        state.historyContentList[rkey] = []
    }
    const len = state.historyContentList[rkey].length
    data.avatar = await getImg(props.db, data.avatar)
    if (inArray(data.msgMedia, [6]) && data.content.Url) {
        data.content.Url = await getImg(props.db, data.content.Url)
    }
    state.historyContentList[rkey].push(data)
    emit("update-parameter-talk-msg", data)
    setTimeout(setScroll, 1);  // 1毫秒后滚动
    if (inArray(data.msgMedia, [2, 3, 4, 5]) && data.content.Url) {
        state.historyContentList[rkey][len].content.url = await getImg(props.db, data.content.url)
    }
}
1

const onMessage = async (event: any) => {
    const data = JSON.parse(event.data);
    console.log("onMessage", event.data)

    const userInfo = await getItemById(props.db, "users", data.fromId)
    data.avatar = userInfo.avatar

    saveMessage(props.db, data)

    const rkey = getKey(data.msgType, data.fromId, data.toId)
    if (!state.historyContentList.hasOwnProperty(rkey)) {
        state.historyContentList[rkey] = []
    }
    const len = state.historyContentList[rkey].length

    data.avatar = await getImg(props.db, data.avatar)
    if (inArray(data.msgMedia, [6]) && data.content.Url) {
        data.content.Url = await getImg(props.db, data.content.Url)
    }
    state.historyContentList[rkey].push(data)

    let num: number = 1
    if (data.msgType == 1) {
        if (data.fromId == props.talkData.toId) {
            num = 0
        }
    }
    if (data.msgType == 2) {
        if (data.toId == props.talkData.toId) {
            num = 0
        }
    }
    emit("update-parameter-talk-tips", data.msgType, num)
    setTimeout(setScroll, 1);  // 1毫秒后滚动

    if (inArray(data.msgMedia, [2, 3, 4, 5]) && data.content.Url) {
        state.historyContentList[rkey][len].content.Url = await getImg(props.db, data.content.Url)
    }
}

const clearMessage = (msgType: number, fromId: number, toId: number) => {
    if (!props.db) {
        return
    }
    const rkey = getKey(msgType, fromId, toId)
    state.historyContentList[rkey] = []
    if (msgType == 1) {
        deleteByMultipleIndexes(props.db, 'message', [{ indexName: "msgType", value: msgType }, { indexName: "fromId", value: fromId }, { indexName: "toId", value: toId }])
        deleteByMultipleIndexes(props.db, 'message', [{ indexName: "msgType", value: msgType }, { indexName: "fromId", value: toId }, { indexName: "toId", value: fromId }])
    }
    if (msgType == 2) {
        deleteByMultipleIndexes(props.db, 'message', [{ indexName: "msgType", value: msgType }, { indexName: "toId", value: toId }])
    }
}


const setScroll = () => {
    nextTick();
    if (chatMessages.value) {
        chatMessages.value.scrollTop = chatMessages.value.scrollHeight + 80;
        console.log(chatMessages.value.scrollHeight, chatMessages.value.scrollHeight + 80, chatMessages.value.scrollTop)
    }
}


const goPhone = () => {
    emit("update-parameter-talk-phone", props.talkData.toId)
    state.isShowPlugin = false
}



// 暴露变量
defineExpose({
    onMessage,
    sendMessage,
    clearMessage,
});

</script>
<style scoped>
.el-footer {
    padding: 0;
    position: fixed;
    bottom: 0;
    left: 0;
    width: 100%;
    height: 45px;
    border-top: 1px solid #ccc;
    background-color: #fff;
}

.chat-messages {
    margin-top: 46px;
    height: calc(100vh - 80px);
    overflow-y: auto;
    padding: 10px;
    background-color: #fff;
}

.message {
    display: flex;
    align-items: flex-end;
    margin-bottom: 10px;
    max-width: 99%;
}

.user-message {
    flex-direction: row-reverse;
}

.message .el-avatar {
    margin-bottom: 20px;
}

.message .message-content {
    max-width: 80%;
    padding: 10px;
    border-radius: 10px;
    background-color: #e1ffc7;
}

.other-message .message-content {
    background-color: #fff;
    padding-bottom: 0;
}

.input-container-inner {
    width: 100%;
    display: flex;
    bottom: 0;
    padding: 5px;
    border-top: 1px solid #ccc;
}

.input-container-inner .van-cell {
    padding: 0;
}

.chat-plugins {
    position: absolute;
    bottom: 44px;
    background-color: #fff;
    width: 100%;
    text-align: left;
    border-top: 1px solid #ccc;
    border-bottom: 1px solid #ccc;
}

.plugins-item {
    width: 25%;
    padding: 10px 0;
    text-align: center;
    display: inline-block;
}

.icon {
    font-size: 55px;
    height: 55px;
    width: 80%;
}

.chat-smile {
    position: absolute;
    bottom: 44px;
    background-color: #fff;
    width: 100%;
    display: flex;
    flex-wrap: wrap;
    align-content: flex-start;
    border-top: 1px solid #ccc;
    border-bottom: 1px solid #ccc;
    padding: 10px 0 0 10px;
}

.chat-smile .van-image {
    padding: 3px;
}
</style>