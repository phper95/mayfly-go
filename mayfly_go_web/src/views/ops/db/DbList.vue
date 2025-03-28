<template>
    <div class="db-list">
        <el-card>
            <el-button v-auth="permissions.saveDb" type="primary" icon="plus" @click="editDb(true)">添加</el-button>
            <el-button v-auth="permissions.saveDb" :disabled="chooseId == null" @click="editDb(false)" type="primary"
                icon="edit">编辑</el-button>
            <el-button v-auth="permissions.delDb" :disabled="chooseId == null" @click="deleteDb(chooseId)" type="danger"
                icon="delete">删除</el-button>
            <div style="float: right">
                <el-select @focus="getTags" v-model="query.tagPath" placeholder="请选择标签" filterable clearable>
                    <el-option v-for="item in tags" :key="item" :label="item" :value="item"> </el-option>
                </el-select>
                <el-button v-waves type="primary" icon="search" @click="search()" class="ml5">查询</el-button>
            </div>
            <el-table :data="datas" ref="table" @current-change="choose" show-overflow-tooltip stripe>
                <el-table-column label="选择" width="60px">
                    <template #default="scope">
                        <el-radio v-model="chooseId" :label="scope.row.id">
                            <i></i>
                        </el-radio>
                    </template>
                </el-table-column>
                <el-table-column prop="tagPath" label="标签路径" min-width="150" show-overflow-tooltip></el-table-column>
                <el-table-column prop="name" label="名称" min-width="160" show-overflow-tooltip></el-table-column>
                <el-table-column min-width="170" label="host:port" show-overflow-tooltip>
                    <template #default="scope">
                        {{ `${scope.row.host}:${scope.row.port}` }}
                    </template>
                </el-table-column>
                <el-table-column prop="type" label="类型" min-width="90"></el-table-column>
                <el-table-column prop="database" label="数据库" min-width="80">
                    <template #default="scope">
                        <el-popover placement="right" trigger="click" :width="300">
                            <template #reference>
                                <el-link type="primary" :underline="false" plain @click="selectDb(scope.row.dbs)">查看
                                </el-link>
                            </template>
                            <el-input v-model="filterDb.param" @keyup="filterSchema" class="w-50 m-2" placeholder="搜索"
                                size="small">
                                <template #prefix>
                                    <el-icon class="el-input__icon">
                                        <search-icon />
                                    </el-icon>
                                </template>
                            </el-input>
                            <div class="el-tag--plain el-tag--success" v-for="db in filterDb.list" :key="db"
                                style="border:1px var(--color-success-light-3) solid; margin-top: 3px;border-radius: 5px; padding: 2px;position: relative">
                                <el-link type="success" plain size="small" :underline="false"
                                    @click="showTableInfo(scope.row, db)">{{ db }}</el-link>
                                <el-link type="primary" plain size="small" :underline="false"
                                    @click="openSqlExec(scope.row, db)" style="position: absolute; right: 4px">数据操作
                                </el-link>
                            </div>
                        </el-popover>
                    </template>
                </el-table-column>
                <el-table-column prop="username" label="用户名" min-width="100"></el-table-column>
                <el-table-column prop="remark" label="备注" min-width="150" show-overflow-tooltip></el-table-column>

                <el-table-column min-width="115" prop="creator" label="创建账号"></el-table-column>
                <el-table-column min-width="160" prop="createTime" label="创建时间" show-overflow-tooltip>
                    <template #default="scope">
                        {{ dateFormat(scope.row.createTime) }}
                    </template>
                </el-table-column>

                <el-table-column label="操作" min-width="120" fixed="right">
                    <template #default="scope">
                        <el-link type="primary" plain size="small" :underline="false" @click="onShowSqlExec(scope.row)">
                            SQL执行记录</el-link>
                    </template>
                </el-table-column>
            </el-table>
            <el-row style="margin-top: 20px" type="flex" justify="end">
                <el-pagination style="text-align: right" @current-change="handlePageChange" :total="total"
                    layout="prev, pager, next, total, jumper" v-model:current-page="query.pageNum"
                    :page-size="query.pageSize"></el-pagination>
            </el-row>
        </el-card>

        <el-dialog width="80%" :title="`${db} 表信息`" :before-close="closeTableInfo" v-model="tableInfoDialog.visible">
            <el-row class="mb10">
                <el-popover v-model:visible="showDumpInfo" :width="470" placement="right" trigger="click">
                    <template #reference>
                        <el-button class="ml5" type="success" size="small">导出</el-button>
                    </template>
                    <el-form-item label="导出内容: ">
                        <el-radio-group v-model="dumpInfo.type">
                            <el-radio :label="1" size="small">结构</el-radio>
                            <el-radio :label="2" size="small">数据</el-radio>
                            <el-radio :label="3" size="small">结构＋数据</el-radio>
                        </el-radio-group>
                    </el-form-item>

                    <el-form-item label="导出表: ">
                        <el-table @selection-change="handleDumpTableSelectionChange" max-height="300" size="small"
                            :data="tableInfoDialog.infos">
                            <el-table-column type="selection" width="45" />
                            <el-table-column property="tableName" label="表名" min-width="150" show-overflow-tooltip>
                            </el-table-column>
                            <el-table-column property="tableComment" label="备注" min-width="150" show-overflow-tooltip>
                            </el-table-column>
                        </el-table>
                    </el-form-item>

                    <div style="text-align: right">
                        <el-button @click="showDumpInfo = false" size="small">取消</el-button>
                        <el-button @click="dump(db)" type="success" size="small">确定</el-button>
                    </div>
                </el-popover>

                <el-button type="primary" size="small" @click="openEditTable(false)">创建表</el-button>
            </el-row>
            <el-table v-loading="tableInfoDialog.loading" border stripe :data="filterTableInfos" size="small"
                max-height="680">
                <el-table-column property="tableName" label="表名" min-width="150" show-overflow-tooltip>
                    <template #header>
                        <el-input v-model="tableInfoDialog.tableNameSearch" size="small" placeholder="表名: 输入可过滤"
                            clearable />
                    </template>
                </el-table-column>
                <el-table-column property="tableComment" label="备注" min-width="150" show-overflow-tooltip>
                    <template #header>
                        <el-input v-model="tableInfoDialog.tableCommentSearch" size="small" placeholder="备注: 输入可过滤"
                            clearable />
                    </template>
                </el-table-column>
                <el-table-column prop="tableRows" label="Rows" min-width="70" sortable
                    :sort-method="(a: any, b: any) => parseInt(a.tableRows) - parseInt(b.tableRows)"></el-table-column>
                <el-table-column property="dataLength" label="数据大小" sortable
                    :sort-method="(a: any, b: any) => parseInt(a.dataLength) - parseInt(b.dataLength)">
                    <template #default="scope">
                        {{ formatByteSize(scope.row.dataLength) }}
                    </template>
                </el-table-column>
                <el-table-column property="indexLength" label="索引大小" sortable
                    :sort-method="(a: any, b: any) => parseInt(a.indexLength) - parseInt(b.indexLength)">
                    <template #default="scope">
                        {{ formatByteSize(scope.row.indexLength) }}
                    </template>
                </el-table-column>
                <el-table-column property="createTime" label="创建时间" min-width="150"> </el-table-column>
                <el-table-column label="更多信息" min-width="140">
                    <template #default="scope">
                        <el-link @click.prevent="showColumns(scope.row)" type="primary">字段</el-link>
                        <el-link class="ml5" @click.prevent="showTableIndex(scope.row)" type="success">索引</el-link>
                        <el-link class="ml5" 
                                 v-if="tableCreateDialog.enableEditTypes.indexOf(tableCreateDialog.type) > -1" 
                                 @click.prevent="openEditTable(scope.row)" type="warning">编辑表</el-link>
                        <el-link class="ml5" @click.prevent="showCreateDdl(scope.row)" type="info">DDL</el-link>
                    </template>
                </el-table-column>
                <el-table-column label="操作" min-width="80">
                    <template #default="scope">
                        <el-link @click.prevent="dropTable(scope.row)" type="danger">删除</el-link>
                    </template>
                </el-table-column>
            </el-table>
        </el-dialog>

        <el-dialog width="90%" :title="`${sqlExecLogDialog.title} - SQL执行记录`" :before-close="onBeforeCloseSqlExecDialog"
            v-model="sqlExecLogDialog.visible">
            <div class="toolbar">
                <el-select v-model="sqlExecLogDialog.query.db" placeholder="请选择数据库" filterable clearable>
                    <el-option v-for="item in sqlExecLogDialog.dbs" :key="item" :label="`${item}`" :value="item">
                    </el-option>
                </el-select>
                <el-input v-model="sqlExecLogDialog.query.table" placeholder="请输入表名" clearable class="ml5"
                    style="width: 180px" />
                <el-select v-model="sqlExecLogDialog.query.type" placeholder="请选择操作类型" clearable class="ml5">
                    <el-option v-for="item in enums.DbSqlExecTypeEnum as any" :key="item.value" :label="item.label"
                        :value="item.value"> </el-option>
                </el-select>
                <el-button class="ml5" @click="searchSqlExecLog" type="success" icon="search"></el-button>
            </div>
            <el-table border stripe :data="sqlExecLogDialog.data" size="small">
                <el-table-column prop="db" label="数据库" min-width="60" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="table" label="表" min-width="60" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="type" label="类型" width="85" show-overflow-tooltip>
                    <template #default="scope">
                        <el-tag v-if="scope.row.type == enums.DbSqlExecTypeEnum['UPDATE'].value" color="#E4F5EB"
                            size="small">UPDATE</el-tag>
                        <el-tag v-if="scope.row.type == enums.DbSqlExecTypeEnum['DELETE'].value" color="#F9E2AE"
                            size="small">DELETE</el-tag>
                        <el-tag v-if="scope.row.type == enums.DbSqlExecTypeEnum['INSERT'].value" color="#A8DEE0"
                            size="small">INSERT</el-tag>
                    </template>
                </el-table-column>
                <el-table-column prop="sql" label="SQL" min-width="230" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="oldValue" label="原值" min-width="150" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="creator" label="执行人" min-width="60" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="createTime" label="执行时间" show-overflow-tooltip>
                    <template #default="scope">
                        {{ dateFormat(scope.row.createTime) }}
                    </template>
                </el-table-column>
                <el-table-column prop="remark" label="备注" min-width="60" show-overflow-tooltip> </el-table-column>
                <el-table-column label="操作" min-width="50" fixed="right">
                    <template #default="scope">
                        <el-link
                            v-if="scope.row.type == enums.DbSqlExecTypeEnum['UPDATE'].value || scope.row.type == enums.DbSqlExecTypeEnum['DELETE'].value"
                            type="primary" plain size="small" :underline="false" @click="onShowRollbackSql(scope.row)">
                            还原SQL</el-link>
                    </template>
                </el-table-column>
            </el-table>
            <el-row style="margin-top: 20px" type="flex" justify="end">
                <el-pagination style="text-align: right" @current-change="handleSqlExecPageChange"
                    :total="sqlExecLogDialog.total" layout="prev, pager, next, total, jumper"
                    v-model:current-page="sqlExecLogDialog.query.pageNum" :page-size="sqlExecLogDialog.query.pageSize">
                </el-pagination>
            </el-row>
        </el-dialog>

        <el-dialog width="55%" :title="`还原SQL`" v-model="rollbackSqlDialog.visible">
            <el-input type="textarea" :autosize="{ minRows: 15, maxRows: 30 }" v-model="rollbackSqlDialog.sql"
                size="small"> </el-input>
        </el-dialog>

        <el-dialog width="40%" :title="`${chooseTableName} 字段信息`" v-model="columnDialog.visible">
            <el-table border stripe :data="columnDialog.columns" size="small">
                <el-table-column prop="columnName" label="名称" show-overflow-tooltip> </el-table-column>
                <el-table-column width="120" prop="columnType" label="类型" show-overflow-tooltip> </el-table-column>
                <el-table-column width="80" prop="nullable" label="是否可为空" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="columnComment" label="备注" show-overflow-tooltip> </el-table-column>
            </el-table>
        </el-dialog>

        <el-dialog width="40%" :title="`${chooseTableName} 索引信息`" v-model="indexDialog.visible">
            <el-table border stripe :data="indexDialog.indexs" size="small">
                <el-table-column prop="indexName" label="索引名" min-width="120" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="columnName" label="列名" min-width="120" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="seqInIndex" label="列序列号" show-overflow-tooltip> </el-table-column>
                <el-table-column prop="indexType" label="类型"> </el-table-column>
                <el-table-column prop="indexComment" label="备注" min-width="130" show-overflow-tooltip>
                </el-table-column>
            </el-table>
        </el-dialog>

        <el-dialog width="55%" :title="`${chooseTableName} Create-DDL`" v-model="ddlDialog.visible">
            <el-input disabled type="textarea" :autosize="{ minRows: 15, maxRows: 30 }" v-model="ddlDialog.ddl"
                size="small"> </el-input>
        </el-dialog>

        <db-edit @val-change="valChange" :title="dbEditDialog.title" v-model:visible="dbEditDialog.visible"
            v-model:db="dbEditDialog.data"></db-edit>
        <create-table :title="tableCreateDialog.title" :active-name="tableCreateDialog.activeName" :dbId="dbId" :db="db"
            :data="tableCreateDialog.data" v-model:visible="tableCreateDialog.visible"></create-table>
    </div>
</template>

<script lang='ts' setup>
import { toRefs, reactive, computed, onMounted } from 'vue';
import { ElMessage, ElMessageBox } from 'element-plus';
import { formatByteSize } from '@/common/utils/format';
import DbEdit from './DbEdit.vue';
import CreateTable from './CreateTable.vue';
import { dbApi } from './api';
import enums from './enums';
import SqlExecBox from './component/SqlExecBox.ts';
import config from '@/common/config';
import { getSession } from '@/common/utils/storage';
import { isTrue } from '@/common/assert';
import { Search as SearchIcon } from '@element-plus/icons-vue'
import router from '@/router';
import { store } from '@/store';
import { tagApi } from '../tag/api.ts';
import { dateFormat } from '@/common/utils/date';

const permissions = {
    saveDb: 'db:save',
    delDb: 'db:del',
}

const state = reactive({
    row: {},
    dbId: 0,
    db: '',
    tags: [],
    chooseId: null as any,
    /**
     * 选中的数据
     */
    chooseData: null,
    /**
     * 查询条件
     */
    query: {
        tagPath: null,
        projectId: null,
        pageNum: 1,
        pageSize: 10,
    },
    datas: [],
    total: 0,
    showDumpInfo: false,
    dumpInfo: {
        id: 0,
        db: '',
        type: 3,
        tables: [],
    },
    // sql执行记录弹框
    sqlExecLogDialog: {
        title: '',
        visible: false,
        data: [],
        total: 0,
        dbs: [],
        query: {
            dbId: 0,
            db: '',
            table: '',
            type: null,
            pageNum: 1,
            pageSize: 12,
        },
    },
    rollbackSqlDialog: {
        visible: false,
        sql: '',
    },
    chooseTableName: '',
    tableInfoDialog: {
        loading: false,
        visible: false,
        infos: [],
        tableNameSearch: '',
        tableCommentSearch: '',
    },
    columnDialog: {
        visible: false,
        columns: [],
    },
    indexDialog: {
        visible: false,
        indexs: [],
    },
    ddlDialog: {
        visible: false,
        ddl: '',
    },
    dbEditDialog: {
        visible: false,
        data: null as any,
        title: '新增数据库',
    },
    tableCreateDialog: {
        title: '创建表',
        visible: false,
        columns: [],
        indexs: [],
        activeName: '1',
        type: '',
        enableEditTypes:['mysql'], // 支持"编辑表"的数据库类型
        data: {  // 修改表时，传递修改数据
            edit: false,
            row: {},
            indexs: [],
            columns: [],
        },
    },
    filterDb: {
        param: '',
        cache: [],
        list: [],
    }
});

const {
    dbId,
    db,
    tags,
    chooseId,
    query,
    datas,
    total,
    showDumpInfo,
    dumpInfo,
    sqlExecLogDialog,
    rollbackSqlDialog,
    chooseTableName,
    tableInfoDialog,
    columnDialog,
    indexDialog,
    ddlDialog,
    dbEditDialog,
    tableCreateDialog,
    filterDb,
} = toRefs(state)


onMounted(async () => {
    search();
});

const filterTableInfos = computed(() => {
    const infos = state.tableInfoDialog.infos;
    const tableNameSearch = state.tableInfoDialog.tableNameSearch;
    const tableCommentSearch = state.tableInfoDialog.tableCommentSearch;
    if (!tableNameSearch && !tableCommentSearch) {
        return infos;
    }
    return infos.filter((data: any) => {
        let tnMatch = true;
        let tcMatch = true;
        if (tableNameSearch) {
            tnMatch = data.tableName.toLowerCase().includes(tableNameSearch.toLowerCase());
        }
        if (tableCommentSearch) {
            tcMatch = data.tableComment.includes(tableCommentSearch);
        }
        return tnMatch && tcMatch;
    });
});

const choose = (item: any) => {
    if (!item) {
        return;
    }
    state.chooseId = item.id;
    state.chooseData = item;
};

const search = async () => {
    let res: any = await dbApi.dbs.request(state.query);
    // 切割数据库
    res.list.forEach((e: any) => {
        e.popoverSelectDbVisible = false;
        e.dbs = e.database.split(' ');
    });
    state.datas = res.list;
    state.total = res.total;
};

const handlePageChange = (curPage: number) => {
    state.query.pageNum = curPage;
    search();
};

const getTags = async () => {
    state.tags = await tagApi.getAccountTags.request(null);
};

const editDb = async (isAdd = false) => {
    if (isAdd) {
        state.dbEditDialog.data = null;
        state.dbEditDialog.title = '新增数据库资源';
    } else {
        state.dbEditDialog.data = state.chooseData;
        state.dbEditDialog.title = '修改数据库资源';
    }
    state.dbEditDialog.visible = true;
};

const valChange = () => {
    state.chooseData = null;
    state.chooseId = null;
    search();
};

const deleteDb = async (id: number) => {
    try {
        await ElMessageBox.confirm(`确定删除该库?`, '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning',
        });
        await dbApi.deleteDb.request({ id });
        ElMessage.success('删除成功');
        state.chooseData = null;
        state.chooseId = null;
        search();
    } catch (err) { }
};

const onShowSqlExec = async (row: any) => {
    state.sqlExecLogDialog.title = `${row.name}[${row.host}:${row.port}]`;
    state.sqlExecLogDialog.query.dbId = row.id;
    state.sqlExecLogDialog.dbs = row.database.split(' ');
    searchSqlExecLog();
    state.sqlExecLogDialog.visible = true;
};

const onBeforeCloseSqlExecDialog = () => {
    state.sqlExecLogDialog.visible = false;
    state.sqlExecLogDialog.data = [];
    state.sqlExecLogDialog.dbs = [];
    state.sqlExecLogDialog.total = 0;
    state.sqlExecLogDialog.query.dbId = 0;
    state.sqlExecLogDialog.query.pageNum = 1;
    state.sqlExecLogDialog.query.table = '';
    state.sqlExecLogDialog.query.db = '';
    state.sqlExecLogDialog.query.type = null;
};

const searchSqlExecLog = async () => {
    const res = await dbApi.getSqlExecs.request(state.sqlExecLogDialog.query);
    state.sqlExecLogDialog.data = res.list;
    state.sqlExecLogDialog.total = res.total;
};

const handleSqlExecPageChange = (curPage: number) => {
    state.sqlExecLogDialog.query.pageNum = curPage;
    searchSqlExecLog();
};

/**
 * 选择导出数据库表
 */
const handleDumpTableSelectionChange = (vals: any) => {
    state.dumpInfo.tables = vals.map((x: any) => x.tableName);
};

/**
 * 数据库信息导出
 */
const dump = (db: string) => {
    isTrue(state.dumpInfo.tables.length > 0, '请选择要导出的表');
    const a = document.createElement('a');
    a.setAttribute(
        'href',
        `${config.baseApiUrl}/dbs/${state.dbId}/dump?db=${db}&type=${state.dumpInfo.type}&tables=${state.dumpInfo.tables.join(
            ','
        )}&token=${getSession('token')}`
    );
    a.click();
    state.showDumpInfo = false;
};

const onShowRollbackSql = async (sqlExecLog: any) => {
    const columns = await dbApi.columnMetadata.request({ id: sqlExecLog.dbId, db: sqlExecLog.db, tableName: sqlExecLog.table });
    const primaryKey = columns[0].columnName;
    const oldValue = JSON.parse(sqlExecLog.oldValue);

    const rollbackSqls = [];
    if (sqlExecLog.type == enums.DbSqlExecTypeEnum['UPDATE'].value) {
        for (let ov of oldValue) {
            const setItems = [];
            for (let key in ov) {
                if (key == primaryKey) {
                    continue;
                }
                setItems.push(`${key} = ${wrapValue(ov[key])}`);
            }
            rollbackSqls.push(`UPDATE ${sqlExecLog.table} SET ${setItems.join(', ')} WHERE ${primaryKey} = ${wrapValue(ov[primaryKey])};`);
        }
    } else if (sqlExecLog.type == enums.DbSqlExecTypeEnum['DELETE'].value) {
        const columnNames = columns.map((c: any) => c.columnName);
        for (let ov of oldValue) {
            const values = [];
            for (let column of columnNames) {
                values.push(wrapValue(ov[column]));
            }
            rollbackSqls.push(`INSERT INTO ${sqlExecLog.table} (${columnNames.join(', ')}) VALUES (${values.join(', ')});`);
        }
    }

    state.rollbackSqlDialog.sql = rollbackSqls.join('\n');
    state.rollbackSqlDialog.visible = true;
};

/**
 * 包装值，如果值类型为number则直接返回，其他则需要使用''包装
 */
const wrapValue = (val: any) => {
    if (typeof val == 'number') {
        return val;
    }
    return `'${val}'`;
};

const showTableInfo = async (row: any, db: string) => {
    state.tableInfoDialog.loading = true;
    state.tableInfoDialog.visible = true;
    try {
        state.tableInfoDialog.infos = await dbApi.tableInfos.request({ id: row.id, db });
        state.tableCreateDialog.type = row.type
        state.dbId = row.id;
        state.row = row;
        state.db = db;
    } catch (e) {
        state.tableInfoDialog.visible = false;
    } finally {
        state.tableInfoDialog.loading = false;
    }
};

// 给子组件调用，勿删
// eslint-disable-next-line no-unused-vars
const refreshTableInfo = async () => {
  state.tableInfoDialog.infos = await dbApi.tableInfos.request({ id: state.dbId, db: state.db });
}

const closeTableInfo = () => {
    state.showDumpInfo = false;
    state.tableInfoDialog.visible = false;
    state.tableInfoDialog.infos = [];
};

const showColumns = async (row: any) => {
    state.chooseTableName = row.tableName;
    state.columnDialog.columns = await dbApi.columnMetadata.request({
        id: state.chooseId,
        db: state.db,
        tableName: row.tableName,
    });

    state.columnDialog.visible = true;
};

const showTableIndex = async (row: any) => {
    state.chooseTableName = row.tableName;
    state.indexDialog.indexs = await dbApi.tableIndex.request({
        id: state.chooseId,
        db: state.db,
        tableName: row.tableName,
    });

    state.indexDialog.visible = true;
};

const showCreateDdl = async (row: any) => {
    state.chooseTableName = row.tableName;
    const res = await dbApi.tableDdl.request({
        id: state.chooseId,
        db: state.db,
        tableName: row.tableName,
    });
    state.ddlDialog.ddl = res[0]['Create Table'];
    state.ddlDialog.visible = true;
};

/**
 * 删除表
 */
const dropTable = async (row: any) => {
    try {
        const tableName = row.tableName;
        await ElMessageBox.confirm(`确定删除'${tableName}'表?`, '提示', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning',
        });
        SqlExecBox({
            sql: `DROP TABLE ${tableName}`,
            dbId: state.chooseId,
            db: state.db,
            runSuccessCallback: async () => {
                state.tableInfoDialog.infos = await dbApi.tableInfos.request({ id: state.chooseId, db: state.db });
            },
        });
    } catch (err) { }
};
const openSqlExec = (row: any, db: any) => {
    // 判断db是否发生改变
    let oldDb = store.state.sqlExecInfo.dbOptInfo.db;
    if (db && oldDb !== db) {
        const { tagPath, id } = row;
        let params = {
            tagPath,
            dbId: id,
            db
        }
        store.dispatch('sqlExecInfo/setSqlExecInfo', params);
    }
    router.push({ name: 'SqlExec' });
}

// 点击查看时初始化数据
const selectDb = (row: any) => {
    state.filterDb.param = ''
    state.filterDb.cache = row;
    state.filterDb.list = row;
}

// 输入字符过滤schema
const filterSchema = () => {
    if (state.filterDb.param) {
        state.filterDb.list = state.filterDb.cache.filter((a) => { return String(a).toLowerCase().indexOf(state.filterDb.param) > -1 })
    } else {
        state.filterDb.list = state.filterDb.cache;
    }
}

// 打开编辑表
const openEditTable = async (row: any) => {

    state.tableCreateDialog.visible = true
    state.tableCreateDialog.activeName = '1'

    if(row === false){
        state.tableCreateDialog.data = {edit: false, row: {}, indexs: [], columns: [] }
        state.tableCreateDialog.title = '创建表'
    }

    if (row.tableName) {
        state.tableCreateDialog.title = '修改表'
        let indexs = await dbApi.tableIndex.request({
            id: state.chooseId,
            db: state.db,
            tableName: row.tableName,
        });
        let columns = await dbApi.columnMetadata.request({
            id: state.chooseId,
            db: state.db,
            tableName: row.tableName,
        });
        state.tableCreateDialog.data = { edit: true, row, indexs, columns }
    }
}
</script>
<style lang="scss">

</style>
