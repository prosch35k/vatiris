<template>
    <div
        style="
            height: 25px;
            margin-top: -5px;
            margin-left: -10px;
            background: #777;
            white-space: nowrap;
            text-overflow: ellipsis;
            overflow: hidden;
        "
    >
        <v-btn variant="text" rounded="0" size="small" color="white" v-if="bookmarks.length > 0">
            <v-icon>mdi-menu</v-icon>
            <v-menu activator="parent" style="user-select: none" location="bottom">
                <v-list density="compact">
                    <v-list-item
                        v-for="bookmark in bookmarks"
                        :key="bookmark.id"
                        @click="clickBookmark(bookmark)"
                        style="min-height: 10px"
                    >
                        <v-list-item-title>
                            <div
                                v-if="bookmark.level >= 3"
                                class="text-caption text-grey-lighten-1 pl-8"
                            >
                                {{ bookmark.title }}
                            </div>
                            <div
                                v-else-if="bookmark.level >= 2"
                                class="text-body-2 text-grey-lighten-2 pl-4"
                            >
                                {{ bookmark.title }}
                            </div>
                            <div v-else>
                                {{ bookmark.title }}
                            </div>
                        </v-list-item-title>
                    </v-list-item>
                </v-list>
            </v-menu>
        </v-btn>
        <span class="float-right mr-1 pt-1 text-caption text-grey-lighten-2">
            {{ updatedBy }}
            {{ updatedTime }}
            <span class="text-white" v-if="revision">#{{ revision }}</span>
            <v-btn
                v-if="Object.keys(aipItems).length > 0"
                variant="text"
                rounded="0"
                size="small"
                color="white"
                style="margin-top: -3px"
            >
                AIP
                <submenu :items="aipItems" />
            </v-btn>
        </span>
    </div>
    <div
        v-if="auth.user"
        ref="div"
        class="wiki-div page-content pa-2"
        :class="loading ? '' : 'bg-white'"
        style="height: calc(100% - 20px); overflow-y: auto"
    >
        <div class="page-content" v-html="content"></div>
    </div>
    <div v-else-if="!auth.pending" class="pa-2">Please login to view WIKI content</div>
</template>

<script setup lang="ts">
import Submenu from "./menu/Submenu.vue"
import useEventBus from "@/eventbus"
import { backendBaseUrl, useAuthStore } from "@/stores/auth"
import { computed, onBeforeUnmount, onMounted, reactive, ref, watch } from "vue"
import axios from "axios"
import moment from "moment"
import { useEaipStore } from "@/stores/eaip"

const eaip = useEaipStore()

const props = defineProps<{ book: string; page: string; src?: string }>()
const auth = useAuthStore()
const div = ref()
const content = ref("Loading...")
const loading = ref(true)
const bookmarks = reactive([] as any[])
const revision = ref("")
const updatedTime = ref("")
const updatedBy = ref("")

const bus = useEventBus()
bus.on("refresh", () => {
    fetch()
})

const aipItems = reactive({} as any)

if (props.book == "lop") {
    watch(eaip.aipIndex, () => {
        const aip = eaip.aipIndex
        for (const document of aip.airports.find((a: any) => a.icao == ad).documents) {
            aipItems[document.name] = `aip${document.prefix}`
        }
    })
}

onMounted(() => {
    fetch()
})

onBeforeUnmount(() => {
    if (div.value) div.value.removeEventListener("scroll", onScroll)
})

watch(div, (newValue, oldValue) => {
    if (div.value && !oldValue) {
        div.value.addEventListener("scroll", onScroll)
        if (props.src && div.value) {
            const winbox = div.value.closest(".winbox")
            if (winbox) {
                const title = winbox.querySelector(".wb-title")
                if (title && !title.innerHTML.includes("mdi-open-in-new")) {
                    title.innerHTML += ` <a href="${props.src}" target="_blank" style="color: #ddd"><span class="mdi mdi-open-in-new"></span></a> `
                }
            }
        }
    }
})

watch(content, () => {
    // Extract bookmarks
    setTimeout(() => {
        bookmarks.splice(0)
        const bookmarkEls = div.value.querySelectorAll(
            "h1[id^='bkmrk-'], h2[id^='bkmrk-'], h3[id^='bkmrk-']",
        )
        for (const el of bookmarkEls) {
            const level = el.tagName.replaceAll("H", "")
            const id = el.id
            const title = el.textContent
            bookmarks.push({ level, id, title })
        }
    }, 10)
})

function clickBookmark(bookmark: any) {
    const selector = bookmark.id.replaceAll(".", "\\.").replaceAll("%", "\\%")
    const el = div.value.querySelector(`#${selector}`)
    if (el) el.scrollIntoView({ behavior: "smooth" })
}

function onScroll(e: any) {
    localStorage[`wikiPage_scroll_${props.book}_${props.page}`] = e.target.scrollTop
}

watch(() => auth.user, fetch)

async function fetch() {
    if (!auth.user) return
    try {
        const page = (
            await axios.get(`${backendBaseUrl}/wiki/book/${props.book}/page/${props.page}`, {
                headers: { Authorization: `Bearer ${auth.token.access_token}` },
            })
        ).data
        loading.value = false
        content.value = `<h1 class='page-name'>${page.name}</h1>\n${page.html}`
        revision.value = page.revision
        updatedTime.value = moment(page.updated).utc().format("YYYY-MM-DD HH:mm")
        updatedBy.value = page.updatedBy
        if (div.value && `wikiPage_scroll_${props.book}_${props.page}` in localStorage) {
            //console.log("set scroll", localStorage[`wikiPage_scroll_${props.book}_${props.page}`])
            setTimeout(() => {
                div.value.scrollTop = parseInt(
                    localStorage[`wikiPage_scroll_${props.book}_${props.page}`],
                )
            }, 10)
        } else {
            //console.log("no scroll")
        }
    } catch (error: any) {
        content.value = error.response ? error.response.data : "Error loading content"
        revision.value = ""
        updatedTime.value = ""
        updatedBy.value = ""
    }
}
</script>

<style lang="scss">
/* Additional fixes added in hindsight... */
.wiki-div {
    tr.mainposition td {
        font-weight: bold;
    }
}
</style>

<style lang="scss">
/* This is copy-pasted and bastardized CSS from the wiki HTML export */

.wiki-div {
    h1 {
        font-size: 28px !important;
        color: #011328 !important;
        font-weight: 600;
    }

    h2 {
        font-size: 24px !important;
        color: #1a475f !important;
        border-bottom: 2px solid #dfebeb;
        font-weight: 500;
    }

    h3 {
        font-size: 22px !important;
    }

    h4 {
        font-size: 20px !important;
    }

    .content-wrap.card {
        border-radius: 2px;
    }

    /* Render of content */

    .page-content {
        font-size: 16px;
    }

    .page-content table {
        font-size: 14px;
        vertical-align: middle;
    }
    .page-content td,
    .page-content th {
        padding: 0.5rem;
        vertical-align: middle;
    }
    .page-content h2 {
        font-size: 2rem;
        padding-top: 1.5rem;
    }
    .page-content h3 {
        font-weight: 500;
        font-size: 1.5rem;
        padding-bottom: 0.1rem;
        border-bottom: 1px dashed #dfebeb;
    }
    .page-content h4 {
        font-size: 1.25rem;
    }
    .page-content h5 {
        font-size: 1rem;
    }

    .page-content a {
        text-decoration: underline;
    }
    .page-content a:hover {
        color: #43c6e7;
    }

    .page-content blockquote p {
        margin-bottom: 0.6em;
    }

    .page-content blockquote {
        margin-bottom: 0.6em;
    }

    table th {
        font-weight: bold;
        color: white;
        background: #1a475f;
    }

    .dark-mode table th {
        color: #dfebeb;
        background: #484b4c;
    }

    .callout::before {
        top: 24px;
    }

    /* Remove unused modified/created infolines on shelfes and book covers */
    .grid-card-footer.text-muted {
        display: none;
    }

    :root {
        --font-body: -apple-system, BlinkMacSystemFont, Segoe UI, Oxygen, Ubuntu, Roboto, Cantarell,
            Fira Sans, Droid Sans, Helvetica Neue, sans-serif;
        --font-code: Lucida Console, DejaVu Sans Mono, Ubuntu Mono, Monaco, monospace;
        --color-primary: #206ea7;
        --color-primary-light: rgba(32, 110, 167, 0.15);
        --color-link: #206ea7;
        --color-page: #206ea7;
        --color-page-draft: #7e50b1;
        --color-chapter: #af4d0d;
        --color-book: #077b70;
        --color-bookshelf: #a94747;
        --color-positive: #0f7d15;
        --color-negative: #ab0f0e;
        --color-info: #0288d1;
        --color-warning: #cf4d03;
        --bg-disabled: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' height='100%25' width='100%25'%3E%3Cdefs%3E%3Cpattern id='doodad' width='19' height='19' viewBox='0 0 40 40' patternUnits='userSpaceOnUse' patternTransform='rotate(143)'%3E%3Crect width='100%25' height='100%25' fill='rgba(42, 67, 101,0)'/%3E%3Cpath d='M-10 30h60v20h-60zM-10-10h60v20h-60' fill='rgba(26, 32, 44,0)'/%3E%3Cpath d='M-10 10h60v20h-60zM-10-30h60v20h-60z' fill='rgba(0, 0, 0,0.05)'/%3E%3C/pattern%3E%3C/defs%3E%3Crect fill='url(%23doodad)' height='200%25' width='200%25'/%3E%3C/svg%3E");
    }
    :root.dark-mode {
        --bg-disabled: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' height='100%25' width='100%25'%3E%3Cdefs%3E%3Cpattern id='doodad' width='19' height='19' viewBox='0 0 40 40' patternUnits='userSpaceOnUse' patternTransform='rotate(143)'%3E%3Crect width='100%25' height='100%25' fill='rgba(42, 67, 101,0)'/%3E%3Cpath d='M-10 30h60v20h-60zM-10-10h60v20h-60' fill='rgba(26, 32, 44,0)'/%3E%3Cpath d='M-10 10h60v20h-60zM-10-30h60v20h-60z' fill='rgba(255, 255, 255,0.05)'/%3E%3C/pattern%3E%3C/defs%3E%3Crect fill='url(%23doodad)' height='200%25' width='200%25'/%3E%3C/svg%3E");
        color-scheme: only dark;
        --color-positive: #4aa850;
        --color-negative: #e85c5b;
        --color-warning: #de8a5a;
    }
    :root:not(.dark-mode) {
        color-scheme: only light;
    }
    * {
        box-sizing: border-box;
        outline-color: var(--color-primary);
        outline-width: 1px;
    }
    *:focus {
        outline-style: dotted;
    }
    html {
        height: 100%;
        overflow-y: scroll;
        background-color: #f2f2f2;
    }
    html.flexbox {
        overflow-y: hidden;
    }
    html.dark-mode {
        background-color: #111;
    }
    body {
        font-size: 14px;
        line-height: 1.6;
        color: #444;
        -webkit-font-smoothing: antialiased;
        height: 100%;
        display: flex;
        flex-direction: column;
    }
    html.dark-mode body {
        color: #aaa;
    }
    body,
    button,
    input,
    select,
    label,
    textarea {
        font-family: var(--font-body);
    }
    pre,
    #markdown-editor-input,
    .text-mono,
    .code-base,
    span.code,
    code {
        font-family: var(--font-code);
    }
    h1 {
        font-size: 3.425em;
        line-height: 1.22222222em;
        margin-top: 0.48888889em;
        margin-bottom: 0.48888889em;
    }
    h2 {
        font-size: 2.8275em;
        line-height: 1.294117647em;
        margin-top: 0.8627451em;
        margin-bottom: 0.43137255em;
    }
    h3 {
        font-size: 2.333em;
        line-height: 1.221428572em;
        margin-top: 0.78571429em;
        margin-bottom: 0.43137255em;
    }
    h4 {
        font-size: 1.666em;
        line-height: 1.375em;
        margin-top: 0.78571429em;
        margin-bottom: 0.43137255em;
    }
    h1,
    h2,
    h3,
    h4,
    h5,
    h6 {
        font-weight: 400;
        position: relative;
        display: block;
        font-family: var(--font-heading, var(--font-body));
        color: #222;
    }
    html.dark-mode h1,
    html.dark-mode h2,
    html.dark-mode h3,
    html.dark-mode h4,
    html.dark-mode h5,
    html.dark-mode h6 {
        color: #bbb;
    }
    h1 .subheader,
    h2 .subheader,
    h3 .subheader,
    h4 .subheader,
    h5 .subheader,
    h6 .subheader {
        font-size: 0.5em;
        line-height: 1em;
        color: #969696;
    }
    h5 {
        font-size: 1.4em;
    }
    h5,
    h6 {
        line-height: 1.2em;
        margin-top: 0.78571429em;
        margin-bottom: 0.66em;
    }
    @media screen and (max-width: 600px) {
        h1 {
            font-size: 2.8275em;
        }
        h2 {
            font-size: 2.333em;
        }
        h3 {
            font-size: 1.666em;
        }
        h4 {
            font-size: 1.333em;
        }
        h5 {
            font-size: 1.161616em;
        }
    }
    .list-heading {
        font-size: 2rem;
    }
    h2.list-heading {
        font-size: 1.333rem;
    }
    a {
        color: var(--color-link);
        fill: currentColor;
        cursor: pointer;
        text-decoration: none;
        transition: filter ease-in-out 80ms;
        line-height: 1.6;
    }
    a:hover {
        text-decoration: underline;
    }
    a.icon {
        display: inline-block;
    }
    a svg {
        position: relative;
        display: inline-block;
    }
    a:focus img:only-child {
        outline: 2px dashed var(--color-link);
        outline-offset: 2px;
    }
    a.no-link-style {
        color: inherit;
    }
    a.no-link-style:hover {
        text-decoration: none;
    }
    .blended-links a {
        color: inherit;
    }
    .blended-links a svg {
        fill: currentColor;
    }
    p,
    ul,
    ol,
    pre,
    table,
    blockquote {
        margin-top: 0.3em;
        margin-bottom: 1.375em;
    }
    hr {
        border: 0;
        height: 1px;
        background: #eaeaea;
        margin-bottom: 24px;
    }
    html.dark-mode hr {
        background: #555;
    }
    hr.faded {
        background-image: linear-gradient(to right, #fff, #e3e0e0 20%, #e3e0e0 80%, #fff);
    }
    hr.darker {
        background: #ddd;
    }
    html.dark-mode hr.darker {
        background: #666;
    }
    hr.margin-top,
    hr.even {
        margin-top: 24px;
    }
    strong,
    b,
    .bold,
    .strong {
        font-weight: bold;
    }
    strong > strong,
    strong > b,
    strong > .bold,
    strong > .strong,
    b > strong,
    b > b,
    b > .bold,
    b > .strong,
    .bold > strong,
    .bold > b,
    .bold > .bold,
    .bold > .strong,
    .strong > strong,
    .strong > b,
    .strong > .bold,
    .strong > .strong {
        font-weight: bolder;
    }
    em,
    i,
    .italic {
        font-style: italic;
    }
    small,
    p.small,
    span.small,
    .text-small {
        font-size: 0.75rem;
    }
    sup,
    .superscript {
        vertical-align: super;
        font-size: 0.8em;
    }
    sub,
    .subscript {
        vertical-align: sub;
        font-size: 0.8em;
    }
    pre {
        font-size: 12px;
        border: 1px solid #ddd;
        background-color: #fff;
        border-color: #ddd;
        border-radius: 4px;
        padding-inline-start: 26px;
        position: relative;
        padding-top: 3px;
        padding-bottom: 3px;
    }
    html.dark-mode pre {
        background-color: #2b2b2b;
    }
    html.dark-mode pre {
        border-color: #111;
    }
    pre:before {
        content: "";
        display: block;
        position: absolute;
        top: 0;
        width: 22.4px;
        inset-inline-start: 0;
        height: 100%;
        background-color: #f5f5f5;
        border-inline-end: 1px solid #ddd;
    }
    html.dark-mode pre:before {
        background-color: #313335;
    }
    html.dark-mode pre:before {
        border-inline-end: none;
    }
    @media print {
        pre {
            padding-left: 12px;
        }
        pre:before {
            display: none;
        }
    }
    blockquote {
        display: block;
        position: relative;
        border-left: 4px solid rgba(0, 0, 0, 0);
        border-left-color: var(--color-primary);
        background-color: #f8f8f8;
        padding: 12px 16px 12px 32px;
        overflow: auto;
    }
    html.dark-mode blockquote {
        background-color: #333;
    }
    blockquote:before {
        content: "“";
        font-size: 2em;
        font-weight: bold;
        position: absolute;
        top: 12px;
        left: 12px;
        color: #777;
    }
    .text-mono {
        font-family: var(--font-code);
    }
    .text-uppercase {
        text-transform: uppercase;
    }
    .text-capitals {
        text-transform: capitalize;
    }
    .code-base,
    span.code,
    code {
        font-size: 0.84em;
        border: 1px solid #ddd;
        border-radius: 3px;
        background-color: #f8f8f8;
        border-color: #ddd;
    }
    html.dark-mode .code-base,
    html.dark-mode span.code,
    html.dark-mode code {
        background-color: #2b2b2b;
    }
    html.dark-mode .code-base,
    html.dark-mode span.code,
    html.dark-mode code {
        border-color: #444;
    }
    code {
        display: inline;
        padding: 1px 3px;
        white-space: pre-wrap;
        line-height: 1.2em;
    }
    span.code {
        padding: 1px 6px;
    }
    pre code {
        background-color: rgba(0, 0, 0, 0);
        border: 0;
        font-size: 1em;
        display: block;
        line-height: 1.6;
    }
    span.highlight {
        font-weight: bold;
        padding: 2px 4px;
    }
    ul,
    ol {
        padding-left: 32px;
        padding-right: 32px;
        display: flow-root;
    }
    ul p,
    ol p {
        margin: 0;
    }
    ul {
        list-style: disc;
    }
    ul ul {
        list-style: circle;
    }
    ul label {
        margin: 0;
    }
    ol {
        list-style: decimal;
    }
    li > ol,
    li > ul {
        margin-top: 0;
        margin-bottom: 0;
        margin-block-end: 0;
        margin-block-start: 0;
        padding-block-end: 0;
        padding-block-start: 0;
        padding-left: 19.2px;
        padding-right: 19.2px;
    }
    li.checkbox-item,
    li.task-list-item {
        display: list-item;
        list-style: none;
        margin-left: -19.2px;
    }
    li.checkbox-item input[type="checkbox"],
    li.task-list-item input[type="checkbox"] {
        margin-right: 6px;
    }
    li.checkbox-item li.checkbox-item,
    li.checkbox-item li.task-list-item,
    li.task-list-item li.checkbox-item,
    li.task-list-item li.task-list-item {
        margin-left: 6px;
    }
    .underlined {
        text-decoration: underline;
    }
    .text-center {
        text-align: center;
    }
    .text-left {
        text-align: start;
    }
    .text-right {
        text-align: end;
    }
    @media screen and (min-width: 360px) {
        .text-xxs-center {
            text-align: center;
        }
        .text-xxs-left {
            text-align: start;
        }
        .text-xxs-right {
            text-align: end;
        }
    }
    @media screen and (min-width: 400px) {
        .text-xs-center {
            text-align: center;
        }
        .text-xs-left {
            text-align: start;
        }
        .text-xs-right {
            text-align: end;
        }
    }
    @media screen and (min-width: 600px) {
        .text-s-center {
            text-align: center;
        }
        .text-s-left {
            text-align: start;
        }
        .text-s-right {
            text-align: end;
        }
    }
    @media screen and (min-width: 880px) {
        .text-m-center {
            text-align: center;
        }
        .text-m-left {
            text-align: start;
        }
        .text-m-right {
            text-align: end;
        }
    }
    @media screen and (min-width: 1000px) {
        .text-l-center {
            text-align: center;
        }
        .text-l-left {
            text-align: start;
        }
        .text-l-right {
            text-align: end;
        }
    }
    @media screen and (min-width: 1100px) {
        .text-xl-center {
            text-align: center;
        }
        .text-xl-left {
            text-align: start;
        }
        .text-xl-right {
            text-align: end;
        }
    }
    .text-bigger {
        font-size: 1.1em;
    }
    .text-large {
        font-size: 1.6666em;
    }
    .no-color {
        color: inherit;
    }
    .break-text {
        white-space: normal;
        word-wrap: break-word;
        overflow-wrap: break-word;
    }
    .text-limit-lines-1 {
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }
    .text-limit-lines-2 {
        display: -webkit-box;
        -webkit-box-orient: vertical;
        -webkit-line-clamp: 2;
        overflow: hidden;
    }
    .header-group {
        margin: 16px 0;
    }
    .header-group h1,
    .header-group h2,
    .header-group h3,
    .header-group h4,
    .header-group h5,
    .header-group h6 {
        margin: 0;
    }
    span.sep {
        color: #bbb;
        padding: 0 6px;
    }
    .list > * {
        display: block;
    }
    .svg-icon {
        width: 1em;
        height: 1em;
        display: inline-block;
        position: relative;
        bottom: -0.105em;
        margin-inline-end: 6px;
        pointer-events: none;
        fill: currentColor;
    }
    table {
        min-width: 100px;
        max-width: 100%;
    }
    table thead {
        background-color: #f8f8f8;
        font-weight: 500;
    }
    html.dark-mode table thead {
        background-color: #333;
    }
    table td,
    table th {
        min-width: 10px;
        padding: 6px 8px;
        border: 1px solid #ddd;
        overflow: auto;
        line-height: 1.2;
        word-break: break-word;
        vertical-align: top;
    }
    table td p,
    table th p {
        margin: 0;
    }
    table.table {
        width: 100%;
    }
    table.table tr td,
    table.table tr th {
        border-bottom: 1px solid rgba(0, 0, 0, 0.05);
    }
    table.table th,
    table.table td {
        text-align: start;
        border: none;
        padding: 12px 12px;
        vertical-align: middle;
        margin: 0;
        overflow: visible;
    }
    table.table th {
        font-weight: bold;
    }
    table.table tr:hover {
        background-color: #f2f2f2;
    }
    html.dark-mode table.table tr:hover {
        background-color: #333;
    }
    table.table .text-right {
        text-align: end;
    }
    table.table .text-center {
        text-align: center;
    }
    table.table td.actions {
        overflow: visible;
    }
    table.table a {
        display: inline-block;
    }
    table.table.expand-to-padding {
        margin-left: -12px;
        margin-right: -12px;
        width: calc(100% + 2 * 12px);
        max-width: calc(100% + 2 * 12px);
    }
    table.no-style td {
        border: 0;
        padding: 0;
    }
    table.list-table {
        margin: 0 -6px;
    }
    table.list-table td {
        border: 0;
        vertical-align: middle;
        padding: 6px;
    }
    .page-content {
        width: 100%;
        max-width: 840px;
        margin: 0 auto;
        overflow-wrap: break-word;
    }
    .page-content .align-left {
        text-align: left;
    }
    .page-content img.align-left,
    .page-content table.align-left,
    .page-content iframe.align-left,
    .page-content video.align-left {
        float: left !important;
        margin: 6px 16px 16px 0;
    }
    .page-content .align-right {
        text-align: right !important;
    }
    .page-content img.align-right,
    .page-content table.align-right,
    .page-content iframe.align-right,
    .page-content video.align-right {
        float: right !important;
        margin: 6px 0 6px 12px;
    }
    .page-content .align-center {
        text-align: center;
    }
    .page-content img.align-center,
    .page-content video.align-center,
    .page-content iframe.align-center {
        display: block;
    }
    .page-content img.align-center,
    .page-content table.align-center,
    .page-content iframe.align-center,
    .page-content video.align-center {
        margin-left: auto;
        margin-right: auto;
    }
    .page-content h1,
    .page-content h2,
    .page-content h3,
    .page-content h4,
    .page-content h5,
    .page-content h6,
    .page-content pre {
        clear: left;
    }
    .page-content hr {
        clear: both;
        margin: 16px 0;
    }
    .page-content table {
        hyphens: auto;
        table-layout: fixed;
        max-width: 100%;
        height: auto !important;
    }
    .page-content ins,
    .page-content del {
        text-decoration: none;
    }
    .page-content ins {
        background: #dbffdb;
    }
    .page-content del {
        background: #ffecec;
    }
    .page-content details {
        border: 1px solid;
        border-color: #ddd;
        margin-bottom: 1em;
        padding: 12px;
    }
    html.dark-mode .page-content details {
        border-color: #555;
    }
    .page-content details > summary {
        margin-top: -12px;
        margin-left: -12px;
        margin-right: -12px;
        margin-bottom: -12px;
        font-weight: bold;
        background-color: #eee;
        padding: 6px 12px;
    }
    html.dark-mode .page-content details > summary {
        background-color: #333;
    }
    .page-content details[open] > summary {
        margin-bottom: 12px;
        border-bottom: 1px solid;
        border-color: #ddd;
    }
    html.dark-mode .page-content details[open] > summary {
        border-color: #555;
    }
    .page-content details > summary + * {
        margin-top: 0.2em;
    }
    .page-content details:after {
        content: "";
        display: block;
        clear: both;
    }
    .page-content li > input[type="checkbox"] {
        vertical-align: top;
        margin-top: 0.3em;
    }
    .page-content p:empty {
        min-height: 1.6em;
    }
    .page-content.page-revision pre code {
        white-space: pre-wrap;
    }
    .page-content .cm-editor {
        margin-bottom: 1.375em;
    }
    .page-content video {
        max-width: 100%;
    }
    .page-content a {
        text-decoration: underline;
    }
    body .page-content img,
    .page-content img:not([data-mce-object]) {
        max-width: 100%;
        height: auto;
    }
    .callout {
        border-left: 3px solid #bbb;
        border-inline-start: 3px solid #bbb;
        border-inline-end: none;
        background-color: #eee;
        padding: 12px;
        padding-left: 32px;
        padding-inline-start: 32px;
        padding-inline-end: 12px;
        display: block;
        position: relative;
        overflow: auto;
    }
    .callout:before {
        background-image: url("data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgMjQgMjQiIGZpbGw9IiMwMTUzODAiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+ICAgIDxwYXRoIGQ9Ik0wIDBoMjR2MjRIMHoiIGZpbGw9Im5vbmUiLz4gICAgPHBhdGggZD0iTTEyIDJDNi40OCAyIDIgNi40OCAyIDEyczQuNDggMTAgMTAgMTAgMTAtNC40OCAxMC0xMFMxNy41MiAyIDEyIDJ6bTEgMTVoLTJ2LTZoMnY2em0wLThoLTJWN2gydjJ6Ii8+PC9zdmc+");
        background-repeat: no-repeat;
        content: "";
        width: 1.2em;
        height: 1.2em;
        left: 8px;
        inset-inline-start: 8px;
        inset-inline-end: unset;
        top: 50%;
        margin-top: -9px;
        display: inline-block;
        position: absolute;
        line-height: 1;
        opacity: 0.8;
    }
    .callout.success {
        border-color: #0f7d15;
        background-color: #eafdeb;
        color: #063409;
    }
    html.dark-mode .callout.success {
        border-color: #4aa850;
    }
    html.dark-mode .callout.success {
        background-color: #122913;
    }
    html.dark-mode .callout.success {
        color: #4aa850;
    }
    .callout.success:before {
        background-image: url("data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgMjQgMjQiIGZpbGw9IiMzNzZjMzkiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+ICAgIDxwYXRoIGQ9Ik0wIDBoMjR2MjRIMHoiIGZpbGw9Im5vbmUiLz4gICAgPHBhdGggZD0iTTEyIDJDNi40OCAyIDIgNi40OCAyIDEyczQuNDggMTAgMTAgMTAgMTAtNC40OCAxMC0xMFMxNy41MiAyIDEyIDJ6bS0yIDE1bC01LTUgMS40MS0xLjQxTDEwIDE0LjE3bDcuNTktNy41OUwxOSA4bC05IDl6Ii8+PC9zdmc+");
    }
    .callout.danger {
        border-color: #ab0f0e;
        background-color: #fcdbdb;
        color: #4d0706;
    }
    html.dark-mode .callout.danger {
        border-color: #e85c5b;
    }
    html.dark-mode .callout.danger {
        background-color: #250505;
    }
    html.dark-mode .callout.danger {
        color: #e85c5b;
    }
    .callout.danger:before {
        background-image: url("data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgMjQgMjQiIGZpbGw9IiNiOTE4MTgiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+ICAgIDxwYXRoIGQ9Ik0xNS43MyAzSDguMjdMMyA4LjI3djcuNDZMOC4yNyAyMWg3LjQ2TDIxIDE1LjczVjguMjdMMTUuNzMgM3pNMTIgMTcuM2MtLjcyIDAtMS4zLS41OC0xLjMtMS4zIDAtLjcyLjU4LTEuMyAxLjMtMS4zLjcyIDAgMS4zLjU4IDEuMyAxLjMgMCAuNzItLjU4IDEuMy0xLjMgMS4zem0xLTQuM2gtMlY3aDJ2NnoiLz4gICAgPHBhdGggZD0iTTAgMGgyNHYyNEgweiIgZmlsbD0ibm9uZSIvPjwvc3ZnPg==");
    }
    .callout.info {
        border-color: #0288d1;
        color: #01466c;
        background-color: #d3efff;
    }
    html.dark-mode .callout.info {
        border-color: #0288d1;
    }
    html.dark-mode .callout.info {
        color: #0288d1;
    }
    html.dark-mode .callout.info {
        background-color: #001825;
    }
    .callout.warning {
        border-color: #cf4d03;
        background-color: #fee3d3;
        color: #6a2802;
    }
    html.dark-mode .callout.warning {
        border-color: #de8a5a;
    }
    html.dark-mode .callout.warning {
        background-color: #30170a;
    }
    html.dark-mode .callout.warning {
        color: #de8a5a;
    }
    .callout.warning:before {
        background-image: url("data:image/svg+xml;base64,PHN2ZyB2aWV3Qm94PSIwIDAgMjQgMjQiIGZpbGw9IiNiNjUzMWMiIHhtbG5zPSJodHRwOi8vd3d3LnczLm9yZy8yMDAwL3N2ZyI+ICAgIDxwYXRoIGQ9Ik0wIDBoMjR2MjRIMHoiIGZpbGw9Im5vbmUiLz4gICAgPHBhdGggZD0iTTEgMjFoMjJMMTIgMiAxIDIxem0xMi0zaC0ydi0yaDJ2MnptMC00aC0ydi00aDJ2NHoiLz48L3N2Zz4=");
    }
    .callout a {
        color: inherit;
        text-decoration: underline;
    }
    html,
    body {
        background-color: #fff;
    }
    body {
        font-family:
            "DejaVu Sans",
            -apple-system,
            BlinkMacSystemFont,
            "Segoe UI",
            "Oxygen",
            "Ubuntu",
            "Roboto",
            "Cantarell",
            "Fira Sans",
            "Droid Sans",
            "Helvetica Neue",
            sans-serif;
        margin: 0;
        padding: 0;
        display: block;
    }
    table {
        border-spacing: 0;
        border-collapse: collapse;
    }
    .page-content {
        overflow: hidden;
    }
    pre {
        padding-left: 12px;
    }
    pre:before {
        display: none;
    }
    pre code {
        white-space: pre-wrap;
    }
    .page-break {
        page-break-after: always;
    }
    @media screen {
        .page-break {
            border-top: 1px solid #ddd;
        }
    }
    ul.contents ul li {
        list-style: circle;
    }
    .chapter-hint {
        color: #888;
        margin-top: 32px;
    }
    .chapter-hint + h1 {
        margin-top: 0;
    }
    body.export-format-pdf {
        font-size: 14px;
        line-height: 1.2;
    }
    body.export-format-pdf h1,
    body.export-format-pdf h2,
    body.export-format-pdf h3,
    body.export-format-pdf h4,
    body.export-format-pdf h5,
    body.export-format-pdf h6 {
        line-height: 1.2;
    }
    body.export-format-pdf table {
        max-width: 800px !important;
        font-size: 0.8em;
        width: 100% !important;
    }
    body.export-format-pdf table td {
        width: auto !important;
    }
    body.export-format-pdf .page-content .float {
        float: none !important;
    }
    body.export-format-pdf .page-content img.align-left,
    body.export-format-pdf .page-content img.align-right {
        float: none !important;
        clear: both;
        display: block;
    }
    body.export-format-pdf.export-engine-dompdf .page-content a > img {
        max-width: 700px;
    }
    body.export-format-pdf.export-engine-dompdf .page-content td a > img {
        max-width: 100%;
    } /*# sourceMappingURL=export-styles.css.map */
}
</style>
