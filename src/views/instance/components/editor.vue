<template>
  <div id="editor">
    <codemirror :style="fontStyle" :options="codeOptions" :value="code" v-model="code"
      @cursorActivity="cursorPosChanged" class="code" :class="mdToolbarVisible&&index===0?'md-active':''"
      ref="codeArea">
    </codemirror>
  </div>
</template>

<script>
import { mapState, mapMutations } from 'vuex'
import { codemirror } from 'vue-codemirror'
import cmConfig, { formatCode } from '@utils/editor/editorOpts'
import { judgeMode, modeStyleList } from '@utils/editor/judgeMode'
import { debounce } from '@utils/tools'
import { changeFormatOptions } from '@utils/editor/codeFormatter'
export default {
  props: {
    codeMode: String,
    index: Number,
    showCodeArea: Boolean,
  },
  data() {
    this.lintList = ['HTML', 'CSS', 'JavaScript']
    return {
      codeOptions: {},
      code: '',
    }
  },
  created() {
    this.initEditor()
    this.code = this.codeContent
  },
  mounted() {
    const cm = this.getCodeMirror()
    if (judgeMode(this.codeMode) !== 'HTML') {
      cm.on('inputRead', this.autoComplete)
    }
    if (this.index === 2) {
      cm.on('cursorActivity', (cm) => {
        window.ternServer.updateArgHints(cm)
      })
    }
    setTimeout(() => {
      if (this.showCodeArea) {
        const codeArea = this.$refs.codeArea
        if(codeArea) {
          codeArea.refresh()
          codeArea.codemirror.focus()
        }
      }
    })
  },
  computed: {
    ...mapState([
      'instanceCode',
      'instanceSetting',
      'mdToolbarVisible',
      'curTab',
      'hasUploadCode',
      'shouldResetCode'
    ]),
    fontStyle() {
      const { family: fontFamily, size: fontSize } = this.instanceSetting.font
      return {
        fontFamily,
        fontSize: `${fontSize}px`,
      }
    },
    codeContent() {
      const mode = judgeMode(this.codeMode)
      return this.instanceCode[mode]
    },
  },
  watch: {
    hasUploadCode(status) {
      if (status) this.code = this.codeContent
    },
    codeMode(newMode) {
      const codeOptions = this.codeOptions
      codeOptions.mode = modeStyleList[newMode]
      codeOptions.lint = this.getLintOpts(newMode)
    },
    shouldResetCode(newState) {
      if (newState) {
        const codeMode = this.codeMode
        const mode = judgeMode(codeMode)
        this.code = this.instanceCode[mode]
        this.$nextTick(() => {
          this.setShouldResetCode(false)
        })
      }
    },
    showCodeArea(newState) {
      // ????????????????????????????????????
      if (newState) {
        const codeArea = this.$refs.codeArea
        codeArea.refresh()
        codeArea.codemirror.focus()
      }
    },
    instanceCode() {
      this.code = this.codeContent
    },
    curTab() {
      // ?????????????????????????????????????????????????????????????????????
      if (!this.showCodeArea) return void 0
      const cm = this.getCodeMirror()
      this.cursorPosChanged(cm)
    },
    'instanceSetting.autoComplete'(newState) {
      // ???????????????input???????????????hint????????????
      const cm = this.getCodeMirror()
      if (newState && judgeMode(this.codeMode) !== 'HTML') {
        cm.on('inputRead', this.autoComplete)
      } else {
        cm.off('inputRead', this.autoComplete)
      }
    },
    'instanceSetting.indent.replace'(newState) {
      this.codeOptions.indentWithTabs = newState
    },
    'instanceSetting.indent.width'(newState) {
      const codeOptions = this.codeOptions
      codeOptions.tabSize = newState
      codeOptions.indentUnit = newState
      changeFormatOptions({
        attr: 'tabWidth',
        val: newState,
      })
    },
    'instanceSetting.lineWrap'(newState) {
      this.codeOptions.lineWrapping = newState
    },
    'instanceSetting.lint'(newState) {
      this.codeOptions.lint = newState && this.getLintOpts(this.codeMode)
    },
    'instanceSetting.autoExecute'(newState) {
      this.watchCode()
      const setting = this.instanceSetting
      this.watchCode = this.$watch(
        'code',
        debounce(function (code) {
          const mode = judgeMode(this.codeMode)
          this.handleInstanceCode({ mode, code })
          if (newState) this.$emit('runCode')
        }, newState ? setting.delayTime : 500)
      )
    },
    'instanceSetting.delayTime'(newState) {
      this.watchCode()
      const setting = this.instanceSetting
      this.watchCode = this.$watch(
        'code',
        debounce(function (code) {
          const mode = judgeMode(this.codeMode)
          this.setInstanceCode({ mode, code })
          if (setting.autoExecute) this.$emit('runCode', true)
        }, setting.autoExecute ? newState : 500)
      )
    },
    'instanceSetting.font.size'(newSize) {
      const codeArea = this.$refs.codeArea
      codeArea.refresh()
    },
  },
  methods: {
    ...mapMutations(['setInstanceCode', 'setShouldResetCode']),
    initEditor() {
      // ????????????????????????
      const instanceCode = this.instanceCode
      const codeMode = this.codeMode
      const mode = judgeMode(codeMode)
      this.codeOptions = cmConfig(codeMode)
      const codeOptions = this.codeOptions
      codeOptions.mode = modeStyleList[codeMode]
      codeOptions.lint = this.getLintOpts(codeMode)
      this.code = instanceCode[mode]
      // ?????????????????????????????????????????????????????????????????????????????????????????????????????????
      const setting = this.instanceSetting
      this.watchCode = this.$watch(
        'code',
        debounce(function (code) {
          const mode = judgeMode(this.codeMode)
          this.setInstanceCode({ mode, code })
          if (this.instanceSetting.autoExecute) this.$emit('runCode', true)
        }, setting.autoExecute ? setting.delayTime : 500)
      )
    },
    getCodeMirror() {
      return this.$refs.codeArea.codemirror
    },
    format() {
      formatCode(this.getCodeMirror(), this.codeMode)
    },
    autoComplete(cm, changeObj) {
      // ???????????????????????????????????????????????????????????????????????????
      if (cm.state.completionActive) return void 0
      const token = cm.getTokenAt(cm.getCursor())
      if (token.type && token.type !== 'comment') {
        if (/^[a-zA-Z]/.test(changeObj.text[0])) {
          cm.showHint()
        }
      }
    },
    getLintOpts(codeMode) {
      let lint = false
      if (this.lintList.includes(codeMode)) {
        switch (codeMode) {
          case 'JavaScript': {
            lint = {
              esversion: 2021,
            }
            break
          }
          default: {
            lint = true
          }
        }
      }
      return this.instanceSetting.lint && lint
    },
    cursorPosChanged(cm) {
      // ?????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????
      if (cm.somethingSelected()) {
        const selectArr = cm.listSelections()
        const selections = selectArr.length
        const selectVal = cm.getSelection()
        let selected = 0
        if (selections > 1) {
          // ????????????????????????????????????????????????????????????????????????????????????????????????
          selected = selectVal.length - selections + 1
          this.$emit('cursorPosChanged', {
            selections,
            selected,
          })
        } else {
          selected = selectVal.length
          this.$emit('cursorPosChanged', {
            ...this.getCursorPos(cm),
            selected,
          })
        }
      } else {
        this.$emit('cursorPosChanged', this.getCursorPos(cm))
      }
    },
    getCursorPos(cm) {
      const cursor = cm.getCursor()
      let { ch: col, line: ln } = cursor
      col++
      ln++
      return { col, ln }
    },
  },
  components: {
    codemirror,
  },
}
</script>

<style lang="scss" scoped>
::v-deep.code {
  width: 100%;
  height: 100%;
  overflow: hidden;
  ::-webkit-scrollbar {
    outline: none;
    width: 12px;
    height: 12px;
    background-color: transparent;
    @include setTransition(all, 0.3s, ease);
  }
  ::-webkit-scrollbar-track {
    background-color: rgba(30, 30, 30, 0);
  }
  ::-webkit-scrollbar-thumb {
    background-color: rgba(80, 80, 80, 0.3);
  }
  ::-webkit-scrollbar-thumb:hover {
    outline: none;
    background-color: rgba(80, 80, 80, 0.7);
  }
  .CodeMirror {
    height: calc(100vh - 91px) !important;
    resize: none;
    outline: none;
    border: none;
    font-family: inherit;
    font-size: inherit;
  }
  .CodeMirror-scroll {
    height: 100%;
    overflow-y: hidden;
  }
}
::v-deep.md-active {
  .CodeMirror {
    height: calc(100vh - 121px) !important;
  }
}
#editor {
  width: 100%;
  height: calc(100% - 30px);
}
</style>
