<!--suppress CssNoGenericFontName -->
<script setup>
import {marked} from "marked";
import markedKatex from "marked-katex-extension";
// 引入 katex 样式文件
import "katex/dist/katex.min.css";
import hljs from "highlight.js";
import {ref, computed, watch, nextTick} from "vue";
import insane from 'insane'
import {useRoute} from "vue-router";
import base64 from 'base64-js'
import pako from 'pako'

function detectColorScheme() {
    let theme = "light";

    if (window.matchMedia("(prefers-color-scheme: dark)").matches) {
        //OS theme setting detected as dark
        theme = "dark";
    }

    //dark theme preferred, set document with a `data-theme` attribute
    if (theme === "dark") {
        document.documentElement.setAttribute("data-theme", "dark");
    }
}

detectColorScheme();

// marked 配置
marked.setOptions({
    renderer: new marked.Renderer(),
    highlight: function (code, lang) {
        const language = hljs.getLanguage(lang) ? lang : "plaintext";
        return hljs.highlight(code, {language}).value;
    },
    langPrefix: "hljs language-", // highlight.js css expects a top-level 'hljs' class.
    pedantic: false,
    gfm: true,
    breaks: false,
    sanitize: false,
    smartypants: false,
    xhtml: false,
});

// marked-katex 插件配置
marked.use(
    markedKatex({
        throwOnError: false,
    })
);

// 替换纯代码段无样式
function indentCodeCompensation(raw, text) {
    const matchIndentToCode = raw.match(/^(\s+)```/);

    if (matchIndentToCode === null) {
        return text;
    }

    const indentToCode = matchIndentToCode[1];

    return text
        .split("\n")
        .map((node) => {
            const matchIndentInNode = node.match(/^\s+/);
            if (matchIndentInNode === null) {
                return node;
            }

            const [indentInNode] = matchIndentInNode;

            if (indentInNode.length >= indentToCode.length) {
                return node.slice(indentToCode.length);
            }

            return node;
        })
        .join("\n");
}

const tokenizer = {
    fences(src) {
        const cap = this.rules.block.fences.exec(src);
        if (cap) {
            const raw = cap[0];
            const text = indentCodeCompensation(raw, cap[3] || "");
            var lang = cap[2]
                ? cap[2].trim().replace(this.rules.inline._escapes, "$1")
                : cap[2];
            if (lang.length === 0) {
                lang = "plaintext";
            }

            return {
                type: "code",
                raw,
                lang,
                text,
            };
        }
    },
};

marked.use({tokenizer});

// 默认初始值
const input = ref(`
# Markdown Renderer
[https://github.com/cnSchwarzer/MarkdownRenderer](https://github.com/cnSchwarzer/MarkdownRenderer)

这是分割线

----------------------

## 这是二级标题
### 这是三级标题
#### 这是四级标题
##### 这是五级标题
###### 这是六级标题

**粗体文本**

*斜体文本*

~~删除线文本~~

这是一个无序列表：
- 列表1
    - a
        - a
            - a

这是一个有序列表：
1. 列表2
    1. a
        1. a
            1. a

这是一个[链接](https://www.baidu.com/)

这是一个图片：

![](https://guild-1304010062.cos.ap-nanjing.myqcloud.com/reito/ending.png)

> 这是一段引用
>  > 嵌套引用

内联代码：\`print("Hello World")\`

代码块：
\`\`\`python
num = 1
if num == 1:
    print("Yes")
\`\`\`

这是表格

| aaa | aaa |
| --- | --- |
| aaa | aaa |
| vvv | vvv |
| ccc | ccc |
| ddd | ddd |

这是一段公式：$y=x_1+x_2+\\dots+x_n=\\sum_{i=1}^n{x_i}$

这是一段麻将：

- 东3局 0本场

<span class="mahjong">
    <span class="mj">
        <span class="mj-red">🀋</span>🀀🀁🀂🀃🀄🀅🀆🀇
    </span>
</span>

- 东3局 1本场

<span class="v-mid">
<span class="mahjong">
    <span class="mj">
        asdfghjkl
    </span>
</span>
<span style="width: 10px"></span>

[用户名]()
</span>

<span class="mahjong">
    <span class="mj">
        qwertyuio
    </span>
</span>

<span class="mahjong">
    <span class="mj">
        zxcvbnm,.
    </span>
</span>

<span class="mahjong">
    <span class="mj">
        1234567890
    </span>
</span>
`);
const ring = ref(false);

// 将 markdown 文本转换为 html
const insaneOptions = {
    allowedAttributes: {
        a: ['href', 'name', 'target', 'rel', 'title'],
        img: ['src', 'alt', 'title'],
        input: ['type', 'checked', 'disabled'],
        code: ['class'],
        span: ['class', 'style'],
        div: ['class', 'style'],
        th: ['align'],
        tr: ['align']
    },
    allowedSchemes: ['http', 'https', 'mailto'],
    allowedTags: [
        'a', 'article', 'b', 'blockquote', 'br', 'caption', 'code', 'del', 'details', 'div',
        'em', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'hr', 'i', 'img', 'input', 'ins', 'kbd',
        'label', 'li', 'main', 'ol', 'p', 'pre', 'section', 'span', 'strike', 'strong', 'sub',
        'summary', 'sup', 'table', 'tbody', 'td', 'th', 'thead', 'tr', 'u', 'ul'
    ],
    transformText: null,
};

for (let i = 1; i <= 6; i++) {
    insaneOptions.allowedAttributes[`h${i}`] = ['id'];
}
const output = computed(() => insane(marked.parse(input.value), insaneOptions));

const appPadding = ref('0.4rem 1rem 1rem 1rem')
const appMargin = ref('0')

window.renderMarkdown = (renderOptions) => {
    input.value = renderOptions.markdown;
    ring.value = renderOptions.ring;
    if (renderOptions.padding) {
        appPadding.value = renderOptions.padding;
    }
    if (renderOptions.margin) {
        appMargin.value = renderOptions.margin;
    }
    if (renderOptions.theme === "dark") {
        document.documentElement.setAttribute("data-theme", "dark");
    } else {
        document.documentElement.removeAttribute("data-theme");
    }
}

let route = useRoute()
if (route.query.data) {
    let raw = base64.toByteArray(route.query.data)
    let dec = pako.ungzip(raw)
    let text = new TextDecoder().decode(dec)
    let j = JSON.parse(text)
    window.renderMarkdown(j)
}
</script>

<template>
    <div>
        <div class="md-root">
            <div v-if="ring"
                 style='height: 20px; display: flex; align-items: center; margin-bottom: 8px; margin-top: 5px; font-size: 19px; font-family: "Consolas", serif'>
                <svg
                        width="20"
                        height="20"
                        viewBox="0 0 41 41"
                        fill="none"
                        xmlns="http://www.w3.org/2000/svg"
                        stroke-width="2"
                        class="scale-appear"
                        style='margin-right: 5px;'
                >
                    <path
                            d="M37.5324 16.8707C37.9808 15.5241 38.1363 14.0974 37.9886 12.6859C37.8409 11.2744 37.3934 9.91076 36.676 8.68622C35.6126 6.83404 33.9882 5.3676 32.0373 4.4985C30.0864 3.62941 27.9098 3.40259 25.8215 3.85078C24.8796 2.7893 23.7219 1.94125 22.4257 1.36341C21.1295 0.785575 19.7249 0.491269 18.3058 0.500197C16.1708 0.495044 14.0893 1.16803 12.3614 2.42214C10.6335 3.67624 9.34853 5.44666 8.6917 7.47815C7.30085 7.76286 5.98686 8.3414 4.8377 9.17505C3.68854 10.0087 2.73073 11.0782 2.02839 12.312C0.956464 14.1591 0.498905 16.2988 0.721698 18.4228C0.944492 20.5467 1.83612 22.5449 3.268 24.1293C2.81966 25.4759 2.66413 26.9026 2.81182 28.3141C2.95951 29.7256 3.40701 31.0892 4.12437 32.3138C5.18791 34.1659 6.8123 35.6322 8.76321 36.5013C10.7141 37.3704 12.8907 37.5973 14.9789 37.1492C15.9208 38.2107 17.0786 39.0587 18.3747 39.6366C19.6709 40.2144 21.0755 40.5087 22.4946 40.4998C24.6307 40.5054 26.7133 39.8321 28.4418 38.5772C30.1704 37.3223 31.4556 35.5506 32.1119 33.5179C33.5027 33.2332 34.8167 32.6547 35.9659 31.821C37.115 30.9874 38.0728 29.9178 38.7752 28.684C39.8458 26.8371 40.3023 24.6979 40.0789 22.5748C39.8556 20.4517 38.9639 18.4544 37.5324 16.8707ZM22.4978 37.8849C20.7443 37.8874 19.0459 37.2733 17.6994 36.1501C17.7601 36.117 17.8666 36.0586 17.936 36.0161L25.9004 31.4156C26.1003 31.3019 26.2663 31.137 26.3813 30.9378C26.4964 30.7386 26.5563 30.5124 26.5549 30.2825V19.0542L29.9213 20.998C29.9389 21.0068 29.9541 21.0198 29.9656 21.0359C29.977 21.052 29.9842 21.0707 29.9867 21.0902V30.3889C29.9842 32.375 29.1946 34.2791 27.7909 35.6841C26.3872 37.0892 24.4838 37.8806 22.4978 37.8849ZM6.39227 31.0064C5.51397 29.4888 5.19742 27.7107 5.49804 25.9832C5.55718 26.0187 5.66048 26.0818 5.73461 26.1244L13.699 30.7248C13.8975 30.8408 14.1233 30.902 14.3532 30.902C14.583 30.902 14.8088 30.8408 15.0073 30.7248L24.731 25.1103V28.9979C24.7321 29.0177 24.7283 29.0376 24.7199 29.0556C24.7115 29.0736 24.6988 29.0893 24.6829 29.1012L16.6317 33.7497C14.9096 34.7416 12.8643 35.0097 10.9447 34.4954C9.02506 33.9811 7.38785 32.7263 6.39227 31.0064ZM4.29707 13.6194C5.17156 12.0998 6.55279 10.9364 8.19885 10.3327C8.19885 10.4013 8.19491 10.5228 8.19491 10.6071V19.808C8.19351 20.0378 8.25334 20.2638 8.36823 20.4629C8.48312 20.6619 8.64893 20.8267 8.84863 20.9404L18.5723 26.5542L15.206 28.4979C15.1894 28.5089 15.1703 28.5155 15.1505 28.5173C15.1307 28.5191 15.1107 28.516 15.0924 28.5082L7.04046 23.8557C5.32135 22.8601 4.06716 21.2235 3.55289 19.3046C3.03862 17.3858 3.30624 15.3413 4.29707 13.6194ZM31.955 20.0556L22.2312 14.4411L25.5976 12.4981C25.6142 12.4872 25.6333 12.4805 25.6531 12.4787C25.6729 12.4769 25.6928 12.4801 25.7111 12.4879L33.7631 17.1364C34.9967 17.849 36.0017 18.8982 36.6606 20.1613C37.3194 21.4244 37.6047 22.849 37.4832 24.2684C37.3617 25.6878 36.8382 27.0432 35.9743 28.1759C35.1103 29.3086 33.9415 30.1717 32.6047 30.6641C32.6047 30.5947 32.6047 30.4733 32.6047 30.3889V21.188C32.6066 20.9586 32.5474 20.7328 32.4332 20.5338C32.319 20.3348 32.154 20.1698 31.955 20.0556ZM35.3055 15.0128C35.2464 14.9765 35.1431 14.9142 35.069 14.8717L27.1045 10.2712C26.906 10.1554 26.6803 10.0943 26.4504 10.0943C26.2206 10.0943 25.9948 10.1554 25.7963 10.2712L16.0726 15.8858V11.9982C16.0715 11.9783 16.0753 11.9585 16.0837 11.9405C16.0921 11.9225 16.1048 11.9068 16.1207 11.8949L24.1719 7.25025C25.4053 6.53903 26.8158 6.19376 28.2383 6.25482C29.6608 6.31589 31.0364 6.78077 32.2044 7.59508C33.3723 8.40939 34.2842 9.53945 34.8334 10.8531C35.3826 12.1667 35.5464 13.6095 35.3055 15.0128ZM14.2424 21.9419L10.8752 19.9981C10.8576 19.9893 10.8423 19.9763 10.8309 19.9602C10.8195 19.9441 10.8122 19.9254 10.8098 19.9058V10.6071C10.8107 9.18295 11.2173 7.78848 11.9819 6.58696C12.7466 5.38544 13.8377 4.42659 15.1275 3.82264C16.4173 3.21869 17.8524 2.99464 19.2649 3.1767C20.6775 3.35876 22.0089 3.93941 23.1034 4.85067C23.0427 4.88379 22.937 4.94215 22.8668 4.98473L14.9024 9.58517C14.7025 9.69878 14.5366 9.86356 14.4215 10.0626C14.3065 10.2616 14.2466 10.4877 14.2479 10.7175L14.2424 21.9419ZM16.071 17.9991L20.4018 15.4978L24.7325 17.9975V22.9985L20.4018 25.4983L16.071 22.9985V17.9991Z"
                            fill="currentColor"
                            data-darkreader-inline-fill=""
                            style="--darkreader-inline-fill: currentColor"
                    ></path>
                </svg>
                Ring
            </div>
            <div class="output" v-html="output"></div>
        </div>
    </div>
</template>

<style>
:root {
    --list-primary-level: "•";
    --list-secondary-level: "◦";
    --list-tertiary-level: "▪";
    --list-item-spacing: 1em;
    --list-ul-base-padding: 1.1em;
    --list-ol-base-padding: 1.2em;
    --list-nested-padding: 1em;

    --blockquote-color: #e8e8e880;
    --table-border-color: #e2e2e2;
    --table-header-bg-color: #f3f3f3;
    --table-odd-row-bg-color: #e7e7e7;
    --table-border-radius: 8px;
    --href-color: #127dda;
    --code-block-color: #2a2c2d;
}

/* 配置深色模式 */
[data-theme="dark"] {
    --blockquote-color: rgba(33, 34, 37, 0.6);
    --table-border-color: rgb(83, 85, 91);
    --table-header-bg-color: #28292f;
    --table-odd-row-bg-color: #33353b;
    --table-even-row-bg-color: #3e4046;
    --href-color: #45a9f9;
    --code-block-color: var(--blockquote-color);
}

pre > code {
    font-family: "Consolas", "思源黑体", serif;
    padding: 0.6em 1em 0.6em 1em !important;
    border-radius: 10px;
    white-space: pre-wrap !important;
    font-size: 13px;
    background: var(--code-block-color) !important;
}

:not(pre) > code {
    font-family: "Consolas", "思源黑体", "Source Hans Sans", serif;
    padding: 2px 5px;
    border-radius: 5px;
    background-color: var(--color-background-mute);
    background-position: center;
    color: #d19a66;
}

pre:has(code) {
    padding: 5px 0 5px 0;
    line-height: 1.3;
    margin: 3px 0;
}

blockquote {
    border-radius: 5px;
    border-left: thick solid #a2a2a2;
    padding: 3px 5px 3px 5px !important;
    margin: 5px 0 5px 0;
    background-color: var(--blockquote-color);
}

img {
    margin: 5px 0 5px 0;
    max-width: 100%;
    border-radius: 5px;
}

/* 列表 */
ul ul, ol ul,
ul ol, ol ol {
    padding-left: var(--list-nested-padding); /* 嵌套列表左边距 */
}

/* 无序列表容器样式 */
ul {
    padding-left: var(--list-ul-base-padding);
}

/* 有序列表（ol）容器样式 */
ul ul {
    list-style-type: circle;
}

ul ul ul {
    list-style-type: square;
}

ul ul ul ul {
    list-style-type: disclosure-closed
}

/* 一级列表项伪元素样式 */
ul li::before {
    padding-right: var(--list-item-spacing); /* 使用间隙距离变量 */
}

/* 二级列表项伪元素样式 */
ul ul li::before {
    padding-right: var(--list-item-spacing); /* 使用间隙距离变量 */
}

/* 三级列表项伪元素样式 */
ul ul ul li::before {
    padding-right: var(--list-item-spacing); /* 使用间隙距离变量 */
}

/* 有序列表（ol）容器样式 */
ol {
    padding-left: var(--list-ol-base-padding);
}

/* 有序列表（ol）容器样式 */
ol ol {
    list-style-type: lower-alpha; /* 移除默认列表样式 */
}

ol ol ol {
    list-style-type: lower-roman; /* 移除默认列表样式 */
}

ol ol ol ol {
    list-style-type: lower-greek; /* 移除默认列表样式 */
}

/* 一级列表项伪元素样式 */
ol li::before {
    padding-right: var(--list-item-spacing); /* 使用间隙距离变量 */
}

/* 二级列表项伪元素样式 */
ol ol li::before {
    padding-right: var(--list-item-spacing); /* 使用间隙距离变量 */
}

/* 三级列表项伪元素样式 */
ol ol ol li::before {
    padding-right: var(--list-item-spacing); /* 使用间隙距离变量 */
}

/* ul:has(input) {
  list-style-type: none; 
  padding-inline-start: 5px;
}

ul:has(input) > li > input {
  margin-right: 5px;
} 

ol:has(input) {
  list-style-type: none;
  padding-inline-start: 5px;
}

ol:has(input) > li > input {
  margin-right: 5px;
}  */

/* 表格 */
table {
    border-collapse: separate;
    border-spacing: 0;
    width: 100%; /* auto 可以自适应内容 */
    margin: 5px 0;
}

th, td {
    text-align: center;
    padding: 5px;
}

th, td {
    border-right: 1px solid var(--table-border-color);
    border-bottom: none;
    border-top: none;
}

th:last-child, td:last-child {
    border-right: none;
}

/* 每行之间的横线 */
table tbody tr:not(:last-child) td {
    border-bottom: 1px solid var(--table-border-color);
}

/* 圆角矩形边框 */
table {
    border: 1px solid var(--table-border-color);
    border-radius: var(--table-border-radius);
}

/* 表头样式 */
table thead th {
    background-color: var(--table-header-bg-color);
    border-bottom: 2px solid var(--table-border-color);
    font-weight: bold;
}

/* 背景颜色交错 */
table tbody tr:nth-child(odd) {
    background-color: var(--table-odd-row-bg-color);
}

table tbody tr:nth-child(even) {
    background-color: var(--table-even-row-bg-color);
}

/* 表格圆角处理 */
table tr:first-child th:first-child {
    border-top-left-radius: calc(var(--table-border-radius) - 1px);
}

table tr:first-child th:last-child {
    border-top-right-radius: calc(var(--table-border-radius) - 1px);
}

table tr:last-child td:first-child {
    border-bottom-left-radius: calc(var(--table-border-radius) - 1px);
}

table tr:last-child td:last-child {
    border-bottom-right-radius: calc(var(--table-border-radius) - 1px);
}

/* 链接 */
a {
    color: var(--href-color);
    text-decoration: none;
}

/* 横线 */
hr {
    border: 1px solid var(--table-border-color);
    margin: 5px 0;
}

* {
    -webkit-transition: none !important;
    -moz-transition: none !important;
    -o-transition: none !important;
    -ms-transition: none !important;
    transition: none !important;
}

@font-face {
    font-family: 'S.Mahjong'; /*a name to be used later*/
    src: url('assets/S-Mahjong.ttf'); /*URL to font*/
}

.v-mid {
    display: flex;
    align-items: center;
}

.mahjong {
    display: inline-flex;
    flex-wrap: wrap;
    align-items: flex-end;
    font-family: "S.Mahjong", "Consolas", "思源黑体", serif;
    color: black;
    line-height: 24px;
    font-size: 24px;
    background-color: white;
    border-radius: 2px;
    padding-left: 1px;
    padding-right: 2px;
}

.mj {
    display: inline-flex;
    position: relative;
    top: -2px;
}

.mj-red {
    filter: invert(50%) sepia(100%) hue-rotate(-49deg) saturate(550%);
}

.table-sm th, .table-sm td {
    padding: 2px;
    font-size: 12px;
    line-height: 1.4;
}

.table-sm table thead th {
    border-bottom: 1px solid var(--table-border-color);
}

.md-root {
    padding: v-bind(appPadding);
    margin: v-bind(appMargin);
}
</style>
