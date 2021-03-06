<template>
    <div class="yl-table" style="overflow:hidden" v-if="showAll">
        <el-table stripe border v-on="clisteners" v-bind="cAttrs" :data="mixData" ref="table" :height="height" @row-click="rowClick" :row-class-name="tableRowClassName" @selection-change="handleSelectionChange" :highlight-current-row="highlightCurrentRow">
            <template v-for="cItem in columns">
                <!-- hide时隐藏该列 -->
                <template v-if="!cItem.isHide">
                    <template v-for="item in [formatItem(cItem)]">
                        <slot v-if="item.slot && !item.hide" :name="item.slot" :label="item.label" :col="item"></slot>
                        <el-table-column v-bind="formatItem(item)" :key="item.mathRound" v-else-if="item.type=='operate' && !item.hide">
                            <!-- <template slot="header" slot-scope="scope">
                                <span>{{item.label}}</span>
                                <i class="el-icon-setting svg-table-title" v-if="item.show" @click.self="handleSet(scope,item)"></i>
                            </template> -->
                            <template slot-scope="scope">
                                <template v-for="(filItem,fillIndex) in [filterBtns(scope.row,item.btns)]">
                                    <template v-for="subItem in filItem.slice(0,(filItem.length>4?3:4))">
                                        <el-button :key="subItem.mathRound" type="text" class="table-btns" size="mini" @click="subItem.event(scope.row)">{{subItem.name}}
                                        </el-button>
                                    </template>
                                    <template v-if="filItem.length>4">
                                        <el-dropdown size="mini" :key="fillIndex">
                                            <span class="el-dropdown-link">
                                                更多<i class="el-icon-arrow-down el-icon--right"></i>
                                            </span>
                                            <el-dropdown-menu slot="dropdown">
                                                <template v-for="subItem in filItem.slice(filItem.length>4?3:4)">
                                                    <el-dropdown-item :key="subItem.mathRound" @click.native="subItem.event(scope.row)">{{subItem.name}}</el-dropdown-item>
                                                </template>
                                            </el-dropdown-menu>
                                        </el-dropdown>
                                    </template>
                                </template>
                            </template>
                        </el-table-column>
                        <el-table-column v-bind="item" :key="item.mathRound" v-else-if="item.field && !item.hide">
                            <template slot-scope="scope">
                                <div v-html="scope.row[item.field]"></div>
                            </template>
                        </el-table-column>
                        <el-table-column v-bind="item" :key="item.mathRound" v-else-if="item.type=='cIndex' && !item.hide" align="center" width="50">
                            <template slot="header" slot-scope="scope">
                                <i class="el-icon-setting" v-if="!item.show" @click.self="handleSet(scope,item)"></i>
                                <span v-else>序号</span>
                            </template>
                        </el-table-column>
                        <el-table-column v-bind="item" :key="item.mathRound" v-else-if="item.type=='selection' && !item.hide" align="center">
                        </el-table-column>

                        <el-table-column v-bind="item" :key="item.mathRound" v-else-if="!item.hide"></el-table-column>
                    </template>
                </template>
            </template>
        </el-table>
        <div v-if="isTableSet">
            <my-table-expand :isShow.sync="isTableSet" :sourceColumns="config.col" :columns.sync="columns" :name="config.name" @change="expandChange" :isShowTotalRow="show.setTotalRow"></my-table-expand>
        </div>
        <slot name="content"></slot>
    </div>
</template>

<script>
import MyTableExpand from './MyTableExpand.vue';
import { cloneDeep, debounce } from 'lodash';
export default {
    name: 'PsmTable',
    data () {
        return {
            height: 'auto',
            container: 'document',
            pageKey: '', // 保存设置唯一key
            showAll: false, // hack方法,确保二次渲染不会出错
            columns: [], // 列表
            cAttrs: {}, // 自定义属性
            clisteners: {}, // 自定义属性
            show: {
                // 显示
                total: false, // 自定义合计
                setTotalRow: true // 显示设置表格合计列
            },
            disAutoHeight: false, // 关闭自动计算高度
            limitHeight: -50, // 缩小高度
            uselocal: false, // 使用本地配置
            isTableSet: false, // 是否显示列设置
            selectRow: [],
            selectData: [],
            mixData: [] // 混合数据
        };
    },
    props: {
        // 配置项
        config: {
            type: Object,
            default () {
                return {
                    name: '', // 名字
                    col: [
                        {
                            label: '',
                            prop: '',
                            slot: '',
                            unSave: true // 是否保存名称
                        }
                    ] // 列表
                };
            },
            required: true
        },
        maxHeight: {
            type: Number
        },
        data: {
            // 数据
            type: Array,
            required: true
        },
        // 分页信息 小计 合计用
        page: {
            type: Object
        }
    },
    watch: {
        selectData (data) {
            if (this.config.rowClick && !this.config.col[0].hide) {
                this.selectRow = [];
                if (data.length > 0) {
                    data.forEach((item, index) => {
                        this.selectRow.push(this.data.indexOf(item));
                    });
                }
            }
        },
        columns: {
            deep: true,
            handler: function () {
                this._updateTotalLine();
            }
        },
        data: {
            deep: true,
            handler: function () {
                this._updateTotalLine();
            }
        },
        showAll (val) {
            if (val) {
                this.$nextTick(() => {
                    if (!this.disAutoHeight) {
                        if (!this.maxHeight) {
                            this.countHeight();
                        }
                    }
                });
            }
        },
        $attrs (val, oldval) {
            if (JSON.stringify(val) !== JSON.stringify(oldval)) {
                this.cAttrs = val;
            }
        }
    },
    methods: {
        handleSelectionChange (data) {
            this.selectData = data;
            this.$emit('selectionChange', data);
        },
        tableRowClassName ({ row, rowIndex }) {
            if (
                this.config.rowClick &&
                !this.config.col[0].hide &&
                this.selectRow.includes(rowIndex)
            ) {
                return 'warning-row';
            }
        },
        rowClick (row) {
            if (this.config.rowClick && !this.config.col[0].hide) {
                this.$refs.table.toggleRowSelection(row);
            } else {
                this.$emit('rowClick', row);
            }
        },
        // 得到页面的key
        _getPageKey () {
            let path = window.location.pathname.split('/').slice(1, 3);
            if (this.config.name) {
                path.push(this.config.name);
                this.pageKey = path.join('.');
            } else {
                // console.info('表格未设置关键词config.name');
            }
        },

        // 处理远程返回的配置数据，以本地数据为准做融合
        _mixConfig (colConfig, cols) {
            let res = [];
            cols.forEach(item => {
                let col = colConfig.filter(data => data.name === item.label);
                if (col.length) {
                    res.push({ ...item, ...col[0] });
                } else {
                    res.push(item);
                }
            });
            return res.sort((a, b) => a.index > b.index);
        },
        // 获取表格配置
        _getConfig () {
            this.selectRow = [];
            this.selectData = [];
            var cols = cloneDeep(this.config.col);
            if (this.uselocal) {
                // 强制使用本地配置
                this.columns = cols;
                this.showAll = true;
            } else {
                this.columns = cols; // 远程设置 TUDO
                this.showAll = true;
                // this.Http(URL.getConfig, {
                //   keys: [this.pageKey]
                // }).then(res => {
                //   if (res.data.length) {
                //     let colConfig = JSON.parse(res.data[0].configureValue);
                //     this.columns = this._mixConfig(colConfig, cols);
                //   } else {
                //     this.columns = cols; // 列表设置
                //   }
                //   this.showAll = true;
                // });
            }
        },

        // 处理列配置
        formatItem (item) {
            let obj = {};
            for (let key of Object.keys(item)) {
                if (typeof item[key] === 'function') {
                    obj[key] = item[key].call(this, item);
                } else {
                    obj[key] = item[key];
                }
            }
            obj.mathRound =
                item.prop ||
                item.type ||
                item.slot ||
                parseInt(Math.random() * 10000000);
            return obj;
        },
        filterBtns (row, msg) {
            let arr = msg.filter(item => {
                item.mathRound =
                    item.mathRound || parseInt(Math.random() * 10000000);
                return this.showBtnFunc(row, item);
            });
            // console.log(arr);
            return arr;
        },
        // 处理按钮
        showBtnFunc (row, item) {
            // 按钮关键词,过滤显示按钮,flag表示什么时候按钮是隐藏的
            let keywords = [
                { key: 'hide', flag: true },
                { key: 'show', flag: false }
            ];
            for (const iterator of keywords) {
                if (item[iterator.key]) {
                    if (typeof item[iterator.key] === 'function') {
                        return iterator.flag
                            ? !item[iterator.key].call(this, row)
                            : item[iterator.key].call(this, row);
                    } else {
                        return iterator.flag
                            ? !item[iterator.key]
                            : item[iterator.key];
                    }
                } else {
                    continue;
                }
            }
            return true;
        },
        // 排序设置改变
        expandChange (allList, saveList) {
            this._updateConfig(saveList).then(() => {
                this.isTableSet = false;
                this.showAll = false;
                this.$nextTick(() => {
                    this.columns = allList;
                    this.showAll = true;
                });
            });
        },
        // 更新表格设置
        _updateConfig (config) {
            return this.Http(URL.setConfig, {
                configures: [
                    {
                        key: this.pageKey,
                        configureValue: JSON.stringify(config)
                    }
                ]
            });
        },

        // 更新是否显示合计行
        _updateTotalLine () {
            let result = [];
            let getColumnData = (name, code) => {
                let headName = this.col.filter(item => item.prop)[0].prop,
                    obj = {
                        [headName]: name,
                        hideBtn: true
                    };
                result.forEach(item => {
                    obj[item.label] = this.page[item[code]];
                });
                return obj;
            };
            if (this.show.total) {
                this.columns.forEach(item => {
                    item.totalProps = item.totalProps || ['', ''];
                    item.totalRow &&
                        result.push({
                            label: item.prop,
                            sub: item.totalProps[0],
                            total: item.totalProps[1]
                        });
                });
            }
            this.mixData = [...this.data];
            // 是否显示合计,小计,补录2条数据
            if (result.length && this.show.total) {
                this.mixData.push(getColumnData('小计', 'sub'));
                // 合计
                let maxPage = Math.ceil(this.page.total / this.page.pageSize);
                if (maxPage === this.page.pageIndex) {
                    this.mixData.push(getColumnData('合计', 'total'));
                }
            }
            this.showAll = false;
            this.$nextTick(() => {
                this.showAll = true;
            });
        },
        // 设置
        handleSet (...scope) {
            this.isTableSet = true;
        },
        countHeight () {
            let $table = this.$refs.table.$el;
            let clientTop = $table && $table.getBoundingClientRect().top;
            let offsetTop = $table && $table.offsetTop;
            let limitH = this.limitHeight + 93;
            // let limitH = 42;
            let elHeight = null;
            switch (this.container) {
            case 'document':
                elHeight = window.innerHeight - clientTop - limitH;
                this.$nextTick(() => {
                    this.height = elHeight;
                });
                break;
            case 'dialog':
                let $dialogHeader = document.querySelector(
                    '.el-dialog__header'
                );
                let $dialogbody = document.querySelector(
                    '.el-dialog__body'
                );
                let bodyHeight = parseInt(
                    window
                        .getComputedStyle($dialogbody, null)
                        .getPropertyValue('max-height')
                );
                elHeight =
                        bodyHeight -
                        (offsetTop - $dialogHeader.clientHeight) -
                        limitH;
                this.$nextTick(() => {
                    this.height = elHeight;
                });
                break;
            default:
                break;
            }
        }
    },
    created () {
        this.cAttrs = this.$attrs;
        this.clisteners = this.$listeners;
        this.config.showTotal &&
            (this.show = { ...this.show, ...this.config.showTotal }); // 合计列
        this.uselocal = !!this.config.uselocal;
        this.disAutoHeight = !!this.config.disAutoHeight;
        this.container = this.config.container
            ? this.config.container
            : this.container;
        this.limitHeight = this.config.limitHeight || -50;
        this._getPageKey();
        this._getConfig();
    },
    mounted () {
        if (!this.disAutoHeight) {
            // 关闭自动高度
            if (!this.maxHeight) {
                const debounceFunc = debounce(this.countHeight, 500);
                window.addEventListener('resize', debounceFunc);
                this.$once('hook:beforeDestroy', () => {
                    window.removeEventListener('resize', debounceFunc);
                });
            } else {
                this.height = this.maxHeight; // 高度也可以自定义设置
            }
        }
    },
    computed: {
        highlightCurrentRow: function () {
            let flag = true;
            if (this.config.col[0].type === 'selection') {
                flag = this.config.col[0].hide;
            }
            return flag;
        }
    },
    components: {
        MyTableExpand
    }
};
</script>

<style lang="scss" scoped>
.yl-table {
    .el-icon-setting {
        margin-top: 3px;
        cursor: pointer;
        font-size: 17px;
    }
}
</style>