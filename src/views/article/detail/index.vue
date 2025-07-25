<template>
  <v-card variant="text" v-if="article" class="pa-2 pa-sm-8">
    <v-card-title>
      <div class="d-flex">
        <v-btn size="60" icon variant="flat" @click="router.push({name:'userinfo',params:{id:article.owner.id}})">
          <v-avatar
              size="60"
          >
            <v-img
                :src="article.owner.avatar"
                :alt="article.owner.username"
            ></v-img>
          </v-avatar>
        </v-btn>
        <div class="d-flex flex-column justify-center ml-4">
          <span>{{ article.owner.username }}</span>
          <div class="text-subtitle-2 text-font text-grey">
            <span>{{ dayjs(article.create_time).fromNow() }}</span><span class="mx-2">|</span>
            <span>{{ t('views') }}:{{ article.views }}</span><span class="mx-2">|</span>
            <span>{{ t('comment') }}:{{ article.comments }}</span>
          </div>
        </div>
      </div>
    </v-card-title>
    <v-card-text class="mt-4">
      <div class="text-h4 text-font font-weight-bold" style="overflow-wrap: break-word">{{ article.title }}</div>
    </v-card-text>
    <div style="padding: 0 1rem 2rem 1rem">
      <v-divider class="my-2" color="surface"/>
      <div v-html="article.content" class="markdown-body detail">

      </div>
      <div class="d-sm-flex align-center mt-6">
        <h3 class="text-grey text-capitalize text-font mr-2">{{ t('tag') }}:</h3>
        <v-chip-group
            active-class="primary--text"
            column
        >
          <v-chip
              v-for="tag in article.tag"
              :key="tag"
              @click="router.push({params: {name:tag.name, type: 'tag'}, name: 'article_search'})"
          >
            {{ tag.name }}
          </v-chip>
        </v-chip-group>
      </div>
      <v-divider class="my-2" color="surface"/>
      <div class="d-flex justify-space-between">
        <div class="d-flex">
          <v-btn variant="text" icon="mdi-comment-outline"></v-btn>
          <span>{{ comments }}</span>
        </div>
        <div class="d-flex">
          <v-btn icon="mdi-circle-edit-outline" variant="text" v-if="user.id === article.owner.id"
                 @click="router.push(`/article/edit/${article.id}`)">
          </v-btn>
          <v-btn icon="mdi-trash-can-outline" variant="text"
                 v-if="article.public&&(user.id === article.owner.id || user.id === 1)" @click="openDel">
          </v-btn>
          <v-btn icon="mdi-thumb-up-outline" variant="text" @click="toggleLike(article.id)"
                 :color="likeFlag&&'primary'"/>
          <span :class="likeFlag&&'text-primary'">{{ likes }}</span>
        </div>
      </div>
    </div>
    <Comment :userId="user.id" :article="article.id" @changeComment="changeComment"/>
  </v-card>
  <v-dialog v-model="delDialog" width="auto">
    <v-card>
      <v-card-title>
        {{ t('del_article') }}
      </v-card-title>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn
            variant="text"
            @click="delDialog = false"
        >
          {{ t('cancel') }}
        </v-btn>
        <v-btn
            variant="text"
            @click="delArticle"
        >
          {{ t('confirm') }}
        </v-btn>
      </v-card-actions>
    </v-card>
  </v-dialog>
</template>

<script setup lang="ts">
import {getCurrentInstance, onMounted, reactive, ref, watch} from "vue";
import {useArticle} from "@/store/article";
import {useI18n} from "vue-i18n";
import {useRouter} from "vue-router";
import {useLike} from "@/store/like";
import Comment from "@/components/Comment"
import {useUser} from "@/store/user";
import {useDisplay, useTheme} from "vuetify";
import {createHighlighter} from 'shiki'

const theme = useTheme();

/**
 * 替换 HTML 字符串中的 <pre><code class="language-xxx">...</code></pre> 为高亮代码
 * @param html 原始 HTML 内容
 */
const highlightCodeInHtml = async (html: string) => {
  const highlighter = await createHighlighter({
    themes: ['github-dark','github-light'],
    langs: ['javascript', 'css', 'typescript', 'json', 'html','java','python','py','c'] // 你可以扩展支持的语言列表
  })

  return html.replace(
    /<pre(?:\s+class="language-(\w+)")?>\s*<code[^>]*?>([\s\S]*?)<\/code>\s*<\/pre>/g,
    (match, lang = 'text', code) => {
      // 反转义 HTML 实体
      const decoded = code
        .replace(/&lt;/g, '<')
        .replace(/&gt;/g, '>')
        .replace(/&amp;/g, '&')

      // 如果传入的lang不支持，默认用 javascript 处理
      const realLang = highlighter.getLoadedLanguages().includes(lang) ? lang : 'javascript'
      return highlighter.codeToHtml(decoded, {
        lang: realLang,
        theme: theme.name.value === 'light' ? 'github-light' : 'github-dark'
      })
    }
  )
}

const {proxy} = getCurrentInstance() as any;
const dayjs = proxy.dayjs
const {t} = useI18n()
const {user} = useUser()
const router = useRouter()
const {smAndDown, xs} = useDisplay()


interface Props {
  id: string | number
}

interface Article {
  comments?: number
  content?: string
  create_time?: string
  id?: number
  likes?: number
  owner?: object
  tag?: []
  title?: string
  update_time?: string
  views?: number
}

const props = defineProps<Props>()

const {get, del} = useArticle()
const {addLike, getLikes} = useLike()
const article = ref(null)

const likes = ref(0)
const likeFlag = ref(false)

const comments = ref(0)

const img = ref({
      width: '',
      height: ''
    }
)

const delDialog = ref(false)
watch(() => props.id, async (value, oldValue) => {
  if (value) {
    await init(value)
  }
})
watch(smAndDown, value => {
  if (value) {
    img.value = {
      width: '100%',
      height: '100%'
    }
  } else {
    img.value = {
      width: '',
      height: ''
    }
  }
}, {
  immediate: true
})

onMounted(async () => {
  await init(props.id)
})

const init = async (id: number | string) => {
  article.value = await get(id)
  article.value.content = await  highlightCodeInHtml(
    article.value.content
  )
  likes.value = article.value.likes
  comments.value = article.value.comments
  likeFlag.value = await isLike(user.id)
}

const isLike = async (userId: any) => {
  const {results} = await getLikes({article: props.id, pagesize: 1000})
  if (results) {
    const val = results.map((item: { user_info: { id: any; }; }) => item.user_info.id)
    return val.includes(userId)
  }
  return false
}

const toggleLike = async (id: number) => {
  const res = await addLike({article: id})
  if (res) {
    likes.value += 1
    likeFlag.value = true
  }
}

const changeComment = (num: number) => {
  comments.value = num
}

const openDel = () => {
  delDialog.value = true
}

const delArticle = async () => {
  await del(article.value.id)
  delDialog.value = false
  router.push('/articles')
}

</script>

<style>
.detail :deep(img) {
  width: v-bind('img.width');
  height: v-bind('img.height');
}
.markdown-body {
  font-family: Arial, sans-serif;
  line-height: 1.6;
}

.markdown-body h1,h2,h3,h4,h5,h6 {
  margin: 1em 0;
}

.markdown-body p {
  margin: 1em 0;
}

.markdown-body ul,
.markdown-body ol {
  padding-left: 2em;
  margin: 1em 0;
}

.markdown-body li {
  margin: 0.5em 0;
}

.markdown-body blockquote {
  margin: 1em 0;
  padding: 0.5em 1em;
  color: #555;
  border-left: 4px solid #eaecef;
}

.markdown-body table {
  width: 100%;
  border-collapse: collapse;
  margin: 1em 0;
}

.markdown-body th,
.markdown-body td {
  border: 1px solid #ddd;
  padding: 0.5em 1em;
  text-align: left;
}

.markdown-body th {

}

.markdown-body img {
  max-width: 100%;
  height: auto;
  display: block;
  margin: 1em 0;
}

.markdown-body pre {
  padding: 1em;
  border-radius: 5px;
  overflow: auto;
}

.markdown-body pre code {
  background: none;
  padding: 0;
  border-radius: 0;
}

.markdown-body code {
  padding: 0.2em 0.4em;
  border-radius: 3px;
}

.markdown-body a {
  text-decoration: none;
}

.markdown-body a:hover {
  text-decoration: underline;
}

.text-font{
  font-family: 'Maple Mono CN SemiBold', sans-serif !important;
}

</style>
