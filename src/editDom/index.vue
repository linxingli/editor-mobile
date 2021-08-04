<template>
  <div class="edit-box">
    <div class="editDom"
         :id="`editor${this.tag}`"
         @paste="onPaste"
         @input="onInput"
         ref="htmlContent">
    </div>
    <!-- 面板弹窗 -->
    <van-popup v-model="show"
               :overlay="false"
               :lazy-render="false"
               :close-on-click-overlay="false"
               get-container="body"
               position="bottom">
      <div class="toolbar-box"
           @click.stop="clickToolbar">
        <div class="toolbar-header">
          <span class="toolbar-title">文本编辑</span>
          <img @click="show = false"
               class="close-btn"
               src="./images/close-icon.png" />
        </div>
        <div :id="`toolbar${tag}`"
             class="toolbar-body">
          <!-- 文本操作按钮 -->
          <van-row>
            <template v-for="(item , index) in buttons">
              <van-col span="4"
                       v-if="[0,1,2,3,4,5].includes(index)"
                       :key="item">
                <img class="text-btn"
                     :ref="item"
                     :src="require(`./images/${item}${toolbarStatus[item] ? '-active':''}.png`)">
              </van-col>
            </template>
          </van-row>

          <div class="slice-line"></div>

          <van-row>
            <template v-for="(item , index) in buttons">
              <van-col span="4"
                       v-if="[6,7,8,9].includes(index)"
                       :key="item">
                <img class="text-btn"
                     :ref="item"
                     :src="require(`./images/${item}${toolbarStatus[item] ? '-active':''}.png`)">
              </van-col>
            </template>
          </van-row>
          <div class="slice-line"></div>
          <!-- 滑动变阻器-字体大小 -->
          <div class="size-slider">
            <img class="small-size"
                 src="./images/small-size.png"
                 alt="">
            <van-slider class="slider"
                        v-model="toolbarStatus.size"
                        @change="sizeChange"
                        :step="1"
                        :min="1"
                        :max="4" />
            <img class="big-size"
                 src="./images/big-size.png"
                 alt="">
          </div>
          <div class="slice-line"></div>
          <!-- 颜色按钮 -->
          <div class="color-box">
            <div class="color-item"
                 v-for="item in colors"
                 :key="item">
              <span :ref="item"
                    :class="toolbarStatus[item] ? 'active' : ''"
                    :style="{backgroundColor: item}"></span>
            </div>
          </div>
        </div>
      </div>
    </van-popup>
  </div>
</template>

<script>
  import { initToolbarStatus } from './utils'
  const Quill = window.Quill

  export default {
    // 实现edit-dom的v-model指令
    model: {
      innerHTML: '',
      event: 'change'
    },
    props: {
      placeholder: {
        type: String,
        default: ''
      },
      // 当同一页面有多个编辑器时，传入不同tag起区分编辑器的作用
      tag: {
        type: [String, Number],
        default: 1
      },
    },
    data () {
      return {
        textContent: '',
        htmlContent: '',
        len: 0,
        show: false,
        quill: null,
        toolbarStatus: initToolbarStatus(),
        // 文本样式
        buttons: ['bold', 'italic', 'underline', 'left', 'center', 'right', 'indentLeft', 'indentRight', 'listOrdered', 'listBullet'],
        // 文字颜色
        colors: [
          // red,
          '#f92b2d',
          // yellow,
          '#ffbe00',
          // green,
          '#47cd48',
          // skyblue,
          '#04aaef',
          // blue,
          '#3e56ed',
          // purple,
          '#8a2be1',
          // blacke
          '#333333'
        ],
        isClickToolbar: false, // 是否点击工具面板
        isClickColor: false, // 是否点击颜色按钮
        selectionStatus: null, // 选区状态
      }
    },
    watch: {
      show () {
        if (!this.show) {
          this.selectionStatus = null
        } else {
          this.quill.blur()
          setTimeout(() => {
            this.clickToolbar()
            this.quill.focus()
            this.show = true
          }, 200)
        }
      }
    },
    mounted () {
      this.quill = new Quill(`#editor${this.tag}`, {
        modules: {
          toolbar: `#toolbar${this.tag}`
        },
        placeholder: this.placeholder,
      })
      this.$refs.htmlContent.querySelector('.ql-editor').addEventListener('blur', () => {
        this.show = false
        this.$emit('blur')
      })
      this.$refs.htmlContent.querySelector('.ql-editor').addEventListener('focus', () => {
        // 判断是否为点击面板按钮导致的focus
        if (this.isClickToolbar) {
          // 阻止键盘弹起
          // this.quill.blur()
        } else {
          // console.log('Text focus!');
          this.show = false
        }
        this.$emit('focus')
      })

      this.addToolbarHandler()
      this.getFormat()
      this.contentChange()
    },
    methods: {
      onInput () {
        this.$emit('input')
      },
      focus () {
        this.$refs.htmlContent.querySelector('.ql-editor').focus()
        const length = this.htmlContent.length
        this.quill.setSelection(length, length)
      },
      contentChange () {
        this.quill.on('editor-change', (eventName) => {
          if (eventName === 'text-change') {
            // 获取内容
            this.htmlContent = this.$refs.htmlContent.querySelector('.ql-editor').innerHTML
            this.textContent = this.$refs.htmlContent.querySelector('.ql-editor').textContent
            this.handleChange(this.textContent, this.htmlContent)
            // 删除完内容后面板状态复位
            let ops = this.quill.getContents().ops
            if (ops && ops.length === 1 && ops[0].insert === '\n' && !ops[0].attributes) {
              this.toolbarStatus = initToolbarStatus()
            }
          }
        })
      },
      showToolbar () {
        // 判断是否有焦点，没有则帮忙focus, 为了实现点击其他编辑器时失焦隐藏toolbar
        if (!this.quill.hasFocus()) {
          this.clickToolbar()
          this.quill.focus()
          this.show = true
        } else {
          this.clickToolbar()
          this.show = true
        }

        // 赋值选区状态
        if (this.selectionStatus && this.selectionStatus.length) {
          let { index, length } = this.selectionStatus
          setTimeout(() => {
            this.quill.setSelection(index, length)
          }, 100)
        }
      },
      getFormat () {
        this.quill.on('selection-change', (range, oldRange, source) => {
          let timer = setTimeout(() => {
            if (this.isClickToolbar) return false
            if (range) {
              console.log('range-oldRange-source', range, oldRange, source)
              // 记录选区状态
              if (range.length) {
                this.selectionStatus = range
              }
              this.toolbarStatus = initToolbarStatus()
              let style = this.quill.getFormat()
              // {
              //   align: "center"
              //   bold: true
              //   color: "green"
              //   indent: 2
              //   italic: true
              //   list: "bullet"
              //   size: "small"
              //   underline: true
              // }
              // 获取style对象，给toolbarStatus同步赋值
              Object.keys(style).forEach(v => {
                switch (v) {
                  case 'align':
                    this.toolbarStatus[style[v]] = true
                    break;
                  case 'underline':
                    this.toolbarStatus[v] = true
                    break;
                  case 'color':
                    // console.log('style[v]', style[v]);
                    this.toolbarStatus['#333333'] = false
                    this.toolbarStatus[style[v]] = true
                    break;
                  case 'bold':
                  case 'italic':
                    this.toolbarStatus[v] = style[v]
                    break;
                  case 'list':
                    if (style[v] === 'bullet') {
                      this.toolbarStatus.listBullet = true
                    } else if (style[v] === 'ordered') {
                      this.toolbarStatus.listOrdered = true
                    }
                    break;
                  case 'size':
                    if (style[v] === 'small') {
                      this.toolbarStatus[v] = 1
                    } else if (style[v] === 'large') {
                      this.toolbarStatus[v] = 3
                    } else if (style[v] === 'huge') {
                      this.toolbarStatus[v] = 4
                    }
                    break;
                }
              })
            } else {
              this.show = false
            }
            clearTimeout(timer)
          }, 10)
        })
      },
      sizeChange (val) {
        let size = false
        switch (val) {
          case 1:
            size = 'small'
            break;
          case 2:
            size = false
            break;
          case 3:
            size = 'large'
            break;
          case 4:
            size = 'huge'
            break;
          default:
            break;
        }
        this.clickToolbar()
        this.toolbarStatus.size = val
        this.quill.format('size', size)
      },
      clickToolbar () {
        this.$refs.htmlContent.querySelector('.ql-editor').setAttribute('contenteditable', false)
        this.isClickToolbar = true
        let timer = setTimeout(() => {
          this.$refs.htmlContent.querySelector('.ql-editor').setAttribute('contenteditable', true)
          this.isClickToolbar = false
          clearTimeout(timer)
        }, 320);
      },
      // 富文本面板按钮事件
      addToolbarHandler () {
        this.buttons.forEach(item => {
          this.$refs[item][0].addEventListener('touchstart', () => {
            this.clickToolbar()
            if (['bold', 'italic', 'underline'].includes(item)) {
              this.toggleHandle(item)
            } else if (['left', 'center', 'right'].includes(item)) {
              this.alignHandle(item)
            } else if (['indentLeft', 'indentRight'].includes(item)) {
              this.indentHandle(item)
            } else if (['listOrdered', 'listBullet'].includes(item)) {
              this.listHandle(item)
            }
          })
        })
        this.colors.forEach(item => {
          this.$refs[item][0].addEventListener('click', (e) => {
            e.stopPropagation()
            this.clickToolbar()
            this.colorHandle(item)
          })
        })
      },
      listHandle (item) {
        let v = item === 'listOrdered' ? 'listBullet' : 'listOrdered'
        this.toolbarStatus[v] = false
        this.toolbarStatus[item] = !this.toolbarStatus[item]
        if (item === 'listOrdered') {
          this.quill.format('list', this.toolbarStatus[item] ? 'ordered' : false)
        } else {
          this.quill.format('list', this.toolbarStatus[item] ? 'bullet' : false)
        }
      },
      indentHandle (item) {
        let arr = ['indentLeft', 'indentRight']
        arr.forEach(v => {
          this.toolbarStatus[v] = false
        })
        this.toolbarStatus[item] = true
        this.quill.format('indent', item === 'indentLeft' ? '-1' : '+1')
      },
      alignHandle (item) {
        let arr = ['left', 'center', 'right']
        arr.forEach(v => {
          this.toolbarStatus[v] = false
        })
        this.toolbarStatus[item] = true
        switch (item) {
          case 'left':
            this.quill.format('align', false)
            break;
          case 'center':
            this.quill.format('align', this.toolbarStatus[item] ? 'center' : false)
            break;
          case 'right':
            this.quill.format('align', this.toolbarStatus[item] ? 'right' : false)
            break;
        }
      },
      colorHandle (item) {
        // 排他
        this.colors.forEach(v => {
          this.toolbarStatus[v] = v === item ? true : false
        })
        this.quill.format('color', item)
      },
      toggleHandle (item) {
        this.toolbarStatus[item] = !this.toolbarStatus[item]
        this.quill.format(item, this.toolbarStatus[item])
      },
      setHtmlContent (html) {
        this.htmlContent = html
        // html = html.replace(/<\/div>/g, '')
        // html = html.replace(/<div>/g, '\n')
        let delta = this.quill.clipboard.convert(html) // 把html转换成delta，因为quill只识别delta
        delta.forEach(v => {
          if (typeof v.insert === 'string') {
            v.insert = v.insert.replace(/(\n){2,}/, '\n')
          }
        })
        this.quill.setContents(delta)
        this.$nextTick(() => {
          this.htmlLength()
        })
      },
      onPaste (e) {
        e.preventDefault()
        const text = e.clipboardData.getData('text/plain')
        document.execCommand('insertText', false, text)
      },
      onFocus () {
        this.$emit('focus', this.$refs.htmlContent)
      },
      onBlur () {
        this.$emit('blur')
      },
      handleChange (textContent, innerHTML) {
        this.textContent = textContent
        this.len = textContent.length
        if (!textContent.length) {
          this.htmlContent = ''
        }
        this.$emit('change', innerHTML)
        this.$emit('update:length', this.len)
      },
      htmlLength () {
        let textContent = this.$refs.htmlContent.querySelector('.ql-editor').textContent || ''
        this.textContent = textContent
        this.len = textContent.length
        this.$emit('update:length', this.len)
      },
    }
  }
</script>

<style lang="less">
  ::selection {
    background-color: #9fd8f0;
  }
  .edit-box {
    position: relative;
    width: 100%;
    margin-top: -20px;

    .placeholder {
      position: absolute;
      top: 0;
      left: 0;
      font-size: 30px;
      color: #999999;
      pointer-events: none;
    }
    .editDom {
      user-select: auto !important;
      font-family: -apple-system, BlinkMacSystemFont, "PingFang SC",
        "Helvetica Neue", STHeiti, "Microsoft Yahei", Tahoma, Simsun, sans-serif;
      width: 100%;
      font-size: 30px;
      // min-height: 120px;
      margin-left: auto;
      margin-right: auto;
      outline: none;
      line-height: 1.3;
      overflow: hidden;
      z-index: 100;
      // -webkit-user-modify: read-write-plaintext-only
      border: 1px solid #eee;
      padding: 16px;
      border-radius: 8px;
    }
  }
  .ql-editor.ql-blank::before {
    color: #a5a5a5;
    content: attr(data-placeholder);
    font-style: inherit;
    pointer-events: none;
    position: absolute;
    left: 0;
    right: 0;
    font-size: 26px;
  }
  .toolbar-box {
    background: #f8f8f9;
    height: 100%;
    padding-top: 30px;
    user-select: none;
    border-top: 1px solid #eeeeee;

    .toolbar-header {
      height: 50px;
      display: flex;
      align-items: center;
      justify-content: space-between;
      padding: 0 44px;
      padding-left: 60px;
      border-top: 1px solid #f8f8f9;
      .toolbar-title {
        font-size: 26px;
        color: #333333;
      }
      .close-btn {
        width: 19px;
        height: 19px;
        padding: 20px;
      }
    }
    .toolbar-body {
      padding: 0px 20px 20px 20px;
      .van-col {
        text-align: center;
      }
      .text-btn {
        width: 44px;
        height: 44px;
        margin: 28px 0;
      }
      .size-slider {
        height: 50px;
        padding: 30px 0;
        position: relative;
        margin: 10px 32px;
        display: flex;
        justify-content: space-around;
        align-items: center;
        .slider {
          padding: 0 30px;
          background: none;
          .van-slider__bar {
            background-color: #7972fe;
          }
        }
        > img {
          width: 50px;
          height: 50px;
        }
      }
      .color-box {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin: 0 32px;
        .color-item {
          text-align: center;
          margin: 32px 0;
          > span {
            display: inline-block;
            width: 42px;
            height: 42px;
            border-radius: 50%;
            background: #333333;
            position: relative;
            &.active {
              &::before {
                content: " ";
                position: absolute;
                top: 50%;
                left: 50%;
                transform: translate(-50%, -50%);
                width: 23px;
                height: 23px;
                border-radius: 50%;
                background: #ffffff;
              }
            }
          }
        }
      }
      .slice-line {
        height: 1px;
        background: #d8d8d8;
        margin: 0 30px;
      }
    }
  }
  .ql-editor {
    padding: 0 !important;
    ol,
    ul {
      padding-left: 0;
      list-style: none !important;
      margin-left: 0 !important;
    }
    .ql-indent-8:not(.ql-direction-rtl) {
      padding-left: 22em;
    }
    &::placeholder {
      font-size: 13px;
      color: violet;
    }
  }
</style>
