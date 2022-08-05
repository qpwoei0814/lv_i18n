lv_i18n - Internationalization for LittlevGL
============================================

这个软件包用于实现LVGL的多语言支持，但官方版本仅支持提取用`_("some text")`或`_p("some text")`标记的文本，但这种标记方式无法适用于静态数组中的文本，因此修改出了这个软件包。

本修改后的软件包支持以提取翻译`$`开头的字符串，例如`"$some text"`。

这个README主要记录一些相对原版包的修改以及使用指令，更详细的说明参见[原版README](README.md)


## Quick overview

1. 在代码中标记需要翻译的文本，在函数参数中的文本直接使用`_("some text")`或`_p("some text")`标记并动态返回翻译后的结果。对于静态定义在数组等数据结构中的文本，以`$`标记文本，使用`_(pointer)`或`_p(pointer)`标记指向文本文本的指针。
2. 建立需要的语言的yml文件，运行JS脚本使用`extract`指令提取文本，填充yml文件。
3. 手动填入各语言翻译文本
4. 使用 `compile`根据yml翻译文本生成c文件，并将c文件和h文件添加进工程

## Install/run the script 安装脚本以及运行环境


**[node.js](https://nodejs.org/en/download/) required.**


全局安装本地的lv_i18n包：

```sh
 npm install ..\lv_i18n -g
 lv_i18n extract -s 'lvgl/proj/ui/*.+(c|cpp|h|hpp)' -t 'lvgl/proj/i18n/*.yml'
 lv_i18n compile -t 'lvgl/proj/i18n/*.yml' -o 'lvgl/proj/i18n/'
```
卸载已安装的lv_i18n包

```sh
npm uninstall -g lv_i18n
```


## Mark up the text in your code

1. 直接使用`_("some text")`或`_p("some text")`标记并动态返回翻译后的结果，可以参见原版README。

2. 使用`$`标记文本，使用`_(pointer)`或`_p(pointer)`标记指向文本文本的指针。
```c
#include "lv_i18n/lv_i18n.h" 

//使用"$"标记需要提取的文本
char* acPowerStateText[] = { "$mainpage_pwrstate_normal", "$mainpage_pwrstate_broken"};


lv_label_set_text(tOverviewPage->lb_state_s1, _(acPowerStateText[0]));
lv_label_set_text(tOverviewPage->lb_state_s2, _(acPowerStateText[1]));
```


## 提取标记文本
建立你需要的语言的语言文件，[这里](https://www.andiamo.co.uk/resources/iso-language-codes/)有各个语言的标准命名，文件内容按如下格式填入：

en-GB.yml:
```yml
en-GB:

```
cn-ZH.yml:
```yml
cn-ZH:

```

之后运行`extract` 命令将工程中需要翻译的文本提取出来 （注意修改工程ui文件以及yml文件路径）:

```sh
lv_i18n extract -s 'src/**/*.+(c|cpp|h|hpp)' -t 'translations/*.yml'
```

命令成功执行后yml文件会被填入如下内容

```yml
en-GB:
  title1: ~
  user_logged_in:
    one: ~
    other: ~
```


## 添加你的翻译文本

Example:

```yml
'en-GB':
  title1: Main menu
  user_logged_in:
    one: One user is logged in
    other: '%d users are logged in'
```
## 运行命令将翻译文本转换为C和H文件

运行 `compile`命令:

```sh
lv_i18n compile -t 'translations/*.yml' -o 'src/lv_i18n'
```

The deafult locale is `en-GB` but you change it with `-l 'language-code'`.

## Follow modifications in the source code
To change a text id in the `yml` files use:
```sh
lv_i18n rename -t src/i18n/*.yml --from 'Hillo wold' --to 'Hello world!'
```

## 相比原版代码上的修改

### 修改正则表达式
增加用于匹配"$"标记的正则式，修改了[parser.js](lib/parser.js)
```js
//原版
function create_singular_re(fn_name) {
  return new RegExp(
    '(?:^|[ =+,;\(])' + _.escapeRegExp(fn_name) + '\\("(.*?)"\\)',
    'g'
  );
}
function extract(text, re) {
  let result = [];
  for (;;) {
    let match = re.exec(text);
    if (!match) break;
    result.push({
      key: unescape_c(match[1]),
      line: getLine(text, match.index)
    });
  }

  return result;
}

//修改后
function create_singular_re(fn_name) {
  return new RegExp(
    '(?:^|[ =+,;\(])' + _.escapeRegExp(fn_name) + '\\("(.*?)"\\)|"\\$(.*?)"',
    'g'
  );
}

function extract(text, re) {
  let result = [];
  let matchtext;
  for (;;) {
    let match = re.exec(text);
    if (!match) break;
    if (typeof (match[1]) === 'undefined'){
      matchtext = match[2];        //通过"$"匹配出的文本会出现在match[2]
    } else {
      matchtext = match[1];
    }
    // eslint-disable-next-line no-console
    console.log(matchtext);
    result.push({
      key: unescape_c(matchtext),
      line: getLine(text, match.index)
    });
  }
  return result;
}
```
### 修改C文件模板，使翻译文本查找函数能忽略"$"提取出正确标签
修改文件: [lv_i18n.template.c](./src/lv_i18n.template.c)
```c
//原版
static const char * __lv_i18n_get_text_core(lv_i18n_phrase_t * trans, const char * msg_id)
{
    uint16_t i;
    for(i = 0; trans[i].msg_id != NULL; i++) {
        if(strcmp(trans[i].msg_id, msg_id) == 0) {
            /*The msg_id has been found. Check the translation*/
            if(trans[i].translation) return trans[i].translation;
        }
    }

    return NULL;
}

//修改后
static const char * __lv_i18n_get_text_core(lv_i18n_phrase_t * trans, const char * msg_id)
{
    uint16_t i;

    //REMOVE "$"
    if (msg_id[0] == 0x24)
    {
        msg_id++;
    }

    for(i = 0; trans[i].msg_id != NULL; i++) {
        if(strcmp(trans[i].msg_id, msg_id) == 0) {
            /*The msg_id has been found. Check the translation*/
            if(trans[i].translation) return trans[i].translation;
        }
    }

    return NULL;
}
```

