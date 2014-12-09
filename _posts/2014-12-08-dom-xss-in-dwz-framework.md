---
layout: post
title: DWZ富客户端框架DOM XSS分析
category: 技术
tags: Infosec
description: DWZ富客户端框架DOM XSS分析
---

### 简述

公司部分后台使用DWZ，在一次代码审计过程中发现了DWZ框架使用不当可造成的XSS漏洞。这个XSS漏洞也让我意识到之前对DOM XSS的诸多误解。

### 过程分析

公司的某个应用大概过程是这样的：用户在前端发帖，管理员可以在后台审核帖子内容。而此应用采用的是输入过滤的方式，重写了java的HttpServletRequest类，在里面加上了转义函数，即输入过滤的方式，（这里就不吐槽输入过滤的种种缺点了）。

问题出在后台的一处话题报表中，其中话题标题为：&lt;script&gt;alert(42873)&lt;/script&gt;如下图：

![后台报表](/images/20141208164721.jpg)

    //HTML代码
    <td>&lt;script&gt;alert(42873)&lt;/script&gt;</td>
    <td>2</td>
    <td>16(0)</td>
    <td>0</td>
    <td>0</td>
    <td><a href="...." target="navTab" title="&lt;script&gt;alert(42873)&lt;/script&gt;" style="color: blue;">查看</a></td>

因为是输入过滤，所以表面上没有任何的XSS，但当我们点击查看的时候却可以触发XSS，原因就在于DWZ框架，当点击查看的时候DWZ会检查之前有没有打开过，没有打开就新建一个tab，如果打开过就跳转到之前的tab，代码如下：

    //dwz.ui.js
    $("a[target=navTab]", $p).each(function(){
        $(this).click(function(event){
            var $this = $(this);
            var title = $this.attr("title") || $this.text();
            var tabid = $this.attr("rel") || "_blank";
            var fresh = eval($this.attr("fresh") || "true");
            var external = eval($this.attr("external") || "false");
            var url = unescape($this.attr("href")).replaceTmById($(event.target).parents(".unitBox:first"));
            DWZ.debug(url);
            if (!url.isFinishedTm()) {
                alertMsg.error($this.attr("warn") || DWZ.msg("alertSelectMsg"));
                return false;
            }
            navTab.openTab(tabid, url,{title:title, fresh:fresh, external:external});

            event.preventDefault();
        });
    });

    //dwz.navTab.js
    openTab: function(tabid, url, options){ //if found tabid replace tab, else create a new tab.
        var op = $.extend({title:"New Tab", type:"GET", data:{}, fresh:true, external:false}, options);
        var iOpenIndex = this._indexTabId(tabid);
        if (iOpenIndex >= 0){
            ...
        } else {
            var tabFrag = '<li tabid="#tabid#"><a href="javascript:" title="#title#" class="#tabid#"><span>#title#</span></a><a href="javascript:;" class="close">close</a></li>';
            //触发点1
            this._tabBox.append(tabFrag.replaceAll("#tabid#", tabid).replaceAll("#title#", op.title));
            this._panelBox.append('<div class="page unitBox"></div>');
            //触发点2
            this._moreBox.append('<li><a href="javascript:" title="#title#">#title#</a></li>'.replaceAll("#title#", op.title));
            ...

通过jQuery的attr方法获取a标签的title属性，并提交给openTab方法，openTab方法将标题放到li中。

### 问题出在哪？
title是之前已经过滤的，但当使用jQuery的attr方法取出a标签的title属性时，这是获取到的title已经经过DOM变换成了&lt;script&gt;alert(42873)&lt;/script&gt;，被转回来了，因此才会触发。

### 误解一
我以为是因为jQuery的attr方法的原因导致此处title的变化，后经过测试，使用js的getAttribute方法同样获取的到也是转回来的title，后查看余弦的《web前端黑客技术揭秘》才发现title是因为DOM渲染才变化的。

### 误解二
之前以为DOM XSS只是\74script\76alert(42873)\74/script\76在js中被解析产生的XSS，除了a标签的href和一些事件属性标签（onXXX）解析转义后的HTML会触发XSS，忽略了其他情况，现在想想，实际上原理一样。

### DOM XSS示例
这里附上《Web前端黑客技术揭秘》中的DOM XSS示例，能够更好的理解DOM XSS。

    //不会触发
    <input type="button" id="exec_btn" value="exec" onclick="document.write(HtmlEncode('<img src=@ onerror=alert(123) />'))" />
    //可触发
    <input type="button" id="exec_btn" value="exec" onclick="document.write('&lt;img src=@ onerror=alert(123)) /&gt;'>" />

仔细对比上述示例中的代码，就会明白DOM渲染在DOM XSS中起到的作用。理解DOM XSS的关键在于理解**HTML解析是先于JS解析**的。
