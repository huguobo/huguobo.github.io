---
title: H5实现图片预览的两种方式和区别
date: 2019-09-24 19:28:44
categories: 
- JavaScript
tags:
- JS
- File
- HTML5
---

> 在实际的业务场景里经常会用到图片或者视频预览的需求，之前一直使用FileReader.readAsDataURL()实现，现在发现URL.createObjectURL()也有同样的能力，那么使用他们两有啥不同呢？

## File
首先大概说下 File 对象，它是一个特殊的 [blob](https://developer.mozilla.org/zh-CN/docs/Web/API/Blob) 类型，围绕文件上传的场景有特别多神奇的[用法](https://developer.mozilla.org/en-US/docs/Web/API/File/Using_files_from_web_applications)。


### 属性
- File.lastModified： 返回当前 File 对象所引用文件最后修改时间，自 UNIX 时间起始值（1970年1月1日 00:00:00 UTC）以来的毫秒数。
- File.lastModifiedDate： 返回当前 File 对象所引用文件最后修改时间的 Date 对象。
- File.name: 文件名
- File.type: 返回文件的 MIME TYPE
- File.size: 返回文件的大小

### 方法
File 接口没有定义任何方法，但是它从 Blob 接口继承了以下方法：
一般为了兼容各个浏览器我们要做如下的处理来使用
```JavaScript
var blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;
```

## FileReader.readAsDataURL()
[FileReader](https://developer.mozilla.org/zh-CN/docs/Web/API/FileReader)也有特别多的使用方法，就不详细展开了（我也记不住），FileReader.readAsDataURL(file)得到的事一段base64的字符串，，主要说下我们使用readAsDataURL实现图片预览的情况：
```Javascript
const readerFile = file => new Promise((resolve, reject) => {
  const fileInfo = {
    name: file.name.substr(0, file.name.lastIndexOf('.')),
    fileSize: file.size,
    lastModified: file.lastModified,
    uploadStatus: 'uploading',
    file,
  };

  let reader = new FileReader();
  reader.onload = (ev) => {
    fileInfo.url = ev.target.result;
    fileInfo.src = ev.target.result;
    resolve(fileInfo);
    reader = null;
  };
  reader.onerror = (err) => {
    reject(err);
    reader = null;
  };
  reader.readAsDataURL(file);
});
```
这样我们从后续的fileInfo中取出src属性赋值给对应的img标签就可以了，FileReader.readAsDataURL是异步操作，有流的概念，可以结合各种框架的state传递和保存（使用react框架，省略了dom操作）。

## URL.createObjectURL()
URL.createObjectURL() 静态方法会创建一个 DOMString，其中包含一个表示参数中给出的对象的URL。这个 URL 的生命周期和创建它的窗口中的 document 绑定。这个新的URL 对象表示指定的 File 对象或 Blob 对象。
简单的理解一下就是将一个file或Blob类型的对象转为UTF-16的字符串，并保存在当前操作的document下。
```HTML
<input type="file" id="fileElem" multiple accept="image/*" style="display:none" onchange="handleFiles(this.files)">
<a href="#" id="fileSelect">Select some files</a> 
<div id="fileList">
  <p>No files selected!</p>
</div>
```
```Javascript
window.URL = window.URL || window.webkitURL;
const fileSelect = document.getElementById("fileSelect"),
    fileElem = document.getElementById("fileElem"),
    fileList = document.getElementById("fileList");

fileSelect.addEventListener("click", function (e) {
  if (fileElem) {
    fileElem.click();
  }
  e.preventDefault(); // prevent navigation to "#"
}, false);
function handleFiles(files) {
  if (!files.length) {
    fileList.innerHTML = "<p>No files selected!</p>";
  } else {
    fileList.innerHTML = "";
    const list = document.createElement("ul");
    fileList.appendChild(list);
    for (let i = 0; i < files.length; i++) {
      const li = document.createElement("li");
      list.appendChild(li);
      
      const img = document.createElement("img");
      img.src = window.URL.createObjectURL(files[i]);
      img.height = 60;
      img.onload = function() {
        window.URL.revokeObjectURL(this.src);
      }
      li.appendChild(img);
      const info = document.createElement("span");
      info.innerHTML = files[i].name + ": " + files[i].size + " bytes";
      li.appendChild(info);
    }
  }
}
```

## URL.createObjectURL() VS FileReader.readAsDataURL()
1. 返回值不同
   - FileReader.readAsDataURL(file)可以得到一段base64的字符串，长度和文件大小正相关
   - URL.createObjectURL(file)可以得到当前文件的一个内存URL。
2. 执行机制
   - FileReader.readAsDataURL(file)通过回调的形式返回，异步执行。
   - URL.createObjectURL(file)直接返回，同步执行。
3. 回收机制
   - FileReader.readAsDataURL(file)依照JS垃圾回收机制自动从内存中清理。
   - URL.createObjectURL(file)存在于当前doucment内，清除方式只有unload()事件或revokeObjectURL()手动清除 。
4. 兼容性
   - IE10以上

总体来说，使用URL.createObjectURL(file)更为方便（同步），性能优秀，但是需要考虑手动释放内存的问题。FileReader.readAsDataURL(file)胜在直接转为base64格式，更方便用于后续的其他数据操作场景（如上传），但是base64的长度会影响页面的性能。如果单纯的实现预览图片的功能，推荐使用URL.createObjectURL(file)。