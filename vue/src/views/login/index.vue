<template>
  <div class="login-container">
    <div class="bg-bubbles">
      <li v-for="n in 10" :key="n"></li>
    </div>

    <div class="login-box animate__animated animate__fadeIn">
      <div class="title">
        <div class="title-row">
          <img src="../../assets/logo.jpg" alt="图表图标" class="title-icon"/>
          <h2>融入智能AI的医学图像处理系统</h2>
        </div>
        <p>脑部疾病分类（YOLO11+YOLO12）+直肠癌分割（Unet+ParaTransCNN）+Deepseek+Qwen</p>
      </div>

      <el-form :model="ruleForm" :rules="registerRules" ref="ruleFormRef">
        <el-form-item prop="username">
          <el-input v-model="ruleForm.username" placeholder="请输入用户名" prefix-icon="User" class="custom-input"/>
        </el-form-item>

        <el-form-item prop="password">
          <el-input v-model="ruleForm.password" type="password" placeholder="请输入密码" prefix-icon="Lock"
                    show-password class="custom-input"/>
        </el-form-item>

        <el-form-item>
          <el-button type="primary" class="login-btn" @click="submitForm(ruleFormRef)"> 登录</el-button>
        </el-form-item>
      </el-form>

      <div class="options">
        <router-link to="/register">注册账号</router-link>
        <span>|</span>
        <a href="#">忘记密码</a>
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import {reactive, computed, ref} from 'vue';
import {useRoute, useRouter} from 'vue-router';
import {ElMessage} from 'element-plus';
import {useI18n} from 'vue-i18n';
import Cookies from 'js-cookie';
import {storeToRefs} from 'pinia';
import {useThemeConfig} from '/@/stores/themeConfig';
import {initFrontEndControlRoutes} from '/@/router/frontEnd';
import {initBackEndControlRoutes} from '/@/router/backEnd';
import {Session} from '/@/utils/storage';
import {formatAxis} from '/@/utils/formatTime';
import {NextLoading} from '/@/utils/loading';
import type {FormInstance, FormRules} from 'element-plus';
import request from '/@/utils/request';

// 定义变量内容
const {t} = useI18n();
const storesThemeConfig = useThemeConfig();
const {themeConfig} = storeToRefs(storesThemeConfig);
const route = useRoute();
const router = useRouter();
const formSize = ref('default');
const ruleFormRef = ref<FormInstance>();

/*
 * 定义全局变量，等价Vue2中的data() return。
 */
const ruleForm = reactive({
  username: '',
  password: '',
});

/*
 * 校验规则。
 */
const registerRules = reactive<FormRules>({
  username: [
    {required: true, message: '请输入账号', trigger: 'blur'},
    {min: 3, max: 5, message: '长度在3-5个字符', trigger: 'blur'},
  ],
  password: [
    {required: true, message: '请输入密码', trigger: 'blur'},
    {min: 3, max: 5, message: '长度在3-5个字符', trigger: 'blur'},
  ],
});

/*
 * 提交后的方法。
 */
// 时间获取
const currentTime = computed(() => {
  return formatAxis(new Date());
});
// 登录
const onSignIn = async () => {
  // 存储 token 到浏览器缓存
  Session.set('token', Math.random().toString(36).substr(0));
  // 模拟数据，对接接口时，记得删除多余代码及对应依赖的引入。用于 `/src/stores/userInfo.ts` 中不同用户登录判断（模拟数据）
  Cookies.set('userName', ruleForm.username);
  if (!themeConfig.value.isRequestRoutes) {
    // 前端控制路由，2、请注意执行顺序
    const isNoPower = await initFrontEndControlRoutes();
    signInSuccess(isNoPower);
  } else {
    // 模拟后端控制路由，isRequestRoutes 为 true，则开启后端控制路由
    // 添加完动态路由，再进行 router 跳转，否则可能报错 No match found for location with path "/"
    const isNoPower = await initBackEndControlRoutes();
    // 执行完 initBackEndControlRoutes，再执行 signInSuccess
    signInSuccess(isNoPower);
  }
};
// 登录成功后的跳转
const signInSuccess = (isNoPower: boolean | undefined) => {
  if (isNoPower) {
    ElMessage.warning('抱歉，您没有登录权限');
    Session.clear();
  } else {
    // 初始化登录成功时间问候语
    let currentTimeInfo = currentTime.value;
    // 登录成功，跳到转首页
    // 如果是复制粘贴的路径，非首页/登录页，那么登录成功后重定向到对应的路径中
    if (route.query?.redirect) {
      router.push({
        path: <string>route.query?.redirect,
        query: Object.keys(<string>route.query?.params).length > 0 ? JSON.parse(<string>route.query?.params) : '',
      });
    } else {
      router.push('/');
    }
    // 登录成功提示
    const signInText = t('message.signInText');
    ElMessage.success(`${currentTimeInfo}，${signInText}`);
    // 添加 loading，防止第一次进入界面时出现短暂空白
    NextLoading.start();
  }
};
const submitForm = (formEl: FormInstance | undefined) => {
  if (!formEl) return;
  formEl.validate((valid) => {
    if (valid) {
      request.post('/api/user/login', ruleForm).then((res) => {
        console.log(res);
        if (res.code == 0) {
          Cookies.set('role', res.data.role); //  设置角色
          //登录成功
          onSignIn();
        } else {
          ElMessage({
            type: 'error',
            message: res.msg,
          });
        }
      });
    } else {
      console.log('error submit!');
      return false;
    }
  });
};
</script>

<style scoped>
.login-container {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  overflow: hidden;
  padding: 20px;
  background: linear-gradient(-45deg, #ff9a9e, #fad0c4, #a1c4fd, #c2e9fb, #fccb90, #a18cd1);
  background-size: 600% 600%;
  animation: gradientBG 12s ease infinite;
}

@keyframes gradientBG {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

.title {
  text-align: center;
  margin-bottom: 35px;
}

.title-row {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 12px;
}

.title-icon {
  width: 32px;
  height: 32px;
  object-fit: contain;
  vertical-align: middle;
  position: relative;
  left: 10px;
  top: -7px; /* 往上移动 2px */
}


/* 炫彩字体标题 */
.title h2 {
  font-size: 22px;
  font-weight: 800;
  margin-bottom: 18px;
  display: block;
  text-align: center;
  background: linear-gradient(to right, #ff6ec4, #7873f5, #4ade80);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  animation: gradientMove 3s infinite linear;
}

.title p {
  font-size: 15px;
  text-align: center;
  margin-bottom: 20px;
  color: #666;
  letter-spacing: 1px;
}

/* 输入框样式加强 */
:deep(.custom-input .el-input__wrapper) {
  padding: 12px 15px;
  border-radius: 12px;
  border: 2px solid transparent;
  background: #ffffffcc;
  box-shadow: 0 0 8px rgba(255, 0, 255, 0.1);
  transition: 0.4s;
}

:deep(.custom-input .el-input__wrapper:hover) {
  border-image: linear-gradient(to right, #f6d365, #fda085, #a1c4fd, #c2e9fb) 1;
  box-shadow: 0 0 15px rgba(255, 182, 193, 0.3);
}

:deep(.custom-input .el-input__wrapper.is-focus) {
  border-image: linear-gradient(to right, #667eea, #764ba2) 1;
  background: #fff;
  animation: pulse 0.5s ease-in-out;
}

/* 登录按钮炫彩渐变 + 悬浮动效 */
.login-btn {
  width: 100%;
  padding: 14px 0;
  font-size: 16px;
  font-weight: 600;
  border: none;
  border-radius: 12px;
  background: linear-gradient(45deg, #ff9a9e, #fad0c4, #fbc2eb, #a6c1ee);
  background-size: 300% 300%;
  color: white;
  cursor: pointer;
  animation: buttonGradient 6s ease infinite;
  box-shadow: 0 8px 20px rgba(0, 0, 0, 0.15);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.login-btn:hover {
  transform: scale(1.05);
  box-shadow: 0 12px 24px rgba(255, 105, 180, 0.3);
}

.login-btn:active {
  transform: scale(0.98);
}

/* 按钮动画 */
@keyframes buttonGradient {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

/* 背景气泡动画保留 */
.bg-bubbles {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  overflow: hidden;
}

.bg-bubbles li {
  position: absolute;
  list-style: none;
  display: block;
  width: 40px;
  height: 40px;
  background: radial-gradient(circle, rgba(255, 255, 255, 0.6), rgba(255, 255, 255, 0.1));
  bottom: -160px;
  animation: square 25s infinite linear;
  border-radius: 50%;
}

.bg-bubbles li:nth-child(1) {
  left: 10%;
  width: 80px;
  height: 80px;
  animation-delay: 0s;
}

.bg-bubbles li:nth-child(2) {
  left: 20%;
  width: 90px;
  height: 90px;
  animation-delay: 2s;
  animation-duration: 17s;
}

.bg-bubbles li:nth-child(3) {
  left: 25%;
  animation-delay: 4s;
}

.bg-bubbles li:nth-child(4) {
  left: 40%;
  width: 60px;
  height: 60px;
  animation-duration: 22s;
}

.bg-bubbles li:nth-child(5) {
  left: 70%;
  width: 120px;
  height: 120px;
}

.bg-bubbles li:nth-child(6) {
  left: 80%;
  width: 90px;
  height: 90px;
  animation-delay: 3s;
}

.bg-bubbles li:nth-child(7) {
  left: 32%;
  width: 60px;
  height: 60px;
  animation-delay: 7s;
}

.bg-bubbles li:nth-child(8) {
  left: 55%;
  width: 20px;
  height: 20px;
  animation-delay: 15s;
  animation-duration: 40s;
}

.bg-bubbles li:nth-child(9) {
  left: 25%;
  width: 10px;
  height: 10px;
  animation-delay: 2s;
  animation-duration: 40s;
}

.bg-bubbles li:nth-child(10) {
  left: 90%;
  width: 160px;
  height: 160px;
  animation-delay: 11s;
}

@keyframes square {
  0% {
    transform: translateY(0) rotate(0deg);
    opacity: 1;
  }
  100% {
    transform: translateY(-1000px) rotate(720deg);
    opacity: 0;
  }
}

.login-box {
  position: relative;
  z-index: 2;
  transform: translateY(20px);
  animation: slideUp 0.8s forwards;
  opacity: 0;
  width: 480px;
  padding: 45px;
  background: rgba(255, 255, 255, 0.96);
  border-radius: 18px;
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.2);
  backdrop-filter: blur(10px);
}

@keyframes slideUp {
  from {
    transform: translateY(20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

/* 输入框进入动画 */
:deep(.el-form-item) {
  opacity: 0;
}

:deep(.el-form-item:nth-child(odd)) {
  transform: translateX(-40px);
  animation: slideRightIn 0.5s forwards;
}

:deep(.el-form-item:nth-child(even)) {
  transform: translateX(40px);
  animation: slideLeftIn 0.5s forwards;
}

:deep(.el-form-item:nth-child(1)) {
  animation-delay: 0.3s;
}

:deep(.el-form-item:nth-child(2)) {
  animation-delay: 0.5s;
}

@keyframes slideRightIn {
  from {
    transform: translateX(-40px);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

@keyframes slideLeftIn {
  from {
    transform: translateX(40px);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

/* 输入框聚焦呼吸动画 */
@keyframes pulse {
  0% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.02);
  }
  100% {
    transform: scale(1);
  }
}

/* 渐变字体动画移动 */
@keyframes gradientMove {
  0% {
    background-position: 0% 50%;
  }
  50% {
    background-position: 100% 50%;
  }
  100% {
    background-position: 0% 50%;
  }
}

.options {
  margin-top: 20px;
  text-align: center;
}

.options a {
  color: #8f5fe8;
  font-size: 15px;
  text-decoration: none;
  font-weight: 500;
  transition: all 0.3s ease;
}

.options span {
  color: #bbb;
  margin: 0 15px;
}

.options a:hover {
  color: #ff69b4;
  text-decoration: underline;
}

/* 响应式 */
@media (max-width: 768px) {
  .login-box {
    width: 90%;
    padding: 35px 25px;
  }

  .title h2 {
    font-size: 20px;
  }
}
</style>





