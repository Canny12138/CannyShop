<template>
	<view class="content">
		<view class="navbar">
			<view v-for="(item, index) in navList" :key="index" class="nav-item"
				:class="{current: tabCurrentIndex === index}" @click="tabClick(index)">
				{{item.text}}
			</view>
		</view>

		<swiper :current="tabCurrentIndex" class="swiper-box" duration="300" @change="changeTab">
			<swiper-item class="tab-content" v-for="(tabItem, tabIndex) in navList" :key="tabIndex">
				<scroll-view class="list-scroll-content" scroll-y @scrolltolower="loadData">
					<!-- 空白页 -->
					<empty v-if="tabItem.loaded === true && tabItem.orderList.length === 0"></empty>

					<!-- 订单列表 -->
					<view v-for="(item, index) in tabItem.orderList" :key="index" class="order-item">
						<view class="i-top b-b">
							<text class="time">{{ item.createTime }}</text>
							<text class="state"
								:style="{ color: orderStateExp(item.status).stateTipColor }">{{ orderStateExp(item.status).stateTip }}</text>
							<!-- <text v-if="item.status === 9" class="del-btn yticon icon-iconfontshanchu1"@click="deleteOrder(index)"></text> -->
						</view>

						<scroll-view v-if="item.orderDetail.length > 1" class="goods-box" scroll-x>
							<view v-for="(detailItem, detailIndex) in item.orderDetail" :key="detailIndex"
								class="goods-item">
								<image class="goods-img" :src="'http://150.158.85.93:81/' + detailItem.book.imgSrc"
									mode="aspectFill"></image>
							</view>
						</scroll-view>

						<view v-if="item.orderDetail.length === 1" class="goods-box-single"
							v-for="(detailItem, detailIndex) in item.orderDetail" :key="detailIndex">
							<image class="goods-img" :src="'http://150.158.85.93:81/' + detailItem.book.imgSrc"
								mode="aspectFill"></image>
							<view class="right">
								<text class="title clamp">{{ detailItem.book.title }}</text>
								<text class="attr-box">x {{ detailItem.quantity }}</text>
								<text class="price">{{ detailItem.book.price }}</text>
							</view>
						</view>
						<!-- <text class="attr-box">{{ item.orderNumber }}</text> -->
						<view class="price-box">
							共 <text class="num">{{ item.orderDetail.length }}</text> 件商品 实付款 <text
								class="price">{{ item.price }}</text>
						</view>

						<view class="action-box b-t" v-if="item.status == 1">
							<button class="action-btn" @click="updateOrder(item, 9)">取消订单</button>
							<navigator :url="`/pages/money/pay?price=${item.price}&orderId=${item.orderId}`"
								hover-class="none">
								<button class="action-btn recom">立即支付</button>
							</navigator>
						</view>
						<view class="action-box b-t" v-if="item.status == 2">
							<button class="action-btn" @click="updateOrder(item, 3)">催发货</button>
						</view>
						<view class="action-box b-t" v-if="item.status == 3">
							<button class="action-btn" @click="updateOrder(item, 4)">确认收货</button>
						</view>
						<view class="action-box b-t" v-if="item.status == 4">
							<button class="action-btn recom">退货</button>
							<navigator :url="`/pages/review/review_edit?orderId=${item.orderId}&orderNumber=${item.orderNumber}`" open-type="redirect" hover-class="none">
								<button	class="action-btn">去评价</button>
							</navigator>
						</view>
						<view class="action-box b-t" v-if="item.status == 5">
							<navigator :url="`/pages/review/review?orderId=${item.orderId}&orderNumber=${item.orderNumber}`" open-type="redirect" hover-class="none">
								<button class="action-btn">查看评价</button>
							</navigator>
						</view>
						<view class="action-box b-t" v-if="item.status == 9">
							<button class="action-btn">删除订单</button>
						</view>
					</view>

					<uni-load-more :status="tabItem.loadingType"></uni-load-more>
				</scroll-view>
			</swiper-item>
		</swiper>
	</view>
</template>

<script>
	import uniLoadMore from '@/components/uni-load-more/uni-load-more.vue';
	import empty from "@/components/empty";

	export default {
		components: {
			uniLoadMore,
			empty
		},
		data() {
			return {
				tabCurrentIndex: 0,
				orderList: [], // 存储订单列表数据
				navList: [{
						state: 0,
						text: '全部',
						loadingType: 'more',
						orderList: []
					},
					{
						state: 1,
						text: '待付款',
						loadingType: 'more',
						orderList: []
					},
					{
						state: 2,
						text: '待发货',
						loadingType: 'more',
						orderList: []
					},
					{
						state: 3,
						text: '待收货',
						loadingType: 'more',
						orderList: []
					},
					{
						state: 4,
						text: '待评价',
						loadingType: 'more',
						orderList: []
					}
				],
			};
		},

		onLoad(options) {
			/**
			 * 修复app端点击除全部订单外的按钮进入时不加载数据的问题
			 * 替换onLoad下代码即可
			 */
			this.tabCurrentIndex = +options.state;
			// #ifndef MP
			this.loadData();
			// #endif
			// #ifdef MP
			if (options.state == 0) {
				this.loadData();
			}
			// #endif
		},

		methods: {
			// 获取订单列表数据
			loadData(source) {
				let index = this.tabCurrentIndex;
				let navItem = this.navList[index];
				let state = navItem.state;

				if (source === 'tabChange' && navItem.loaded === true) {
					// tab切换只有第一次需要加载数据
					return;
				}
				if (navItem.loadingType === 'loading') {
					// 防止重复加载
					return;
				}
				if (navItem.loadingType === 'noMore') {
					// 没有更多数据
					return;
				}

				navItem.loadingType = 'loading';

				this.$u.api.getOrders({
					pageNum: 1,
					pageSize: 100,
					status: this.tabCurrentIndex
				}).then(({
					data,
					meta
				}) => {
					let orderList = data.records.map(item => {
						let orderDetail = item.orderDetail.map(detailItem => {
							return {
								quantity: detailItem.quantity,
								book: detailItem.book
							};
						});

						return {
							orderId: item.id,
							createTime: item.createTime,
							orderStatus: item.orderStatus,
							price: item.price,
							status: item.status,
							orderNumber: item.orderNumber,
							orderDetail: orderDetail
						};
					});

					navItem.orderList = orderList;

					// if (meta.total_pages == meta.current_page) {
					// 	// 没有更多数据
					// 	navItem.loadingType = 'noMore';
					// } else {
					// 	// 还有更多数据
					// 	navItem.loadingType = 'more';
					// }
					navItem.loadingType = 'noMore';
					navItem.loaded = true;
				}).catch(error => {
					// 处理请求错误
				});
			},

			// swiper 切换
			changeTab(e) {
				this.tabCurrentIndex = e.target.current;
				this.loadData('tabChange');
			},

			// 顶部tab点击
			tabClick(index) {
				this.tabCurrentIndex = index;
			},

			// 删除订单
			deleteOrder(index) {
				uni.showLoading({
					title: '请稍后'
				});

				setTimeout(() => {
					let navItem = this.navList[this.tabCurrentIndex];
					navItem.orderList.splice(index, 1);

					uni.hideLoading();
				}, 600);
			},

			// 更新订单
			updateOrder(item, changeStatus) {
				uni.showLoading({
					title: '请稍后'
				});

				setTimeout(() => {
					this.$u.api.updateOrderStatus({
						orderId: item.orderId,
						status: changeStatus,
					})
					console.log(item)
					// 更新订单后删除原状态中该项
					let list = this.navList[item.status].orderList;
					let index = list.findIndex(val => val.id === item.id);
					if (index !== -1) {
						list.splice(index, item.status);
					}

					let {
						stateTip,
						stateTipColor
					} = this.orderStateExp(item.status);
					item.status = changeStatus;
					item.orderStatus = stateTip;
					item.stateTipColor = stateTipColor;
					uni.hideLoading();
				}, 600);
			},

			// 订单状态文字和颜色
			orderStateExp(state) {
				let stateTip = '';
				let stateTipColor = '#fa3534';
				switch (+state) {
					case 1:
						stateTip = '待付款';
						break;
					case 2:
						stateTip = '待发货';
						break;
					case 3:
						stateTip = '待收货';
						break;
					case 4:
						stateTip = '待评价';
						break;
					case 5:
						stateTip = '已完成';
						stateTipColor = '#909399';
						break;
					case 9:
						stateTip = '订单已关闭';
						stateTipColor = '#909399';
						break;
						// 更多自定义
				}
				return {
					stateTip,
					stateTipColor
				};
			}
		}
	};
</script>

<style lang="scss">
	page,
	.content {
		background: $page-color-base;
		height: 100%;
	}

	.swiper-box {
		height: calc(100% - 40px);
	}

	.list-scroll-content {
		height: 100%;
	}

	.navbar {
		display: flex;
		height: 40px;
		padding: 0 5px;
		background: #fff;
		box-shadow: 0 1px 5px rgba(0, 0, 0, .06);
		position: relative;
		z-index: 10;

		.nav-item {
			flex: 1;
			display: flex;
			justify-content: center;
			align-items: center;
			height: 100%;
			font-size: 15px;
			color: $font-color-dark;
			position: relative;

			&.current {
				color: $base-color;

				&:after {
					content: '';
					position: absolute;
					left: 50%;
					bottom: 0;
					transform: translateX(-50%);
					width: 44px;
					height: 0;
					border-bottom: 2px solid $base-color;
				}
			}
		}
	}

	.uni-swiper-item {
		height: auto;
	}

	.order-item {
		display: flex;
		flex-direction: column;
		padding-left: 30upx;
		background: #fff;
		margin-top: 16upx;

		.i-top {
			display: flex;
			align-items: center;
			height: 80upx;
			padding-right: 30upx;
			font-size: $font-base;
			color: $font-color-dark;
			position: relative;

			.time {
				flex: 1;
			}

			.state {
				color: $base-color;
			}

			.del-btn {
				padding: 10upx 0 10upx 36upx;
				font-size: $font-lg;
				color: $font-color-light;
				position: relative;

				&:after {
					content: '';
					width: 0;
					height: 30upx;
					border-left: 1px solid $border-color-dark;
					position: absolute;
					left: 20upx;
					top: 50%;
					transform: translateY(-50%);
				}
			}
		}

		/* 多条商品 */
		.goods-box {
			height: 160upx;
			padding: 20upx 0;
			white-space: nowrap;

			.goods-item {
				width: 120upx;
				height: 120upx;
				display: inline-block;
				margin-right: 24upx;
			}

			.goods-img {
				display: block;
				width: 100%;
				height: 100%;
			}
		}

		/* 单条商品 */
		.goods-box-single {
			display: flex;
			padding: 20upx 0;

			.goods-img {
				display: block;
				width: 120upx;
				height: 120upx;
			}

			.right {
				flex: 1;
				display: flex;
				flex-direction: column;
				padding: 0 30upx 0 24upx;
				overflow: hidden;

				.title {
					font-size: $font-base + 2upx;
					color: $font-color-dark;
					line-height: 1;
				}

				.attr-box {
					font-size: $font-sm + 2upx;
					color: $font-color-light;
					padding: 10upx 12upx;
				}

				.price {
					font-size: $font-base + 2upx;
					color: $font-color-dark;

					&:before {
						content: '￥';
						font-size: $font-sm;
						margin: 0 2upx 0 8upx;
					}
				}
			}
		}

		.price-box {
			display: flex;
			justify-content: flex-end;
			align-items: baseline;
			padding: 20upx 30upx;
			font-size: $font-sm + 2upx;
			color: $font-color-light;

			.num {
				margin: 0 8upx;
				color: $font-color-dark;
			}

			.price {
				font-size: $font-lg;
				color: $font-color-dark;

				&:before {
					content: '￥';
					font-size: $font-sm;
					margin: 0 2upx 0 8upx;
				}
			}
		}

		.action-box {
			display: flex;
			justify-content: flex-end;
			align-items: center;
			height: 100upx;
			position: relative;
			padding-right: 30upx;
		}

		.action-btn {
			width: 160upx;
			height: 60upx;
			margin: 0;
			margin-left: 24upx;
			padding: 0;
			text-align: center;
			line-height: 60upx;
			font-size: $font-sm + 2upx;
			color: $font-color-dark;
			background: #fff;
			border-radius: 100px;

			&:after {
				border-radius: 100px;
			}

			&.recom {
				background: #fff9f9;
				color: $base-color;

				&:after {
					border-color: #f7bcc8;
				}
			}
		}
	}


	/* load-more */
	.uni-load-more {
		display: flex;
		flex-direction: row;
		height: 80upx;
		align-items: center;
		justify-content: center
	}

	.uni-load-more__text {
		font-size: 28upx;
		color: #999
	}

	.uni-load-more__img {
		height: 24px;
		width: 24px;
		margin-right: 10px
	}

	.uni-load-more__img>view {
		position: absolute
	}

	.uni-load-more__img>view view {
		width: 6px;
		height: 2px;
		border-top-left-radius: 1px;
		border-bottom-left-radius: 1px;
		background: #999;
		position: absolute;
		opacity: .2;
		transform-origin: 50%;
		animation: load 1.56s ease infinite
	}

	.uni-load-more__img>view view:nth-child(1) {
		transform: rotate(90deg);
		top: 2px;
		left: 9px
	}

	.uni-load-more__img>view view:nth-child(2) {
		transform: rotate(180deg);
		top: 11px;
		right: 0
	}

	.uni-load-more__img>view view:nth-child(3) {
		transform: rotate(270deg);
		bottom: 2px;
		left: 9px
	}

	.uni-load-more__img>view view:nth-child(4) {
		top: 11px;
		left: 0
	}

	.load1,
	.load2,
	.load3 {
		height: 24px;
		width: 24px
	}

	.load2 {
		transform: rotate(30deg)
	}

	.load3 {
		transform: rotate(60deg)
	}

	.load1 view:nth-child(1) {
		animation-delay: 0s
	}

	.load2 view:nth-child(1) {
		animation-delay: .13s
	}

	.load3 view:nth-child(1) {
		animation-delay: .26s
	}

	.load1 view:nth-child(2) {
		animation-delay: .39s
	}

	.load2 view:nth-child(2) {
		animation-delay: .52s
	}

	.load3 view:nth-child(2) {
		animation-delay: .65s
	}

	.load1 view:nth-child(3) {
		animation-delay: .78s
	}

	.load2 view:nth-child(3) {
		animation-delay: .91s
	}

	.load3 view:nth-child(3) {
		animation-delay: 1.04s
	}

	.load1 view:nth-child(4) {
		animation-delay: 1.17s
	}

	.load2 view:nth-child(4) {
		animation-delay: 1.3s
	}

	.load3 view:nth-child(4) {
		animation-delay: 1.43s
	}

	@-webkit-keyframes load {
		0% {
			opacity: 1
		}

		100% {
			opacity: .2
		}
	}
</style>
