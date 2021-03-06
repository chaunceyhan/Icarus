<template>
<loading v-if="loading"/>
<redirecting v-else-if="!canEditWiki()" class="ic-container box">
    <span>抱歉，你的账号没有权限访问这个页面</span>
</redirecting>
<div v-else class="ic-container">
    <div class="edit-page-title">
        <div v-title>{{ isEdit ? '编辑文章' : '添加文章' }} - {{state.config.title}}</div>
        <h3 class="" v-if="!isEdit">添加文章</h3>
        <h3 class="" v-else>编辑文章<span v-if="asAdmin"> - 管理员模式</span></h3>
        <button class="ic-btn primary right-top-btn" type="primary" :loading="loading" @click="send">{{postButtonText}}</button>
    </div>

    <form class="ic-form" id="form_topic" method="POST" @submit.prevent="send">
        <check-row :results="formErrors.title" :multi="true">
            <input type="text" name="title" v-model="wikiInfo.title" :placeholder="`这里填写标题，${state.misc.BACKEND_CONFIG.TOPIC_TITLE_LENGTH_MIN} - ${state.misc.BACKEND_CONFIG.TOPIC_TITLE_LENGTH_MAX} 字`">
        </check-row>

        <check-row :results="formErrors.ref" :multi="true" v-if="!save.flag">
            <input type="text" v-model="wikiInfo.ref" :placeholder="`这里是这篇文章的URL地址，若为abc，地址将为/wiki/r/abc，不填则与标题相同`">
        </check-row>

        <check-row :results="formErrors.content" :multi="true">
            <markdown-editor ref="editor" v-model="wikiInfo.content" rows="15" autofocus></markdown-editor>
        </check-row>
        <div class="ic-form-row">
            <button class="ic-btn primary" style="float: right" type="primary" :loading="loading">{{postButtonText}}</button>
        </div>
    </form>
</div>
</template>

<style lang="scss" scoped>
#form_topic input[type='text'] {
    padding: 6px 12px;
    width: 100%;
    font-weight: bolder;
    border-radius: 5px;
    box-shadow: none;
    border: 1px solid #e8e8e8;
    outline: none;
}

.error {
    #form_topic input[name='title'] {
        border: 1px solid $danger;
    }
}

.edit-page-title {
    display: flex;
    padding: 20px 0;
    padding-top: 0;
    justify-content: space-between;
    align-items: center;
}

.ic-form-row {
    margin-bottom: 10px;
}

.form-select-row {
    /* 此举是为了避免 ic-form 对 input 的样式覆盖。
        不然select会出现外框套内框这样的事情 */
    margin-bottom: 10px;
}

.el-select > .el-input > input[readonly] {
    background-color: #fff;
    color: #1f2d3d;
}
</style>

<style>
div.markdown-editor > div.editor-toolbar {
    border-color: #ddd !important;
}

/* 这部分内容是动态生成的 因此写在scope中无效 */
.ic-form > .ic-form-row.error > div.markdown-editor > div.editor-toolbar {
    opacity: 1; /* 上下颜色一致 */
    border-color: #FF6060 !important;
}

.ic-form > .ic-form-row.error > .markdown-editor > .cm-s-easymde {
    border-left-color: #FF6060 !important;
    border-right-color: #FF6060 !important;
    border-bottom-color: #FF6060 !important;
}

.editor-preview img {
    width: 100%;
}

.multiselect__tags {
    border: 1px solid #ddd !important;
}
</style>

<script>
import markdownEditor from '@/components/misc/markdown-editor.vue'
import api from '@/netapi.js'
import state from '@/state.js'
import nprogress from 'nprogress/nprogress.js'
import * as qiniu from 'qiniu-js'
import Objectid from 'objectid-js'
// import marked from '@/md.js'
import './../forum/topic-edit-fa.js'

export default {
    data () {
        return {
            state,
            asAdmin: false,
            loading: true,
            working: false,
            save: {},

            wikiInfo: {
                title: '',
                ref: '',
                content: ''
            },

            formErrors: {
                title: [],
                content: []
            }
        }
    },
    computed: {
        isEdit () {
            return this.$route.name === 'wiki_article_edit'
        },
        action: function () {
            if (this.$route.name === 'wiki_article_edit') return 'edit'
            if (this.$route.name === 'wiki_article_fork') return 'fork'
            if (this.$route.name === 'wiki_article_new') return 'new'
        },
        postButtonText: function () {
            return this.working ? '请等待'
                : (this.isEdit ? '编辑' : '发布')
        }
    },
    methods: {
        canEditWiki: $.canEditWiki,
        send: async function (e) {
            let ret
            let wikiId
            let successText

            if (this.working) return
            this.working = true

            let wikiInfo = $.objDiff(this.wikiInfo, this.save)
            if (Object.keys(wikiInfo).length <= 0) {
                $.message_success('编辑成功！但编辑者并未进行任何改动。')
                if (this.save.flag) {
                    this.$router.push({ name: 'wiki' })
                } else {
                    this.$router.push({ name: 'wiki_article_by_id', params: { id: this.save.id } })
                }
                this.working = false
                return
            }

            let role = this.asAdmin ? 'superuser' : 'user'
            wikiInfo.returning = true

            if (this.isEdit) {
                ret = await api.wiki.set({ id: this.wikiInfo.id }, wikiInfo, role)
            } else {
                if (this.action === 'fork') {
                    wikiInfo['root_id'] = this.wikiInfo.root_id
                }
                // return
                ret = await api.wiki.new(wikiInfo, 'superuser')
            }
            successText = '编辑成功！已自动跳转至文章页面。'

            if (ret.code === 0) {
                localStorage.setItem('topic-post-cache-clear', 1)
                wikiId = ret.data.id
                if (ret.data.ref) {
                    this.$router.push({ name: 'wiki_article_by_ref', params: { ref: ret.data.ref } })
                } else {
                    this.$router.push({ name: 'wiki_article_by_id', params: { id: wikiId } })
                }
                $.message_success(successText)
            } else if (ret.code === api.retcode.INVALID_ROLE) {
                $.message_error('当前用户身份无此权限')
                this.working = false
            } else if (ret.code === api.retcode.ALREADY_EXISTS) {
                $.message_error('指定的URL地址已经存在')
                this.working = false
            } else {
                if (ret.code === api.retcode.FAILED) {
                    this.formErrors = ret.data
                    $.message_error('内容不符合要求，请根据输入框下方提示进行修改')
                } else {
                    this.formErrors = {}
                    $.message_by_code(ret.code, ret.data)
                }
                // 注意：发布成功会跳转，故不做复位，失败则复位
                this.working = false
            }
        },
        fetchData: async function () {
            this.loading = true
            let params = this.$route.params
            this.asAdmin = this.$route.query.manage

            if (!state.user) {
                $.message_error('抱歉，无权访问此页面，请返回')
                return
            }

            if (this.action !== 'new') {
                let ret = await api.wiki.get({
                    id: params.id,
                    loadfk: { user_id: null }
                })

                if (ret.code) {
                    $.message_error('抱歉，发生了错误')
                    return
                }

                this.wikiInfo = ret.data
                if (this.action === 'edit') {
                    this.save = _.clone(ret.data)
                }
            }

            this.loading = false
        }
    },
    beforeRouteEnter (to, from, next) {
        if (!state.user) {
            state.working = 0
            nprogress.done()
            $.message_error('在登录后才能发帖。请登录账号，如果没有账号，先注册一个。')
            return next('/')
        }
        next()
    },
    mounted: async function () {
        // if (localStorage.getItem('topic-post-cache-clear')) {
        //     // 我不知道为什么，在地址跳转前进行 storage 的清除工作，
        //     // 并不会实质上起效，因此这是一个替代手段，效果比较理想。
        //     localStorage.removeItem('topic-post-title')
        //     localStorage.removeItem('smde_topic-post-content')
        //     localStorage.removeItem('topic-post-cache-clear')
        // }

        if (!this.isEdit) {
            // this.title = localStorage.getItem('topic-post-title') || ''
        }
    },
    created: async function () {
        await this.fetchData()

        let func = async () => {
            let editor = this.$refs.editor
            if (editor) {
                let uploadImage = async function (editor, fileList) {
                    let theFile = null

                    if (fileList.length === 0) return
                    for (let i of fileList) {
                        if (i.type.indexOf('image') !== -1) {
                            theFile = i
                            break
                        }
                    }
                    if (!theFile) return false
                    let token = await $.asyncGetUploadToken()

                    let placeholder = `![Uploading ${theFile['name']} - ${(new Objectid()).toString()} ...]()`
                    editor.replaceRange(placeholder, {
                        line: editor.getCursor().line,
                        ch: editor.getCursor().ch
                    })
                    let ob = qiniu.upload(theFile, null, token, null)

                    ob.subscribe({
                        complete: (ret) => {
                            // 注意，这里的res是本地那个callback的结果，七牛直接转发过来了
                            // console.log('done', ret)
                            if (ret.code === api.retcode.SUCCESS) {
                                // let url = `${config.qiniu.host}/${ret.data}` // -${config.qiniu.suffix}
                                let url = `${state.misc.BACKEND_CONFIG.UPLOAD_STATIC_HOST}/${ret.data}`
                                let suffix = state.misc.BACKEND_CONFIG.UPLOAD_QINIU_IMAGE_STYLE_TOPIC
                                if (suffix) url += `-${suffix}`
                                let newTxt = `![](${url})`
                                let offset = newTxt.length - placeholder.length
                                let cur = editor.getCursor()
                                editor.setValue(editor.getValue().replace(placeholder, newTxt + '\n'))
                                editor.setCursor(cur.line, cur.ch + offset)
                            }
                        }
                    })
                }

                let cm = editor.simplemde.codemirror
                cm.on('drop', async (editor, e) => {
                    await uploadImage(editor, e.dataTransfer.files)
                })
                cm.on('paste', async (editor, e) => {
                    await uploadImage(editor, e.clipboardData.files)
                    return false
                })
            } else {
                setTimeout(func, 500)
            }
        }
        setTimeout(func, 500)
    },
    watch: {
        'wikiInfo.ref': function (val, oldVal) {
            if (val && !/^[a-zA-Z0-9-_%]+$/.test(val)) {
                this.wikiInfo.ref = oldVal
            }
        },
        'wikiInfo.title': _.debounce(function (val, oldVal) {
            localStorage.setItem('wiki-post-title', val)
        }, 5000)
    },
    components: {
        markdownEditor
    }
}
</script>
