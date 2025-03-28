<template>
    <div class="layout-navbars-breadcrumb-user" :style="{ flex: layoutUserFlexNum }">
        <el-dropdown :show-timeout="70" :hide-timeout="50" trigger="click" @command="onComponentSizeChange">
            <div class="layout-navbars-breadcrumb-user-icon">
                <el-icon title="组件大小">
                    <plus />
                </el-icon>
            </div>
            <template #dropdown>
                <el-dropdown-menu>
                    <el-dropdown-item command="" :disabled="disabledSize === ''">默认</el-dropdown-item>
                    <el-dropdown-item command="large" :disabled="disabledSize === 'large'">大型</el-dropdown-item>
                    <el-dropdown-item command="small" :disabled="disabledSize === 'small'">小型</el-dropdown-item>
                </el-dropdown-menu>
            </template>
        </el-dropdown>
        <!-- <div class="layout-navbars-breadcrumb-user-icon" @click="onSearchClick">
            <el-icon title="菜单搜索">
                <search />
            </el-icon>
        </div> -->
        <div class="layout-navbars-breadcrumb-user-icon" @click="onLayoutSetingClick">
            <el-icon title="布局设置">
                <setting />
            </el-icon>
        </div>
        <div class="layout-navbars-breadcrumb-user-icon">
            <el-popover
                placement="bottom"
                trigger="click"
                :visible="isShowUserNewsPopover"
                :width="300"
                popper-class="el-popover-pupop-user-news"
            >
                <template #reference>
                    <el-badge :is-dot="true" @click="isShowUserNewsPopover = !isShowUserNewsPopover">
                        <el-icon title="消息">
                            <bell />
                        </el-icon>
                    </el-badge>
                </template>
                <transition name="el-zoom-in-top">
                    <UserNews v-show="isShowUserNewsPopover" />
                </transition>
            </el-popover>
        </div>
        <div class="layout-navbars-breadcrumb-user-icon mr10" @click="onScreenfullClick">
            <el-icon v-if="!isScreenfull" title="关全屏">
                <full-screen />
            </el-icon>
            <el-icon v-else title="开全屏">
                <crop />
            </el-icon>
        </div>
        <el-dropdown :show-timeout="70" :hide-timeout="50" @command="onHandleCommandClick">
            <span class="layout-navbars-breadcrumb-user-link" style="cursor: pointer">
                <img :src="getUserInfos.photo" class="layout-navbars-breadcrumb-user-link-photo mr5" />
                {{ getUserInfos.name || 'test' }}
                <i class="el-icon-arrow-down el-icon--right"></i>
            </span>
            <template #dropdown>
                <el-dropdown-menu>
                    <el-dropdown-item command="/home">首页</el-dropdown-item>
                    <el-dropdown-item command="/personal">个人中心</el-dropdown-item>
                    <el-dropdown-item divided command="logOut">退出登录</el-dropdown-item>
                </el-dropdown-menu>
            </template>
        </el-dropdown>
        <SearchMenu ref="searchRef" />
    </div>
</template>

<script lang="ts">
import { ref, getCurrentInstance, computed, reactive, toRefs, onMounted } from 'vue';
import { useRouter } from 'vue-router';
import { ElMessageBox, ElMessage } from 'element-plus';
import screenfull from 'screenfull';
import { resetRoute } from '@/router/index.ts';
import { useStore } from '@/store/index.ts';
import { clearSession, setLocal, getLocal, removeLocal } from '@/common/utils/storage.ts';
import UserNews from '@/views/layout/navBars/breadcrumb/userNews.vue';
import Search from '@/views/layout/navBars/breadcrumb/search.vue';
export default {
    name: 'layoutBreadcrumbUser',
    components: { UserNews, SearchMenu: Search },
    setup() {
        const { proxy } = getCurrentInstance() as any;
        const router = useRouter();
        const store = useStore();
        const searchRef = ref();
        const state = reactive({
            isScreenfull: false,
            isShowUserNewsPopover: false,
            disabledI18n: 'zh-cn',
            disabledSize: '',
        });
        // 获取用户信息 vuex
        const getUserInfos = computed(() => {
            return store.state.userInfos.userInfos;
        });
        // 获取布局配置信息
        const getThemeConfig = computed(() => {
            return store.state.themeConfig.themeConfig;
        });
        // 设置分割样式
        const layoutUserFlexNum = computed(() => {
            let { layout, isClassicSplitMenu } = getThemeConfig.value;
            let num = '';
            if (layout === 'defaults' || (layout === 'classic' && !isClassicSplitMenu) || layout === 'columns') num = '1';
            else num = '';
            return num;
        });
        // 全屏点击时
        const onScreenfullClick = () => {
            if (!screenfull.isEnabled) {
                ElMessage.warning('暂不不支持全屏');
                return false;
            }
            screenfull.toggle();
            state.isScreenfull = !state.isScreenfull;
        };
        // 布局配置 icon 点击时
        const onLayoutSetingClick = () => {
            proxy.mittBus.emit('openSetingsDrawer');
        };
        // 下拉菜单点击时
        const onHandleCommandClick = (path: string) => {
            if (path === 'logOut') {
                ElMessageBox({
                    closeOnClickModal: false,
                    closeOnPressEscape: false,
                    title: '提示',
                    message: '此操作将退出登录, 是否继续?',
                    showCancelButton: true,
                    confirmButtonText: '确定',
                    cancelButtonText: '取消',
                    beforeClose: (action, instance, done) => {
                        if (action === 'confirm') {
                            instance.confirmButtonLoading = true;
                            instance.confirmButtonText = '退出中';
                            setTimeout(() => {
                                done();
                                setTimeout(() => {
                                    instance.confirmButtonLoading = false;
                                }, 300);
                            }, 700);
                        } else {
                            done();
                        }
                    },
                })
                    .then(() => {
                        clearSession(); // 清除缓存/token等
                        resetRoute(); // 删除/重置路由
                        router.push('/login');
                        setTimeout(() => {
                            ElMessage.success('安全退出成功！');
                        }, 300);
                    })
                    .catch(() => {});
            } else {
                router.push(path);
            }
        };
        // 菜单搜索点击
        const onSearchClick = () => {
            searchRef.value.openSearch();
        };
        // 组件大小改变
        const onComponentSizeChange = (size: string) => {
            removeLocal('themeConfig');
            getThemeConfig.value.globalComponentSize = size;
            setLocal('themeConfig', getThemeConfig.value);
            // proxy.$ELEMENT.size = size;
            initComponentSize();
            window.location.reload();
        };
        // 初始化全局组件大小
        const initComponentSize = () => {
            switch (getLocal('themeConfig').globalComponentSize) {
                case '':
                    state.disabledSize = '';
                    break;
                case 'default':
                    state.disabledSize = 'default';
                    break;
                case 'small':
                    state.disabledSize = 'small';
                    break;
                case 'large':
                    state.disabledSize = 'large';
                    break;
            }
        };
        // 页面加载时
        onMounted(() => {
            if (getLocal('themeConfig')) {
                initComponentSize();
            }
        });
        return {
            getUserInfos,
            onLayoutSetingClick,
            onHandleCommandClick,
            onScreenfullClick,
            onSearchClick,
            onComponentSizeChange,
            searchRef,
            layoutUserFlexNum,
            ...toRefs(state),
        };
    },
};
</script>

<style scoped lang="scss">
.layout-navbars-breadcrumb-user {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    &-link {
        height: 100%;
        display: flex;
        align-items: center;
        white-space: nowrap;
        &-photo {
            width: 25px;
            height: 25px;
            border-radius: 100%;
        }
    }
    &-icon {
        padding: 0 10px;
        cursor: pointer;
        color: var(--bg-topBarColor);
        height: 50px;
        line-height: 50px;
        display: flex;
        align-items: center;
        &:hover {
            background: rgba(0, 0, 0, 0.04);
            i {
                display: inline-block;
                animation: logoAnimation 0.3s ease-in-out;
            }
        }
    }
    ::v-deep(.el-dropdown) {
        color: var(--bg-topBarColor);
    }
    ::v-deep(.el-badge) {
        height: 40px;
        line-height: 40px;
        display: flex;
        align-items: center;
    }
    ::v-deep(.el-badge__content.is-fixed) {
        top: 12px;
    }
}
</style>
