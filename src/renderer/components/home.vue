<style lang="less">

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  .wrapper {
    padding: 20px 80px;
    .btn-gs {
      margin: 10px auto;
      width: 340px;
      height: 45px;
    }
    .ipt, .btn-upload, #zipFile, #pdfFile {
      width: 160px;
      height: 45px;
      line-height: 45px;
    }
    .ipt {
      display: inline-block;
      position: relative;
      .btn-upload {
        color: #ffffff;
        border: 1px solid #9b33e5;
        background-color: #aa5de5;
        border-radius: 5px;
      }
      overflow: hidden;
      #zipFile, #pdfFile {
        position: absolute;
        top: 0;
        left: 0;
        bottom: 0;
        right: 0;
        color: transparent;
        opacity: 0;
      }
    }

    .tip {
      margin-top: 60px;
      ul, li {
        list-style: none;
      }
    }
  }

</style>
<template>
  <div class="wrapper">
    <div class="btn-gs">
      <div class="ipt">
        <button class="btn-upload">
          <span>解压ZIP压缩包</span>
        </button>
        <input type="file" accept="application/zip" id="zipFile" @change="openZip">
      </div>
      <div class="ipt" style="float: right;">
        <button class="btn-upload">
          <span>合并PDF文件</span>
        </button>
        <input type="file" accept="application/zip" id="pdfFile" @change="mergePdf" multiple>
      </div>
    </div>
    <div class="tip">
      <p style="margin-bottom: 20px;">说明:</p>
      <ul>
        <li>
          1.请上传标准的发票zip文件，结构如图
          <br>
          <img src="../assets/tip-1.jpg" alt="" width="420">
        </li>
        <li>
          2.解压完成后，会自动打开发票文件所在的文件夹
        </li>
        <li>
          3.合并PDF文件的功能需要Java的环境(一般的电脑都默认安装，如果没有提示安装不需要管)
        </li>
        <li>
          4.合并文件之后，会自动打开该文件。合并的文件位于刚刚打开的文件中，名字为:"合并后的文件-"+一串数字
        </li>
      </ul>
    </div>
  </div>
</template>

<script>
  import fs from 'fs'
  import AdmZip from 'adm-zip'
  import {shell} from 'electron'
  import PDFMerge from './easy-pdf-merge/index'
  import rimraf from 'rimraf'

  export default {
    name: 'home-page',
    data () {
      return {
        pdfList: []
      }
    },
    methods: {
      openZip () {
        let $file = document.getElementById('zipFile')
        let file = $file.files[0]
        let filePath = file.path
        let outPath = filePath.replace(/.zip$/g, '/')
        let admObj = this.getEntries(filePath)
        // console.log(outPath)
        admObj.entries.forEach((entry) => {
          if (this.isZipFile(entry)) {
            // console.log(entry)
            admObj.example.extractEntryTo(entry.entryName, outPath, true, true)
          }
        })
        // admObj.example.extractAllTo(outPath, true)
        // 重置输入框
        $file.value = ''
        setTimeout(() => {
          // 解压官网标准包 得到 含有发票的 zip包 同时忽略清单
          fs.readdir(outPath, (err, files) => {
            if (err) {
              console.log(err)
            } else {
              files.forEach((filename) => {
                if (this.isZipFile(filename)) {
                  let filePath = outPath + filename
                  let admObj = this.getEntries(filePath)
                  // admObj.example.extractAllTo(outPath + 'PDF/', true)
                  admObj.entries.forEach((entry) => {
                    if (this.isZipFile(entry.name)) {
                      admObj.example.extractEntryTo(entry.entryName, outPath + 'PDF/', false, true)
                    }
                  })
                }
              })
            }
          })
        }, 100)
        // 解压官网的发票zip包
        // console.log(outPath + 'PDF/')
        setTimeout(() => {
          fs.readdir(outPath + 'PDF/', (err, files) => {
            if (err) {
              console.log(err)
            } else {
              files.forEach((filename) => {
                // console.log('-->', filename)
                if (this.isZipFile(filename)) {
                  let filePath = outPath + 'PDF/' + filename
                  let admObj = this.getEntries(filePath)
                  admObj.entries.forEach((entry) => {
                    if (this.isPdfFile(entry.name)) {
                      admObj.example.extractEntryTo(entry.entryName, outPath, false, true)
                    }
                  })
                }
              })

              setTimeout(() => {
                // 移除多余文件
                fs.readdir(outPath, (err, files) => {
                  if (err) {
                    console.log(err)
                  } else {
                    files.forEach((filename) => {
                      let filePath = outPath + filename
                      console.log(!this.isPdfFile(filename), filePath)
                      if (/PDF$/.test(filename)) {
                        rimraf(filePath, () => {
                          console.log('rimraf:', filename)
                        })
                      } else if (/^[._]\S*[.pdf]$/.test(filename) || !this.isPdfFile(filename)) {
                        fs.unlinkSync(filePath)
                      }
                    })
                  }
                })
              }, 100)
            }
          })
        }, 200)

        setTimeout(() => {
          shell.openItem(outPath)
        }, 300)
      },
      mergePdf () {
        let $file = document.getElementById('pdfFile')
        let files = $file.files
        let pathList = []
        let outPath = files[0].path.split('/')
        outPath.splice(outPath.length - 1, 1)
        outPath = outPath.join('/')
        for (let file of files) {
          pathList.push(file.path)
        }
        if (pathList.length < 2) {
          alert('最少需要选择2个文件')
          return
        }
        //Save as new file
        // PDFMerge(pathList, {output: (outPath + '/合并后的文件-' + +new Date() + '.pdf')})
        //   .then((buffer) => {
        //     console.log(buffer)
        //     shell.openItem(outPath)
        //   })

        console.log(pathList)
        PDFMerge(pathList, outPath + '/合并后的文件-' + +new Date() + '.pdf', (err) => {
          if (err) {
            alert(err)
          } else {
            shell.openItem(outPath)
          }
        })
        $file.value = ''
      },
      getEntries (filePath) {
        if (filePath) {
          let admZip = new AdmZip(filePath)
          return {
            example: admZip,
            entries: admZip.getEntries()
          }
        }
      },
      isZipFile (entry) {
        let filename = entry
        if (typeof entry === 'object') {
          filename = entry.entryName
          if (entry.isDirectory) {
            return false
          }
        }
        return filename.indexOf('.zip') > 0 && filename.indexOf('__MACOSX') === -1
      },
      isPdfFile (name) {
        return /.pdf$/g.test(name)
      }

    },
    mounted () {

    }

  }
</script>
