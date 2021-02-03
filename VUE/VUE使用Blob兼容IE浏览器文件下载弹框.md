Excel文件下载

```vue
<el-button
	class="filter-item"
    type="warning"
    icon="el-icon-download"
    :loading="downVisible"
    @click="downLoadTaskList"
>导出</el-button>
```

```js
downLoadTaskList() {
    this.downVisible = true
    download('api/project/task/downList', this.listQuery).then(result => {
        this.downloadFile(result, '任务进度', 'xlsx')
        this.downVisible = false
    }).catch(() => {
        this.downVisible = false
    })
}
```

点击**导出**按钮，发送导出请求，其中download，downloadFile函数源码如下：

```js
function download(url, params) {
  return request({
    url: url + '?' + qs.stringify(params, { indices: false }),
    method: 'get',
    responseType: 'blob'
  })
}

function downloadFile(obj, name, suffix) {
  const blob = new Blob([obj], {
    type: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet;charset=utf-8'
  })
  if (!!window.ActiveXObject || 'ActiveXObject' in window) {
    const fileName = parseTime(new Date()) + '-' + name + '.' + suffix
    window.navigator.msSaveOrOpenBlob(blob, fileName)
  } else {
    const url = window.URL.createObjectURL(blob)
    const link = document.createElement('a')
    link.style.display = 'none'
    link.href = url
    const fileName = parseTime(new Date()) + '-' + name + '.' + suffix
    link.setAttribute('download', fileName)
    document.body.appendChild(link)
    link.click()
    document.body.removeChild(link)
  }
}
```

>  使用了Blob来存放二进制数据容器，使用!!window.ActiveXObject || 'ActiveXObject' in window来判断是否为IE浏览器，IE浏览器用window.navigator.msSaveOrOpenBlob()， 弹出下载框，其他浏览器使用document弹出。