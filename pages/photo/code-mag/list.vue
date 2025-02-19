<template>
	<view>
		<view class="uni-header">
			<view class="uni-group">
				<view class="uni-title"></view>
				<view class="uni-sub-title"></view>
			</view>
			<view class="uni-group">
				<input class="uni-search" type="text" v-model="query" @confirm="search" placeholder="请输入类型/code/标题" />
				<button class="uni-button" type="default" size="mini" @click="search">搜索</button>
				<button class="uni-button" type="default" size="mini" @click="navigateTo('./add')">新增</button>
				<button class="uni-button" type="default" size="mini" :disabled="!selectedIndexs.length" @click="delTable">批量删除</button>
				<download-excel class="hide-on-phone" :fields="exportExcel.fields" :data="exportExcelData" :type="exportExcel.type" :name="exportExcel.filename">
					<button class="uni-button" type="primary" size="mini">导出 Excel</button>
				</download-excel>
			</view>
		</view>
		<view class="uni-container">
			<unicloud-db
				ref="udb"
				collection="code-mag"
				field="type,code,title,desc,image_url,path,state,is_del,create_date,update_date"
				:where="where"
				page-data="replace"
				:orderby="orderby"
				:getcount="true"
				:page-size="options.pageSize"
				:page-current="options.pageCurrent"
				v-slot:default="{ data, pagination, loading, error, options }"
				:options="options"
				loadtime="manual"
				@load="onqueryload"
			>
				<uni-table ref="table" :loading="loading" :emptyText="error.message || '没有更多数据'" border stripe type="selection" @selection-change="selectionChange">
					<uni-tr>
						<uni-th align="center">类型</uni-th>
						<uni-th align="center">标识</uni-th>
						<uni-th align="center">标题</uni-th>
						<uni-th align="center">描述</uni-th>
						<uni-th align="center">图片</uni-th>
						<uni-th align="center">路由</uni-th>
						<uni-th align="center" filter-type="select" :filter-data="options.filterData.state_localdata" @filter-change="filterChange($event, 'state')">状态</uni-th>
						<uni-th align="center" sortable @sort-change="sortChange($event, 'create_date')">创建时间</uni-th>
						<uni-th align="center" sortable @sort-change="sortChange($event, 'update_date')">更新时间</uni-th>
						<uni-th align="center">操作</uni-th>
					</uni-tr>
					<uni-tr v-for="(item, index) in data" :key="index">
						<uni-td align="center">{{ item.type }}</uni-td>
						<uni-td align="center">{{ item.code }}</uni-td>
						<uni-td align="center">{{ item.title }}</uni-td>
						<uni-td align="center">{{ item.desc }}</uni-td>
						<uni-td align="center"><image :src="item.image_url" mode="widthFix" style="width: 100rpx;"></image></uni-td>
						<uni-td align="center">{{ item.path }}</uni-td>
						<uni-td align="center">{{ getState(item.state) }}</uni-td>
						<uni-td align="center"><uni-dateformat :threshold="[0, 0]" :date="item.create_date"></uni-dateformat></uni-td>
						<uni-td align="center"><uni-dateformat :threshold="[0, 0]" :date="item.update_date"></uni-dateformat></uni-td>
						<uni-td align="center">
							<view class="uni-group">
								<button @click="navigateTo('./edit?id=' + item._id, false)" class="uni-button" size="mini" type="primary">修改</button>
								<button v-if="!item.state" @click="confirmDelete(item, { state: true })" class="uni-button" size="mini" type="primary">启用</button>
								<button v-else @click="confirmDelete(item, { state: false })" class="uni-button" size="mini" type="warn">禁用</button>
								<button @click="confirmDelete(item, { is_del: true })" class="uni-button" size="mini" type="warn">删除</button>
							</view>
						</uni-td>
					</uni-tr>
				</uni-table>
				<view class="uni-pagination-box">
					<uni-pagination show-icon :page-size="pagination.size" v-model="pagination.current" :total="pagination.count" @change="onPageChanged" />
				</view>
			</unicloud-db>
		</view>
	</view>
</template>

<script>
import { enumConverter, filterToWhere } from '../../../js_sdk/validator/code-mag.js';

const db = uniCloud.database();
// 表查询配置
const dbOrderBy = ''; // 排序字段
const dbSearchFields = ['type', 'code', 'title']; // 模糊搜索字段，支持模糊搜索的字段列表。联表查询格式: 主表字段名.副表字段名，例如用户表关联角色表 role.role_name
// 分页配置
const pageSize = 20;
const pageCurrent = 1;

const orderByMapping = {
	ascending: 'asc',
	descending: 'desc'
};
const filterInfo = 'is_del == false';

export default {
	data() {
		return {
			query: '',
			where: filterInfo,
			orderby: dbOrderBy,
			orderByFieldName: '',
			selectedIndexs: [],
			options: {
				pageSize,
				pageCurrent,
				filterData: {
					state_localdata: [
						{
							text: '启用',
							value: true
						},
						{
							text: '禁用',
							value: false
						}
					],
					is_del_localdata: [
						{
							text: '是',
							value: true
						},
						{
							text: '否',
							value: false
						}
					]
				},
				...enumConverter
			},
			imageStyles: {
				width: 64,
				height: 64
			},
			exportExcel: {
				filename: 'code-mag.xls',
				type: 'xls',
				fields: {
					类型: 'type',
					标识: 'code',
					标题: 'title',
					描述: 'desc',
					图片: 'image_url',
					路由: 'path',
					创建时间: 'create_date',
					更新时间: 'update_date'
				}
			},
			exportExcelData: []
		};
	},
	onLoad() {
		this._filter = {};
	},
	onReady() {
		this.$refs.udb.loadData();
	},
	methods: {
		getState(state) {
			return this.options.filterData.state_localdata.filter(el => el.value === state)[0].text;
		},
		onqueryload(data) {
			this.exportExcelData = data;
		},
		getWhere() {
			const query = this.query.trim();
			if (!query) {
				return '';
			}
			const queryRe = new RegExp(query, 'i');
			return dbSearchFields.map(name => queryRe + '.test(' + name + ')').join(' || ');
		},
		search() {
			const newWhere = this.getWhere();
			this.where = newWhere;
			this.$nextTick(() => {
				this.loadData();
			});
		},
		loadData(clear = true) {
			this.$refs.udb.loadData({
				clear
			});
		},
		onPageChanged(e) {
			this.selectedIndexs.length = 0;
			this.$refs.table.clearSelection();
			this.$refs.udb.loadData({
				current: e.current
			});
		},
		navigateTo(url, clear) {
			// clear 表示刷新列表时是否清除页码，true 表示刷新并回到列表第 1 页，默认为 true
			uni.navigateTo({
				url,
				events: {
					refreshData: () => {
						this.loadData(clear);
					}
				}
			});
		},
		// 多选处理
		selectedItems() {
			var dataList = this.$refs.udb.dataList;
			return this.selectedIndexs.map(i => dataList[i]._id);
		},
		// 批量删除
		delTable() {
			this.$refs.udb.remove(this.selectedItems(), {
				success: res => {
					this.$refs.table.clearSelection();
				}
			});
		},
		// 多选
		selectionChange(e) {
			this.selectedIndexs = e.detail.index;
		},
		confirmDelete(data, info) {
			let value = { ...data, ...info, update_date: new Date().getTime() };
			delete value._id;
			uniCloud
				.database()
				.collection('code-mag')
				.doc(data._id)
				.update(value)
				.then(res => {
					this.$refs.table.clearSelection();
					this.$nextTick(() => {
						this.$refs.udb.loadData();
					});
				})
				.catch(err => {
					uni.showModal({
						content: err.message || '请求服务失败',
						showCancel: false
					});
				})
				.finally(() => {
					uni.hideLoading();
				});
			// this.$refs.udb.remove(id, {
			// 	success: res => {
			// 		this.$refs.table.clearSelection();
			// 	}
			// });
		},
		sortChange(e, name) {
			this.orderByFieldName = name;
			if (e.order) {
				this.orderby = name + ' ' + orderByMapping[e.order];
			} else {
				this.orderby = '';
			}
			this.$refs.table.clearSelection();
			this.$nextTick(() => {
				this.$refs.udb.loadData();
			});
		},
		filterChange(e, name) {
			this._filter[name] = {
				type: e.filterType,
				value: e.filter
			};
			let newWhere = filterToWhere(this._filter, db.command);
			if (Object.keys(newWhere).length) {
				this.where = newWhere;
			} else {
				this.where = '';
			}
			this.$nextTick(() => {
				this.$refs.udb.loadData();
			});
		}
	}
};
</script>

<style></style>
