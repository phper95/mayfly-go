<template>
    <div>
        <el-dialog title="待执行SQL" v-model="dialogVisible" :show-close="false" width="600px">
            如需执行多条sql，需要在【数据库管理】配置连接参数：multiStatements=true
            <codemirror height="350px" class="codesql" ref="cmEditor" language="sql" v-model="sqlValue"
                :options="cmOptions" />
            <el-input ref="remarkInputRef" v-model="remark" placeholder="请输入执行备注" class="mt5" />
            <template #footer>
                <span class="dialog-footer">
                    <el-button @click="cancel">取 消</el-button>
                    <el-button @click="runSql" type="primary" :loading="btnLoading">执 行</el-button>
                </span>
            </template>
        </el-dialog>
    </div>
</template>

<script lang="ts" setup>
import { toRefs, ref, nextTick, reactive } from 'vue';
import { dbApi } from '../api';
import { ElDialog, ElButton, ElInput, ElMessage, InputInstance } from 'element-plus';
// import base style
import 'codemirror/lib/codemirror.css';
// 引入主题后还需要在 options 中指定主题才会生效
import 'codemirror/theme/base16-light.css';
import 'codemirror/addon/selection/active-line';
import { codemirror } from '@/components/codemirror';
import { format as sqlFormatter } from 'sql-formatter';

import { SqlExecProps } from './SqlExecBox';

const props = defineProps({
    visible: {
        type: Boolean,
    },
    dbId: {
        type: [Number],
    },
    db: {
        type: String,
    },
    sql: {
        type: String,
    },
})

const cmOptions = {
    tabSize: 4,
    mode: 'text/x-sql',
    lineNumbers: true,
    line: true,
    indentWithTabs: true,
    smartIndent: true,
    matchBrackets: true,
    theme: 'base16-light',
    autofocus: true,
    extraKeys: { Tab: 'autocomplete' }, // 自定义快捷键
}

const remarkInputRef = ref<InputInstance>();
const state = reactive({
    dialogVisible: false,
    sqlValue: '',
    dbId: 0,
    db: '',
    remark: '',
    btnLoading: false,
});

const {
    dialogVisible,
    sqlValue,
    remark,
    btnLoading
} = toRefs(state)

state.sqlValue = props.sql as any;
let runSuccessCallback: any;
let cancelCallback: any;
let runSuccess: boolean = false;

/**
 * 执行sql
 */
const runSql = async () => {
    if (!state.remark) {
        ElMessage.error('请输入执行的备注信息');
        return;
    }

    try {
        state.btnLoading = true;
        await dbApi.sqlExec.request({
            id: state.dbId,
            db: state.db,
            remark: state.remark,
            sql: state.sqlValue.trim(),
        });
        ElMessage.success('执行成功');
        runSuccess = true;
    } catch (e) {
        runSuccess = false;
    }
    if (runSuccess) {
        if (runSuccessCallback) {
            runSuccessCallback();
        }
        cancel();
    }
    state.btnLoading = false;
};

const cancel = () => {
    state.dialogVisible = false;
    // 没有执行成功，并且取消回调函数存在，则执行
    if (!runSuccess && cancelCallback) {
        cancelCallback();
    }
    setTimeout(() => {
        state.dbId = 0;
        state.sqlValue = '';
        state.remark = '';
        runSuccessCallback = null;
        cancelCallback = null;
        runSuccess = false;
    }, 200);
};

const open = (props: SqlExecProps) => {
    runSuccessCallback = props.runSuccessCallback;
    cancelCallback = props.cancelCallback;
    state.sqlValue = sqlFormatter(props.sql);
    state.dbId = props.dbId;
    state.db = props.db;
    state.dialogVisible = true;
    nextTick(() => {
        setTimeout(() => {
            remarkInputRef.value?.focus();
        });
    });
};
</script>
<style lang="scss">
.codesql {
    font-size: 9pt;
    font-weight: 600;
}
</style>
