<template>
	<view>
		<view class="top-wrap">
			<tab id="category" :tab-data="categoryMenu" :tab-cur-index="categoryCur" :size="80" scroll="true" @change="toggleCategory">
			</tab>
		</view>

		<swiper :current="categoryCur" :duration="duration" @animationfinish="animationFinish">
			<swiper-item v-for="(item, index) in categoryData" :key="index">
				<scroll :requesting="item.requesting" :end="item.end" :empty-show="item.emptyShow" :list-count="item.listData.length"
				 has-top="true" :refresh-size="80" @refresh="refresh" @more="more">
					<view class="cells">
						<view class="cell" v-for="(item, index2) in item.listData" :key="index2" :data-link="item.link" @tap="showArticle">
							<view class="cell__bd">
								<view class="name">
									<rich-text :nodes="item.title"></rich-text>
								</view>
								<view class="tags">
									<view class="tags__bd">
										<view v-for="(item, name) in item.tags" :key="name" class="tag">{{item.name}}</view>
									</view>
									<view class="date">{{item.niceDate}}</view>
								</view>
							</view>
						</view>
					</view>
				</scroll>
			</swiper-item>
		</swiper>
	</view>
</template>

<script>
	const app = getApp().globalData;
	let pageStart = 1;
	import tab from "../../components/tab/index";
	import scroll from "../../components/scroll/index";
	export default {
		data() {
			return {
				duration: 300,
				// swiper-item 切换过渡时间
				categoryCur: 0,
				// 当前数据列索引
				categoryMenu: [],
				// 分类菜单数据, 字符串数组格式
				categoryData: [] // 所有数据列
			};
		},

		components: {
			tab,
			scroll
		},
		props: {},

		onLoad() {
			getApp().globalData.httpGet("/wxarticle/chapters/json").then(res => {
				console.log(res)
				let menus = res.data || [];
				let categoryMenu = [];
				let categoryData = [];
				menus.forEach((item, index) => {
					categoryMenu.push(item.name.replace("&amp;", "&"));
					categoryData.push({
						id: item.id,
						categoryCur: index,
						requesting: false,
						end: false,
						emptyShow: false,
						page: pageStart,
						listData: []
					});
				});
				this.categoryMenu = categoryMenu
				this.categoryData = categoryData
				// 第一次加载延迟 350 毫秒 防止第一次动画效果不能完全体验
				setTimeout(() => {
					this.refresh();
				}, 350);
			});
		},

		methods: {
			getList(type, currentPage) {
				let currentCur = this.categoryCur;
				let pageData = this.getCurrentData(currentCur);
				if (pageData.end) return;
				pageData.requesting = true;
				this.setCurrentData(currentCur, pageData);
				getApp().globalData.httpGet(`/wxarticle/list/${pageData.id}/${currentPage}/json`).then(res => {
					let data = res.data || {
						datas: [],
						over: false
					};
					let listData = data.datas || [];
					pageData.requesting = false;

					if (type === 'refresh') {
						pageData.listData = listData;
						pageData.end = data.over;
						pageData.page = currentPage + 1;
					} else {
						pageData.listData = pageData.listData.concat(listData);
						pageData.end = data.over;
						pageData.page = currentPage + 1;
					}

					this.setCurrentData(currentCur, pageData);
				});
			},

			// 更新页面数据
			setCurrentData(currentCur, pageData) {
				this.categoryData[currentCur] = pageData
			},

			// 获取当前激活页面的数据
			getCurrentData() {
				return this.categoryData[this.categoryCur];
			},

			// 顶部tab切换事件
			toggleCategory(e) {
				this.duration = 0
				setTimeout(() => {
					this.categoryCur = e.index
				}, 0);
			},

			// 页面滑动切换事件
			animationFinish(e) {
				this.duration = 300
				setTimeout(() => {
					this.categoryCur = e.detail.current
					let pageData = this.getCurrentData();
					if (pageData.listData.length === 0) {
						this.getList('refresh', pageStart);
					}
				}, 0);
			},

			// 刷新数据
			refresh() {
				this.getList('refresh', pageStart);
			},

			// 加载更多
			more() {
				this.getList('more', this.getCurrentData(this.categoryCur).page);
			},

			showArticle(e) {
				wx.setClipboardData({
					data: e.currentTarget.dataset.link,
					success(res) {
						wx.showToast({
							icon: "none",
							title: "链接已复制到剪切板"
						});
					}
				});
			}
		}
	};
</script>
<style>
	@import "./index.css";
</style>
