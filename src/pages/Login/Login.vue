<template>
  <section class="loginContainer">
    <div class="loginInner">
      <div class="login_header">
        <h2 class="login_logo">硅谷外卖</h2>
        <div class="login_header_title">
          <a href="javascript:;" :class="{on: loginWay}" @click="loginWay=true">短信登录</a>
          <a href="javascript:;" :class="{on: !loginWay}" @click="loginWay=false">密码登录</a>
        </div>
      </div>
      <div class="login_content">
        <form>
          <div :class="{on: loginWay}">
            <section class="login_message">
              <input type="tel" maxlength="11" placeholder="手机号" 
                v-model="phone" name="phone" v-validate="'required|mobile'">
              <button :disabled="!isRightPhone || computeTime>0" class="get_verification" 
                  :class="{right_phone_number: isRightPhone}" @click.prevent="sendCode">
                    {{computeTime>0 ? `短信已发送(${computeTime}s)` : '发送验证码'}}
              </button>
              <span style="color: red;" v-show="errors.has('phone')">{{ errors.first('phone') }}</span>
            </section>
            <section class="login_verification">
              <input type="text" maxlength="8" placeholder="验证码"
                v-model="code" name="code" v-validate="{required: true,regex: /^\d{6}$/}">
              <span style="color: red;" v-show="errors.has('code')">{{ errors.first('code') }}</span>
            </section>
            <section class="login_hint">
              温馨提示：未注册硅谷外卖帐号的手机号，登录时将自动注册，且代表已同意
              <a href="javascript:;">《用户服务协议》</a>
            </section>
          </div>
          <div :class="{on: !loginWay}">
            <section>
              <section class="login_message">
                <input type="text" placeholder="用户名"
                  v-model="name" name="name" v-validate="'required'">
                <span style="color: red;" v-show="errors.has('name')">{{ errors.first('name') }}</span>
              </section>
              <section class="login_verification">
                <input :type="isShowPwd ? 'text' : 'password'" placeholder="密码" 
                   v-model="pwd" name="pwd" v-validate="'required'">
                <div class="switch_button" :class="isShowPwd ? 'on' : 'off'" @click="isShowPwd = !isShowPwd">
                  <div class="switch_circle" :class="{right: isShowPwd}"></div>
                  <span class="switch_text">{{isShowPwd ? 'abc' : ''}}</span>
                </div>
                <span style="color: red;" v-show="errors.has('pwd')">{{ errors.first('pwd') }}</span>
              </section>
              <section class="login_message">
                <input type="text" maxlength="11" placeholder="验证码"
                  v-model="captcha" name="captcha" v-validate="{required: true,regex: /^[0-9a-zA-Z]{4}$/}">
                <img ref="captcha" class="get_verification" src="http://localhost:4000/captcha" alt="captcha"
                  @click="updateCaptcha">
                <span style="color: red;" v-show="errors.has('captcha')">{{ errors.first('captcha') }}</span>
              </section>
            </section>
          </div>
          <button class="login_submit" @click.prevent="login">{{$t('login_login')}}</button>
        </form>
        <a href="javascript:;" class="about_us">{{$t('login_aboutUs')}}</a>

        <br>
        <button class="login_submit" @click.prevent="toggleLocale">切换语言</button>
      </div>
      <a href="javascript:" class="go_back" @click="$router.replace('/profile')">
        <i class="iconfont icon-jiantou2"></i>
      </a>
    </div>
  </section>


</template>

<script type="text/ecmascript-6">
  // import {setInterval} from 'timers'
  import { Toast, MessageBox } from 'mint-ui'
  // import {reqSendCode, reqPwdLogin, reqSmsLogin} from '../../api'
  export default {
    data () {
      return {
        loginWay: false, // true: 短信登陆, false: 密码登陆
        phone: '', // 手机号
        code: '', // 短信验证码
        name: '', // 用户名
        pwd: '', // 密码
        captcha: '', // 图形验证码
        computeTime: 0, // 计时剩余时间
        isShowPwd: false, // 是否显示密码
      }
    },

    computed: {
      /* 判断phone是否是一个正确的手机号 */
      isRightPhone () {
        return /^1\d{10}$/.test(this.phone)
      }
    },

    methods: {
      /* 发送短信验证码 */
      async sendCode () {
        // 设置computeTime为最大值
        this.computeTime = 10
        // 启动循环定时器, 每隔1s将computeTime减1
        const intervalId = setInterval(() => {
          console.log('-------')
          this.computeTime--
          // 当计时为0, 停止计时
          if (this.computeTime<=0) {
            clearInterval(intervalId)
          }
        }, 1000);

        // 发请求 ==> 发短信的接口
        const result = await this.$API2.user.sendCode(this.phone)
        if (result.code===0) {
          Toast('短信发送成功!')
        } else {
          // 停止计时
          this.computeTime = 0
          MessageBox('提示', result.msg);
        }
      },

      /* 
      登陆
      */
      async login () {
        // 进行前台表单验证
        let names
        if (this.loginWay) {
          names = ['phone', 'code']
        } else {
          names = ['name', 'pwd', 'captcha']
        }

        const success = await this.$validator.validateAll(names) // 对指定的所有表单项进行验证
        if (success) {
          const {phone, code, name, pwd, captcha} = this
          // 验证通过后发登陆的请求
          let result
          if (this.loginWay) {
            result = await this.$API2.user.loginSms({phone, code})
            // 请求结束后, 停止计时
            this.computeTime = 0
          } else {
            result = await this.$API2.user.loginPwd({name, pwd, captcha})
            if (result.code!==0) { // 登陆失败了
              this.updateCaptcha() // 更新图形验证码
              this.captcha = ''
            }
          }

          // 根据请求的结果进行处理
          if (result.code===0) { // 登陆请求成功
            const user = result.data
            // 保存user到state
            this.$store.dispatch('saveUser', user)
            // 跳转到个人中心
            this.$router.replace('/profile')
          } else { // 登陆请求失败
            MessageBox.alert(result.msg)
          }
        }
      },

      /* 
      更新图形验证码
      */
      updateCaptcha () {
        // 如何让浏览器对图片重新请求: 图片地址携带一个时间戳参数
        this.$refs.captcha.src = 'http://localhost:4000/captcha?time='+ Date.now()
      },

      /* 
      切换语言
      */
      toggleLocale () {
        // 根据当前的locale确定新的locale
        const locale = this.$i18n.locale === 'en' ? 'zh_CN' : 'en'
        // 指定新的locale
        this.$i18n.locale = locale
        // 保存新的locale
        localStorage.setItem('locale_key', locale)
      }
    },

    /* 2. 进入登陆界面时, 如果已经登陆了自动跳转到个人中心 */

    // 在当前组件对象被创建前调用, 不能直接访问this(不是组件对象)
    // 但可以通过next(component => {}), 在回调函数中访问组件对象
    beforeRouteEnter (to, from, next) {
      // 指定回调函数在组件对象创建之后执行, 且会将组件对象传入
      next((component) => { 
         // 如果已登陆了, 自动跳转到profile
        if (component.$store.state.user.token) {
          next('/profile')
        } else { // 否则放行
          next()
        }
      })
    },
  }
</script>

<style lang="stylus" rel="stylesheet/stylus" scoped>
  @import "../../common/stylus/mixins.styl"
  .loginContainer
    width 100%
    height 100%
    background #fff
    .loginInner
      padding-top 60px
      width 80%
      margin 0 auto
      .login_header
        .login_logo
          font-size 40px
          font-weight bold
          color #02a774
          text-align center
        .login_header_title
          padding-top 40px
          text-align center
          >a
            color #333
            font-size 14px
            padding-bottom 4px
            &:first-child
              margin-right 40px
            &.on
              color #02a774
              font-weight 700
              border-bottom 2px solid #02a774
      .login_content
        >form
          >div
            display none
            &.on
              display block
            input
              width 100%
              height 100%
              padding-left 10px
              box-sizing border-box
              border 1px solid #ddd
              border-radius 4px
              outline 0
              font 400 14px Arial
              &:focus
                border 1px solid #02a774
            .login_message
              position relative
              margin-top 16px
              height 48px
              font-size 14px
              background #fff
              .get_verification
                position absolute
                top 50%
                right 10px
                transform translateY(-50%)
                border 0
                color #ccc
                font-size 14px
                background transparent
                &.right_phone_number
                  color black
            .login_verification
              position relative
              margin-top 16px
              height 48px
              font-size 14px
              background #fff
              .switch_button
                font-size 12px
                border 1px solid #ddd
                border-radius 8px
                transition background-color .3s,border-color .3s
                padding 0 6px
                width 30px
                height 16px
                line-height 16px
                color #fff
                position absolute
                top 50%
                right 10px
                transform translateY(-50%)
                &.off
                  background #fff
                  .switch_text
                    float right
                    color #ddd
                &.on
                  background #02a774
                >.switch_circle
                  //transform translateX(27px)
                  position absolute
                  top -1px
                  left -1px
                  width 16px
                  height 16px
                  border 1px solid #ddd
                  border-radius 50%
                  background #fff
                  box-shadow 0 2px 4px 0 rgba(0,0,0,.1)
                  transition transform .3s
                  &.right
                    transform translateX(27px)
            .login_hint
              margin-top 12px
              color #999
              font-size 14px
              line-height 20px
              >a
                color #02a774
          .login_submit
            display block
            width 100%
            height 42px
            margin-top 30px
            border-radius 4px
            background #4cd96f
            color #fff
            text-align center
            font-size 16px
            line-height 42px
            border 0
        .about_us
          display block
          font-size 12px
          margin-top 20px
          text-align center
          color #999
      .go_back
        position absolute
        top 5px
        left 5px
        width 30px
        height 30px
        >.iconfont
          font-size 20px
          color #999
    
</style>
