<template>
  <v-card class="ma-6 pa-2 d-flex flex-column" height="85vh">
    <v-card-title class="d-flex align-center">
      {{ t('public_chat') }}
      <v-spacer/>
      <v-icon icon="mdi-account-outline" size="small"/>
      {{ ws.publicChatOnline }}
    </v-card-title>
    <v-divider/>
    <v-card-text class="overflow-hidden overflow-y-auto">
      <div ref="el" style="padding-bottom: 56px">
        <v-list>
          <v-list-item
              class="my-1"
              v-for="(i,k) in ws.publicChatMes"
              :key="k"
          >
            <div class="w-100 text-center text-grey text-caption text-font" v-if="i.state">
              {{ i.state === 'join' ? `${i.user}${t('etcr')}` : `${i.user}${t('ltcr')}` }}
            </div>
            <div :class="i.username === user.username ? 'd-flex w-100 flex-row-reverse' : 'd-flex w-100' " v-else>
              <v-avatar>
                <v-img
                  :src="i.avatar"
                  alt="avatar"
                ></v-img>
              </v-avatar>
              <div
                  :class="i.username === user.username ? 'd-flex flex-column mr-2 text-right' : 'd-flex flex-column ml-2' ">
                <span class="font-weight-bold text-primary">{{ i.username }}</span>
                <span style="word-break: break-all">{{ i.message }}</span>
              </div>
            </div>
          </v-list-item>
        </v-list>
      </div>

    </v-card-text>
    <v-card-actions class="bg-surface">
      <v-btn icon="mdi-emoticon-sick-outline" color="primary" class="mr-2" @click="emojiDialog=true"></v-btn>
      <v-text-field color="primary" variant="outlined" density="compact" hide-details v-model="mes" @keyup.enter="sendMes">

      </v-text-field>
      <v-btn @click="sendMes" color="primary" variant="flat" class="ml-2">
        {{ t('send') }}
      </v-btn>
    </v-card-actions>
    <v-dialog v-model="emojiDialog" @click:outside="emojiDialog=false" width="auto">
      <EmojiPicker :native="true" @select="onSelectEmoji" />
    </v-dialog>
  </v-card>
</template>

<script lang="ts" setup>
import {useWs} from "@/store/ws";
import {useUser} from "@/store/user";
import {useSnackBar} from "@/store/snackbar";
import {useI18n} from "vue-i18n";
import {onMounted, onUnmounted, ref, watch} from "vue";
import * as _ from "lodash";

// import picker compopnent
import EmojiPicker from 'vue3-emoji-picker'

// import css
import 'vue3-emoji-picker/css'

const ws = useWs()
const snackbar = useSnackBar()
const {user} = useUser()
const {t} = useI18n()

const mes = ref('')
const el = ref<HTMLDivElement | null>(null)
const chatList = ref([])
const emojiDialog = ref(false)

watch(() => ws.publicChatMes.length, (value, oldValue) => {
  if (value) {
    el.value.scrollIntoView({behavior: "smooth", block: "end", inline: "nearest"})
  }
})

onMounted(() => {
  join()
  el.value.scrollIntoView({behavior: "smooth", block: "end", inline: "nearest"})
})

// event callback
const onSelectEmoji=(emoji:any)=> {
  mes.value+=emoji.i
}

const join = () => {
  ws.setPublicWs('chat/public/')
}
const logout = () => {
  if (ws.publicChatRoom && ws.publicChatRoom.url) {
    ws.publicChatRoom.send(JSON.stringify({"type": "leave", "user": user.id}))
    ws.publicChatRoom.close()
  }
}

const sendMes = _.debounce(() => {
  if (mes.value.trim().length > 0) {
    if (ws.publicChatRoom && ws.publicChatRoom.url) {
      ws.publicChatRoom.send(JSON.stringify({
        "type": "message",
        "message": mes.value,
        "username": user.username,
        "avatar": user.avatar
      }))
    }
  } else {
    snackbar.warning('信息为空')
  }
  mes.value = ''
}, 200)

onUnmounted(() => {
  logout()
})

window.onbeforeunload = function() {
  logout()
};

</script>

<style scoped>
.text-font{
  font-family: 'Maple Mono CN SemiBold', sans-serif !important;
}
</style>
