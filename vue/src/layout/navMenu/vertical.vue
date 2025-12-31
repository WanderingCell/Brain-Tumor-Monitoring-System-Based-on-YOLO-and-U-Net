<template>
	<el-menu
		router
		:default-active="state.defaultActive"
		background-color="transparent"
		:collapse="state.isCollapse"
		:unique-opened="getThemeConfig.isUniqueOpened"
		:collapse-transition="false"
	>
		<template v-for="val in menuLists">
			<el-sub-menu :index="val.path" v-if="val.children && val.children.length > 0" :key="val.path">
				<template #title>
					<SvgIcon :name="val.meta.icon" />
					<span>{{ $t(val.meta.title) }}</span>
				</template>
				<SubItem :chil="val.children" />
			</el-sub-menu>
			<template v-else>
				<el-menu-item :index="val.path" :key="val.path">
					<SvgIcon :name="val.meta.icon" />
					<template #title v-if="!val.meta.isLink || (val.meta.isLink && val.meta.isIframe)">
						<span>{{ $t(val.meta.title) }}</span>
					</template>
					<template #title v-else>
						<a class="w100" @click.prevent="onALinkClick(val)">{{ $t(val.meta.title) }}</a>
					</template>
				</el-menu-item>
			</template>
		</template>
	</el-menu>
</template>

<script setup lang="ts" name="navMenuVertical">
import { defineAsyncComponent, reactive, computed, onMounted, watch } from 'vue';
import { useRoute, onBeforeRouteUpdate, RouteRecordRaw } from 'vue-router';
import { storeToRefs } from 'pinia';
import { useThemeConfig } from '/@/stores/themeConfig';
import other from '/@/utils/other';

// 引入组件
const SubItem = defineAsyncComponent(() => import('/@/layout/navMenu/subItem.vue'));

// 定义父组件传过来的值
const props = defineProps({
	// 菜单列表
	menuList: {
		type: Array<RouteRecordRaw>,
		default: () => [],
	},
});

// 定义变量内容
const storesThemeConfig = useThemeConfig();
const { themeConfig } = storeToRefs(storesThemeConfig);
const route = useRoute();
const state = reactive({
	// 修复：https://gitee.com/lyt-top/vue-next-admin/issues/I3YX6G
	defaultActive: route.meta.isDynamic ? route.meta.isDynamicPath : route.path,
	isCollapse: false,
});

// 获取父级菜单数据
const menuLists = computed(() => {
	return <RouteItems>props.menuList;
});
// 获取布局配置信息
const getThemeConfig = computed(() => {
	return themeConfig.value;
});
// 菜单高亮（详情时，父级高亮）
const setParentHighlight = (currentRoute: RouteToFrom) => {
	const { path, meta } = currentRoute;
	const pathSplit = meta?.isDynamic ? meta.isDynamicPath!.split('/') : path!.split('/');
	if (pathSplit.length >= 4 && meta?.isHide) return pathSplit.splice(0, 3).join('/');
	else return path;
};
// 打开外部链接
const onALinkClick = (val: RouteItem) => {
	other.handleOpenLink(val);
};
// 页面加载时
onMounted(() => {
	state.defaultActive = setParentHighlight(route);
});
// 路由更新时
onBeforeRouteUpdate((to) => {
	// 修复：https://gitee.com/lyt-top/vue-next-admin/issues/I3YX6G
	state.defaultActive = setParentHighlight(to);
	const clientWidth = document.body.clientWidth;
	if (clientWidth < 1000) themeConfig.value.isCollapse = false;
});
// 设置菜单的收起/展开
watch(
	themeConfig.value,
	() => {
		document.body.clientWidth <= 1000 ? (state.isCollapse = false) : (state.isCollapse = themeConfig.value.isCollapse);
	},
	{
		immediate: true,
	}
);
</script>

<style scoped>
/* 侧边栏整体背景和阴影 */
.el-menu {
  background-color: #1f2a40; /* 深蓝色，沉稳又有质感 */
  color: #bfcbd9; /* 浅灰字体 */
  border-right: none;
  height: 100vh;
  box-shadow: 2px 0 8px rgba(0,0,0,0.15);
  border-radius: 0 8px 8px 0; /* 右边圆角 */
}

/* 菜单项字体和图标排版 */
.el-menu-item,
.el-sub-menu__title {
  font-size: 14px;
  color: #cfd8f7;
  padding-left: 20px !important;
  display: flex;
  align-items: center;
  transition: background-color 0.3s ease;
}

/* 图标和文字间距 */
.el-menu-item svg,
.el-sub-menu__title svg {
  width: 18px;
  height: 18px;
  margin-right: 10px;
  fill: #9fb6d4;
  transition: fill 0.3s ease;
}

/* 激活状态的菜单项 - 优化 */
.el-menu-item.is-active,
.el-sub-menu__title.is-active {
  background: linear-gradient(90deg, #3a7bd5, #00d2ff); /* 更柔和的蓝色渐变 */
  color: #ffffff !important;
  font-weight: 700;
  box-shadow: inset 4px 0 8px rgba(0, 123, 255, 0.6);
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}

/* 激活状态图标颜色 */
.el-menu-item.is-active svg,
.el-sub-menu__title.is-active svg {
  fill: #ffffff;
}

.el-menu-item:hover,
.el-sub-menu__title:hover {
  background-color: #2a3a6d !important; /* 深蓝色 */
  color: #e0e9ff !important; /* 亮色文字 */
  cursor: pointer;
}

/* 悬浮状态图标颜色 */
.el-menu-item:hover svg,
.el-sub-menu__title:hover svg {
  fill: #a3c1ff; /* 浅蓝，和文字颜色搭配 */
}


/* 子菜单展开动画 */
.el-sub-menu .el-menu {
  background-color: transparent;
  padding-left: 15px;
}

/* 去掉默认边框 */
.el-menu-item,
.el-sub-menu__title {
  border: none !important;
}

/* 侧边栏折叠时图标居中 */
.el-menu.is-collapse .el-menu-item,
.el-menu.is-collapse .el-sub-menu__title {
  justify-content: center;
  padding-left: 0 !important;
}

/* 链接样式 */
a.w100 {
  color: inherit;
  text-decoration: none;
  width: 100%;
  display: inline-block;
}

a.w100:hover {
  color: #66b1ff;
}



.el-menu::-webkit-scrollbar {
  width: 6px;
}

.el-menu::-webkit-scrollbar-thumb {
  background-color: #409eff;
  border-radius: 3px;
}

</style>

