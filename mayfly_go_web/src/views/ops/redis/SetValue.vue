<template>
    <el-dialog :title="title" v-model="dialogVisible" :before-close="cancel" width="800px" :destroy-on-close="true">
        <el-form label-width="85px">
            <el-form-item prop="key" label="key:">
                <el-input :disabled="operationType == 2" v-model="key.key"></el-input>
            </el-form-item>
            <el-form-item prop="timed" label="过期时间:">
                <el-input v-model.number="key.timed" type="number"></el-input>
            </el-form-item>
            <el-form-item prop="dataType" label="数据类型:">
                <el-input v-model="key.type" disabled></el-input>
            </el-form-item>

            <el-button @click="onAddSetValue" icon="plus" size="small" plain class="mt10">添加</el-button>
            <el-table :data="value" stripe style="width: 100%">
                <el-table-column prop="value" label="value" min-width="200">
                    <template #default="scope">
                        <el-input v-model="scope.row.value" clearable type="textarea"
                            :autosize="{ minRows: 2, maxRows: 10 }" size="small"></el-input>
                    </template>
                </el-table-column>
                <el-table-column label="操作" width="90">
                    <template #default="scope">
                        <el-button type="danger" @click="value.splice(scope.$index, 1)" icon="delete" size="small"
                            plain>删除</el-button>
                    </template>
                </el-table-column>
            </el-table>
        </el-form>
        <template #footer>
            <div class="dialog-footer">
                <el-button @click="cancel()">取 消</el-button>
                <el-button @click="saveValue" type="primary" v-auth="'redis:data:save'">确 定</el-button>
            </div>
        </template>
    </el-dialog>
</template>
<script lang="ts" setup>
import { reactive, watch, toRefs } from 'vue';
import { redisApi } from './api';
import { ElMessage } from 'element-plus';
import { isTrue, notEmpty } from '@/common/assert';

const props = defineProps({
    visible: {
        type: Boolean,
    },
    title: {
        type: String,
    },
    redisId: {
        type: [Number],
        require: true,
    },
    db: {
        type: [String],
        require: true,
    },
    keyInfo: {
        type: [Object],
    },
    // 操作类型，1：新增，2：修改
    operationType: {
        type: [Number],
    },
    setValue: {
        type: [Array, Object],
    },
})

const emit = defineEmits(['update:visible', 'cancel', 'valChange'])

const state = reactive({
    dialogVisible: false,
    operationType: 1,
    redisId: '',
    db: '0',
    key: {
        key: '',
        type: 'string',
        timed: -1,
    },
    value: [{ value: '' }],
});

const {
    dialogVisible,
    operationType,
    key,
    value,
} = toRefs(state)

const cancel = () => {
    emit('update:visible', false);
    emit('cancel');
    setTimeout(() => {
        state.key = {
            key: '',
            type: 'string',
            timed: -1,
        };
        state.value = [];
    }, 500);
};

watch(props, async (newValue: any) => {
    state.dialogVisible = newValue.visible;
    state.key = newValue.key;
    state.redisId = newValue.redisId;
    state.db = newValue.db;
    state.key = newValue.keyInfo;
    state.operationType = newValue.operationType;
    // 如果是查看编辑操作，则获取值
    if (state.dialogVisible && state.operationType == 2) {
        getSetValue();
    }
});

const getSetValue = async () => {
    const res = await redisApi.getSetValue.request({
        id: state.redisId,
        db: state.db,
        key: state.key.key,
    });
    state.value = res.map((x: any) => {
        return {
            value: x,
        };
    });
};

const saveValue = async () => {
    notEmpty(state.key.key, 'key不能为空');
    isTrue(state.value.length > 0, 'set内容不能为空');
    const sv = { value: state.value.map((x) => x.value), id: state.redisId, db: state.db };
    Object.assign(sv, state.key);
    await redisApi.saveSetValue.request(sv);

    ElMessage.success('数据保存成功');
    cancel();
    emit('valChange');
};

const onAddSetValue = () => {
    state.value.unshift({ value: '' });
};
</script>
<style lang="scss">
#string-value-text {
    flex-grow: 1;
    display: flex;
    position: relative;

    .text-type-select {
        position: absolute;
        z-index: 2;
        right: 10px;
        top: 10px;
        max-width: 70px;
    }
}
</style>
