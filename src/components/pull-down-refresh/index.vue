<template>
  <div class="vc-pull-down-refresh-warpper" ref="wrapper" v-swipe:vertical="swipeConfig">
    <div class="vc-pull-down-refresh-panel" ref="panel" :style="{ 'background-color': backgroundColor }">
      <div class="vc-pull-down-refresh-panel-icons" ref="icons" v-show="showMsgIcon">
        <div class="loading-circle-icon" ref="circle"></div>
        <div class="arrows-icon" ref="arrow"></div>
        <div class="done-icon" ref="done"></div>
        <div class="error-icon" ref="error"></div>
      </div>
      <div class="vc-pull-down-refresh-panel-message" ref="message"></div>
    </div>
    <div class="vc-pull-down-refresh-content" ref="content">
      <slot></slot>
    </div>
  </div>
</template>

<script>
  import { throttle } from '../../utils/utils'
  import { swipeDirective } from 'vue-swipe-directive'

  export default {
    name: 'vc-pull-down-refresh',

    created () {
      this.swipeConfig = {
        move: this._onSwipe,
        end: this._onSwipeDone
      }
    },

    /** status
     * 0 滚动状态
     * 1 未达到阈值
     * 2 达到阈值
     * 3 正在加载
     * 4 加载失败/成功
     */

    /** 事件
     * @function{onLoad} 用户一定交互之后会触发加载更多
     * @function{onScrollLoad} 滑动到底部触发的更多加载
     * @function{onPullStart} 开始触发下拉的时候
     * @function{onPullEnd} finger leave screen
     */

    data () {
      return {
        status: 0,
        startY: null,
        scrollLock: null,
        noMoreTry: false,
        noMoreScrollTry: false,
        messages: [
          '下拉刷新',
          '松开刷新',
          '正在刷新',
          '刷新成功',
          '加载失败',
          '没有更多'
        ]

        //有color选项的数据格式,课混合使用~
        // messages: [
        //   {
        //     text: '下拉刷新',
        //     color: ''
        //   },{
        //     text: '松开刷新',
        //     color: ''
        //   },{
        //     text: '正在刷新',
        //     color: ''
        //   },{
        //     text: '刷新成功',
        //     color: ''
        //   },{
        //     text: '加载失败',
        //     color: ''
        //   },{
        //     text: '没有更多',
        //     color: ''
        //   }
        // ]
      }
    },

    props: {
      showMsgIcon: {
        type: Boolean,
        default: true
      },
      triggerOffset: {
        type: Number,
        default: 40
      },
      backgroundColor: {
        type: String,
        default: 'transparent'
      },
      maxDragOffset: Number,
      customMsg: Array,
      triggerScrollLoadOffset: Number,
      throttleDelay: {
        type: Number,
        default: 100
      }
    },

    mounted () {
      if (this.$parent)
        this.$parent.$nextTick(() => {
          requestAnimationFrame(() => {
            this.$refs.panel.style.marginTop = -this.$refs.panel.clientHeight + 'px'
          })
        })
      else
        requestAnimationFrame(() => {
          this.$refs.panel.style.marginTop = -this.$refs.panel.clientHeight + 'px'
        })

      Object.assign(this.messages, this.customMsg)
      this.$refs.wrapper.onscroll = throttle(this._scroll, this.throttleDelay)
    },

    methods: {
      _onSwipe (info, lock) {
        var $wrapper = this.$refs.wrapper

        if (
          $wrapper.scrollTop !== 0 ||
          info.directionFour !== 'down'
        ) {
          this.status = 0
          return
        };

        if (this.status === 3) return

        var $panel = this.$refs.panel,
          $content = this.$refs.content,
          offset

        if (this.status === 0) {
          this.startY = info.movingY
          this.status = 1
          this.$emit('onPullStart')
        } else {
          offset = parseInt((info.movingY - this.startY) / 2)

          if (typeof this.maxDragOffset === 'number')
            offset = offset > this.maxDragOffset ? this.maxDragOffset : offset

          $panel.style.webkitTransform = `translateY(${offset}px) translateZ(0)`
          $content.style.webkitTransform = `translateY(${offset}px) translateZ(0)`

          if (this.noMoreTry === true) {
            this._noMore(false)
          } else {
            if (offset > this.triggerOffset && this.status === 1)
              this.status = 2

            if (offset < this.triggerOffset && this.status === 2)
              this.status = 1
          }
        }
        lock(true)
      },

      _onSwipeDone () {
        var $panel = this.$refs.panel,
          $content = this.$refs.content

        requestAnimationFrame(() => {
          $panel.style.webkitTransitionDuration = null
          $content.style.webkitTransitionDuration = null

          this.$emit('onPullEnd')

          requestAnimationFrame(() => {
            if (this.status === 1 || this.noMoreTry === true)
              this.status = 0
            else if (this.status === 2)
              this.status = 3
          })
        })
      },

      _success () {
        this._message(3)
        setTimeout(() => {
          this.status = 0
        }, 720)
      },

      _error () {
        this._message(4)
        setTimeout(() => {
          this.status = 0
        }, 720)
      },

      _noMore (backInital = true) {
        this._message(5, false)

        if (backInital)
          setTimeout(() => {
            this.status = 0
          }, 720)
      },

      _noMoreTry () {
        this.noMoreTry = true
      },

      _noMoreScrollTry () {
        this.noMoreScrollTry = true
        this.$refs.wrapper.onscroll = null
      },

      _message (which, withAnimation = true) {
        var $message = this.$refs.message,
          $panel = this.$refs.panel,
          $active = this.$refs.icons.getElementsByClassName('active')[0],
          msg = this.messages[which]

        $panel.setAttribute('data-status', which)
        $active && $active.classList.remove('active')

        if (typeof msg === 'object') {
          $message.innerText = msg.text
          $message.style.color = msg.color
        } else if (typeof msg === 'string') {
          $message.innerText = msg
          $message.style.color = ''
        }

        if (which === 2)
          this.$refs.circle.classList.add('active')
        else if (which === 0 || which === 1) {
          this.$refs.arrow.classList.add('active')

          if (which === 1) {
            this.$refs.arrow.classList.add('reverse')
          } else
            this.$refs.arrow.classList.remove('reverse')
        } else {
          (which === 3 || which === 5) &&
            this.$refs.done.classList.add('active')
          which === 4 &&
            this.$refs.error.classList.add('active')

          if (withAnimation)
            requestAnimationFrame(() => {
              $panel.style.opacity = 0
              $panel.style.webkitTransitionDuration = '0s'
              requestAnimationFrame(() => {
                $panel.style.opacity = 1
                $panel.style.webkitTransitionDuration = null
              })
            })
        }
      },

      //下滑到一定程度自动加载更多
      _scroll (e) {
        var $wrapper = this.$refs.wrapper,
          {scrollTop, scrollHeight, clientHeight} = $wrapper

        if (this.triggerScrollLoadOffset > (scrollHeight - clientHeight - scrollTop)) {
          this.$emit('onScrollLoad', this._noMoreScrollTry)
        }
      },

      getScrollContainer () {
        return this.$refs.wrapper
      }
    },

    watch: {
      status (val) {
        var $panel = this.$refs.panel,
          $content = this.$refs.content

        switch (val) {
          case 0:
            $panel.style.webkitTransform = null
            $content.style.webkitTransform = null
            $panel.style.visibility = null
            $panel.style.willChange = null
            $content.style.willChange = null
            break
          case 1:
            $panel.style.willChange = 'all'
            $content.style.willChange = 'all'
            $panel.style.webkitTransitionDuration = '0ms'
            $content.style.webkitTransitionDuration = '0ms'
            $panel.style.visibility = 'visible'
            if (this.noMoreTry === false)
              this._message(0)
            break
          case 2:
            this._message(1)
            break
          case 3:
            this._message(2)

            $panel.style.webkitTransform = `translateY(50.4px) translateZ(0)`
            $content.style.webkitTransform = `translateY(50.4px) translateZ(0)`

            this.$emit('onLoad', this._success, this._error, this._noMore, this._noMoreTry)
            break
        }
      }
    },

    directives: {
      swipe: swipeDirective
    }
  }
</script>

<style lang="scss">
  .vc-pull-down-refresh-warpper {
    overflow: auto;
    z-index: 0;
    position: relative;
  }

  .vc-pull-down-refresh-content{
    transition: all 250ms ease 0ms;
    z-index: 0;
    position: relative;
    height: 200vh;
    will-change: transform;
  }

  .vc-pull-down-refresh-panel{
    position: absolute;
    height: 50px;
    width: 100%;
    transition: all 250ms ease 0ms;
    display: inline-flex;
    justify-content: center;
    align-items: center;
    visibility: hidden;
    z-index: 1;
    color:#e0e0e0;
    will-change: transform;

    &[data-status='0'] {
      color: grey;
    }
    &[data-status='1'] {
      color: #252525;
    }
    &[data-status='2'] {
      color: #607d8b;
    }
    &[data-status='3'] {
      color: #1AAD19;
    }
    &[data-status='4'] {
      color: #E64340;
    }
    &[data-status='5'] {
      color: #586C94;
    }
  }

  .vc-pull-down-refresh-panel-message {
    display: inline-block;
    margin-left: 10px;
  }

  .vc-pull-down-refresh-panel-icons{
    width: 24px;
    height: 24px;
    display: inline-block;
    position: relative;
    text-align: center;

    & > div{
      position: absolute;
      opacity: 0;
    }

    & > .active{
      opacity: 1;
    }

    & .loading-circle-icon{
      width: 20px;
      height: 20px;
      border-radius: 100%;
      border: 1px solid ;
      border-top-color: transparent;
      animation: 'vc-pull-down-refresh-circle' 1s linear infinite;
    }

    & .arrows-icon{
      transition: all 0.3s ease;

      &.reverse{
        transform: rotate(180deg);
      }
    }

    & .done-icon{
      transition: opacity 0.3s ease;
      display: inline-block;
      box-sizing: content-box;
      vertical-align: middle;
      font-size: 20px;
      height: calc(100% - 2px);
      width: calc(100% - 2px);
      border: 1px solid;
      border-radius: 100%;
      position: relative;
      margin-left: -13px;

      &:before{
        content: '';
        position: absolute;
        top: 3px;
        left: 8px;
        height: 12px;
        width: 6.5px;
        border: solid;
        border-width: 0 1px 1px 0;
        transform: rotate(40deg);
      }
    }

    & .arrows-icon {
      display: inline-block;
      font-size: 2em;
      vertical-align: middle;
      height: 100%;
      border-left: 1px solid;
      position: relative;
      transition: transform .3s ease;

      &:before,
      &:after {
        content: '';
        position: absolute;
        font-size: .5em;
        width: 12.5px;
        bottom: 0px;
        border-top: 1px solid;
      }

      &:before {
        right: 1px;
        transform: rotate(50deg);
        transform-origin: right;
      }

      &:after {
        left: 0px;
        transform: rotate(-50deg);
        transform-origin: left;
      }

    }

    & .error-icon {
      display: inline-block;
      font-size: 2em;
      vertical-align: middle;
      position: relative;
      transition: transform .3s ease;
      height: calc(100% - 2px);
      width: calc(100% - 2px);
      border: 1px solid;
      border-radius: 100%;
      margin-left: -13px;

      &:before,
      &:after {
        content: '';
        position: absolute;
        font-size: .5em;
        width: 12.5px;
        top: 50%;
        left: 23%;
        border-top: 1px solid;
      }

      &:before {
        transform: rotate(45deg);
      }

      &:after {
        transform: rotate(-45deg);
      }
    }
  }

  @keyframes vc-pull-down-refresh-circle {
    0%  {transform: rotate(0deg);}
    100%{transform: rotate(1080deg);}
  }
</style>
