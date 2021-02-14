## 给GitBook添加自己喜欢的插件
GitBook可以通过添加插件的方式来丰富文档的内容，添加插件的步骤：
* 在文档根目录下新建一个**book.json**文件。
* 在book.json的根结构上加上**plugins**属性(祥见分享插件)
* **gitbook install ./** 安装插件

## 添加分享插件
分享插件的配置文件.

```json
{
    "title": "格局.",
    "author": "LuoSu",
    "plugins": ["-sharing", "sharing-plus"],
    "pluginsConfig": {
        "sharing": {
            "douban": false,
            "facebook": false,
            "google": true,
            "hatenaBookmark": false,
            "instapaper": false,
            "line": true,
            "linkedin": true,
            "messenger": false,
            "pocket": false,
            "qq": false,
            "qzone": true,
            "stumbleupon": false,
            "twitter": false,
            "viber": false,
            "vk": false,
            "weibo": true,
            "whatsapp": false,
            "all": [
                "facebook", "google", "twitter",
                "weibo", "instapaper", "linkedin",
                "pocket", "stumbleupon"
            ]
        }
    }
}
```
