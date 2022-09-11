# VSCode 配置Markdown模板


### 打开setting.json
```json
"[markdown]":  {
    "editor.quickSuggestions": true
}
```
<!--more-->

### 配置模板
- Ctrl + Shift + P
- 输入snippet
- 点击首选项：配置用户代码片片段
- 选择markdown.json


### 编写模板
```json
{
	// Place your snippets for markdown here. Each snippet is defined under a snippet name and has a prefix, body and 
	// description. The prefix is what is used to trigger the snippet and the body will be expanded and inserted. Possible variables are:
	// $1, $2 for tab stops, $0 for the final cursor position, and ${1:label}, ${2:another} for placeholders. Placeholders with the 
	// same ids are connected.
	// Example:
	// "Print to console": {
	// 	"prefix": "log",
	// 	"body": [
	// 		"console.log('$1');",
	// 		"$2"
	// 	],
	// 	"description": "Log output to console"
	// }
	"blog meta template": {
		"prefix": "meta",
		"body": [
			"---",
			"title: ${TM_FILENAME_BASE}",
			"slug: ${RANDOM_HEX}",
			"date: ${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE}T${CURRENT_HOUR}:${CURRENT_MINUTE}:${CURRENT_SECOND}+08:00",
			"draft: false",
			"author: kbsonlong",
			"authorEmail: kbsonlong@gmail.com",
			"tags: [${1}]",
			"categories: [${2}]",
			"featuredImage: https://imgapi.cn/api.php?zd=zsy&fl=meizi&random=${RANDOM}",
			"---"
		]
	},
	"last modifier time": {
		"prefix": "last",
		"body": "${CURRENT_YEAR}-${CURRENT_MONTH}-${CURRENT_DATE}T${CURRENT_HOUR}:${CURRENT_MINUTE}:${CURRENT_SECOND}+08:00"
	},
	"more themplate" :{
		"prefix": "more",
		"body": "<!--more-->",
		"description": "文章摘要手动分割"
	}
}
```

### 常用变量

```bash
TM_SELECTED_TEXT 当前选定的文本或空字符串
TM_CURRENT_LINE 当前行的内容
TM_CURRENT_WORD 光标下的单词的内容或空字符串
TM_LINE_INDEX 基于零索引的行号
TM_LINE_NUMBER 基于一索引的行号
TM_FILENAME 当前文档的文件名
TM_FILENAME_BASE 当前文档的文件名（不含后缀名)
TM_DIRECTORY 当前文档的目录
RELATIVE_FILEPATH 当前文件的相对目录
TM_FILEPATH 当前文档的完整文件路径
CLIPBOARD 剪切板里的内容

CURRENT_YEAR 当前年(四位数)
CURRENT_MONTH 当前月
CURRENT_DATE 当前日
CURRENT_DAY_NAME_SHORT 当天的短名称（’Mon’）
CURRENT_HOUR 当前小时
CURRENT_MINUTE 当前分钟
CURRENT_SECOND 当前秒



插入随机值
RANDOM 6位随机10进制数
RANDOM_HEX 6位16进制数
UUID 一个版本4的UUID

/** 
286055
f570d8
0a831688-a7f1-4668-9964-f6100114792c
*/

BLOCK_COMMENT_START 块注释开始标识,如 PHP /* 或 HTML <!--
BLOCK_COMMENT_END 块注释结束标识,如 PHP */ 或 HTML -->
LINE_COMMENT 行注释，如： PHP // 或 HTML <!-- -->
```
